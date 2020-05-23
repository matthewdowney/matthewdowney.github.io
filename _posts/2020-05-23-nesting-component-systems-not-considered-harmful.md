---
layout: post
title: Nesting Component Systems (Sometimes) Not Considered Harmful
tags:
 - Software
 - Clojure
---

Stuart Sierra's dependency injection & life cycle design pattern minimizes 
tight coupling, but in large systems the "everything is a component" mindset 
comes with its own overhead, which is luckily avoidable with good design.

Component is an elegant design pattern. It is a quintessentially Clojure-esque 
kind of dependency injection: it's small, it plays nicely with immutable data 
structures, and it enables the [reloaded](https://github.com/stuartsierra/reloaded) 
workflow that's so enjoyable to use from the REPL. Component is tempting to 
misuse however, either by introducing unnecessary [nouns](https://steve-yegge.blogspot.com/2006/03/execution-in-kingdom-of-nouns.html)
to the top level system or by using dependency injection as an excuse to not 
develop strong interfaces and abstractions.

I'm going to make the case that (1) component subsystems can and should be 
separated and nested, rather than being merged together, and that (2) libraries
can make use of component without forcing calling code to know anything about
it, and should do so where it improves their structure and maintainability.

## Prefer cohorts to subsystems?

Stuart makes the well-founded point that component systems shouldn't be directly 
nested because it could end up confusing when and how often `start` and `stop` 
are called on components of the subsystem. 

Therefore a traditional approach to grouping related components (what 
[Howard Ship calls](https://medium.com/@hlship/tips-and-tricks-for-component-d00832abcdfa) 
a _cohort_) in a flat system goes something like this:

```clojure
;;; E.g., a system where we connect to a couple APIs, aggregate their WebSocket
;;; output in a connection pool, and then perform some logic using the 
;;; connections.

(defn api-foo-subsystem [config]
  (component/system-map 
    :api-foo/config config
    :api-foo/socket (foo/websocket-connection)
    :api-foo/message-handler (foo/message-handler)
    :api-foo/out-channel (async/chan)))
    
(defn api-bar-subsystem [config]
  (component/system-map 
    :api-bar/config config
    :api-bar/socket (bar/websocket-connection)
    :api-bar/message-handler (bar/message-handler)
    :api-bar/out-channel (async/chan)))
    
(defn api-system [config]
  (merge 
    (api-foo-subsystem (:api-foo config))
    (api-bar-subsystem (:api-bar config))
    (component/system-map 
      :pool (component/using 
              (connection-pool)
              [:api-foo/out-channel :api-bar/out-channel]))))

(defn full-system [api-config]
  (merge 
    (api-system api-config)
    (component/system-map 
      :main-logic (component/using (main-logic) [:pool]))))

```

In a perfect world, where this is the entire system, this flat approach makes 
a lot of sense, but it gets messy fast. 

Forgetting for a second about component, it's clear that in almost any other 
design pattern, the concepts making up the "api connections" abstraction would 
be hidden behind an interface (or an interface per connection, if the 
connections aren't interchangeable). Standard stuff.

```clojure
(defprotocol IAPIConnections 
  (subscribe [this chan] 
    "Subscribe `chan` to messages from the APIs.")
  (request [this request] 
    "Route `request` through the API connection 
    specified by (:api request)."))

(defn connections [& apis] ...)

;; The calling code would then use it like this. Notice
;; that there's still an obligation for resource 
;; management.
(with-open [live-conns (connections :api-foo :api-bar)]

  (let [ch (async/chan)]
    (async/go-loop [] 
      (when-let [msg (async/<! ch)] 
        (println "debug >" msg) 
        (recur)))
    (subscribe live-conns ch))
    
  (request live-conns 
           {:api :foo 
            :requesting :some-resource 
            :auth :etc}))
```

## Neither cohorts (unwieldy) nor nested `SystemMap`s (non-obvious)

Coming back to component, there's no reason to compromise on strong abstraction
or separation of concerns. Instead of a library or a namespace exposing a cohort 
of components, it should expose an otherwise normal interface that's also 
designed to be component-friendly. 

The interface should implement the component `Lifecycle` for convenient 
composition into larger systems, but it should also have its own start / stop 
functions (and probably implement `java.io.Closeable`) for use by calling code 
that is unaware of component. 

This allows us to separate out a useful, general purpose abstraction without 
amplifying its scope or losing the benefits we get from component in the first 
place.

```clojure
;;; Tweaking our initial example to hide the API subsystem 
;;; behind the IAPIConnections protocol while still allowing 
;;; it to use a component system.

;; Keep the component subsystem hidden inside the record
(defrecord APIConnections [config sys])

(defn connections [config]
  (map->APIConnections {:config config}))
  
(defn merge-config [conns config]
  (update conns :config merge config))

;; It doesn't really matter when the subsystem is constructed,
;; but it should be started in some kind of `start` equivalent.
(defn connect [{:keys [config] :as conns}]
  (let [subsys (merge 
                 (api-foo-subsystem (:api-foo config)) 
                 (api-bar-subsystem (:api-bar config)) 
                 (component/system-map 
                 :pool (component/using 
                         (connection-pool) 
                         [:api-foo/out-channel 
                         :api-bar/out-channel])))]
    (assoc conns :sys (component/start-system subsys))))
    
(defn disconnect [conns]
  (update conns :sys component/stop-system))

(extend-type APIConnections
  IAPIConnections
  (subscribe [this chan]
    ;; Delegate to the connection pool subscribe
    (subscribe* (get this :pool) chan))
  
  (request [this req-data]
    (let [sock (keyword 
                 (name (:api req-data)) 
                 "socket")]
      (send! (get sys sock) req-data)))
  
  ;; Delegate to (dis)connect
  c/Lifecycle 
  (start [this] (connect this))
  (stop [this] (disconnect this))
  
  java.io.Closeable
  (close [this] (disconnect this)))
```

This approach adds some overhead, but also some usefulness:

- The contract between `APIConnections` and calling code is better defined 
  than it was when we were just merging subsystems together directly. 
- You can expose this as its own library without forcing the calling code to
  use component.
- You can use your own semantics for start / stop, e.g. by replacing 
  `start-system` with a version that [starts all of the components in parallel](../starting-stuartsierra-component-systems-in-parallel.html),
  only blocking for dependencies.
- But, you still get to use component when adding features to this abstraction;
  you haven't sacrificed any functionality.
- Anywhere that you would have previously `merge`d or modified the system map 
  directly, you now have to add one very small function to the top level of the 
  interface that you're exposing. 

That extra effort is a good thing! It keeps the code clean and decouples the 
abstractions inside of your system. That's the point of component in the first 
place.

I want to reiterate here that I am **not** suggesting that anybody do this for
separate components that just happen to work together in a system. This is a
solution for factoring out an abstraction with a well-defined scope — the sort 
of thing that you might otherwise expose as its own library.

> Beyond factoring out groups of components and putting them behind interfaces, 
  the other thing that keeps large component systems architecturally clean is 
  fighting the urge to see everything as a component. The point is well made in
  [Reloaded workflow out of the box](https://medium.com/@maciekszajna/reloaded-workflow-out-of-the-box-be6b5f38ea98),
  where Maciej Szajna shows that resource management sufficient for the 
  reloaded workflow is pretty easily doable with nothing more than 
  `java.io.Closeable` (which is already compatible with `with-open`).

## Composing component systems

Wrapped subsystems like the one above are easy to compose into the top-level
component system. The construction code becomes:

```clojure
(defn full-system [api-config]
  (component/system-map 
    :conns (api/connections api-config)
    :main-logic (component/using (main-logic) [:conns])))
```

The alternative, not being able to pick apart those abstractions without making 
monstrously large components, quickly degrades the value of being able to 
explicitly define dependency relationships at the top level of your code (since
the cohort constructors tend to hold all of the `using` declarations) and forces
pieces of different conceptual abstraction layers to awkwardly coexist on the 
same level.
