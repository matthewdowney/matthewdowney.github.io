---
layout: post
title: Forcing a JVM Heap Dump Programmatically from Clojure
tags:
 - Software
 - Clojure
---

I was monitoring a new version of our trading software running on a staging 
server and noticed through `htop` that it was using much more memory than I 
was expecting.

But I hadn't 
[set up remote profiling](http://issamben.com/how-to-monitor-remote-jvm-over-ssh/)! 
I didn't want to restart the JVM with profiling enabled, because it took several 
days for the potential issue I was now seeing to occur. 

Luckily we control our system from a REPL, and if you're using Clojure, chances 
are that you also control your system from a REPL. 

You could read through and translate the 
[70 or so lines of Java on Oracle's blog](https://blogs.oracle.com/sundararajan/programmatically-dumping-heap-from-java-applications)
(which gives an excellent overview of how to cause a heap dump programmatically),
but you should also feel free to just copy and paste this Clojure equivalent. :) 

```clojure
(import 'java.lang.management.ManagementFactory)
(import 'com.sun.management.HotSpotDiagnosticMXBean)

(let [server (ManagementFactory/getPlatformMBeanServer)
      bean-name "com.sun.management:type=HotSpotDiagnostic"
      bean (ManagementFactory/newPlatformMXBeanProxy server bean-name HotSpotDiagnosticMXBean)
      live-objects-only? false]
  (.dumpHeap bean "dump.hprof" live-objects-only?))
```

Hope this is useful to somebody else out there!
