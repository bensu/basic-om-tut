For this next tutorial we're going to get a bit more
ambitious. We're going to build an Om frontend to a simple
[Ring](http://github.com/ring-clojure) application that talks to
[Datomic](http://datomic.com) for persistence. You can of course swap
in another database, but Datomic is particularly easy to use from
Clojure and also provides time travel capabilities making it quite
nice to pair with Om.

First download a copy of
[Datomic Free](http://my.datomic.com/downloads/free). You might get an
error with versions of Datomic later than `0.9.5130`, so try that one
first (seems to be working as of `0.9.5130`). Unzip it and run the
following inside the directory:

```
bin/transactor config/samples/free-transactor-template.properties
```

This will start up the Datomic transactor.

Now in some other directory run the following to generate the tutorial
from the `om-intermediate-template` Lein template:

```
lein new om-intermediate-template om-async
```

`cd` into `om-async` and launch a Lein repl with `lein repl`. Once the
repl is up run the following:

```
user=> (use '[om-async.util])
nil
user=> (init-db)
:done
```

The tutorial database now exists and is populated. Quit the
REPL.

If you got an error, try an earlier version of Datomic Free.

We don't have time to cover all the details of Ring or Datomic here
but hopefully you get the basic idea. If you're curious about Datomic
I highly recommend Jonas Enlund's
[tutorial](http://www.learndatalogtoday.org) and the
[Day of Datomic](http://github.com/Datomic/day-of-datomic) tutorial.

We will use [Figwheel](https://github.com/bhauman/lein-figwheel) to
reload our frontend ClojureScript while we code. Figwheel uses a
server to auto compile our code and push it to the browser. But we
also need a server running our backend code. For that we will use the
`lein-ring` [plugin](https://github.com/weavejester/lein-ring). To
tell ring what it should run, we specify it in our `project.clj`:

```clj
  :ring {:handler om-async.core/handler
         :port 8000}
```

To understand the backend code, open `src/clj/om-async/core.clj` in
your favorite editor. This code creates `om-async.core/handler` that
accepts requests to read and write to Datomic. It also serves the
static files and the compiled JavaScript files that our ClojureScript code will
generate.

To start it run:

    lein ring server 

When the server is up, point your browser at
[localhost:8000](http://localhost:8000). If you open the JavaScript
console you should see that `main.js` is missing. That is because we
have not compiled our Clojurescript code yet. Open another terminal
and run:

    lein figwheel
    
When it is done compiling, check if the Browser REPL is connected by typing:

```
ClojureScript:cljs.user> (js/alert "Am I connected?")
```

You should see the alert in your browser. 

Now let's read the server side code located in
`src/clj/om-async/core.clj`. At the top of the file we have the usual
namespace stuff:

```clj
(ns om-async.core
  (:require [ring.util.response :refer [file-response]]
            [ring.adapter.jetty :refer [run-jetty]]
            [ring.middleware.edn :refer [wrap-edn-params]]
            [compojure.core :refer [defroutes GET PUT]]
            [compojure.route :as route]
            [compojure.handler :as handler]
            [datomic.api :as d]))
```

Then we establish a connection to Datomic:

```clj
(def uri "datomic:free://localhost:4334/om_async")
(def conn (d/connect uri))
```

We then define our main route handler `index`:

```clj
(defn index []
  (file-response "public/html/index.html" {:root "resources"}))
```

Instead of [JSON](http://www.json.org) as a data format, we'll
use [EDN](http://github.com/edn-format/edn). We write a little helper
for the EDN middleware we'll be using:

```clj
(defn generate-response [data & [status]]
  {:status (or status 200)
   :headers {"Content-Type" "application/edn"}
   :body (pr-str data)})
```

Skip ahead briefly and let's take a look at `classes`:

```clj
(defn classes []
  (let [db (d/db conn)
        classes
        (vec (map #(d/touch (d/entity db (first %)))
               (d/q '[:find ?class
                      :where
                      [?class :class/id]]
                 db)))]
    (generate-response classes)))
```

This just finds all the classes and returns an EDN response.

Back up and look at `update-class`. It finds a class by id and modifies the
title:

```clj
(defn update-class [id params]
  (let [db    (d/db conn)
        title (:class/title params)
        eid   (ffirst
                (d/q '[:find ?class
                       :in $ ?id
                       :where 
                       [?class :class/id ?id]]
                  db id))]
    (d/transact conn [[:db/add eid :class/title title]])
    (generate-response {:status :ok})))
```

We then have our `routes`:

```clj
(defroutes routes
  (GET "/" [] (index))
  (GET "/classes" [] (classes))
  (PUT "/class/:id/update"
    {params :params edn-params :edn-params}
    (update-class (:id params) edn-params))
  (route/files "/" {:root "resources/public"}))
```

Finally we add our EDN middleware to get our final handler:

```clj
(def handler 
  (-> routes
      wrap-edn-params))
```

Let's look at the client side portion now, open
`src/cljs/om-async/core.cljs` in your editor. The `ns` form should
look familiar, and we enable `console.log` printing. The last line is
what tells Figwheel that we want to do code reloading:

```clj
(ns om-async.core
  (:require [cljs.reader :as reader]
            [figwheel.client :as fw]
            [goog.events :as events]
            [goog.dom :as gdom]
            [om.core :as om :include-macros true]
            [om.dom :as dom :include-macros true])
  (:import [goog.net XhrIo]
           goog.net.EventType
           [goog.events EventType]))

(enable-console-print!)

(println "Hello world!")

(fw/start {:websocket-url "ws://localhost:3449/figwheel-ws"})
```

We're going to use simple callbacks in this tutorial instead of
relying on core.async.

First we write a simple utility for making async requests to the
server with EDN. We rely on Google Closure to deal with cross browser
issues:

```clj
(def ^:private meths
  {:get "GET"
   :put "PUT"
   :post "POST"
   :delete "DELETE"})

(defn edn-xhr [{:keys [method url data on-complete]}]
  (let [xhr (XhrIo.)]
    (events/listen xhr goog.net.EventType.COMPLETE
      (fn [e]
        (on-complete (reader/read-string (.getResponseText xhr)))))
    (. xhr
      (send url (meths method) (when data (pr-str data))
        #js {"Content-Type" "application/edn"}))))
```

Next we define our `app-state`. In this case we won't have initial
data because we will need to fetch it from the server. Still we need
to provide the basic structure:

```clj
(def app-state
  (atom {:classes []}))
```

Next we need the `display` style helper from the previous tutorial:

```clj
(defn display [show]
  (if show
    #js {}
    #js {:display "none"}))
```

We are now going to write a version of the `editable` component that
does not require hacking around the differences between JavaScript
primitive strings and JavaScript String objects. The following is
strongly recommended over extending native types to `ICloneable`.

Instead of `editable` taking a string cursor from the application state,
it will take some larger piece of data and a key to locate the
string to edit. `handle-change` now looks like this:

```clj
(defn handle-change [e data edit-key owner]
  (om/transact! data edit-key (fn [_] (.. e -target -value))))
```

Instead of using channels because `editable` is so simple and to
demonstrate alternative approaches, `editable` will notify its parent
component with a callback when editing is complete:

```clj
(defn end-edit [text owner cb]
  (om/set-state! owner :editing false)
  (cb text))
```
This is `editable`:

```clj
(defn editable [data owner {:keys [edit-key on-edit] :as opts}]
  (reify
    om/IInitState
    (init-state [_]
      {:editing false})
    om/IRenderState
    (render-state [_ {:keys [editing]}]
      (let [text (get data edit-key)]
        (dom/li nil
          (dom/span #js {:style (display (not editing))} text)
          (dom/input
            #js {:style (display editing)
                 :value text
                 :onChange #(handle-change % data edit-key owner)
                 :onKeyDown #(when (= (.-key %) "Enter")
                                (end-edit text owner on-edit))
                 :onBlur (fn [e]
                           (when (om/get-state owner :editing)
                             (end-edit text owner on-edit)))})
          (dom/button
            #js {:style (display (not editing))
                 :onClick #(om/set-state! owner :editing true)}
            "Edit"))))))
```

Take the time to read through and understand this code. It's almost
identical to the previous version with the exception that we take
`data` which is a map cursor. We also take an `edit-key` and the
`on-edit` callback via `opts`.

We now have a generic editable component that doesn't require us to hack
JavaScript strings.

When the user commits a change we want to communicate this back to the
server:

```clj
(defn on-edit [id title]
  (edn-xhr
    {:method :put
     :url (str "class/" id "/update")
     :data {:class/title title}
     :on-complete
     (fn [res]
       (println "server response:" res))}))
```

Our `classes-view` will load the data from server on
`om.core/IWillMount`.

```clj
(defn classes-view [app owner]
  (reify
    om/IWillMount
    (will-mount [_]
      (edn-xhr
        {:method :get
         :url "classes"
         :on-complete #(om/transact! app :classes (fn [_] %))}))
    om/IRender
    (render [_]
      (dom/div #js {:id "classes"}
        (dom/h2 nil "Classes")
        (apply dom/ul nil
          (map
            (fn [class]
              (let [id (:class/id class)]
                (om/build editable class
                  {:opts {:edit-key :class/title
                          :on-edit #(on-edit id %)}})))
            (:classes app)))))))

(om/root classes-view app-state
  {:target (gdom/getElement "classes")})
```

That's it. Save your file. You should see the list of
classes loaded from the server. You should be able to edit a class
title. Press enter to commit the title change. Refresh your browser
and you should see that the change persisted.

Both your backend and frontend can support sophisticated operations on
application state history without incurring incidental complexity.

## Speculative UI Programming

Many difficulties in modern UI programming arise from the fact that
we're often building distributed systems - clients and servers. Not
only do we have to keep clients and servers in sync but we have to
gracefully handle those cases where sync is not possible.

It's quite common in modern applications to make an optimistic commit,
and then discover that synchronization cannot happen - perhaps
because of a backend bug, undefined behavior in the API
, or commonly the complete lack of network connectivity. In this
distributed setting UI programming becomes a speculative endeavor.

One well known form of speculative UI programming is multi-level
undo. It's telling that many single page user interfaces on the web do
not support this kind of undo - it's hard enough to implement in a
desktop application. Add the distributed component and most developers
raise the white flag.

However an immutable UI approach like Om and immutable databases like
Datomic provide the infrastructure to make these problems very
approachable even in a distributed context.

We'll come back to this but first let's explore modularity in the
context of synchronizing application state.

### Modularity

Ideally we can build our UIs without considering synchronization. And
when we come to the problem of synchronization we should be able to add
it to our application without changing any code we have already written.

This is effectively the goal of
[om-sync](http://github.com/swannodette/om-sync), a reusable
synchronization component for Om. You can take your existing
component, put together an application and impose a synchronization
policy without changing your own code.

Let's modify the previous tutorial so that we can demonstrate this now.

#### Updating the Server

Make sure the Datomic transactor is running as before.

First we need to change our `project.clj` to include a dependency on
`om-sync`.

```clj
  :dependencies [[org.clojure/clojure "1.6.0"]
                 [org.clojure/clojurescript "0.0-2727"]
                 [org.clojure/core.async "0.1.346.0-17112a-alpha"]
                 [org.omcljs/om "0.8.7"]
                 [om-sync "0.1.1"] ;; <=== ADD THIS
                 [ring "1.3.2"]
                 [compojure "1.3.1"]
                 [figwheel "0.2.2-SNAPSHOT"]
                 [fogus/ring-edn "0.2.0"]
                 [com.datomic/datomic-free "0.9.5130" :exclusions [joda-time]]]
```

Let's update the server side code to uniformly handle EDN requests.

After `generate-response` let's add `get-classes`:

```clj
(defn get-classes [db]
  (->> (d/q '[:find ?class
              :where
              [?class :class/id]]
          db)
       (map #(d/touch (d/entity db (first %))))
       vec))
```

We want to be able to properly initialize the client state as well as
give it information so that it can communicate back with the server:

Let's make a request handler called `init`:

```clj
(defn init []
  (generate-response
     {:classes {:url "/classes" :coll (get-classes (d/db conn))}}))
```

We're also going to add a handler for creating classes but we're going
to return a 500 for now to demonstrate some neat properties of Om:

```clj
(defn create-class [params]
  {:status 500})
```

Let's refactor `update-class` a bit:

```clj
(defn update-class [params]
  (let [id    (:class/id params)
        db    (d/db conn)
        title (:class/title params)
        eid   (ffirst
                (d/q '[:find ?class
                       :in $ ?id
                       :where
                       [?class :class/id ?id]]
                  db id))]
    (d/transact conn [[:db/add eid :class/title title]])
    (generate-response {:status :ok})))
```

Let's also refactor the `classes` request handler:

```clj
(defn classes []
  (generate-response (get-classes (d/db conn))))
```

We are introducing the `POST` HTTP method so need to make sure it is imported.
Add `POST` to the `compojure.core` import at the top of the file:

```clj
(ns om-async.core
  (:require [ring.util.response :refer [file-response]]
            [ring.adapter.jetty :refer [run-jetty]]
            [ring.middleware.edn :refer [wrap-edn-params]]
            [compojure.core :refer [defroutes GET PUT POST]] ;; <=== ADD POST
            [compojure.route :as route]
            [compojure.handler :as handler]
            [datomic.api :as d]))
```

And let's provide the new routes:

```clj
(defroutes routes
  (GET "/" [] (index))
  (GET "/init" [] (init))
  (GET "/classes" [] (classes))
  (POST "/classes" {params :edn-params} (create-class params))
  (PUT "/classes" {params :edn-params} (update-class params))
  (route/files "/" {:root "resources/public"}))
```

Since we modified our `ns` declaration you might need to restart `lein
ring server`. That's it for our server side code. Let's update the
client code.

#### Updating the Client

You should also restart `lein figwheel` since our our dependencies have changed.

First we need to modify the namespace form. Since we'll be using
`om-sync` we clean up some things:

```clj
(ns om-async.core
  (:require-macros [cljs.core.async.macros :refer [go]])
  (:require [cljs.core.async :as async :refer [put! chan alts!]]
            [figwheel.client :as fw]
            [goog.dom :as gdom]
            [om.core :as om :include-macros true]
            [om.dom :as dom :include-macros true]
            [om-sync.core :refer [om-sync]]
            [om-sync.util :refer [tx-tag edn-xhr]]))
```

Remove the edn-xhr function created earlier. We now use the version
defined in om-sync.util.

In order for `om-sync` to work you need modify how you call
`om.core/root`. `om-sync` needs to be able to subscribe to the
application's transactions so that it can observe transactions that
are relevant to it.

We use core.async to publish a channel that components like `om-sync`
can subscribe to. This is done by making the channel a global service via
`:shared`. We also make an EDN request to get our initial state using
the new `init` route we wrote on the backend. Place the following at the
very bottom of the page.

```clj
(let [tx-chan (chan)
      tx-pub-chan (async/pub tx-chan (fn [_] :txs))]
  (edn-xhr
    {:method :get
     :url "/init"
     :on-complete
     (fn [res]
       (reset! app-state res)
       (om/root app-view app-state
         {:target (gdom/getElement "classes")
          :shared {:tx-chan tx-pub-chan}
          :tx-listen
          (fn [tx-data root-cursor]
            (put! tx-chan [tx-data root-cursor]))}))}))
```

We create a channel `tx-chan`. We then make a subscribeable channel
`tx-pub-chan` so that `om-sync` instances can call
`cljs.core.async/sub` on it. We request the initial state of the
application from the server. Here we use some `om.core/root` options we
have not seen before. `:shared` allows us to provide a global service
to any components in our application. `:tx-listen` is a callback that
will be invoked anytime the application state transitions. We simply
put this information into `tx-chan`.

Technically we don't need to change `editable` to make this work,
but we're going to make `editable` a better citizen in order to
explain how `om-sync` works.

First modify `end-edit`:

```clj
(defn end-edit [data edit-key text owner cb]
  (om/set-state! owner :editing false)
  (om/transact! data edit-key (fn [_] text) :update)
  (when cb
    (cb text)))
```

The changes here are most around the fact that we now invoke
`om/transact!`. The interesting part is that this `om/transact!`
supplies a tag, `:update`. `om-sync` specifically listens for
`:create`, `:update`, and `:delete` tags. We could make this work
without the transaction tag, but not as simply.

`editable` need to change to accommodate the new `end-edit` signature:

```clj
(defn editable [data owner {:keys [edit-key on-edit] :as opts}]
  (reify
    om/IInitState
    (init-state [_]
      {:editing false})
    om/IRenderState
    (render-state [_ {:keys [editing]}]
      (let [text (get data edit-key)]
        (dom/li nil
          (dom/span #js {:style (display (not editing))} text)
          (dom/input
            #js {:style (display editing)
                 :value text
                 :onChange #(handle-change % data edit-key owner)
                 :onKeyDown #(when (= (.-key %) "Enter")
                                (end-edit data edit-key text owner on-edit))
                 :onBlur (fn [e]
                           (when (om/get-state owner :editing)
                             (end-edit data edit-key text owner on-edit)))})
          (dom/button
            #js {:style (display (not editing))
                 :onClick #(om/set-state! owner :editing true)}
            "Edit"))))))
```

We also want to allow people to add classes. We know that the backend
doesn't support this but we'll try it on the front end anyway to
demonstrate the capabilities of `om-sync`.

```clj
(defn create-class [classes owner]
  (let [class-id-el   (om/get-node owner "class-id")
        class-id      (.-value class-id-el)
        class-name-el (om/get-node owner "class-name")
        class-name    (.-value class-name-el)
        new-class     {:class/id class-id :class/title class-name}]
    (om/transact! classes [] #(conj % new-class)
      [:create new-class])
    (set! (.-value class-id-el) "")
    (set! (.-value class-name-el) "")))
```

We update `classes-view` to include some new input fields:

```clj
(defn classes-view [classes owner]
  (reify
    om/IRender
    (render [_]
      (dom/div #js {:id "classes"}
        (dom/h2 nil "Classes")
        (apply dom/ul nil
          (map
            (fn [class]
              (let [id (:class/id class)]
                (om/build editable class
                  {:opts {:edit-key :class/title}})))
            classes))
        (dom/div nil
          (dom/label nil "ID:")
          (dom/input #js {:ref "class-id"})
          (dom/label nil "Name:")
          (dom/input #js {:ref "class-name"})
          (dom/button
            #js {:onClick (fn [e] (create-class classes owner))}
           "Add"))))))
```

All this should look pretty straightforward by now.

Finally we write `app-view`, add this right before the `om/root`
expression:

```clj
(defn app-view [app owner]
  (reify
    om/IWillUpdate
    (will-update [_ next-props next-state]
      (when (:err-msg next-state)
        (js/setTimeout #(om/set-state! owner :err-msg nil) 5000)))
    om/IRenderState
    (render-state [_ {:keys [err-msg]}]
      (dom/div nil
        (om/build om-sync (:classes app)
          {:opts {:view classes-view
                  :filter (comp #{:create :update :delete} tx-tag)
                  :id-key :class/id
                  :on-success (fn [res tx-data] (println res))
                  :on-error
                  (fn [err tx-data]
                    (reset! app-state (:old-state tx-data))
                    (om/set-state! owner :err-msg
                      "Ooops! Sorry something went wrong try again later."))}})
         (when err-msg
           (dom/div nil err-msg))))))
```

You can remove the `on-edit` function too.  `om-sync` will handle the PUT.

We render `classes-view` via `om-sync`, this is specified by the
`:view` option. We supply a `:filter` so that individual key presses
don't get synchronized - this is why we tagged the `end-edit`
`om/transact!` above with `:update`. `om-sync` needs to know which
property represents an identifier the server understands in order to
send incremental updates. Finally we establish `:on-success` and
`:on-error` updates.

`:on-error` is the most interesting bit here - if there's a server error
we just roll back the application state and present an error message.

This is speculative UI programming, we can optimistically update the
client yet trivially roll back if something goes wrong. And all of
this can be done without polluting your actual components with
synchronization or undo logic.

Refresh your browser and make sure your JavaScript Console is
open. Edit a class. You should see `{:status ok}` if it
worked. Refresh the browser, you will see that the state persisted.

Now put some random data into the new input fields and press
"Add". You should see the data briefly appear in the list and then
disappear when we roll back the state because we could not succeed. An
error message appears.

Hopefully this tutorial demonstrates how Om allows application
developers to focus on their problem domain and remove sources of
incidental complexity.

Having made it this far you can probably read through the `om-sync`
source yourself and hopefully be inspired to send a pull request to
improve so it can be leveraged in more contexts.

In case you run intro trouble, you can find the final version of the
code [here](https://github.com/bensu/om-intermediate-tut)