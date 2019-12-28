---
layout: post
title: Using Clojure's Transducers in Asychronous Systems
tags:
 - Clojure
 - Software
---

Today I was reminded once again of how well designed Clojure’s abstractions are.

I was writing code to replay logged messages — which are, at various stages, transformed and handed off between several threads — through the same logic that handles real time messages in our trading system. Using a real instance of the system to do message handling, there were plenty of resources to manage, disqualifying the approach I would otherwise reach for:

```clojure
(with-open [r (io/reader "messages")
            system (comment "... create a system ...")]
  (->> (line-seq r) 
       (map pure-function)
       (mapcat #(get-output system %))
       (comment "etc")))
```

Eventually a natural pattern emerged that, while not earth shattering, is useful in a variety of situations where you deal simultaneously with high level code (say, test assertions) and rigging together various queues and worker threads. 


## Objective

One snippet of client code I wanted to be able to write was a test to check the order book state at a specific point in time. Our system receives messages about market conditions and keeps limit order books up to date for each market, so I wanted to assert that replaying the raw messages through all of the real system logic resulted in a state where the USD/MXN midprice was $19.5540 and the EUR/MXN midprice was $21.6670, at precisely 1570232041452.

```clojure
;; The output we're interested in is composed of messages with the following shape
;; {:type :book-diff, :market :eur-mxn, :book <order book structure>, :timestamp _}
(let [xform (comp
              ;; Take messages concerning the order book
              (filter #(= (:type %) :book-diff))
              ;; For either of the markets we care about
              (filter (comp #{:usd-mxn :eur-mxn} :market))
              ;; Until the specified timestamp 
              (take-while #(<= (:timestamp %) 1570232041452)))

      ;; Process the messages by updating a map of the order book state
      rf (completing (fn [state msg] (assoc state (:market msg) (:book msg))))

      ;; Replay the messages from our flat file, and inspect the result
      state (replay (io/reader "messages.log") (xform rf) {})]
  (assert (= (midprice (:usd-mxn state)) 19.5540M))
  (assert (= (midprice (:eur-mxn state)) 21.6670M)))
```

In production, the output that we're reducing over here is not just saved in a map, but consumed in an `(async/go ...)` block to drive further events. In this test, I don't care about how the messages are communicated or what happens once they're seen by other components of the system, but I do want to see them in order and be able to *stop* consuming them at some point, so a [transducer](https://clojure.org/reference/transducers) feels like the right level of abstraction (as opposed to something that generates a lazy sequence, but can't run "stop" logic).

## Pattern: Using Transducers Asynchronously

Making the desired `replay` syntax a reality involves making two processes communicate. (1) consumes output from the the system, reducing over it, while (2) uses a buffered reader to go through the lines in "messages.log" and load them into the system. Either process can gracefully halt the other.

Orchestrating that communication via lexical scope looks something like this for a system that produces
output via [core.async](https://github.com/clojure/core.async):

```clojure
(defn replay [reader rf init]
  (let [;; All shared state goes here
        constructed-system (comment "...")
        stopped (promise)

        ;; (1) Asynchronous reducing task that consumes messages until the system
        ;; channel closes OR the reducing function says it's time to stop.
        reducing-task
        (async/go-loop [state init]
          (if (reduced? state)
            (do (deliver stopped true) (rf @state))
            (if-let [m (async/<! (comment "wherever the system outputs msgs"))]
              ;; I'm omitting (try ... (catch ...)) here, but 
              ;; in reality invoking rf might throw, and we 
              ;; should handle it gracefully
              (recur (rf state m))
              (rf state))))]

    ;; (2) Code that loads messages into the system, but allows the transducer
    ;; to tell it when to stop early.
    (with-open [reader reader]
      (->> (line-seq reader)
           (map edn/read-string)
           (take-while (fn [_] (not (realized? stopped))))
           (map (comment "Some function to inject the messages"))
           (dorun)))

    ;; If (2) stops because the reducer is done, reducing-task will have already
    ;; terminated, otherwise we wait until (1) is finished processing the messages
    ;; that are making their way through the system, if it hasn't finished already.
    (async/<!! reducing-task)))
```

That's really the whole pattern: provide an interface that accepts a reducing function, manage resources in a closure, and abstract away anything asynchronous by awaiting the result of the reducing function. 
You just need clear semantics in your system around start, stop, and "wait for this thing to be done".

It's a powerful pattern because it gives you *a lot* of flexibility when it comes to managing the stateful elements of whatever produces the input to your reducing function. 

For instance, I had to be able to completely stop & recreate the running system whenever I saw a message indicating that the system had shut down (because it meant the next message was the first of a new session, and should be replayed against a clean state), which in turn meant that the reducing task had to switch over to consuming the new system's output channel when it was done with the contents of the previous channel. My situation also entailed seeking through multiple files containing logged messages, so `reader` needed to be able to switch over to the next file if it had exhausted the current one before being told to stop.

## Clojure = Abstraction Oriented Programming

I think it's pretty cool that in Clojure, you can take some statement like,

```clojure
(filter (comp #{:usd-mxn :eur-mxn} :market))
```

compose it with another statement like,

```clojure
(take-while #(<= (:timestamp %) 1570232041452))
```

and use the result in the context of a multi-threaded, multi-component trading system littered with asynchronous communication! 
