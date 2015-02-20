### Table of Contents

#### om.core

* [Life Cycle Protocols](#life-cycle-protocols)
  * [IInitState](#iinitstate)
  * [IWillMount](#iwillmount)
  * [IDidMount](#ididmount)
  * [IShouldUpdate](#ishouldupdate)
  * [IWillReceiveProps](#iwillreceiveprops)
  * [IWillUpdate](#iwillupdate)
  * [IDidUpdate](#ididupdate)
  * [IRender](#irender)
  * [IRenderState](#irenderstate)
  * [IDisplayName](#idisplayname)
  * [IWillUnmount](#iwillunmount)
* [Functions](#functions)
  * [get-props](#get-props)
  * [get-state](#get-state)
  * [get-shared](#get-shared)
  * [root](#root)
  * [build](#build)
  * [build*](#build-1)
  * [build-all](#build-all)
  * [transact!](#transact)
  * [update!](#update)
  * [path](#path)
  * [state](#state)
  * [value](#value)
  * [get-node](#get-node)
  * [set-state!](#set-state)
  * [update-state!](#update-state)
  * [refresh!](#refresh)
  * [get-render-state](#get-render-state)
  * [detach-root](#detach-root)
  * [root-cursor](#root-cursor)
  * [ref-cursor](#ref-cursor)
  * [observe](#observe)
* [Macros](#macros)
  * [component](#component)

#### om.dom

* [Props](#props)
* [render-to-str](#render-to-str)

# om.core

## Life Cycle Protocols

The Om life cycle protocols map more or less directly to the life
cycle API present in Facebook's React.

Om component functions return `reify` instances that implement the Om
life cycle protocols. When `this` is used in a lifecycle protocol it refers
to the reify instance. As a rule of thumb it is not needed and can be discarded.

```clj
(defn my-widget [data owner]
  (reify
    om/IRender
    (render [_]
      (dom/h1 nil "Hello world!"))))
```

It's important to understand the `my-widget` will be called many
times. Thus it's an anti-pattern to wrap `reify` in a `let` to
allocate stateful entities like core.async channels. Stateful entities
should be stored in component local state - `om.core/IInitState` or
`om.core/IWillMount` are good places to do this.

### IInitState

```clj
(defprotocol IInitState
  (init-state [this]))
```

Called only once on an Om component. Implementations should return a map
of initial state.

```clj
(defn my-widget [data owner]
  (reify
    om/IInitState
    (init-state [_]
      {:text "Hello world!"})
    om/IRenderState
    (render-state [_ state]
      (dom/h1 nil (:text state)))))
```

### IWillMount

```clj
(defprotocol IWillMount
  (will-mount [this]))
```

Called once when the component is about to be mounted into the DOM. A
useful place to establish persistent information and control like
core.async channels and go loops.

### IDidMount

```clj
(defprotocol IDidMount
  (did-mount [this]))
```

Called once when the component has been mounted into the DOM. The DOM node associated with this component can be retrieved by using `(om.core/get-node owner)`.

This is a good place to initialize persistent information and control
that needs the DOM to be present.

### IShouldUpdate

```clj
(defprotocol IShouldUpdate
  (should-update [this next-props next-state]))
```

You should only implement this if you really know what you're
doing. Even then you probably shouldn't.

Implementations should return a boolean value. If true then the
component's `om.core/IRender` or `om.core/IRenderState` implementation
will be called. This provides the opportunity to prevent components
from re-rendering in response to certain changes in app (props) or local
state. Please note that preventing components from re-rendering
in response to props or state change could result in the DOM being out
of sync with application or local state. 

`next-props` is the next application state that the component is
associated with. `next-state` is the next component local state, it is
always a map.

In your implementation if you wish to detect prop transitions **you
must use `om.core/get-props`** to get the previous props. This is because your component
constructor function is called with the updated props.

### IWillReceiveProps

```clj
(defprotocol IWillReceiveProps
  (will-receive-props [this next-props]))
```

Not called on the first render, will be called on all subsequent
renders. This is a good place to detect app state changes and make 
updates to local component state using om/set-state! or 
om/update-state!.

`next-props` is the next application state associated with this
component.

In your implementation if you wish to detect prop transitions **you
must use `om.core/get-props`** to get the previous props. This is 
because your component constructor function is called with the 
updated props.

### IWillUpdate

```clj
(defprotocol IWillUpdate
  (will-update [this next-props next-state]))
```

Not called on the first render, will be called on all subsequent
renders. This is a good place to detect and act on state transitions.

`next-props` is the next application state associated with this
component. `next-state` is the next component local state, it is
always a map.

In your implementation if you wish to detect prop transitions **you
must use `om.core/get-props`** to get the previous props. This is 
because your component constructor function is called with the 
updated props.

Note:: You cannot update local component state in this method. 
If you wish to change local state in response to prop changes use
IWillReceiveProps. 

### IDidUpdate

```clj
(defprotocol IDidUpdate
  (did-update [this prev-props prev-state]))
```

Called when React has rendered the component into the
DOM. `prev-props` is the previous application state associated with
this component. `prev-state` is the previous component local state, it
is always a map.

### IRender

```clj
(defprotocol IRender
  (render [this]))
```

Called on all changes to application state or component local
state. Must return an Om component, a React component, or a some value
that React knows how to render.

If you implement `om.core/IRender` you should not implement
`om.core/IRenderState`.

### IRenderState

```clj
(defprotocol IRenderState
  (render-state [this state]))
```

The only difference between `om.core/IRender` and
`om.core/IRenderState` is that `IRenderState` implementations get the
state as an argument. `state` is always a map, you can use destructuring.

If you implement `om.core/IRenderState` you should not implement
`om.core/IRender`.

### IDisplayName

```clj
(defprotocol IDisplayName
  (display-name [this]))
```

Return a string name to be used for debugging. The Chrome [React Developer Tools](https://chrome.google.com/webstore/detail/react-developer-tools/fmkadmapgofadopljbjfkapdkoienihi) extension uses this to name the components.

### IWillUnmount

```clj
(defprotocol IWillUnmount
  (will-unmount [this]))
```

Called immediately before a component is unmounted from the DOM.

Perform any necessary cleanup in this method, such as invalidating timers or cleaning up any DOM elements that were created in `om.core/IDidMount`.

## Functions

### get-props

```clj
(defn get-props [owner]
   ...)
```

`owner` is the backing Om component. `om.core/get-props` returns the
*cursor* into the application state associated with the Om
component. A *cursor* is a piece of the application state that knows
how to update itself.

### get-state

```clj
(defn get-state
  ([owner] ...)
  ([owner korks] ...))
```

`owner` is the backing Om component. `korks` is a key or sequence of
keys. Will return the specified piece of component local state. Will
always return pending state.

### get-shared

```clj
(defn get-shared
  ([owner] ...)
  ([owner korks] ...))
```

`owner` is the backing Om component. `korks` is a key or sequence of
keys. It will return data that is shared across the entire render tree. 
You can set global shared data with `om.core/root`.

### root

```clj
(defn root
  ([f value options] ...))
```

`f` is a function returning an instance of `IRender` or `IRenderState`.
`f` takes two arguments, a root cursor on the application state and
the backing Om component for the root. `value` is either a tree of
associative ClojureScript data structures or an atom wrapping a tree
of associative ClojureScript data structures.

`options` is a map containing any key allowed to `om.core/build`.
Additionally the following keys are allowed/required:

* `:target` (required)
* `:shared` (optional) in order to provide global services
* `:tx-listen` a function that will listen in on transactions, should take 2 arguments:
    1. a map containing the path, old and new state at path, old and new global state, and transaction tag if provided (`:path`, `:old-value`, `:new-value`, `:old-state`, `:new-state` and `:tag`).
    2. the root cursor.
* `:path` to specify the path of the cursor into app-state (see
  [#72](https://github.com/swannodette/om/issues/72))
* `:instrument` a function of three arguments that if provided will
  intercept all calls to `om.core/build`. The function arguments
  correspond exactly to the three argument arity of `om.core/build`

`om.core/root` is idempotent. You may safely call it multiple
times. Only one Om render loop is ever allowed on a particular DOM
target.

```clj
(om.core/root
  (fn [app-state owner]
    (reify
      om.core/IRender
      (render [_]
        (dom/h1 nil (:text app-state)))))
  {:text "Hello world!"}
  {:target (. js/document getElementById "my-app")})   
```

### build

```clj
(defn build
  ([f x] ...)
  ([f x m] ...))
```

Constructs an Om component. `f` must be a function that returns an
instance of `om.core/IRender` or `om.core/IRenderState`. `f` must take
two arguments - a value and the backing Om component usually referred
to as the owner. `f` can take a third argument if `:opts` is specified
in `m`. The component is identified by the function `f`. Changing `f`
to a different function will construct a new component, while changing
the return value will not change component.  `x` can be any value. `m`
is an optional map of options.

Only the following keys are allowed in `m`.

`:key` - a keyword that will be used to lookup a value in `x` to
be used as a [React key](http://facebook.github.io/react/docs/multiple-components.html#dynamic-children).

`:react-key` - a value to use as a [React key](http://facebook.github.io/react/docs/multiple-components.html#dynamic-children).

`:fn` - a function to apply to `x` before invoking `f`.

`:init-state` - a map of initial state to set on the component (state from `IInitState` is merged onto it).

`:state` - a map of state to merge into the component.

`:opts` - a map of side information.

### build*

```clj
(defn build*
  ([f x] ...)
  ([f x m] ...))
```

Identical to `om.core/build` except cannot be intercepted by the
`:instrument` argument provided to `om.core/root`. Needed to avoid
infinite loops in components constructed via `:instrument`.

### build-all

```clj
(defn build-all
  ([f xs] ...)
  ([f xs m] ...)
```

Conceptually the same as `om.core/build`, the only difference is that
it returns a sequence of Om components. `xs` is a sequence of
values. `f` and `m` are the same as `om.core/build`.

### transact!

```clj
(defn transact!
  ([cursor f] ...)
  ([cursor korks f])
  ([cursor korks f tag]))
```

The primary way to transition application state. `cursor` is an Om
cursor into the application state. `f` is a function that will receive
the specified piece of application state. `korks` is an optional
key or sequence of keys to access in the cursor. `tag` is optional
information to tag the transaction. `tag` should be either a keyword
or a vector whose first element is a `keyword`.

```clj
(transact! cursor :text (fn [_] "Changed this!"))
```

### update!

```clj
(defn update!
  ([cursor v] ...)
  ([cursor korks v] ...)
  ([cursor korks v tag] ...))
```

Similar to `om.core/transact!` but just sets a cursor to a new value,
analagous to `reset!` for atoms.

```clj
(update! cursor [:text] "Changed this!")
```

### path

```clj
(defn path
  ([cursor] ...))
```

### state

```clj
(defn state
  ([cursor] ...))
```

### value

```clj
(defn value
  ([cursor] ...))
```

### get-node

```clj
(defn get-node
  ([owner ref] ...)
  ([owner] ...))
```

`owner` is the backing Om component. `ref` is a JavaScript String that
references a DOM node. This functionality is identical to the
[`ReactComponent.getDOMNode`](http://facebook.github.io/react/docs/component-api.html#getdomnode) in React.

### set-state!

```clj
(defn set-state!
  ([owner korks v] ...))
```

Sets component local state. `owner` is the backing Om
component. `korks` is a key or sequences of keys. `v` is the value to
set. Will trigger an Om re-render.

### update-state!

```clj
(defn update-state! 
  ([owner f] ...)
  ([owner korks f] ...))
```

Takes a pure owning component, an optional sequential list of keys and a function to transition the state of the component. Conceptually analogous to React `setState`. Will schedule an Om re-render.

### refresh!

```clj
(defn refresh! [owner]
  ...)
```

Utility to re-render an owner. Delegates to `update-state!`.

### get-render-state

```clj
(defn get-render-state
  ([owner] ...)
  ([owner korks] ...))
```

Returns rendered component local state. `owner` is the backing Om
component. `korks` is an optional key or sequences of keys. Similar to
`om.core/get-state` except always returns the rendered state. Useful
for detecting state transitions.

### detach-root

```clj
(defn detach-root [target]
  ...)
```

Given a DOM target, remove its render loop if one exists.

### root-cursor

```clj
(defn root-cursor [atom]
  ...)
```

Given an application state atom, return a root cursor for it.

### ref-cursor

```clj
(defn ref-cursor [cursor]
  ...)
```

Given a cursor, return a reference cursor that inherits all of the
properties and methods of the cursor. Reference cursors may be
observed via `om.core/observe`.

### observe

```clj
(defn observe [owner ref]
  ...)
```

Given a component and a reference cursor, have the component observe
the reference cursor for any data changes.  Evaluates to the given reference cursor.
  
## Macros

### component

```clj
(defmacro component [& body]
  ...)
```

Sugar over `reify` for quickly putting together components that
only need to implement `om.core/IRender` and don't need access to
the owner argument.

# om.dom

The dom functions map directly to the DOM api presented by React. For
example `React.DOM.div` becomes `om.dom/div`. The arguments are exactly
the same as React, the first argument is `props`, all subsequent
arguments are `children`.

### Props

For a list of supported `props` see [React's supported DOM attributes](http://facebook.github.io/react/docs/tags-and-attributes.html#html-attributes) and [React's special attributes](http://facebook.github.io/react/docs/special-non-dom-attributes.html). As an example of the special attributes, the following code will create a `div` component that contains raw HTML:

```clj
(om.dom/div #js {:dangerouslySetInnerHTML #js {:__html "<b>Bold!</b>"}}
            nil)
```
Be careful! The attribute is well-named: this is potentially dangerous and should be used with caution.

### render-to-str

```clj
(defn render-to-str [c]
  ...)
```

Equivalent to [`React.renderComponentToString`](http://facebook.github.io/react/docs/top-level-api.html#react.rendercomponenttostring). For example:

```clj
(dom/render-to-str (om/build some-widget data))
```