## Functions vs classes

Bear in mind though, that an anonymous _function_ isn't the same as an anonymous _class_ in Java. An anonymous class in Java still needs to be instantiated to an object. It may not have a proper name but it's only as an _object_ that it can be useful.

A _function_ on the other hand has no instance associated with it. Functions are disassociated with the data they act on whereas an object is intimately associated with the data it acts upon.


### Summary

[slide - read the bullets]

So...

* Classes must be instantiated, whereas functions are not.
* When classes are newed up, memory is allocated for the object.
* Memory need only be allocated once for functions. They are stored in the "permanent" area of the heap.
* Objects act on their own data, functions act on unrelated data.
* Static class methods in Java are roughly equivalent to functions.


When we take a look at the new lambda syntax, remember that although lambdas are used in a very similar way to anonymous classes in Java, they are technically different. Lambdas in Java need not be instantiated every time they're evaluated unlike an instance of an anonymous class.

This should serve to remind you that lambdas in Java 8 are NOT just syntactic sugar. We'll look at this in more detail later but for now, let's have a look at the syntax.








## Notes (not to be included)

Lambdas in Java need not be instantiated every time they are evaluated unlike an instance of an anonymous class which will create a new instance each time. This obviously has time and memory implications. There's a caveat with closures [need to verify in bytecode] in that they do need to be instantiated every time as they need to have their environment passed in. (see http://programmers.stackexchange.com/questions/177879/type-inference-in-java-8/)
