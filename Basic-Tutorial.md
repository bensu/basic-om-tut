This tutorial assumes familiarity with Clojure or ClojureScript. If
you are not familiar with them, we recommend going through the
[ClojureScript tutorial for Light Table](http://github.com/swannodette/lt-cljs-tutorial)
first. We will use
[Figwheel](https://github.com/bhauman/lein-figwheel) to 
easily get an interactive environment for ClojureScript and Om.
Figwheel does two things:

- It automatically compiles and reloads your ClojureScript and CSS in
  the browser as soon as you save the file, no need to refresh.
- It connects the browser to a REPL so you can try expressions and
  manipulate your running app. 

To get started, install [Leiningen](http://leiningen.org). Then
navigate in a terminal to the directory where you would like the tutorial
and run the following command:

```
lein new figwheel om-tut -- --om
```

This will create a project including Om and Figwheel in a  folder
called `om-tut`. `cd` into it and run the following command:

```
lein figwheel 
```

It will start auto building the ClojureScript project. The first build
will take a few seconds. Once the build has succeeded 
open [localhost:3449](http://localhost:3449/) in your favorite
browser (we recommend Google Chrome as it has excellent support for
source maps). You should see an `h1` tag with the text content `Hello
World!` in it.

Open `src/om_tut/core.cljs` in your preferred editor. Change the
`:text` value of `app-state` to be something else other than `Hello
World!`. Save the file. Refresh your browser and you should see the
new contents. The reason we need to refresh the browser is because
`app-state` is defined with `defonce`. This is meant to prevent each
reload from resetting the state. 

That's pretty boring isn't it? Let's do some live coding instead.
Arrange your windows so that you can see both the Chrome window 
and your source code at the same time.

Now change `dom/h1` to `dom/p` and watch the browser window. It should
change without the need for reloading. This changed the behavior of
the code without affecting `app-state`. To edit `app-state` you can
either refresh the browser to reset it, or manipulate it directly from
the browser-connected-REPL. In the same terminal window where you
entered `lein figwheel` you should see a REPL. If you don't, try
refreshing your browser window. An easy way to try the REPL is:

```clj
ClojureScript:cljs.user> (js/alert "Am I connected?")
```

You should see the alert in your browser. After you click "Ok" in the
browser, the expression should return `nil` in the REPL. If you see
exceptions and error messages, please troubleshoot Figwheel's Browser
REPL before proceeding. If everything is working move from the generic
`cljs.user` namespace to `om-tut.core` (where all our code lives) and
then change the state:

```clj
ClojureScript:cljs.user=> (in-ns 'om-tut.core)
om-tut.core
ClojureScript:om-tut.core=> (swap! app-state assoc :text "Do it live!")
```

## Om basics

In Om the application state is held in an `atom`, the one reference
type built into ClojureScript. If you change the value of the atom via
`swap!` or `reset!` this will always trigger a re-render of any Om
roots attached to it (we'll explain this in a second). You can think
of this atom as the database of your client side 
application. Everything in the atom should be an associative data
structure - either a ClojureScript map or indexed sequential data
structure such as a vector (but not a set). This means you should
never put lists or lazy sequences into the application state. It's
particularly easy to forget this when updating indexed sequences in
the application state.

### om.core/root

`om.core/root` (which is aliased to `om/root` here), establishes an
Om rendering loop on a specific element in the DOM. The `om.root`
expression in the tutorial at this point looks like this:

```clj
(om/root
  (fn [app owner]
    (reify
      om/IRender
      (render [_]
        (dom/h1 nil (:text app)))))
  app-state
  {:target (. js/document (getElementById "app"))})
```

`om.core/root` is idempotent; that is, it's safe to evaluate it
multiple times (that is why we don't need `defonce`). It takes three
arguments. The first argument is a function that takes the application
state data and the backing React component, here called `owner`. This
function must return an Om component - i.e. a model of the
`om/IRender` interface, like `om.core/component` macro generates. The
second argument  is the application state atom. The third argument 
is a map; it must contain a `:target` DOM node key value pair. It also
takes other interesting options which will be covered later.

There can be multiple roots. Edit `resources/public/index.html`, replace `<div
id="app"><h2>Figwheel..</h2><p>Checkout..</p></div>` with the following:

```html
<div id="app0"></div>
<div id="app1"></div>
```

You need to refresh the browser after changing `resources/public/index.html`
since Figwheel doesn't handle HTML reloading. And edit
`src/om_tut/core.cljs` replacing the `om/root` expression
with the following:

```clj
(om/root
  (fn [data owner]
    (om/component (dom/h2 nil (:text data))))
  app-state
  {:target (. js/document (getElementById "app0"))})
```

Refresh your browser. You should see just one `h2` tag on the
page. Copy and paste the `om/root` expression and edit the second
one to look like the following:

```clj
(om/root
  (fn [data owner]
    (om/component (dom/h2 nil (:text data))))
  app-state
  {:target (. js/document (getElementById "app1"))}) ;; <-- "app0" to "app1"
```

You should see the second `h2` tag magically appear after saving.

Evaluate this in the REPL:

```clj
ClojureScript:om-tut.core=> (swap! app-state assoc :text "Multiple roots!")
```

You should see both `h2` tags update on the fly. Multiple roots are
fully supported and synchronized to render on the same
`requestAnimationFrame`.

Before proceeding remove the `<div id="app1"></div>` from
`resources/public/index.html` and remove the second `om/root`
expression. Save and refresh the browser.

## Rendering a list of things

Change the `app-state` expression to the following and refresh the browser:

```clj
(defonce app-state (atom {:list ["Lion" "Zebra" "Buffalo" "Antelope"]}))
```

Change the `om/root` expression to the following and save. Don't bother refreshing,
[John McCarthy](http://library.stanford.edu/collections/john-mccarthy-papers-0)
would be pissed! You should see a list of animals now.

```clj
(om/root
  (fn [data owner]
    (om/component 
      (apply dom/ul nil
        (map (fn [text] (dom/li nil text)) (:list data)))))
  app-state
  {:target (. js/document (getElementById "app0"))})
```

You might have noticed that the first argument to `dom/ul` and
`dom/li` is `nil`. This argument is how you set DOM attributes. Change
the `om/root` expression to the following and save it:

```clj
(om/root
  (fn [data owner]
    (om/component 
      (apply dom/ul #js {:className "animals"}
        (map (fn [text] (dom/li nil text)) (:list data)))))
  app-state
  {:target (. js/document (getElementById "app0"))})
```

If you right click on the list in Google Chrome and select **Inspect
Element** you should see that the `ul` tag in the DOM does indeed have
its CSS class attribute set to "animals".

`#js {...}` and `#js [...]` is what is referred to as a reader
literal. ClojureScript supports data literals for JavaScript via
`#js`. 

`#js {...}` is for JavaScript objects:

```clj
#js {:foo "bar"}  ;; is equivalent to
#js {"foo" "bar"}
```

`#js [...]` is for JavaScript arrays:

```clj
#js [1 2 3]
```

The `#js` reader literal support is shallow, take note of the
following:

```clj
#js {:foo [1 2 3]} ;; a JS object with a persistent vector in it
```

## Life without a templating language

In Om you have the full power of the ClojureScript language when building
your user interface. At the same time, Om leaves the door open for
alternate syntaxes for describing the DOM if that's your cup of tea.

Let's edit our code so we get zebra striping on the list. Lets add a
helper function `stripe` before the `om/root` expression:

```clj
(defn stripe [text bgc]
  (let [st #js {:backgroundColor bgc}]
    (dom/li #js {:style st} text)))
```

Then change the `om/root` expression to the following and save:

```clj
(om/root
  (fn [data owner]
    (om/component 
      (apply dom/ul #js {:className "animals"}
        (map stripe (:list data) (cycle ["#ff0" "#fff"])))))
  app-state
  {:target (. js/document (getElementById "app0"))})
```

As we can see ClojureScript offers powerful functional tools that put
most templating languages completely to shame.

## Your first Om component

Change `<div id="app0"></div>` to `<div id="contacts"></div>`, remove
`stripe` from `src/om_tut/core.cljs` and save.

Change the `om/root` expression to the following, don't save it
yet since we haven't defined `contacts-view`.

```clj
(om/root contacts-view app-state
  {:target (. js/document (getElementById "contacts"))})
```

Let's edit `app-state` so it looks like this:

```clj
(defonce app-state
  (atom
    {:contacts
     [{:first "Ben" :last "Bitdiddle" :email "benb@mit.edu"}
      {:first "Alyssa" :middle-initial "P" :last "Hacker" :email "aphacker@mit.edu"}
      {:first "Eva" :middle "Lu" :last "Ator" :email "eval@mit.edu"}
      {:first "Louis" :last "Reasoner" :email "prolog@mit.edu"}
      {:first "Cy" :middle-initial "D" :last "Effect" :email "bugs@mit.edu"}
      {:first "Lem" :middle-initial "E" :last "Tweakit" :email "morebugs@mit.edu"}]}))
```

After `app-state` lets add the following code:

```clj
(defn contacts-view [data owner]
  (reify
    om/IRender
    (render [this]
      (dom/div nil
        (dom/h2 nil "Contact list")
        (apply dom/ul nil
          (om/build-all contact-view (:contacts data)))))))
```

In order to build an Om component we must use `om.core/build` for a
single component and `om.core/build-all` to build many components. In
this case we want to display a contact list so we want to use
`om.core/build-all`. `contacts-view` returns a `div` with a `h2` and
`ul` tag in it. We want to render several `li` elements so we call
`apply` on `dom/ul`.

Let's write `contact-view` now and add it after `contacts-view`.

```clj
(defn contact-view [contact owner]
  (reify
    om/IRender
    (render [this]
      (dom/li nil (display-name contact)))))
```

Now save the file and reload the browser. You'll see no changes and
some errors in the the browser console or in the same screen (look for
the yellow message in the bottom). Figwheel refuses to load code that
has warnings. Go to the console: 

```
WARNING: Use of undeclared Var om-tut.core/contact-view at line 24 src/om_tut/core.cljs
WARNING: Use of undeclared Var om-tut.core/display-name at line 30 src/om_tut/core.cljs
```

It warns that `contact-view` is undefined, because it's defined *after* `contacts-view`. 

So let's move it before and save it again to check that the warning is gone and we only have the warning about the actually missing `display-name` function.

So now for `display-name`. Let's add it *before* `contact-view`.

```clj
(defn display-name [{:keys [first last] :as contact}]
  (str last ", " first (middle-name contact)))
```

Here we get a taste of map destructuring. Pretty handy. Finally let's
write the `middle-name` helper, it should come before `display-name`
in your file.

```clj
(defn middle-name [{:keys [middle middle-initial]}]
  (cond
    middle (str " " middle)
    middle-initial (str " " middle-initial ".")))
```

Again some map destructuring.

After you save you should see the list of contacts.

## Enhancing your first Om component

Let's try deleting contacts. Change `contact-view` to the following:

```clj
(defn contact-view [contact owner]
  (reify
    om/IRender
    (render [this]
      (dom/li nil
        (dom/span nil (display-name contact))
        (dom/button nil "Delete")))))
```

Save it. You should see delete
buttons now, however clicking on them won't do anything.

Contacts don't need to be able to delete themselves, however they should
be able to communicate to some entity that does have that power.

## Intercomponent communication

For communication between components we will use core.async
channels. Change your namespace form to the following:

```clj
(ns om-tut.core
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [figwheel.client :as fw]
    		[om.core :as om :include-macros true]
            [om.dom :as dom :include-macros true]
            [cljs.core.async :refer [put! chan <!]]))
```

Save your file and refresh the browser. (Note: as we have changed
the namespace form and added a dependency in `project.clj` you will
need to stop/restart the `lein figwheel` process). Change
`contact-view` to the following:

```clj
(defn contact-view [contact owner]
  (reify
    om/IRenderState
    (render-state [this {:keys [delete]}]
      (dom/li nil
        (dom/span nil (display-name contact))
        (dom/button #js {:onClick (fn [e] (put! delete @contact))} "Delete")))))
```

We've changed `om/IRender` to `om/IRenderState`. This is because we
will receive the delete notification channel as part of our component
state. We've also added a `onClick` handler to the button which writes
the contact onto the channel. This is actually a bug as we will soon
see.

Change `contacts-view` to the following. It's a big change, don't
worry we'll walk through all of it.

```clj
(defn contacts-view [data owner]
  (reify
    om/IInitState
    (init-state [_]
      {:delete (chan)})
    om/IWillMount
    (will-mount [_]
      (let [delete (om/get-state owner :delete)]
        (go (loop []
          (let [contact (<! delete)]
            (om/transact! data :contacts
              (fn [xs] (vec (remove #(= contact %) xs))))
            (recur))))))
    om/IRenderState
    (render-state [this {:keys [delete]}]
      (dom/div nil
        (dom/h2 nil "Contact list")
        (apply dom/ul nil
          (om/build-all contact-view (:contacts data)
            {:init-state {:delete delete}}))))))
```

**Reminder:** Notice that we're using `vec` to transform the result of `remove` (a lazy sequence) back into a vector, consistent with our aforementioned rule that state should only consist of associative data structures like maps and vectors. 

First we set the initial state by implementing `om/IInitState`. We
just allocate a core.async channel. It's extremely important we don't
do this in a `let` binding around `reify`. This is a common
mistake. Remember the `contacts-view` function will potentially be
invoked many, many times. We also change `om/IRender` to
`om/IRenderState` as we want to get the delete channel so we can pass
it down.

We then implement `om/IWillMount` so that we can establish a go loop
that will listen for events from the children contact views. If we get
a delete event, we remove the associated contact from the application state
with `om.core/transact!`. You should now be able to delete people from the
contact list.

## Adding Contacts

Let's modify our application so we can add new contacts. Change the
top namespace form to the following, restart `lein figwheel`, and refresh your browser:

```clj
(ns om-tut.core
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [figwheel.client :as fw]
  			[om.core :as om :include-macros true]
            [om.dom :as dom :include-macros true]
            [cljs.core.async :refer [put! chan <!]]
            [clojure.data :as data]
            [clojure.string :as string]))
```

Let's add a new function called `parse-contact` and save it.

```clj
(defn parse-contact [contact-str]
  (let [[first middle last :as parts] (string/split contact-str #"\s+")
        [first last middle] (if (nil? last) [first middle] [first last middle])
        middle (when middle (string/replace middle "." ""))
        c (if middle (count middle) 0)]
    (when (>= (count parts) 2)
      (cond-> {:first first :last last}
        (== c 1) (assoc :middle-initial middle)
        (>= c 2) (assoc :middle middle)))))
```

There are of course many ways to write `parse-contact` and this is not
particularly the best way, however it illustrates many common
idioms. If you're not experienced with Clojure or ClojureScript it's
worth taking the time to understand it before proceeding.

Try it on some input!

```clj
(parse-contact "Gerald J. Sussman")
```

If Figwheel throws a ReferenceError with the message `clojure is not
defined` or `string is not defined`, shut down the REPL. Then enter the command:

```
lein clean
```

... restarting Figwheel and refreshing the browser will clear the problem.

Once you've seen that it basically works lets write `add-contact`, it
should look like the following:

```clj
(defn add-contact [data owner]
  (let [new-contact (-> (om/get-node owner "new-contact")
                        .-value
                        parse-contact)]
    (when new-contact
      (om/transact! data :contacts #(conj % new-contact)))))
```

We need to use `om.core/get-node` so that we can extract the value
from the text field. We'll see how we set this up, your contacts view
should look like the following:

```clj
(defn contacts-view [data owner]
  (reify
    om/IInitState
    (init-state [_]
      {:delete (chan)})
    om/IWillMount
    (will-mount [_]
      (let [delete (om/get-state owner :delete)]
        (go (loop []
              (let [contact (<! delete)]
                (om/transact! data :contacts
                  (fn [xs] (vec (remove #(= contact %) xs))))
                (recur))))))
    om/IRenderState
    (render-state [this state]
      (dom/div nil
        (dom/h2 nil "Contact list")
        (apply dom/ul nil
          (om/build-all contact-view (:contacts data)
            {:init-state state}))
        (dom/div nil
          (dom/input #js {:type "text" :ref "new-contact"})
          (dom/button #js {:onClick #(add-contact data owner)} "Add contact"))))))
```

Notice that the input field specified `:ref`, this is a feature of
React for the few cases where you need direct access to a DOM node.

Save the file. You should now be able to add contacts as long as you
provide at least a first and last name.

This is a lot of information. As a challenge I recommend trying to
clear the text field when a real contact has been added. This is
harder than it looks so don't get discouraged. If you spend more than
15, 20 minutes on it feel free to proceed to the next section and
we'll show how to do it.

## Dealing with text input fields

React's declarative model, and thus Om's, makes dealing with input
both a little more challenging, but also more flexible. The "easiest"
way to clear the text field would be by changing `add-contact` to
the following:

```clj
(defn add-contact [data owner]
  (let [input (om/get-node owner "new-contact")
        new-contact (-> input .-value parse-contact)]
    (when new-contact
      (om/transact! data :contacts #(conj % new-contact))
      (set! (.-value input) ""))))
```

This works great, but any time you find yourself leaning heavily on
refs it's probably worth stepping back and considering whether the
same thing could be accomplished in a more declarative manner.

Since `contacts-view` "owns" the text field we should consider making
its value a part of `contacts-view`'s local state. Let's change
`contacts-view` to the following and save it:

```clj
(defn contacts-view [data owner]
  (reify
    om/IInitState
    (init-state [_]
      {:delete (chan)
       :text ""})
    om/IWillMount
    (will-mount [_]
      (let [delete (om/get-state owner :delete)]
        (go (loop []
              (let [contact (<! delete)]
                (om/transact! data :contacts
                  (fn [xs] (vec (remove #(= contact %) xs))))
                (recur))))))
    om/IRenderState
    (render-state [this state]
      (dom/div nil
        (dom/h2 nil "Contact list")
        (apply dom/ul nil
          (om/build-all contact-view (:contacts data)
            {:init-state state}))
        (dom/div nil
          (dom/input #js {:type "text" :ref "new-contact" :value (:text state)})
          (dom/button #js {:onClick #(add-contact data owner)} "Add contact"))))))
```

Try typing in the text field.

Yikes! You can no longer enter anything. Let's take a moment to
consider what's going on.

We've added a new piece of state to `contacts-view`. Regardless of
what the user may type we are now setting the value of the input field to
the value of the `:text` state property. We need to keep this in sync
with the user input. Let's change `contacts-view` again by adding an
event listener to watch when the input field changes:

```clj
(defn contacts-view [data owner]
  (reify
    om/IInitState
    (init-state [_]
      {:delete (chan)
       :text ""})
    om/IWillMount
    (will-mount [_]
      (let [delete (om/get-state owner :delete)]
        (go (loop []
              (let [contact (<! delete)]
                (om/transact! data :contacts
                  (fn [xs] (vec (remove #(= contact %) xs))))
                (recur))))))
    om/IRenderState
    (render-state [this state]
      (dom/div nil
        (dom/h2 nil "Contact list")
        (apply dom/ul nil
          (om/build-all contact-view (:contacts data)
            {:init-state state}))
        (dom/div nil
          (dom/input 
            #js {:type "text" :ref "new-contact" :value (:text state)
                 :onChange #(handle-change % owner state)})
          (dom/button #js {:onClick #(add-contact data owner)} "Add contact"))))))
```

Before saving it that let's add `handle-change` before `contacts-view`:

```clj
(defn handle-change [e owner {:keys [text]}]
  (om/set-state! owner :text (.. e -target -value)))
```

Now save all the changes. You should now be able to type in the text
field again.

Let's finally add the piece of code that clears the text field. As you see it looks like our first "easy" attempt, except that we're no longer directly manipulating a ref but we're changing the state of the app:

```clj
(defn add-contact [data owner]
  (let [new-contact (-> (om/get-node owner "new-contact")
                        .-value
                        parse-contact)]
    (when new-contact
      (om/transact! data :contacts #(conj % new-contact))
      (om/set-state! owner :text ""))))
```

That seemed like a lot of work for little gain ... except we just saw
we have really fine grained control over user input entry. For example
a name can't have a number in it, let's prevent that now by modifying
`handle-change`:

```clj
(defn handle-change [e owner {:keys [text]}]
  (let [value (.. e -target -value)]
    (if-not (re-find #"[0-9]" value)
      (om/set-state! owner :text value)
      (om/set-state! owner :text text))))
```

Now save. You can type names however no change will occur if you
attempt to enter a number. Pretty slick.

**Note**: If you are familiar with React you'll notice that this is a
little bit clunkier than in React, here we have to make sure to set
the state of the text field even if we don't want it to change. This
is a side effect of React's internals clashing a bit with Om's
optimization of always rendering on requestAnimationFrame.

## Higher Order Components

The most powerful components are those whose sub-components can be
parameterized. In order to focus on this concept let's leave aside user
input and other complications for now. As a challenge you should try
to re-add these facilities yourself after you've worked through this
section.

Let's start fresh. Your `resources/public/index.html` should look like the
following: 

```html
<!DOCTYPE html>
<html>
  <head>
    <link href="css/style.css" rel="stylesheet" type="text/css">
  </head>
  <body>
    <div id="registry"></div>
    <script src="js/compiled/om_tut.js" type="text/javascript"></script>
  </body>
</html>
```

Your source file should look like the following:

```clj
(ns om-tut.core
  (:require [figwheel.client :as fw]
    		[om.core :as om :include-macros true]
            [om.dom :as dom :include-macros true]
            [clojure.string :as string]))

(enable-console-print!)

(def app-state
  (atom
    {:people
     [{:type :student :first "Ben" :last "Bitdiddle" :email "benb@mit.edu"}
      {:type :student :first "Alyssa" :middle-initial "P" :last "Hacker" 
       :email "aphacker@mit.edu"}
      {:type :professor :first "Gerald" :middle "Jay" :last "Sussman" 
       :email "metacirc@mit.edu" :classes [:6001 :6946]}
      {:type :student :first "Eva" :middle "Lu" :last "Ator" :email "eval@mit.edu"}
      {:type :student :first "Louis" :last "Reasoner" :email "prolog@mit.edu"}
      {:type :professor :first "Hal" :last "Abelson" :email "evalapply@mit.edu" 
       :classes [:6001]}]
     :classes
     {:6001 "The Structure and Interpretation of Computer Programs"
      :6946 "The Structure and Interpretation of Classical Mechanics"
      :1806 "Linear Algebra"}}))

(defn middle-name [{:keys [middle middle-initial]}]
  (cond
    middle (str " " middle)
    middle-initial (str " " middle-initial ".")))

(defn display-name [{:keys [first last] :as contact}]
  (str last ", " first (middle-name contact)))

(defn registry-view [data owner]
  (reify
    om/IRenderState
    (render-state [_ state]
      (dom/div nil
        (dom/h2 nil "Registry")))))

(defn main []
  (om/root registry-view app-state
    {:target (. js/document (getElementById "registry"))}))
```

Now what we want is for `registry-view` to render different views for
different types of people without hardcoding. Here we'll see how
multimethods in ClojureScript really shine.

After `display-name` let's write the following:

```clj
(defmulti entry-view (fn [person _] (:type person)))

(defmethod entry-view :student
  [person owner] (student-view person owner))

(defmethod entry-view :professor
  [person owner] (professor-view person owner)) 
```

Don't save these yet as we haven't written `student-view` or
`professor-view`. Take a moment to let the idea sink, we're adding a
level of indirection so that `entry-view` can delegate to any other
view as long as we supplied a `defmethod` for it!

Let's write the underlying views now before `entry-view`:

```clj
(defn student-view [student owner]
  (reify
    om/IRender
    (render [_]
      (dom/li nil (display-name student)))))

(defn professor-view [professor owner]
  (reify
    om/IRender
    (render [_]
      (dom/li nil
        (dom/div nil (display-name professor))
        (dom/label nil "Classes")
        (apply dom/ul nil
          (map #(dom/li nil %) (:classes professor)))))))
```

Finally let's fix up `registry-view`. It's succinct and we've kept it
clean of conditionals.

```clj
(defn registry-view [data owner]
  (reify
    om/IRenderState
    (render-state [_ state]
      (dom/div #js {:id "registry"}
        (dom/h2 nil "Registry")
        (apply dom/ul nil
          (om/build-all entry-view (people data)))))))
```

The only missing bit now is the `people` function. We want to make
sure to render professors with their list of actual class
titles. Before `registry-view` write the following:

```clj
(defn people [data]
  (->> (:people data)
    (mapv (fn [x]
            (if (:classes x)
              (update-in x [:classes]
                (fn [cs] (mapv (:classes data) cs)))
               x)))))
```

Save everything and you should see the results.

Hopefully the view composition and extensibility offered by Om puts
a gleam in your eye. There's more than enough here to keep you occupied for
hours if you wish to take it further yourself.

Otherwise in the next section we'll show you how we can easily modify
each professor's class list and make those changes visible in
multiple locations on the screen.

## Interactivity & Higher Order Components

Let's change `resources/public/index.html` to the following:

```html
<!DOCTYPE html>
<html>
  <head>
    <link href="css/style.css" rel="stylesheet" type="text/css">
  </head>
  <body>
    <div id="registry"></div>
    <div id="classes"></div>
    <script src="js/compiled/om_tut.js" type="text/javascript"></script>
  </body>
</html>
```

and `resources/public/css/style.css` to:

```css
ul li input {
    width: 400px;
}
ul li button {
    margin-left: 10px;
}
```

Let's add `classes-view` after `registry-view`:

```clj
(defn classes-view [data owner]
  (reify
    om/IRender
    (render [_]
      (dom/div #js {:id "classes"}
        (dom/h2 nil "Classes")
        (apply dom/ul nil
          (map #(dom/li nil %) (vals (:classes data))))))))
```

Let's use it by adding a new `om/root` expression after the existing
one:

```clj
(om/root classes-view app-state
  {:target (. js/document (getElementById "classes"))})
```

Save all the files, you should see the new bits of UI.

Let's make class names editable inline. To accomplish this we want to
make a reusable component that we can plug in wherever we like. This
will be the most complicated component we have seen so far.

When the user edits a class name we should hide the original text and
present an input field. So we need a little helper function to do
this. After `app-state` add `display`:

```clj
(defn display [show]
  (if show
    #js {}
    #js {:display "none"}))
```

Every time a user presses a key while editing a class name we want to
update the application state:

```clj
(defn handle-change [e text owner]
  (om/transact! text (fn [_] (.. e -target -value))))
```

When the input loses focus we want to exit editing mode:

```clj
(defn commit-change [text owner]
  (om/set-state! owner :editing false))
```

These are all helpers for our soon to be written `editable`
component. `editable` takes a JavaScript string and presents it while
also making it editable. In order for this to work the JavaScript
string needs to support the Om cursor interface. We don't need to
implement this ourselves but we do need to make sure that JavaScript
strings implement `ICloneable` so that Om can do the hard work for us
(*Note the following is for demonstration purposes only, it is not
recommended in most real applications*. Please refer to the
[Intermediate Tutorial](http://github.com/swannodette/om/wiki/Intermediate-Tutorial)
for a better approach that does not require extending JavaScript
native strings to ICloneable*).

**Be sure to put these `extend-type` forms *before* the `om/root` forms.** Putting them near the top of the file will do nicely.

```clj
(extend-type string
  ICloneable
  (-clone [s] (js/String. s)))
```

Sadly this is not enough because JavaScript String objects and
JavaScript primitive strings are not the same thing:

```clj
(extend-type js/String
  ICloneable
  (-clone [s] (js/String. s))
  om/IValue
  (-value [s] (str s)))
```

We'll explain `IValue` in a moment. The ClojureScript compiler will
emit a warning. Normally you don't want to ignore it but in this case
we'll make an exception in order to keep the `editable` component as
simple as possible. Since Figwheel doesn't reload code with warnings
by default replace the call to Figwheel in `src/om_tut/core.cljs` to:

```clj
(fw/start {
           :load-warninged-code true  ;; <- Add this
           :on-jsload (fn []
                        ;; (stop-and-start-my app)
                        )})
```

Now refresh the browser. If you are still unable to load the code, do
`lein clean` and restart `lein figwheel`.

This is the editable component; this might look like a lot but take a
moment to read it and you'll see that it's quite simple.

```clj
(defn editable [text owner]
  (reify
    om/IInitState
    (init-state [_]
      {:editing false})
    om/IRenderState
    (render-state [_ {:keys [editing]}]
      (dom/li nil
        (dom/span #js {:style (display (not editing))} (om/value text))
        (dom/input
          #js {:style (display editing)
               :value (om/value text)
               :onChange #(handle-change % text owner)
               :onKeyDown #(when (= (.-key %) "Enter")
                              (commit-change text owner))
               :onBlur (fn [e] (commit-change text owner))})
        (dom/button
          #js {:style (display (not editing))
               :onClick #(om/set-state! owner :editing true)}
          "Edit")))))
```

We have to use `om.core/value` here because React doesn't know how to
handle JavaScript String objects. This is also why we implement
`IValue` above.

We also need to update `professor-view` since the class title might be
a String object instead of a primitive string:

```clj
(defn professor-view [professor owner]
  (reify
    om/IRender
    (render [_]
      (dom/li nil
        (dom/div nil (display-name professor))
        (dom/label nil "Classes")
        (apply dom/ul nil
          (map #(dom/li nil (om/value %)) (:classes professor)))))))
```

Let's use `editable` in `classes-view`:

```clj
(defn classes-view [data owner]
  (reify
    om/IRender
    (render [_]
      (dom/div #js {:id "classes"}
        (dom/h2 nil "Classes")
        (apply dom/ul nil
          (om/build-all editable (vals (:classes data))))))))
```

That's it, save it. You should now be able to edit class titles in the
`classes-view`. Notice that the class titles in `registry-view` stay
perfectly in sync.

As a challenge render the classes in `professor-view` with `editable`
instead of just rendering strings. It's just a one line change. If you
get it working notice how editing the class titles from deep within
the `registry-view` "just works". This is the modularity that Om
cursors provide.
