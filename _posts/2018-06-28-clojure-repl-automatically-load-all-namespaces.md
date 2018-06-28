---
layout: post
title: Clojure REPL Automatically Loading Project Namespaces
tags:
 - Clojure
---

If you're working on a Clojure project with several namespaces, you might find it useful to
automatically require those namespaces each time the REPL starts, rather than having to fully qualify, `use`, or `require` them.

I.e. if your directory structure is

```haskell
src/
└── a_project
    ├── core.clj
    ├── data_structures
    │   ├── order_book.clj
    │   ├── time_series.clj
    ├── connect.clj
    └── utils.clj
```

you might want to be able to start up the REPL and right away have access to methods in the `utils` or `order-book` namespaces, for example.

```clojure
:~/a-project$ lein repl
a-project.core=> (utils/dbg (+ 1 3))
> (+ 1 3) => 4
4
the-system.core=> (book/best (book/random-book) :asks)
{:price 100M, :qty 22.81478M, :fee-price 100.25M}
```

With Lein this is not so hard.

I just wrote some code to recursively `require` all the namespaces in my project, aliased by the last bit of their namespace (i.e. not fully qualified), and set it to be invoked upon startup by structuring `project.clj` like this:

```clojure
(defproject a-project "0.0.1"
  :repl-options {:init (a-project.utils/require-package "a-project")}

  ;; ...
)
```
In utils.clj

```clojure
(defn in-package
  "List the namespaces in a given package, excluding a collection of regex exclude strings."
  ([prefix]
   (in-package prefix []))
  ([prefix excludes]
   (let [excluded? #(some
                      (fn [exclude] (re-find (re-pattern exclude) %))
                      excludes)
         xform (comp (map #(str (.getName %)))
                     (filter #(string/starts-with? % prefix))
                     (filter (complement excluded?)))]
     (sequence xform (all-ns)))))


(defmacro require-package
  "Require everything in a package, aliased by namespace. I.e. would
      (require '[a-project.data-structures.order-book :as order-book])"
  ([package]
   `(require-package ~package []))
  ([package excludes]
   (let [alias (fn [ns] (last (string/split ns #"[.]")))
         build-aliases (map #(-> [%, (alias %)]))
         to-symbols (map #(map symbol %))
         to-form (map (fn [[ns alias]] `(require '[~ns :as ~alias])))
         xform (comp build-aliases to-symbols to-form)
         requires (sequence xform (in-package package excludes))]
     `(do
        ~@requires))))
```

Please do copy & paste.
