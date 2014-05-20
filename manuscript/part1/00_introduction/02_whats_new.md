## What's new in Java 8

It's been two years, seven months, and eighteen days since the last release of Java. Oracle have put this time to good use and really crammed in the features.

The headliners are of-course lambdas and and a retrofit to support functional programming ideas. With languages like Scala taking center stage and the current trends towards functional programming, Java had to do something to keep up.

Although Java is not and never will be a pure functional programming language, Java 8 does allow you to use functional idioms more easily. With disciple and experience, you can now get a lot of the benefits of functional programming without the third-party libraries.


### Features

It's a big shift for Java and an exciting time to be a Java developer, so lets take a whistle-stop tour of what's in the Java 8 release.

* Lambda support finally comes to Java
* and along with it, a bunch of the core APIs have been updated to take advantage of them, including the collection APIs and a new functional package to help build functional constructs.
* Entirely new APIs have also been developed that use lambdas, things like the stream API which bring functional style processing of data.
* Functions like `map` and `flatmap` from the stream API allow a declarative way to process lists for example and move away from external iteration to internal iteration. This in turn allows the _library_ vendors to worry about the details and they can optimise processing however they like. For example, Java now comes with a parallel way to process streams without bothering the developer with the details.
* It feels like nearly all of the core APIs have been touched by the release. New helper methods have popped up for strings, collections, comparators, numbers and maths.
* Some of the additions may have a big influence on how we code in the future. For example, the `Optional` class will be familiar to some and allows a better way to avoid dealing with nulls.
* There are various concurrency library improvements. Things like an improved concurrent hash map, completable futures, thread safe accumulators, an improved read write lock (called a `StampedLock`), an implementation of a work stealing thread pool and much more besides.
* Apart from lambda support, other language features have also been added. Things like support for adding static methods to interfaces and default methods (otherwise known as virtual extension or defender methods).
* To support lambdas a bunch of stuff had to go on behind the scenes. Type inference had to be improved and new constructs like functional interfaces and method references have all been worked in.
* A new date and time API has been introduced.
* The IO and NIO packages have welcome additions to allow you to work with IO streams using the new streams API.
* Reflection and annotations have been improved.
* If that's not enough, an entirely new JavaScript engine ships with Java 8. Nashorn replaced Rhino and is faster and has better support for ECMA-Script.
* The JVM has had a bunch of improvements; it could be the fastest JVM to date as the integration with JRocket is now complete.
* The JVM has dropped the idea of perm gen, instead using native OS memory for class metadata. This is a huge deal and should in theory mean better memory utilitsation.
* The JRocket integration also brings Mission control to the JDK as standard. It compliments `jconsole` and `visualvm` with similar functionality but adds very inexpensive profiling.

The list is almost endless, improvements to JavaFX, base64 encoding support and I'm sure I must have missed some out.
