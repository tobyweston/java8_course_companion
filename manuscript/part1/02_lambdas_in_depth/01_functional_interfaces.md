## Functional Interfaces {#functional_interfaces}

Java 8 treats lambdas as an instance of an interface type. It formalises this into something it calls _functional interfaces_. A functional interface is just an interface with a single method. Java calls the method a "functional method" but the name "single abstract method" or SAM is often used. 

All the existing single method interfaces like `Runnable` and `Callable` in the JDK are now functional interfaces and lambdas can be used anywhere a single abstract method interface is used. In fact, it's functional interfaces that allow for what's called _target typing_; they provide enough information for the compiler to infer argument and return types.



### @FunctionalInterface

Oracle have introduced a new annotation `@FunctionalInterface` to mark an interface as such. It's basically to communicate intent but also allows the compiler to do some additional checks.

For example, this interface compiles,


    public interface FunctionalInterfaceExample {
        // compiles ok
    }


but when you indicate that it should be a _functional interface_ bu adding the the new annotation,


    @FunctionalInterface // <- error here
    public interface FunctionalInterfaceExample {
        // doesn't compile
    }


the compiler will raise an error. It tells us "Example is not a functional interface" as "no abstract method was found". Often the IDE will also hint, [IntelliJ](http://www.jetbrains.com/idea/) will say something like "no target method was found". It's hinting that we left off the functional method. A "single abstract method" needs a single, abstract method.

So what if we try and add a second method to the interface?


    @FunctionalInterface
    public interface FunctionalInterfaceExample {
        void apply();
        void illegal(); // <- error here
    }


The compiler will error again, this time with a message along the lines of "multiple, non-overriding abstract methods were found". Functional interfaces can have only **one** method.


### Extension

What about the case of an interfaces that extends another interface?

Let's create a new functional interface called `A` and another called `B` which extends `A`. The interface `B` is still a functional interface. It inherits the parents `apply` method as you'd expect.


    @FunctionalInterface
    interface A {
        abstract void apply();
    }

    interface B extends A {
    }

If you wanted to make this clearer, you can also override the functional method from the parent.


    @FunctionalInterface
    interface A {
        abstract void apply();
    }

    interface B extends A {
        @Override
        abstract void apply();
    }


We can verify it works as a functional interface if we use it as a lambda. We'll implement a method to show that a lambda can be assigned to a type of `A` and a type of `B` below. The implementation just prints out "A" or "B".


    @FunctionalInterface
    public interface A {
        void apply();
    }

    public interface B extends A {
        @Override
        void apply();
    }

    public static void main(String... args) {
        A a = () -> System.out.println("A");
        B b = () -> System.out.println("B");
    }

You can't add a new abstract method to the extending interface (`B`) though, as the resulting type would have two abstract methods and the IDE would warn us and the compiler would error.


    @FunctionalInterface
    public interface A {
        void apply();
    }

    public interface B extends A {
        void illegal();     // <- can't do this
    }

    public static void main(String... args) {
        A a = () -> System.out.println("A");
        B b = () -> System.out.println("B");    // <- error
    }



In both cases, you can override methods from `Object` without causing problems. You can also add default methods (also new to Java 8). As you'd probably expect, it doesn't make sense to try and mark an abstract class as a functional interface.


### Other Interface Improvements

Interfaces generally have had some new features added, they include:

 * Default methods (virtual extension methods)
 * Static interface methods
 * and a bunch of new functional interfaces in the `java.util.function` package; things like `Function` and `Predicate`


### Summary


In this section we've talked about how any interface with a single method is now a "functional interface" and that the single method is often called a "functional method" or SAM (for single abstract method).

We looked at the new annotation and saw a couple of examples of how existing JDK interfaces like `Runnable` and `Callable` have been retrofitted with the annotation.

We also introduced the idea of _target typing_ which is how the compiler can use the signature of a functional method to help work out what lambdas can be used where. We skimmed over this a little as we're going to talk about it later in the type inference section.

We discussed some examples of functional interfaces, how the compiler and IDE can help us out when we make mistakes and got a feel for the kinds of errors we might encounter. Things like adding more than one method to a functional interface. We also saw the exceptions to this rule, namely when we override methods from `Object` or implement default methods.

We had a quick look at interface inheritance and how that affects things and mentioned some of the other interface improvements that we'll be covering later.



An important point to take away here is the idea that any place a functional interface is used, you can now use a lambda. Lambdas can be used in-lieu of anonymous implementations of the functional interface. Using a lambda instead of the anonymous class may seem like syntactic sugar, but they're actually quiet different. See the [Functions vs. classes](#functions_vs_classes) section for more details.



