## Java 8

Java 8 was released on March 18, 2014, two years seven months after the previous release. It was plagued with delays and technical problems but when it finally came, it represented one of the biggest shifts in Java since Java 5.

The headliners were of-course lambdas and and a retrofit to support functional programming ideas. With languages like Scala taking center stage and the modern trend towards functional programming, Java had to do something to keep up.

Although Java is not and never will be a _pure_ functional programming language, Java 8 does allow you to use functional idioms more easily than it's predecessors. With disciple and experience, you can now get a lot of the benefits of functional programming without resorting to third-party libraries.


### Features

A mostly-complete list of the new features of Java 8:

* Lambda support 
* The core APIs have been updated to take advantage of lambdas, including the collection APIs and a new functional package to help build functional constructs.
* Entirely new APIs have also been developed that use lambdas, things like the stream API which bring functional style processing of data.
    * For example, functions like `map` and `flatMap` from the stream API allow a declarative way to process lists and move away from external iteration to internal iteration. 
    * This in turn allows the _library vendors_ to worry about the details and optimise processing however they like. For example, Java now comes with a parallel way to process streams without bothering the developer with the details.
* Minor changes to the core APIs; new helper methods have popped up for strings, collections, comparators, numbers and maths.
* Some of the additions may have a big influence on how we code in the future. For example, the `Optional` class will be familiar to some and allows a better way to deal with nulls.
* There are various concurrency library improvements. Things like an improved concurrent hash map, completable futures, thread safe accumulators, an improved read write lock (called a `StampedLock`), an implementation of a work stealing thread pool and much more besides.
* Support for adding static methods to interfaces.
* Default methods (otherwise known as _virtual extension_ or _defender methods_).
* Type inference has been improved and new constructs like functional interfaces and method references have been introduced to better support lambdas.
* An improved date and time API has been introduced (similar to the popular Joda-time library).
* The IO and NIO packages have welcome additions to allow you to work with IO streams using the new streams API.
* Reflection and annotations have been improved.
* An entirely new JavaScript engine ships with Java 8. Nashorn replaces Rhino and is faster and has better support for ECMA-Script.
* JVM improvements; it could be the fastest JVM to date as the integration with JRocket is now complete.
* The JVM has dropped the idea of perm gen, instead using native OS memory for class metadata. This is a huge deal and should in theory mean better memory utilitsation.
* The JRocket integration also brings Mission control (`jmc`) to the JDK as standard. It compliments `jconsole` and `visualvm` with similar functionality but adds very inexpensive profiling.
* Other miscalanous improvements, like improvements to JavaFX, base64 encoding support and more.
