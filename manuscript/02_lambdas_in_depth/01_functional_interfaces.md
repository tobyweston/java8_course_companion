# λ In-depth

## Functional Interfaces

We talked earlier about how Java treats lambdas as an instance of an interface type. Java 8 formalises this into something it calls "functional interfaces".

A functional interface is just an interface with a single method. Java calls the method a "functional method" but the name "single abstract method" or SAM is often used. All the existing single method interfaces like `Runnable` and `Callable` in the JDK are therefore functional interfaces and lambdas can be used anywhere the interface is used. In fact, it's functional interfaces that allow for what's called "target typing"; they provide enough information for the compiler to infer argument and return types.

Oracle have introduced a new annotation `@FunctionalInterface` to mark an interface as such. It's basically to communicate intent but also allows the compiler to do some additional checks.

Lets jump into the IDE to look at some examples.

So I'll go ahead and create an interface, lets call it `Example`.

    public interface FunctionalInterfaceExample {
    }


I'll use the new annotation to mark this as a functional interface.

    @FunctionalInterface // <- error here
        public interface FunctionalInterfaceExample {
    }


Now, notice two things. Firstly, the IDE has added this orange highlight to give us a hint that something is wrong. If I hover over this, it tells us "no target method was found". It's hinting that we left off the functional method.

The second thing to notice is that if we compile the class, we get an error. It says that "Example is not a functional interface" as "no abstract method was found"

Lets go ahead and add the functional method and recompile. Because of the way I've got the IDE setup, the error window disappears to tell us the compilation was successful.


ok, so what if we try and add a second method to the interface?

    @FunctionalInterface
    public interface FunctionalInterfaceExample {
        void apply();
        void illegal(); // <- error here
    }


Notice the hint highlight is back? This time it's saying "multiple, non-overriding abstract methods were found". Functional interfaces can have only **one** method.

If we compile, you'll see the compiler error saying the same thing.

Let's go ahead and strip that off and recompile successfully this time.



Ok. so let's have a look at the case of interfaces that extends other interfaces.

I'll create a new functional interface called "A" and another called "B" which extends "A". "B" is still "functional". It inherits the parents `apply` method as you'd expect.

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


We can verify it works as a functional interface if we use it as a lambda. So I'll implement a little method here to show that a lambda can be assigned to a type of `A` and a type of `B`. The implementation just prints out "A" or "B".

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

You still can't add a new abstract method to the extending interface though, as the resulting type would have two abstract methods and so the IDE will warn us and the compiler will error.

Let's drop this line for a second and add a new method.

    @FunctionalInterface
    public interface A {
        void apply();
    }

    public interface B extends A {
        void illegal();
    }

    public static void main(String... args) {
        A a = () -> System.out.println("A");
        B b = () -> System.out.println("B");    // <- error
    }


Notice the IDE hint and if we compile, we get an error.

Let me take that off and recompile.


What you can do in both cases though is override methods from `Object` without causing problems. You can also add default methods (which we'll cover later).


As you'd probably expect, it doesn't make sense to try and mark an abstract class as a functional interface.


Interfaces generally have had some new features added. We'll look at those in their own sections later but just so that you're aware they include.

[video notes - use split screen presentation/demo mode again]

 * Default methods (virtual extension methods)
 * Static interface methods
 * and a bunch of new functional interfaces in the `java.util.function` package; things like `Function` and `Predicate`


## Summary


To recap then, in this section we've talked about how any interface with a single method is now a "functional interface".

We've talked about how _that_ single method is often called a "functional method" or SAM (for single abstract method).

We looked at the new annotation and saw a couple of examples of how existing JDK interfaces like `Runnable` and `Callable` have been retrofitted _with_ the annotation.

An important point to take away was the idea that any place a functional interface is used, we can now use lambdas. Lambdas can be used in-lieu of anonymous implementations of the functional interface.

We also introduced the idea of "target typing" which is how the compiler can use the signature of a functional method to help work out what lambdas can be used where. We skimmed over this a little as we're going to talk about it later in the type inference section.

During the demo part, we saw some examples of functional interfaces, how the compiler and IDE can help us out when we make mistakes and got a feel for the kinds of errors we might encounter. Things like adding more than one method to a functional interface. We also saw the exceptions to this rule, namely when we override methods from `Object` or implement default methods.

We had a quick look at interface inheritance and how that affects things and I mentioned some of the other interface improvements that we'll be covering later.