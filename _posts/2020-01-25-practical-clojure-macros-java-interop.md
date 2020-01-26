---
layout: post
title: Practical Clojure Macros for Java Interop
tags:
 - Clojure
 - Software
---

I've found no lack of resources when it comes to library interoperability between 
the various JVM languages, so instead I want to share a few ways that I've found
it useful interact with the JVM & the Java standard library when writing pure 
Clojure code.

These are the macros I use to 

- set [JVM shutdown hooks](#jvm-shutdown-hook),
- spawn [JVM threads](#jvm-threads),
- cause [deliberate read/write contention](#deliberate-contention),
- create [thread-local delays](#threadlocal),
- and [replace Java setters with Clojure maps](#setters).

## <a name="jvm-shutdown-hook"></a> JVM Shutdown Hook

The JVM allows setting hooks which are called during any controlled shutdown. 
It's kind of a sledgehammer approach to cleaning up resources, but still a 
pragmatic tool if you really need something to happen on shutdown. 

> Maybe your logger uses a `BufferedWriter` and you want to flush it during an 
  unexpected shutdown, since that's precisely the situation in which you _most_ 
  need complete logfiles.

In these cases, I use a `with-shutdown-hook` macro that sets / unsets the
shutdown hook, but also names it in case I want to call it myself during a 
normal shutdown.

```clojure
(defmacro with-shutdown-hook [[hook-name hook-fn] & body]
  `(let [~hook-name ~hook-fn ;; Make the hook available by name in body
         hook# (Thread. ~hook-name) 
         rt# (Runtime/getRuntime)]
     (.addShutdownHook rt# hook#)
     (let [ret# (do ~@body)]
       (.removeShutdownHook rt# hook#)
       ret#)))
```

For example:
  
```clojure
;; Imagine you're writing via this BufferedWriter throughout the program
(let [resource (io/writer "debug.log" :append true)]
  ;; To clean it up, we just need to invoke .close
  (with-shutdown-hook [clean-up (fn []
                                  (println "Cleaning up...")
                                  (.close resource))]
    ;; This code might hang, maybe we Ctrl+C while it's running, etc.
    (comment "... code using resource ...")
    (System/exit 1) ;; Simulate some unexpected shutdown
    
    (println "We'll never reach this code!")
    (clean-up)))
    
;=> Cleaning up...
;   Process finished with exit code 1
```

You'll have to sacrifice a REPL session to test that example.

> Note that the shutdown hook is not _always_ called. On OOM errors or SIGTERMs 
  the JVM doesn't call shutdown hooks, but it does during a normal shutdown
  (Ctrl+C) or an OS shutdown.
  

## <a name="jvm-threads"></a> JVM Threads

Alex Miller wrote 
[a nice `thread` macro in clojure.core.server](https://github.com/clojure/clojure/blob/0035cd8d73517e7475cb8b96c7911eb0c43a1a9d/src/clj/clojure/core/server.clj#L38)
that I use when I want to explicitly name a thread, set its daemon status, or
run it outside of the pool that `future` uses for some reason.

```clojure
;; src/clj/clojure/core/server.clj line 38
(defmacro ^:private thread
  [^String name daemon & body]
  `(doto (Thread. (fn [] ~@body) ~name)
    (.setDaemon ~daemon)
    (.start)))
```

> It's declared as `:private` there, so you'll have to reproduce it.

This can be useful if the thread is long-lived and you'd like its name to be 
clear to somebody observing via [VisualVM](https://visualvm.github.io/).


## <a name="deliberate-contention"></a> Deliberate Maximum Contention

Sometimes you want to test some explicitly concurrent piece of code (hopefully
not too often!), or you just want to spin up a bunch of threads to hammer on
something at the same exact time. 

If you just `(dotimes [i n-threads] (future (some-action)))`, you have no 
guarantees about when each future begins to execute (the CachedThreadPool that
Clojure uses for futures decides when to create new threads, when to recycle
cached ones, etc.)

If you want to guarantee maximum possible concurrency, you can use the Java 
standard library's CountDownLatch to oblige each thread to wait for the others 
to have started before starting.

```clojure
(import java.util.concurrent.CountDownLatch)

(defmacro dothreads
  "Like (dotimes [i n] ...) except the body is executed concurrently, and the
  result is a map from `i` to the result."
  [[tid-binding n-threads] & body]
  `(let [n-threads# ~n-threads
         latch# (java.util.concurrent.CountDownLatch. n-threads#)]
     (->> (range n-threads#)
          (map
            (fn [~tid-binding]
              (future
                (vector
                  ~tid-binding
                  (do
                    (.countDown latch#) ;; Signal that we're ready
                    (.await latch#) ;; Await all other threads ready
                    ~@body))))) ;; Do whatever thing
          (doall) ;; Make sure all threads start before we deref
          (map deref)
          (into {}))))
```

For example:

```clojure
;; Execute the body in 4 different threads
(dothreads [n 4]
  (let [started (System/currentTimeMillis)
        my-name (.getName (Thread/currentThread))]
    {:name my-name :started started}))

;; => {0 {:name "clojure-agent-send-off-pool-29", :started 1565723836090},
;;     1 {:name "clojure-agent-send-off-pool-30", :started 1565723836090},
;;     2 {:name "clojure-agent-send-off-pool-31", :started 1565723836090},
;;     3 {:name "clojure-agent-send-off-pool-32", :started 1565723836090}}
```

It's probably worth noting that the difference, if any, depends on the JVM. It's
just that if you don't specify any sort of dependency relationship between when
the threads start, you might accidentally end up relying on an implementation 
detail.


## <a name="threadlocal"></a> ThreadLocal

This one is not something I ordinarily use from pure Clojure code, except when
I want to use and re-use something from the Java standard library that I know 
isn't thread-safe.

For example, to format / parse some date, you might create a `SimpleDateFormat` 
inside the lexical scope of your function. Lexical scope is definitely the best
way to go for controlling unsafe objects, because you have complete control over
their use. 

However, if you're ever in a situation where you'd like to reuse objects, 
instead of creating and recreating them inside a `let`, you can use a 
`thread-local` macro — it's kind of like `delay`, but each thread that 
dereferences it computes its own value.

```clojure
(defmacro thread-local
  [& body]
  `(let [tl# (proxy [ThreadLocal] [] (initialValue [] (do ~@body)))]
     (reify clojure.lang.IDeref
       (deref [this]
         (.get tl#)))))

(def ^:private sdf ;; <-- You can share this between threads
  (thread-local 
    (java.text.SimpleDateFormat. "yyyy-MM-dd")))

(defn ymd-fmt [^java.util.Date date]
  (.format @sdf date)) ;; <-- Just make sure to dereference it 

(ymd-format (java.util.Date.))
;=> "2020-01-25"
```

> Luckily [clj-time](https://github.com/clj-time/clj-time) exists as a much 
  better alternative for this particular situation.


## <a name="setters"></a> Setters

This is another one you could use to make elements of the Java standard lib more
Clojure-friendly — I've used it to 
[declaratively generate Excel spreadsheets](https://github.com/matthewdowney/excel-clj)
and I've seen it used to 
[create a Clojure interface over a Java AWS libary](https://github.com/mcohen01/amazonica).

It's sort of the inverse of [`bean`](https://clojuredocs.org/clojure.core/bean).

Imagine you're working with some cumbersome Font object:

```clojure
(doto font-object
  (.setBold true)
  (.setFontName "arial")
  (.setFontHeightInPoints 14)
  (.setFont "large" "red"))
```

But you'd rather work with a font _specification_:

```clojure
{:bold true
 :font-name "arial"
 :font-height-in-points 14 
 :font ["large" "red"]}
```

Well, you can use a `setters` macro to camel-case the keys in the map and call
the corresponding setters:

```clojure
(defmacro setters
  [obj attrs]
  (assert (map? attrs) "Compile-time map literal")
  (let [capitalize (fn [coll] (map string/capitalize coll))
        camel-case (fn [kw] (-> (name kw) 
	                        (string/split #"\W") 
				capitalize 
				string/join))
        setter-sym (fn [kw] (->> (camel-case kw) 
	                         (str ".set") 
				 symbol))
        expanded (map (fn [[a val]]
                        (if (vector? val)
                          `( ~(setter-sym a) ~@val)
                          `( ~(setter-sym a) ~val)))
                      attrs)]
    `(doto obj# ~@expanded)))

(setters 
  font-object 
  {:bold true 
   :font-name "arial"
   :font-height-in-points 14 
   :font ["large" "red"]})
```

> For anyone interested, you can do something similar with nested objects — 
  whose setters might expect other objects, rather than primitives — by 
  registering the keys whose values need to be specially constructed and then
  walking through the map like a tree. Sample code from my excel-clj library
  [here](https://github.com/matthewdowney/excel-clj/blob/master/src/excel_clj/style.clj).
