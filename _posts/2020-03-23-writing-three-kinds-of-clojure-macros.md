---
layout: post
title: Writing Three Kinds of Clojure Macros
tags:
 - Software
 - Clojure
---

When I was first learning Clojure and trying to understand macros, I wish that 
somebody had told me that:

1. There are basically (oversimplifying) 3 types of macros,
    - those that exist so that the caller doesn't have to type `(fn [] ...)`,
    - those that walk through your code and expand some special pieces of 
      syntax, and
    - those that have non-trivial instruction sets and generate complex control 
      flow.
2. [`clojure.walk/postwalk`](https://clojuredocs.org/clojure.walk/postwalk) is 
   the easiest way to implement non-trivial macros, and
3. complex macros that affect the control flow of the program in non-trivial 
   ways (for example, DSLs like core.async) are more about creating instructions
   out of _data_ that are then processed by functions than about creating the
   high-level syntax.
    
Also, that you'll only infrequently (or never) need to write the last type of 
macro.
   
## Trivial Macros Replacing `fn`

You see and do this all the time. Almost not worth mentioning!

You have a function that takes a function:

```clojure
(defn thread* [task]
  (doto (Thread. ^Runnable task) 
    (.start)))
```

And you also want to offer an interface that feels more like "syntax":

```clojure
(defmacro thread [& body]
  `(thread* (fn [] ~@body)))
```

## Less Trivial Macros Replacing `fn`

It's slightly less common, but you can also use this pattern with things that
require binding. This is not how `with-open` is implemented, but a reasonable 
alternative implementation coming from a functional perspective might have 
been:

```clojure
(defn with-resource* [^java.io.Closeable resource f]
  (let [ret (try (f resource) (catch Throwable t {::error t}))]
    (.close resource)
    (if-let [err (::error ret)]
      (throw err)
      ret)))

(with-resource* (io/reader "a-file.txt") (fn [r] (count (line-seq r))))
```

You'd probably then decide to wrap it in a macro to make invocation a little nicer:

```clojure
(defmacro with-resource [[resource-sym resource] & body]
  `(with-resource* ~resource (fn [~resource-sym] ~@body)))

(with-resource [r (io/reader "project.clj")]
  (count (line-seq r)))
```

You could then imagine updating it to handle multiple resources:

```clojure
(defmacro with-resource [[resource-sym resource & more-bindings] & body]
  (let [body (if (seq more-bindings)
               `(with-resource ~(vec more-bindings) ~@body)
               `(do ~@body))]
    `(with-resource* ~resource (fn [~resource-sym] ~body))))

(with-resource [r (io/reader "project.clj")
                r' (io/reader "project.clj")]
  (+ (count (line-seq r))
     (count (line-seq r'))))
```

Though all of this could reasonably be described as "macro writing", the lion's
share of the work is managed by functions at runtime, which is why I'd classify
the macro components as "trivial". 

That's good! Making the macro writing easy by delegating the real work to 
functions is the best tactic to use where possible.
   
## Macros using `postwalk`

Some macros cannot be implemented with functional patterns, and are therefore
more than just an interface. I think the `postwalk` subset of macros is an 
important one here.

Let's say you're tired of typing out literal milliseconds, and you want to be 
able to write numbers with units.

```clojure
(ms-time
  (let [delay-time [10 :seconds]]
    (Thread/sleep delay-time)
    (println "Done delaying after" [10 :seconds] "milliseconds.")))
; Done delaying after 10000 milliseconds.
;=> nil
```

Since `Thread/sleep` doesn't accept a vector, you have to write a macro that 
walks through your code, looks for vectors of `[number :unit]`, and replaces
them with milliseconds.
  

```clojure
(defmacro ms-time [form]
  (let [conversions {:seconds #(* % 1000)}
        time-vector? (fn [thing]
                       (and (sequential? thing)
                            (= (count thing) 2)
                            (contains? conversions (second thing))))]
    ;; All of this happens at compile time
    (walk/postwalk
      (fn [x]
        (if (time-vector? x)
          ;; If it's a time vector, look up the conversion fn and apply it to 
          ;; the number
          (let [conversion-fn (get conversions (second x))]
            (conversion-fn (first x)))
          
          ;; Otherwise leave it alone
          x))
      form)))

(macroexpand
  '(ms-time
     (let [delay-time [10 :seconds]]
       (Thread/sleep delay-time)
       (println "Done delaying after" [10 :seconds] "milliseconds."))))
;=> (let* [delay-time 10000] 
;     (Thread/sleep delay-time) 
;     (println "Done delaying after" 10000 "milliseconds."))
```

You can get as creative as you want with this. For example, you can create 
syntax for different kinds of format strings:

```clojure
(macroexpand '(fmt (println "${x} plus ${y} is ${(+ x y)}")))
;=> (println (clojure.core/str "" x " plus " y " is " (+ x y) ""))

(fmt
  (let [x 1 
        y 2
        to-print "${x} plus ${y} is ${(+ x y)}"]
    (println to-print)))
; 1 plus 2 is 3
;=> nil


(defmacro fmt
  "Rewrite any strings so that occurrences of ${expr} are replaced with a
  (runtime) evaluated version."
  [form]
  (let [format-string
        (fn [s]
          (loop [parts [] s s]
            (if-let [[_ next-expr] (re-find #"\$\{([^}]*)\}" s)]
              (let [[pre post] (string/split s #"\$\{([^}]*)\}" 2)]
                (recur
                  (into parts [pre (read-string next-expr)])
                  post))
              `(str ~@(conj parts s)))))]
    (walk/postwalk
      (fn [x]
        (if (string? x)
          (format-string x)
          x))
      form)))
```

> It's also worth noting that if you just want to evaluate something at compile
  time, you can use the `#=` reader macro. See code starting on 
  [line 2594 of Peter Taoussanis's encore library](https://github.com/ptaoussanis/encore/blob/master/src/taoensso/encore.cljx#L2594)
  for some good examples.

## Macros Generating Instruction Sets

These macros vary greatly in complexity, and it's unlikely that you'll have to 
write them, unless you're really into that sort of thing.

To get comfortable, I would 

1. Read `clojure.core/case` ([source here](https://github.com/clojure/clojure/blob/clojure-1.10.1/src/clj/clojure/core.clj#L6697)).
2. Read `clojure.core.match/match` ([source here](https://github.com/clojure/core.match/blob/1c6b2b522990ed9c78fa3499d2797d0fab87d114/src/main/clojure/clojure/core/match.clj#L2101)).
3. Watch the video series by Timothy Baldridge on building clojure.core.async,
   an incredibly cool macro ([first video here](https://www.youtube.com/watch?v=R3PZMIwXN_g)).

## A Word on Clojure DSLs

LISP dialects have always prided themselves on the ability to create expressive 
DSLs, like core.async, via macros. 

However, I've found that the best way to do this in Clojure is to build DSLs 
first by thinking in terms of data (usually maps) and, only after coming up with
an unambiguous specification of the tasks I want done, _possibly_ passing those
maps to a macro for some sort of compile-time work. Even then, more often than
not, I end up passing those maps to functions.

For example, say you're dreaming up a DSL for GUIs. Instead of starting with
syntax, say

```clojure
(defwindow 
  (pane "Title"
    (input :an-input "Default text")
    (label (str "You entered" (get *context* :an-input)))))
```

I would recommend starting with a specification

```clojure
{:type :window
 :contents
 [{:type :pane
   :title "Title"
   :contents
   [{:type :input
     :id   :an-input
     :text "Default text"}
    {:type :label
     :text (fn [context] (get context :an-input))}]}]}
```

It might not be as pretty, but I think it helps to keep things focused on 
exactly what sorts of things are being described, what they have in common,
how state and dependency injection might work, etc. 

The challenge is coming up with something that can consume your specifications,
draw the window, manage state, and come up with an event loop. If you can get
all of that done, shifting some of the work to compile-time and coming up with
a nice syntax should be easy.
