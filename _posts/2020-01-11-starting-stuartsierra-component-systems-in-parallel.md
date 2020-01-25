---
layout: post
title: Starting stuartsierra/component in Parallel
tags:
 - Clojure
 - Software
---

The [stuartsierra/component](https://github.com/stuartsierra/component) library
(design pattern?) has been a tremendous aid to me in the development of 
[Sixtant's](https://sixtant.io) trading system, but over time I've found our 
dependency graph growing in such a way that we have ten or fifteen different 
connections — that the rest of the system depends on to start up — each of which 
makes HTTP requests, establishes connections, fetches data, etc. before 
completing `Lifecycle/start`.

Those five or ten seconds waiting at the REPL for the connections to fire up 
serially, over enough restarts, gave me time to realize that there was a better
way.

I remembered reading Stuart's 
["Lifecycle Composition"](https://stuartsierra.com/2013/09/15/lifecycle-composition) 
brainstorm / blog post that eventually evolved into the component library and 
noting that his initial workflow had all of the calls to `start` inside a
`(future ...)` such that the expression would dereference to a modified component 
record. I decided to write a small alternative to `start-system` that would do 
something similar, starting components in parallel and dereferencing them 
before injecting them downstream — with the result that components that don't 
depend on each other start concurrently but enjoy the guarantee that _their_ 
dependencies are already started and dereferenced before being injected.

### Approach

Given that `start-system`'s implementation is pretty much,

```clojure
(fn [sys] (update-system sys (keys sys) #'start))
```

I initially replaced `#'start` with this:

```clojure
(fn [component]
  (future ;; <-- start the component inside a future
    (start 
      (merge 
        component
        (zipmap ;; <-- modify the component to deref any future dependencies
          (keys component)
          (map #(if (future? %) (deref %) %) (vals component)))))))
```

... which works fine, except that components can be things that aren't maps, 
including futures. So I wrapped everything in `{::future (future ...)}` to 
unambiguously identify other components waiting to finish starting.

```clojure
(require '[com.stuartsierra.component :as c])

(defn deref-deps [component]
  (if-not (map? component)
    component
    (reduce
      (fn [component [field-key field]]
        (if-let [dep (::future field)]
          (assoc component field-key (deref dep))
          component))
      component
      component)))

(defn start-future [component]
  (let [starting (future (-> component (deref-deps) (c/start)))]
    {::future starting}))

(defn parallel-start [sys]
  (c/update-system sys (keys sys) #'start-future))
```

Here's hoping that somebody else finds this snippet useful!
