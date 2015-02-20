Any time you read or write local state, you are working with React component instances. But you probably aren't writing any code to manage these instances. Your code routes the appropriate global state into your component `f` via `om/build`... but your `f` also receives an `owner` that you didn't specify. Where does that come from?

Asked differently: where does local state live between renders, and how is it reassociated with `om/build`s on subsequent renders?

This page shows internal data structures to make the concepts more palpable.

#### Where are the instances between renders?

React maintains a `instancesByReactRootID` list.

You can inspect it via browser dev tools. Set a breakpoint in your code and jump up the callstack to react.js.

```javascript
instancesByReactRootID
//= {.0: ReactCompositeComponent.createClass.Constructor}
```

These root instances are associated with the DOM element that was passed into `om/root`. This instance contains a tree of all rendered components. For example, using the code from the "pathological case" below, you can see a child instance `childc` inside of the root instance `rootc`.

```javascript
instancesByReactRootID[".0"].getDisplayName()
//= "rootc"
instancesByReactRootID[".0"]._renderedComponent._renderedChildren[".1"].getDisplayName()
//= "childc"
```

These `ReactCompositeComponent`s are the `owner`s that get passed into your `f`. They are instantiated before mount, and they are reused until unmount.

In other words, these `owner`s are stored by React in a list of trees. Your component event handlers and `go` blocks may also hold a reference to `owner`s via closures, as do Om's `render-queue` and `ref-cursors`.


#### How are instances reassociated with subsequent `om/build`s?

In the React model, you specify "what comes next" rather than manipulating "what exists now". In this series-of-snapshots model, identity doesn't come for free, so how do components reclaim their local state across renders?

Think of it like this. We have:
* a tree of instances  
  * an `instancesByReactRootID` entry or one of its subtrees
* a specification of what comes next  
  * via `render`  
  
Identity is inferred via React's [reconciliation](http://facebook.github.io/react/docs/reconciliation.html).

Local state goes along for the ride. If React chooses to reuse a component instance, then the instance's previous state will persist. This is usually correct, but you can imagine cases where you have to nudge it via `:react-key`, and not just for performance reasons.


#### The pathological case

To better understand this identity approach, it's useful to see where it breaks down.

The [reconciliation](http://facebook.github.io/react/docs/reconciliation.html) documentation explains the reuse heuristics in terms of performance, but they can also affect what actually gets rendered. This is not specific to Om, but it's amplified by the fact that Om has `:init-state`.


```clojure
(defonce app-state (atom {:count 1}))

(defn childc [_ owner]
  (reify
	om/IDisplayName
	(display-name [_]
	  "childc")

	om/IRenderState
	(render-state [_ {:keys [text]}]
	  (dom/div nil text))))

(defn rootc [app owner]
  (reify
	om/IDisplayName
	(display-name [_]
	  "rootc")

	om/IRender
	(render [_]
	  (apply dom/div nil
			 (dom/button #js {:onClick #(om/transact! app :count inc)}
						 "Add another")
			 (map #(om/build childc app {:init-state {:text %}

										 ;; :react-key % ;; Uncomment to fix.

										 ;; Without this, clicking causes
										 ;; 0 0 0 0 ... to render.
										 ;;
										 ;; With this, clicking causes
										 ;; 0 1 2 3 ... to render reversed,
										 ;; as expected.
										 })
				  (reverse (range (:count app))))))))
```

On the first render, `rootc` builds a `childc`, initializing it with `:text` 0.
On the second render, `rootc` builds two `childc`s, initializing with `:text`s 1 and 0.

The first render results in a `childc` React component with `.-state` of `#js {... :__om_state {:text 0}}`.

```javascript
instancesByReactRootID[".0"].getDisplayName()
//= "rootc"
instancesByReactRootID[".0"]._renderedComponent._renderedChildren[".1"].getDisplayName()
//= "childc"
instancesByReactRootID[".0"]._renderedComponent._renderedChildren[".1"].state.__om_state.arr
//= [[Object cljs.core.Keyword], 0]
```

The second render results in a **two** `childc` React components with this same state.
```javascript
instancesByReactRootID[".0"]._renderedComponent._renderedChildren[".1"].state.__om_state.arr
//= [cljs.core.Keyword, 0]
instancesByReactRootID[".0"]._renderedComponent._renderedChildren[".2"].state.__om_state.arr
//= [cljs.core.Keyword, 0]
```

Where'd the 1 go?
```javascript
instancesByReactRootID[".0"]._renderedComponent._renderedChildren[".1"].props.__om_init_state.arr
//= [cljs.core.Keyword, 1]
```

It never got merged into the component's state because the first instance was reused, so the `:init-state` was not consulted.

At a fundamental level, local state requires a solid notion of identity, so it's a good idea to understand React's identity heuristics.