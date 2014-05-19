## λ Basic Syntax

[slide]

Lets get started

I'm sure you're keen to have a look at the basic syntax, so lets jump into that first.

So, a lambda is basically an anonymous block of functionality. It's a lot like using an anonymous class instance.

[demo - basic structure]

For example, if we want to sort an array in Java, we can use the `Arrays.sort` method which takes an instance of a `Comparator`.

It would look something like this.

    Arrays.sort(numbers, new Comparator<Integer>() {
      @Override
      public int compare(Integer first, Integer second) {
        return first.compareTo(second);
      }
    });

The `Comparator` instance here is a an abstract piece of the functionality; it means nothing on its only when it's used by the sort method that it has purpose.

Using Java's new syntax, you can replace this with a lambda which looks like this

    Arrays.sort(numbers, (first, second) -> first.compareTo(second));

It's a more succinct way of achieving the same thing. In fact, Java treats this as if it were an instance of the `Comparator` class.

We can see this if we extract a variable. Notice it's type? The lambda is of type `Comparator<Integer>` just like the anonymous instance above.

Because `Comparator` has only a single abstract method on it, `compareTo`, the compiler can piece together that when we have an anonymous block like this, that what we really mean an instance of `Comparator`.  It can do this thanks to a couple of the other new features that we'll talk about later; functional interfaces and improvements to type inference.


[demo - syntax breakdown]

So in terms of syntax, you can always convert from using a single abstract method to a using lambda.

For example, let's say we have an an interface `Example` with a method `apply`, returning some type and taking some argument

    interface Example {
      R apply(A arg);
    }

We could instantiate an instance with something like this;

new Example, override the method, implement the body.

    new Example() {
      @Override
      public R apply(A args) {
        body
      }
    };

And to convert to a lambda, we basically trim the fat. We drop the instantiation and annotation... we drop the method details which leaves just the argument list and the body.

      (args) {
        body
      }

we then introduce the new arrow symbol to indicate both that the whole thing is a lambda and that what follows is the body.

    (args) -> { body }


and that's our basic lambda syntax.


Let's take the sorting example from earlier through these steps.

We start with the anonymous instance;

    Arrays.sort(numbers, new Comparator<Integer>() {
      @Override
      public int compare(Integer first, Integer second) {
        return first.compareTo(second);
      }
    });


and trim the instantiation and method signature

    Arrays.sort(numbers, (Integer first, Integer second) {
      return first.compareTo(second);
    });

introduce the lambda

    Arrays.sort(numbers, (Integer first, Integer second) -> {
      return first.compareTo(second);
    });

and we're done. There's a couple of optimisations we can do though. You can drop the types if the compiler knows enough to infer them.

    Arrays.sort(numbers, (first, second) -> {
      return first.compareTo(second);
    });

and for simple expressions, you can drop the braces to produce a lambda expression

    Arrays.sort(numbers, (first, second) -> first.compareTo(second));


In this case, the compiler can infer enough to know what you mean. The single statement returns a value consistent with the interface, so it says, "no need to tell me that you're going to return something, I can see that for myself!".


For single argument interface methods, you can even drop the first brackets. For example

A lambda taking an argument `x` and returning `x + 1`

    (x) -> x + 1

can be written without the brackets

    x -> x + 1



[slide]

Let's recap with a summary of the syntax options.

The first example is the most verbose way to create a lambda. The arguments to the function along with their types are in parenthesise, followed by the new arrow syntax and then the body; the code block to be executed.

You can often drop the types from the argument list. The compiler will use type inference here to try and guess the types. It does this based on the context that you're trying to use the lambda in.

and if your code block returns something or is a single line expression, you can drop the braces and return statement.

In the case of only a single argument, you can drop the parentheses...

...and if you have no arguments at all, the 'hamburger' symbol is needed.

In the interest of completeness, there is another variation; a kind of shortcut to a lambda called a "method reference". So this last one is actually a short cut to a lambda.

We're going to talk about those in more detail later, so for now, just be aware that they exist and can be used anywhere you can use a lambda.
