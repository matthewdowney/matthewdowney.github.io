---
layout: post
title: Calling Native C Code from Java with the JNI
tags:
- Java
- C
- JNI
---
The JNI ([*Java Native Interface*](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/)) allows Java libraries to implement functionality via native code.
The idea is to define an interface in Java, and implement it in C/C++. Your client code will then load the implementation and construct a class from it, from which you may then instantiate objects normally. Something like this:

{% highlight java %}
// Implemented in C elsewhere
public class INative {
   public native void myMethod();
}
{% endhighlight %}

{% highlight java %}
public class Client {
   public static void main(String[] args) {
      System.loadLibrary("INative");
      INative iNative = new INative();
      iNative.myMethod();
   }
}
{% endhighlight %}

## A 60 Second Tutorial
=====================
**Define your interface** (we are actually forced to implement this as a class--the `native` keyword isn't allowed in interfaces):

{% highlight java %}
// INative.java
public class INative {
   public native void hello();
}
{% endhighlight %}

... and compile

{% highlight text %}
~$ javac INative.java
{% endhighlight %}


**Generate the C interface** (header file) which will be named `INative.h`.

{% highlight text %}
~$ javah INative
{% endhighlight %}


**Write an implementation** for the generated `INative.h` in a file `INative.c`, simply copying the method signature from the generated header:

{% highlight C %}
// Don't forget these first two imports!
\#include "INative.h"
\#include <jni.h>
\#include <stdio.h>
JNIEXPORT void JNICALL Java_INative_hello(JNIEnv *env, jobject obj) {
   printf("Hello World\n");
}
{% endhighlight %}


**Compile your native code** into a library. On \*nix the resulting file should be `libINative.so`, and on Windows it should be `INative.dll`. Make sure to include the `jni` library that comes with your JDK.

{% highlight text %}
~$ gcc -shared -fpic -o libINative.so -I${JAVA_HOME}/include -I${JAVA_HOME}/include/linux INative.c
{% endhighlight %}


Now just make sure to **include your library** when running any client code.

{% highlight text %}
~$ java -Djava.library.path=. Client
Hello World
{% endhighlight %}

Example client:

{% highlight java %}
public class Client {
   public static void main(String[] args) {
      System.loadLibrary("INative");
      INative in = new INative();
      in.hello();
   }
}
{% endhighlight %}

> For more in depth explanations of the JNI--working with different data types, invoking Java from C, working with Exceptions, etc.--check out [MIT's tutorial](http://web.mit.edu/javadev/doc/tutorial/native1.1/implementing/index.html).
