---
layout: post
title: Using Clojure's <code>swap-vals!</code> in Lock-Free Algorithms
tags:
 - Clojure
 - Software
---

Clojure's default approaches of "everything is immutable" and "all the 
references you plan to change are `atom`s" make it simple to reason about 
concurrent code _that should be simple to reason about_. 

```clojure
;; The average programmer would have no trouble changing this implementation
;; to store the count in a map of {:count <...>, :initial-count <...>}.

(defn counter [initial-value]
  (let [c (atom initial-value)]
    (fn increment-count []
      (swap! c inc))))
      
;; That's impressive, not because that would be particularly useful, but because
;; of what you'd have to do to make that work in languages without the same 
;; abstractions. 

;; Imagine that you're working with Java, and you have to refactor code that's 
;; currently using an AtomicLong. Would you just make one of the boxed values an 
;; AtomicLong and operate on it directly? Would you put both values inside an 
;; immutable box and use an AtomicReference to change it? 
;; 
;; Is there any chance you'd make the mistake of putting a mutable box inside an 
;; AtomicReference and changing the value unsafely, even though you think you're 
;; incrementing it atomically? If you took the first approach of wrapping an 
;; AtomicLong for the counter field, and your next feature involved wrapping 
;; _two_ values that you have to operate on atomically, where would you go from 
;; there?
```

> Like others who remember their first experience reading 
  [Java Concurrency in Practice](https://www.amazon.com/gp/product/0321349601/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0321349601&linkCode=as2&tag=matthewdowney-20&linkId=bc1814e29b63a674d93fbf86964f0bb3) 
  and promptly entering in crisis, I continue to be impressed by how the 
  the path of least resistance created by the language design is almost always 
  "the right choice", without closing the door to dipping into finer grained
  control if necessary as your code matures.

Clojure [1.9](https://clojure.org/news/2017/12/08/clojure19) introduced another
function, `swap-vals!`, that handles most situations previously requiring 
`compare-and-set!` and joins `swap!` and `reset!` in the cohort of high-level, 
hard to misuse, functions operating on `atom`s. 

It's useful for situations where you have to (1) make some change to the value 
and then (2) do something with the _previous_ value of the atom. For example,
popping the first element off a queue:

```clojure
(def q (atom [{:task :a} {:task :b} {:task :c}]))

(defn pop' [q]
  (let [[old-value new-value] (swap-vals! q rest)]
    (first old-value)))

;; Alternatively, if swap-vals! is a common idiom in your codebase, maybe just:
;; (defn pop' [q] (ffirst (swap-vals! q rest)))

(pop' q) ;=> {:task :a}
(pop' q) ;=> {:task :b}
```

How would `pop'` look otherwise? 

```clojure
;; It's not unreasonably complicated, but the difference between `swap-vals!`
;; code and `compare-and-set!` code only gets larger as the complexity 
;; increases.
(defn pop' [q]
  (let [state @q
        item (first state)
        new-state (rest state)]
    (if (compare-and-set! q state new-state)
      item 
      (recur q))))
```

Importantly, it provides the correct "path of least resistance" for situations
where it might otherwise be tempting to introduce assumptions that the code 
doesn't really need. 

For instance, perhaps at the time of writing, `pop'` is only going to be called 
from one thread at a time (whereas multiple threads might be adding items to the 
queue) and it's tempting to write a brittle piece of code that relies on those 
circumstances.

```clojure
(defn pop' [q]
  ;; This allows concurrent write access to the (end of the) queue, but only
  ;; one thread calling `pop'` at a time.
  (let [[item & _] @q] ;; <-- likely source of future bugs!
    (swap! q rest)))
```

Both `swap-vals!` and `reset-vals!` make it easy to do the right thing without 
giving it a second thought in those explicit concurrency situations that 
require _slightly_ more complexity than `swap!` operations but still don't need
the level of generality that `compare-and-set!` offers. 

This is a great example of the language proactively jumping out of your way so
that you can stay in the flow and focused on the **purpose** of your code, 
rather than manipulating a vector.
