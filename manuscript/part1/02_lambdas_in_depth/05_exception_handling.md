## Exception Handling

There's no new syntax for exception handling in lambdas. Exceptions thrown in a lambda are propagated to the caller, just as you'd expect with a regular method call. There's nothing special about calling lambdas or handling their exceptions.

[slide-hint: two points 1) and 2) ]

However, there are some subtleties that you need to be aware of. Firstly, as a caller of a lambda, you are potentially unaware of what exceptions might be thrown, if any and secondly, as the author of a lambda, you're potentially unaware what context your lambda will be run in. What I mean here is that you give up responsibility of how your lambda will be executed to the method that you pass it to. For all you know, your lambda may be run in parallel or at some point in the future and so any exception you throw may not get handled as you might expect. You can't rely on exception handling as a way to control your program's flow.


[demo part-I](async-example.demo)

To demonstrate this, lets write some code to call two things, one after the other. We'll use `Runnable` as a convenient lambda type.

	public static void runInSequence(Runnable first, Runnable second) {
		first.run();
		second.run();
	}

If the first call to `run` were to throw an exception, the method would terminate and the second method would never be called. The caller is left to deal the exception.

If we use this method to transfer money between two bank accounts for example, we might write two lambdas. One for the debit action and one for the credit.

	public void transfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = () -> a.debit(amount);
		Runnable credit = () -> b.credit(amount);
	}

we could call our `runInSequence` method like this

	public void transfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = () -> a.debit(amount);
		Runnable credit = () -> b.credit(amount);
		runInSequence(debit, credit);
	}

any exceptions could be caught and dealt with easily at this level.

    public void transfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = () -> a.debit(amount);
		Runnable credit = () -> b.credit(amount);
		try {
			runInSequence(debit, credit);
		} catch (Exception e) {
            // check account balances and rollback
		}
	}


Here's the thing. As an author of the lambdas, I potentially have no idea how `runInSequence` is implemented. It may well be implemented to run asynchronously like this.

	public static void runInSequence(Runnable first, Runnable second) {
		new Thread(() -> {
			first.run();
			second.run();
		}).start();
	}


In which case would mean that any exception in the first call would terminate the thread, the exception would disappear to the default exception handler and our original client code wouldn't get the chance to deal with the exception.

[end-demo]


### Optional Section

[demo part-II](???.demo)

Incidentally, one way round the specific problem with raising an exception on a different thread than the caller can be addressed with a callback function. Firstly, you'd defend against exception in the `runInSequence` method.

	public static void runInSequence(Runnable first, Runnable second) {
		new Thread(() -> {
			try {
				first.run();
				second.run();
			} catch (Exception e) {
				// ...
			}
		}).start();
	}

Then introduce an exception handler which can be called in the event of an exception

	public static void runInSequence(Runnable first, Runnable second, Consumer<Throwable> exceptionHandler) {
		new Thread(() -> {
			try {
				first.run();
				second.run();
			} catch (Exception e) {
				exceptionHandler.accept(e);
			}
		}).start();
	}

`Consumer` is a functional interface which in this case takes the exception as an argument to it's `accept` method.

When we wire this up to the client, we can pass in a callback lambda to handle any exception.

	public void nonBlockingTransfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = () -> a.debit(amount);
		Runnable credit = () -> b.credit(amount);
		runInSequence(debit, credit, (exception) -> { /* check account balances and rollback */ });
	}


This has it's own caveats! It's a good example of deferred execution. The exception handler method may (or may not) get executed at some later point in time. The `nonBlockingTransfer` method will have finished and the bank accounts themselves may be in some other state. You can't rely on the exception handler being called when it's convenient for you; we've opened up a whole can of concurrency worms.

[end-demo]


Getting back to the point...

[slide hint - cut away from demo to show slide with the two perspectives before going back to the first demo]

Let's look at exception handling from the two perspectives in turn. First we'll look at dealing with exceptions when writing lambdas, then we'll look at dealing with exceptions when calling lambdas.


## As an Author (Dealing with Exceptions When Writing Lambdas)

To start with, lets look at it as if we wanted to implement the transfer method using lambdas but this time wanted to reuse an existing library that supplies the `runInSequence` method.


[demo](implementing-lambdas.demo)

Before we start, let's take a look at the `BankAccount` class. [cmd+shift+P]

You'll notice that this time, the debit and credit methods both throw a checked exception; `InsufficientFundsException`.

Let's start fresh and recreate the transfer method. [cmd+shift+P].

We'll try to create the `debit` and `credit` lambdas and pass these into the `runInSequence` method. Remember that the `runInSequence` method was written by some library author and we can't see or modify it's implementation.

The `debit` and `credit` both throw a checked exception, so this time, you can see a compiler error. It makes no difference if we add this to the method signature; the exception would happen inside the lambda. Remember I said exceptions in lambdas are propagated to the caller? In our case, this will be the `runInSequence` method and not the point we define the lambda. The two aren't communicating between themselves that there could be an exception raised.

So we'll take that off again.

So if we can't force a checked exception to be _transparent_ between the lambda and the caller, one option is to wrap the checked exception as a runtime exception. I'll go ahead and do that within the lambdas.

I'll get the IDE to help me here and surround the statement with a try/catch.

...and again here.

That gets us out of the compilation error but it's not the full story yet.

We also have to catch and deal with, what's now a runtime exception, around the call to `runInSequence`.

There's still one or two niggles though; we're throwing and catching a `RuntimeException` which is perhaps a little loose. We don't really know what other exceptions, if any, might be thrown in the `runInSequence` method. Perhaps it's better to be more explicit.

Let's create a new sub-type of `RuntimeException` and use that instead.

We'll call it insufficient funds runtime exception.

I'll just have to go back and throw the new exception from within the lambdas.

and restrict the final catch to deal with only exceptions we know about. Namely the insufficient funds exception. I _could_ now implement some kind of balance check and rollback functionality, confident that I know all the scenarios that cause it.


---

The trouble with all this, is that the code has more exception handling boilerplate than actual business logic. Lambdas are supposed to make things less verbose but this is just full of noise.

We can make things better if we generalise the wrapping of checked exceptions to runtime equivalents. We could create a functional interface that captures an exception type on the signature using generics.

Let's name if `Callable` and its single method; `call`.

	@FunctionalInterface
	interface Callable<E extends Exception> {
		void call() throws E;
	}

We'll chop out the old implementation and create a lambda to match the "shape" of the new functional interface.

    () -> a.debit(amount);

Remember from the type inference lecture that Java is able to see this as a lambda of type `Callable` as it has no parameters and as does `Callable`, it has the same return type (none) and throws an exception of the same type as the interface. So we can assign this to an instance of a `Callable`.

    Callable<InsufficientFundsException> function = () -> a.debit(amount);

Creating this lambda like this doesn't cause a compilation error as the functional interface declares that it could be thrown. It doesn't need to warn us early, at the point we create the lambda, as the signature of the functional method will cause the compiler ti error if required when we actually try and call it. Just like a regular method.

Let's go ahead and create the other lambda and pass them into the `runInSequence` method.

As you can see, there's an error when we try and pass them in. They're the wrong type. We still need a lambda of type `Runnable`. We'll have to write a method that can convert from a `Callable` to a `Runnable` and wrap the checked exception to a runtime one.

Something like this.

	public static Runnable unchecked(Callable<InsufficientFundsException> function) {
		return () -> {
			try {
				function.call();
			} catch (InsufficientFundsException e) {
				throw new InsufficientFundsRuntimeException(e);
			}
		};
	}

All that's left to do is wire it in for our lambdas.

    Runnable debit = unchecked(() -> a.debit(amount));
	Runnable credit = unchecked(() -> b.credit(amount));


We're back to a more concise method body and have dealt with the potential exceptions in the same way as before. You can see the difference if we compare the two side by side. Especially when we look at just the critical regions of the code; namely the `transfer` methods.

The downside is this isn't a totally generalised solution; we'd still have to create variations of the `unchecked` method for different functions. We've also just hidden the verbose syntax away. The verbosity is still there its just been moved.

Yes, we've got some reuse out of it but if exception handling were transparent or we didn't have checked exceptions, we wouldn't need to brush the issue under the carpet quiet so much.

It's worth pointing out that we'd probably end up doing something similar if we were in Java 7 and using anonymous classes instead of lambdas. A lot of this stuff can still be done pre-Java 8 and you'll end up creating helper methods to push the verbosity to one side.

It's certainly the case that lambdas offer more concise representations for small anonymous pieces of functionality but because of Java's checked exception model, dealing with exceptions in lambdas will often cause all the same verbosity problems we had before.

[blog fodder - It seems to me we can't easily have exception transparency whilst we have checked exceptions in Java. The JDK boys would have to jump through hoops to enable it where the fundamental tension is with checked exceptions. If we had only unchecked exceptions, exception handling in lambdas would be inherently transparent.]


## As a Caller (Dealing with Exceptions when Calling Lambdas)

Ok, so we've seen things from the perspective of writing lambdas, now lets have a look at things when calling lambdas.

Let's imagine that now we're writing the library that offers the `runInSequence` method. We have more control this time and aren't limited to using `Runnable` as a lambda type. Because we don't want to force our clients to jump through hoops dealing with exceptions in their lambdas (or wrap them as runtime exceptions), we'll provide a functional interface that declares that a checked exception might be thrown.

We'll call it `FinancialTransfer` with a `transfer` method.

	@FunctionalInterface
	interface FinancialTransfer {
		void transfer() throws InsufficientFundsException;
	}

We're saying that whenever a banking transaction occurs, there's the possibility that insufficient funds are available.

Then when we implement our `runInSequence` method, we accept lambdas of this type.

	public static void runInSequence(FinancialTransfer first, FinancialTransfer second) throws InsufficientFundsException {
		first.transfer();
		second.transfer();
	}

This means that when clients use the method, they're not forced to deal with exceptions within their lambdas. For example, writing a method like this.

    // example client usage
	public void transfer(BankAccount a, BankAccount b, Integer amount) {
		FinancialTransfer debit = () -> a.debit(amount);
		FinancialTransfer credit = () -> b.credit(amount);
        runInSequence(debit, credit);
	}

Notice that there is no compiler error on the lambdas. There's no need to wrap the exceptions from `BankAccount` methods as runtime exceptions. The functional interface has already declared the exception.

`runInSequence` now throws a checked exception, so it's explicit that the client has to deal with the possibility and you'll see a compiler error.

Let's wrap that in a try and catch to make the compiler happy.

The end result is something like we saw previously but without the need for the `unchecked` method. As a library developer, we've made it easier for clients to integrate with our code.

But what about if we try something more exotic?

Let's make the `runInSequence` method asynchronous again,

	new Thread(() -> {
        first.transfer();
        second.transfer();
    }).start();

There's no need to throw the exception anymore so lets take that off the signature and remove the catch block above.

With that gone and compiler errors still in the `runInSequence` method, we need another way to handle the exception. One technique is to pass in a function that will be called in the event of an exception. We can use this lambda to bridge the code running asynchronously back to the caller.

To start with, we'll add the catch block back in and pass in a functional interface to use as the exception handler.

I'll use the `Consumer` interface here, it's new in Java 8 and part of the `java.util.function` package. We then call the interface method in the catch block, passing in the cause. To make that useful, we need to supply a lambda in the `transfer` method.

The parameter of the lambda, i've called it `exception` here, will be whatever is passed into the `accept` method in `runInSequence`. It will be an instance of `InsufficientFundsException` and the client can deal with it however they chose.

Lets pop that in place to finish the example.

There we are. We've provided the client to our library with an alternative exception handling mechanism rather than forcing them to catch exceptions.

We've _internalised_ the exception handling into the library code. It's a good example of deferred execution; should there be an exception, the client doesn't necessarily know when his exception handler would get invoked. For example, as we're running in another thread, the bank accounts themselves may have be altered by the time it executes. Again it highlights that using exceptions to control your program's flow is a flawed approach.

You can't rely on the exception handler being called when it's convenient for you; we've just opened a whole can of concurrency worms.


Looking at the two alternatives side by side should give you a feel for how they differ. The original version forces the client to deal with the exception directly (via a throws clause on the functional interface). Whereas the callback version, although running in a thread, handles the exception, providing a callback via a client lambda.


## Old Text

Exception transparency describes the problem of how to handle exceptions thrown within a closure in the calling code. A closure throwing an exception may not be obvious or _transparent_ to the calling code. For example [make example up].

Java's solution is to rely on checked exceptions and force any calling code to deal with the exception; just like you've been doing all along. You defer the exception handling to the enclosing scope.

This means that either a) you're functional interface methods have to throw exceptions explicitly or b) you catch and rethrow as `RuntimeException`. The later is not a fix. Another alternative is a callback. When you consider that lambdas could be run in parallell you can see how bad it can get... [example?].


## Notes

(from [this post](http://www.techempower.com/blog/2013/03/26/everything-about-java-8/))

Exception transparency - If a checked exception may be thrown from inside a lambda, the functional interface must also declare that checked exception can be thrown. The exception is not propagated to the containing method. This code does not compile:

    void appendAll(Iterable<String> values, Appendable out) throws IOException { // doesn't help with the error
        values.forEach(s -> {
            out.append(s); // error: can't throw IOException here
                           // Consumer.accept(T) doesn't allow it
        });
    }

There are ways to work around this, where you can define your own functional interface that extends Consumer and sneaks the IOException through as a RuntimeException. I tried this out in code and found it to be too confusing to be worthwhile.

Control flow (break, early return) - In the forEach examples above, a traditional continue is possible by placing a "return;" statement within the lambda. However, there is no way to break out of the loop or return a value as the result of the containing method from within the lambda. For example:

final String secret = "foo";
boolean containsSecret(Iterable<String> values) {
    values.forEach(s -> {
        if (secret.equals(s)) {
            ??? // want to end the loop and return true, but can't
        }
    });
}
For further reading about these issues, see this explanation written by Brian Goetz: response to [Checked exceptions within Block<T>](http://mail.openjdk.java.net/pipermail/lambda-dev/2013-January/007662.html)


[Could also mention breaking control flow ie from http://www.techempower.com/blog/2013/03/26/everything-about-java-8/]