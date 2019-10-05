---
layout: post
title: Clojure-Java Interop Pattern for Avoiding Setters
tags:
 - Clojure
 - Java
 - Software
 - Macros
---

The Clojure standard library is 
[filled with](https://clojuredocs.org/clojure.core/memfn) cool 
[macros](https://clojuredocs.org/clojure.core/doto) 
and [functions](https://clojuredocs.org/clojure.core/aset) 
to smooth Java interoperability and ameliorate the contrast between Java's 
verbosity and Clojure's conciseness, which becomes particularly disconcerting in
code that interacts with both languages non-trivially.

I came across one such frustration when writing some Excel sheet generation code
that relies heavily on an [Apache library (POI)](https://poi.apache.org/): the
difference between Java's setters and Clojure's corresponding system for 
specifying attributes. I had some boilerplate code littered with `doto` and 
`(.setX Y Z)` statements:

```clojure
(defn cell-style [workbook attrs]
  (let [^CellStyle style (.createCellStyle workbook)
        font (doto (.createFont workbook)
               (.setBold (get attrs :bold false))
               (.setItalic (get attrs :italic false))
               (.setUnderline (get attrs :underline false))
               (.setFontHeightInPoints (get attrs :font-height-in-points 11))
               (.setFontName (get attrs :font-name "Calibri")))
        ;; ditto for objects for DataFormat, Alignment, Border, etc
        ]
    (doto style 
      (.setFont font)
      ;; ditto for objects for DataFormat, Alignment, Border, etc
      )))
```

Aside from being inherently boring, boilerplate is not necessary—or is at least 
greatly reducible—in Lisp dialects! I can imagine a `setters` macro that takes care of all of this:

```clojure 
(macroexpand 
  '(setters 
     {:bold true :italic true 
      :font-name "Arial" :font-height-in-points 12}))
; =>
; (fn*
;  ([obj__1251__auto__]
;   (clojure.core/doto obj__1251__auto__
;    (.setBold true)
;    (.setItalic true)
;    (.setFontName "Arial")
;    (.setFontHeightInPoints 12))))
```

```clojure 
(defmacro setters
  "Given a compile-time literal map of attributes and values, return a function
  that calls the corresponding setters on some java object.

  E.g. (macroexpand
         '(setters
           {:bold true, :font-height-in-points 14, :font [\"large\" \"red\"]}))
       ; => (fn [obj]
              (doto obj
                (.setBold true)
                (.setFontHeightInPoints 14)
                (.setFont \"large\" \"red\")))"
  [attrs]
  (when-not (map? attrs)
    (throw (ex-info "attrs must be a literal map, not a symbol" {})))
  (let [capitalize (fn [coll] (map string/capitalize coll))
        camel-case (fn [kw] (-> (name kw) (string/split #"\W") capitalize string/join))
        setter-sym (fn [kw] (->> (camel-case kw) (str ".set") symbol))
        expanded (map (fn [[a val]]
                        (if (vector? val)
                          `( ~(setter-sym a) ~@val)
                          `( ~(setter-sym a) ~val)))
                      attrs)]
    `(fn [obj#] (doto obj# ~@expanded))))
```

Shout out to `memfn` in `clojure.core`, which clojureizes a Java function and 
was the first thing I looked at when figuring out how to generate the setter code.

```clojure
;;; clojure.core line 3837
(defmacro memfn
  "Expands into code that creates a fn that expects to be passed an
  object and any args and calls the named instance method on the
  object passing the args. Use when you want to treat a Java method as
  a first-class fn. name may be type-hinted with the method receiver's
  type in order to avoid reflective calls."
  {:added "1.0"}
  [name & args]
  (let [t (with-meta (gensym "target")
            (meta name))]
    `(fn [~t ~@args]
       (. ~t (~name ~@args)))))
```


### Update

The excel generation code is now on github at [excel-clj](https://github.com/matthewdowney/excel-clj). You
can see the setter-avoiding code in acction (and build a styled excel spreadsheet) with the following
snippet:

```clojure
(require '[excel-clj.core :as excel])

(def table-data
  [{"Date" "2018-01-01" "% Return" 0.05M "USD" 1500.5005M}
   {"Date" "2018-02-01" "% Return" 0.04M "USD" 1300.20M}
   {"Date" "2018-03-01" "% Return" 0.07M "USD" 2100.66666666M}])

(letfn [(highlight-below-5% [row-data col-name]
          (when (< (row-data "% Return") 0.05M)
            {:fill-pattern :solid-foreground
             :fill-foreground-color :yellow}))]
  (excel/quick-open
    {"My Generated Sheet" (excel/table table-data :data-style highlight-below-5%)}))
```

Which produces

![an excel sheet with a highlighted row](https://github.com/matthewdowney/excel-clj/blob/master/resources/manual-formatting.png?raw=true)
