---
layout: post
title: Asynchronous (NIO) HTTP workflows in Clojure
tags:
 - Clojure
 - Software
---

This year I had occasion to develop a handy data collection program: a server application to catalog [limit order books](https://www.investopedia.com/terms/l/limitorderbook.asp) and trade history in real time for around 500 exchange traded digital asset and foreign currency markets, performing live analysis of the data as well as saving them for historical analysis.

The first challenge I hit when scaling up the number of markets was performing the high volume of HTTP requests in parallel, without much overhead, while [avoiding unpleasantly structured code](http://callbackhell.com/).  

(Scroll through to the bottom to just see the working code.)

Naive Approaches
===

```clojure
;; These approaches didn't get me very far...

(defn do-requests [& urls] ;; Worst approach -- not completely parallel
  (pmap clj-http.client/get urls))

(defn do-requests' [& urls] ;; Faster, but spawns `(count urls)` threads
  (let [reqs (mapv #(future (clj-http.client/get %)) urls)]
    (map deref reqs)))

(defonce do-requests'' ;; Limits total thread overhead, but still inefficient
  (let [n-threads 30
        p (Executors/newFixedThreadPool n-threads)
        exec (fn [f] (.submit p (reify Callable (call [_] (f)))))]
    (fn [& urls]
      (let [reqs (mapv #(exec (fn [] (clj-http.client/get %))) urls)]
        (map #(.get %) reqs)))))
```

These snippets are intuitively recognizable as not optimal—after all, lots of smart people have worked hard to make sure the NIC can handle network traffic and iterrupt the CPU when the data are ready, one thread is more than capable of managing many recv buffers. 

But clear, if not performance optimal approaches, are often better than code designed to extract the maximum performance out of the machine, and my pre-Clojure experience conditioned me to expect the standard library to leave me on my own when it came to NIO. I was pleasantly surprised to find an idiomatic solution in Clojure that was even more concise than my thread-pool version of do-requests, largely in thanks to Clojure's way of thinking about concurrency.

Concurrency Strategies
===

After hitting the performance bottleneck, I started doing a little research. I came across a blog post by Martin Trojer from 2011 titled [Asynchronous workflows in Clojure](http://martintrojer.github.io/clojure/2011/12/22/asynchronous-workflows-in-clojure) that both summarized some of the problems with using threads as the basic unit of concurrency ...

> Why is it so bad with many threads you might wonder. Well, first of all they are expensive, both in the JVM and on .NET. If you want to write some code that can scale to thousands of connections, one thread per connection simply doesn’t work. If you are using futures like above, when you try to make thousands of simultaneous connections, they will just queue on the thread pool, and the threads that are claimed will spend almost all their time blocked on IO (plus it will be dominated by accesses to slow servers). So either you kill you system spawning thousands of threads or you take for ever to complete when the thread pool slowly easts through the queued up work.

... and linked to the [aleph library](https://github.com/ztellman/aleph) for all things asynchronous in Clojure.

Interestingly, Martin also provided an idiomatic F# code snippet for async IO:

```f#
let download url = async {
  let request = HttpWebRequest.Create(Uri(url))
  let! response = request.AsyncGetResponse()
  use stream = response.GetResponseStream()
  let! res = asyncReadToEnd stream
  return res
}
```

It struck me that Clojure _does_ have the same abstraction (though it didn't come out until two years after Martin wrote his blog post): [core/async](https://github.com/clojure/core.async). 

It's an abstraction that's largely missing from languages that prefer a lower level concurrency interface, which normally just includes the ability to create OS threads (anything from C's fork() to the feature-packed thread pools in java.concurrnt.Executors) and some mutex to coordinate their behavior (again, ranging from C's primitive `pthread_mutex_lock(pthread_mutex_t *mutex)` to Java's built-in object monitor, [ReentrantLock](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantLock.html), or more creative [CountDownLatch](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/CountDownLatch.html) and [CyclicBarrier](https://docs.oracle.com/javase/8/docs/api/?java/util/concurrent/package-summary.html) constructs).

However, the concurrency models in languages like Go, F#, and Clojure recognize that the fundamental work unit in a program ought to be a code block, rather than a thread. This opens the door to automatic yielding semantics—largely ignorable by the programmer—that optimize for CPU utilization by allowing pieces of code running blocking or IO bound operations to relinquish use of the CPU until they become unblocked.

(Node.js chose a third, callback-based alternative to this apparent dichotomy. The language designers did a great job given the situation they were in—expecting JS developers to suddenly learn how multithreaded programming works was almost as implausible as writing a single-threaded server—and this pressure spurred them to develop a paradigm that is performant and almost impossible to misuse because there are no blocking operations. This came at the cost of code quality, but that ship had sailed with respect to JS anyway.)

Crashing Your System
===

The difference between the two approaches becomes clear if you imagine you want a function to schedule something to take place in `ms` milliseconds, without blocking the current thread.

```clojure
(require '[clojure.core.async :as async])

(defmacro blocking-schedule [ms & body]
  `(future 
     (Thread/sleep ~ms) 
     ~@body))

(defmacro async-schedule [ms & body]
  `(async/go 
     (async/<! (async/timeout ~ms))
     ~@body))
```

The two approaches have similar characteristics: both return to the calling thread immediately and both do something at some time in the future in a different thread. But the threaded approach is wasteful: we create a whole thread just to sleep for the majority of its life!

Just by running this innocuous looking snippet

```clojure
(dotimes [_ 1000000]
  (let [ms (rand-int 10000)]
    (blocking-schedule ms 
      (println "It has been " ms "milliseconds!"))))
```

a couple of times I managed to get an `OutOfMemoryError`. Obviously. 

However the same code, using `async-schedule`, is perfectly safe because core/async is smart enough not to try to create 1 million OS threads for a job that could be done by just one or two of them.

The Async Building Block
===

The `async-schedule` macro above is essentially callback based, and is _not_ the place to look when developing an asynchronous workflow in Clojure. Really the analogy we want to make is

```clojure
(Thread/sleep ms)         is to (async/timeout ms) as
(clj-http.client/get url) is to ???
```

```clojure
;; After requiring [aleph "0.4.6"] as a dependency in project.clj...
(require '[aleph.http :as http])
(require '[manifold.deferred :as d])
(require '[byte-streams :as bs])

(defn async-get [url ops]
  (let [chan (async/chan)]
    (d/chain 
      (http/get url ops) :body bs/to-string 
      (fn [x] (async/>!! chan x) (async/close! chan)))
    chan))
```

This allows us to write asynchronous code as if it were sequential—much in the way that `async/timeout` simulates the familiar `Thread/sleep`—say in the case of pagination where we need the last item from the previous page before making the next request

```clojure
(defn paginated-request 
  "For the sake of brevity, let's assume the api takes a query-param 
  `since` and returns a json array of data, which is empty when there 
  are no more items."
  [url]
  (async/go-loop [items [], marker nil]
    (let [qps (if marker {"since" marker} {})
          ;; Here other threads are allowed to use the CPU until the response is ready
          response (async/<! (async-get url {:query-params qps}))]
      (if-let [page (not-empty (clojure.data.json/read-str response))]
        (recur (into items page) (last page))
        items))))

;; Which we can in turn use in sequential-style code if we want to
(async/go
  (println "some-api:"  (async/<! (paginated-request "http://some-api.com/endpoint")))
  (println "other-api:" (async/<! (paginated-request "http://other-api.com/endpoint"))))
```

... and also covers the initial toy example, a function [url] => [response].

```clojure
(defn do-requests [& urls]
  (let [in-progress (mapv #(async-get % {}) urls)]
    (async/merge in-progress)))

;; We can process results one by one like so
(async/go-loop [results (do-requests "https://xe.com" "https://bitstamp.com" ...)]
  (when-let [res (async/<! results)]
    (println "Got result" res)
    (recur results)))
```

By creating functions like paginated-request which return core/async channels, it's easy to construct asynchronous functions that don't rely on callbacks, and ultimately improve the performance of the calling code.

In my case, querying limit order book and trade history data involved not only parallel requests but pagination for the trade history. Both of these things made a thread pool approach less convenient in addition to worse. The asynchronous alternative is so easy in Clojure that it should be the default for any code using HTTP concurrently, whether or not it is running against some of the performance constraints of thread pools that I was hitting.
