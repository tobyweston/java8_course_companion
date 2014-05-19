## Type Inference Improvements

There have been several type inference improvements in Java 8. Firstly to be able to support lambdas, the way the compiler infers things has been beefed up to use "target typing" extensively and secondly, specific improvements were made over Java 7. These were managed under the Open JDK Enhancement Proposal 101.

Before we get into those, lets recap on the basics.

Type inference refers to the ability for a programming language to automatically deduce the type of an expression.

Statically typed languages know the types of things only at _compile_ time. Dynamically typed languages know the types only at _runtime_. A statically typed language can use type inference and drop type information in source code and use the compiler to figure out what's missing.

So this means that type inference can be used by statically typed languages, like Scala, to "look" like dynamic languages (like JavaScript). At least in the source code.

Here's a line of Scala, it just doesn't need to tell the compiler explicitly that the value is a string. It figures it out.

You could write it out explicitly, but there's no need.

---

Type inference is a fairly broad topic, Java doesn't support the type of inference I've just been talking about, at least for things like dropping the type annotations for variables. We have to keep that in.

    String name = "Henry";


So Java doesn't support type inference in the widest sense. It can't guess _everything_ like some languages. Type inference for Java then typically refers to the way the compiler can work out types for generics. Java 7 improved this when it introduced the diamond operator but there are still lots of limitations in what Java _can_ figure out.

The Java compiler was built with type erasure; it actively removes the type information during compilation. For historical reasons, when generics were introduced in Java 5, the developers couldn't easily reverse the decision to use erasure. Java was left with the need to understand what types to substitute for a given generic type but no information how to do it because it had all been erased. Type inference was the solution.

That's why type inference became important to the Java community.

Because of type erasure then, a `List<String>` [pause] would become a `List<Object>` after compilation.

All generic values are really of type `Object` behind the scenes but by using type inference, the compiler can check that all the usages are consistent with what it thinks the generic should be. At runtime, everything is going to get passed around as instances of `Object` with appropriate casting. Type inference just allows the compiler to check that the casts would be valid ahead of time.


So type inference is about guessing the types, Java's support for type inferences was due to be improved in a couple of ways with Java 8.

1. Target-typing for lambdas

and using generalised target-typing to

1. Add support for parameter type inference in method calls
1. Add support for parameter type inference in chained calls

Lets have a look at the current problems and how Java 8 addresses them.


## Target-typing for lambdas

The general super-charging of type inference in Java 8 means that lambdas can infer their type parameters; it means means you drop the `Integer` type annotation here!

    (Integer x, Integer y) -> x + y;


    (x, y) -> x + y;

This is because the functional interface describes the types, it gives the compiler all the information it needs.

Let's have a look with a demo

[demo](target_typing.demo)

    @FunctionalInterface
	interface Calculation {
		Integer apply(Integer x, Integer y);
	}

Let's create an example interface.

So when a lambda is created based on this interface, the first thing the compiler does is work out the "target" type of the lambda. So once the compiler knows the target is the 'Calculation' interface, it can use the method signature to guess the types of the lambdas parameters. As there is only one method on the interface, there's no ambiguity.

The only other bit to explain is how it identifies the target type. That's straight forward when we look at how the lambda might be used.

Here we have a `calculate` method that takes two numbers and applies them to the `Calculation` lambda.

	static Integer calculate(Calculation operation, Integer x, Integer y) {
		return operation.apply(x, y);
	}

we might create an addition and subtraction lambda

    Calculation addition = (x, y) -> x + y;
    Calculation subtraction = (x, y) -> x - y;

and use them like this

    calculate(addition, 2, 2);
    calculate(subtraction, 5, calculate(addition, 3, 2));


It's the `calculate` method that gives the final hint. The compiler can use it to identify the "target type" and from there the types for the lambda arguments.

[end-demo]

We're going to come across lots of examples of target typing as we go on so I'm not going to say much more on this. Just be aware that the mechanism Java uses to achieve lots of the lambda goodness relies on improvements to type inference and this idea of a "target" type.


## Type parameters in method calls

The were some situations prior to Java 8 where the compiler couldn't infer types. One of these was when calling methods with generics type parameters as arguments.

For example, the `Collections` class has a generified method for producing an empty list. It looks like this.

    public static final <T> List<T> emptyList() { ... }


In Java 7 this compiles

    List<String> names = Collections.emptyList();       // compiles in Java 7

as the Java 7 compiler can work out that the generic needed for the `emptyList` method is of type `String`.

What it struggles with though is if the result of a generic method is passed as a parameter to another method call.

So if we had a method to process a list that looks like this.

    static void processNames(List<String> names) {
        System.out.println("hello " + name);
    }

and then call it with the empty list method.

    processNames(Collections.emptyList());           // doesn't compile in Java 7

it won't compile because the generic type of the parameter has been erased to `Object`.

    processNames(Collections.<Object>emptyList names);

This doesn't match the `processList` method.

    processNames(Collections.<String>emptyList names);

So it won't compile until we give it an extra hint using an explicit "type witness".

    processNames(Collections.<String>emptyList());   // compiles in Java 7

Now the compiler knows enough about what generic type is being passed into the method.

The improvements in Java 8 include better support for this, so generally speaking where you would have needed a "type witness", you no longer do.

Our example of calling the `processNames` now compiles!

    processNames(Collections.emptyList());           // compiles in Java 8



## Type parameters in chained method calls

Another common problem with type inference is when methods are chained together. Lets suppose we have a `List` class,

	static class List<E> {

		static <T> List<T> emptyList() {
			return new List<T>();
		}

		List<E> add(E e) {
			// add element
			return this;
		}
	}

and we want to chain a call to add an element to the method creating an empty list. Type erasure rears it's head again; the type is erased and so can't be known by the next method in the chain. It doesn't compile.

    List<String> list = List.emptyList().add(":(");             // doesn't compile

This was due to be fixed in Java 8, but unfortunately it was dropped. So, at least for now, you'll still need explicitly offer up a type to the compiler.

    List<String> list = List.<String>emptyList().add(":(");    // still needs a "type witness"

You'll still need a type witness.



