## Functions vs classes

Bear in mind that an anonymous _function_ isn't the same as an anonymous _class_ in Java. An anonymous class in Java still needs to be instantiated to an object. It may not have a proper name but it's only as an _object_ that it can be useful.

A _function_ on the other hand has no instance associated with it. Functions are disassociated with the data they act on whereas an object is intimately associated with the data it acts upon.


## Lambdas in Java 8

You can use lambdas in Java 8 anywhere you would have previously used a [single method interface](???functional interfaces????) so it may just look like syntactic sugar but it's not. Let's have a look at how they differ and compare anonymous classes to lambdas; classes vs. functions.

A typical implementation of an anonymous class (a single method interface) in Java pre-8, might look something like this. The `anonymousClass` method is calling the `waitFor` method passing in some implementation of `Condition`, in this case it's saying wait for some server to have shutdown.

{% codeblock lang:java %}
void anonymousClass() {
    final Server server = new HttpServer();
    waitFor(new Condition() {
        @Override
        public Boolean isSatisfied() {
            return !server.isRunning();
        }
    });
}
{% endcodeblock %}

The functionally equivalent lambda would look like this.

{% codeblock lang:java %}
void closure() {
    Server server = new HttpServer();
    waitFor(() -> !server.isRunning());
}
{% endcodeblock %}

Where in the interest of completeness, a naive polling `waitFor` method might look like this.

{% codeblock lang:java %}
class WaitFor {
	static void waitFor(Condition condition) throws InterruptedException {
		while (!condition.isSatisfied())
			Thread.sleep(250);
	}
}
{% endcodeblock %}


## Some Theoretical Differences

Firstly, both implementations are in-fact closures, the later is also a lambda. We'll look at this distinction in more detail later in the [lambdas vs. closures](???) section. This means that both have to capture their "environment" at runtime. In Java pre-8, this means copying the things the closure needs into an instance of an class (an anonymous instances of `Condition`). In our example, the `server` variable.

As it's a copy, it has to be declared final to ensure that it can not be changed between when it's captured and when it's used. These two points in time could be very different given that closures are often used to defer execution until some later point (see [lazy evaluation](http://en.wikipedia.org/wiki/Lazy_evaluation) for example). Java 8 uses a neat trick whereby if it can reason that a variable is never updated, it might as well be final so it treats it as "effectively final" and you don't need to declare it as `final` explicitly.

A lambda on the other hand, doesn't need to copy it's environment or _capture any terms_. This means it can be treated as a genuine function and not an instance of a class. What's the difference? Plenty.


### Functions vs. Classes

For a start, functions; [genuine functions](http://en.wikipedia.org/wiki/Pure_function), don't need to be instantiated many times. I'm not sure if instantiation is even the right word to use when talking about allocating memory and loading a chunk of machine code as a function. The point is, once it's available, it can be re-used, it's idempotent in nature as it retains no state. Static class methods are the closest thing Java has to functions.

For Java, this means that a lambda need not be instantiated every time it's evaluated which is a big deal. Unlike instantiating an anonymous class, the memory impact should be minimal.

In terms of some conceptual differences then;

* Classes must be instantiated, whereas functions are not.
* When classes are newed up, memory is allocated for the object.
* Memory need only be allocated once for functions. They are stored in the "permanent" area of the heap.
* Objects act on their own data, functions act on unrelated data.
* Static class methods in Java are roughly equivalent to functions.


## Some Concrete Differences

### Capture Semantics

Another difference is around capture semantics for `this`. In an anonymous class, `this` refers to the instance of the anonymous class. For example, `Foo$InnerClass` and not `Foo`. That's why you have crazy syntax like `Foo.this.x` when you refer to the enclosing scope from the anonymous class.

In lambdas on the other hand, `this` refers to the enclosing scope (`Foo` directly in our example). In fact, lambdas are **entirely lexically scoped**, meaning they don't inherit any names from a super type or introduce a new level of scoping at all; you can directly access fields, methods and local variables from the enclosing scope.

For example, this class shows that the lambda can reference the `firstName` variable directly.

{% codeblock lang:java %}
public class Example {

	private String firstName = "Jack";

	public void example() {
		Function<String, String> addSurname = surname -> {
			return firstName + " " + surname;       // equivalent to this.firstName
		};
	}
}
{% endcodeblock %}

The anonymous class equivalent would need to explicitly refer to `firstName` from the enclosing scope.

{% codeblock lang:java %}
public class Example {

	private String firstName = "Charlie";

    public void anotherExample() {
        Function<String, String> addSurname = new Function<String, String>() {
            @Override
            public String apply(String surname) {
                return Example.this.firstName + " " + surname;
            }
        };
    }
}
{% endcodeblock %}


Shadowing also becomes much more straight forward to reason about (when referencing shadowed variables).


## Summary

Functions in the academic sense are very different things from anonymous classes (which we often treat like functions in Java pre-8). It's useful to keep the distinctions in your head to be able to justify the use of Java 8 lambdas for something more than just their concise syntax. Of course, there's lots of additional advantages in using lambdas (not least the retrofit of the JDK to heavily use them).

When we take a look at the new lambda syntax next, remember that although lambdas are used in a very similar way to anonymous classes in Java, they are technically different. Lambdas in Java need not be instantiated every time they're evaluated unlike an instance of an anonymous class.

This should serve to remind you that lambdas in Java 8 are **not** just syntactic sugar. 
