## Exception Handling

There's no new syntax for exception handling in lambdas. Exceptions thrown in a lambda are propagated to the caller, just as you'd expect with a regular method call. There's nothing special about calling lambdas or handling their exceptions.

However, there are some subtleties that you need to be aware of. Firstly, as a caller of a lambda, you are potentially unaware of what exceptions might be thrown, if any and secondly, as the author of a lambda, you're potentially unaware what context your lambda will be run in.

When you create a lambda, you typically give up responsibility of how that lambda will be executed to the method that you pass it to. For all you know, your lambda may be run in parallel or at some point in the future and so any exception you throw may not get handled as you might expect. You can't rely on exception handling as a way to control your program's flow.

To demonstrate this, lets write some code to call two things, one after the other. We'll use `Runnable` as a convenient lambda type.

	public static void runInSequence(Runnable first, Runnable second) {
		first.run();
		second.run();
	}

If the first call to `run` were to throw an exception, the method would terminate and the second method would never be called. The caller is left to deal the exception. If we use this method to transfer money between two bank accounts, we might write two lambdas. One for the debit action and one for the credit.

	public void transfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = () -> a.debit(amount);
		Runnable credit = () -> b.credit(amount);
	}

we could then call our `runInSequence` method like this

	public void transfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = () -> a.debit(amount);
		Runnable credit = () -> b.credit(amount);
		runInSequence(debit, credit);
	}

any exceptions could be caught and dealt with by using a try/catch like this.

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


### Using a Callback

Incidentally, one way round the specific problem with raising an exception on a different thread than the caller can be addressed with a callback function. Firstly, you'd defend against exceptions in the `runInSequence` method.

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

	public static void runInSequence(Runnable first, Runnable second,
	        Consumer<Throwable> exceptionHandler) {
		new Thread(() -> {
			try {
				first.run();
				second.run();
			} catch (Exception e) {
				exceptionHandler.accept(e);
			}
		}).start();
	}

`Consumer` is a functional interface (new in Java 8) that in this case takes the exception as an argument to it's `accept` method.

When we wire this up to the client, we can pass in a callback lambda to handle any exception.

	public void nonBlockingTransfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = () -> a.debit(amount);
		Runnable credit = () -> b.credit(amount);
		runInSequence(debit, credit, (exception) -> {
		    /* check account balances and rollback */
        });
	}


This is a good example of deferred execution and so has it's own foibles. The exception handler method may (or may not) get executed at some later point in time. The `nonBlockingTransfer` method will have finished and the bank accounts themselves may be in some other state by the time it fires. You can't rely on the exception handler being called when it's convenient for you; we've opened up a whole can of concurrency worms.


### Dealing with Exceptions When Writing Lambdas

Let's look at dealing with exceptions from the perspective of a lambda author, someone writing lambdas. After this, we'll look at dealing with exceptions when calling lambdas.

Lets look at it as if we wanted to implement the transfer method using lambdas but this time wanted to reuse an existing library that supplies the `runInSequence` method.


Before we start, let's take a look at the `BankAccount` class. You'll notice that this time, the debit and credit methods both throw a checked exception; `InsufficientFundsException`.

    class BankAccount {
        public void debit(int amount) throws InsufficientFundsException {
            // ...
        }

        public void credit(int amount) throws InsufficientFundsException {
            // ...
        }
    }

    class InsufficientFundsException extends Exception { }

Let's recreate the transfer method. We'll try to create the `debit` and `credit` lambdas and pass these into the `runInSequence` method. Remember that the `runInSequence` method was written by some library author and we can't see or modify it's implementation.

	public void transfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = () -> a.debit(amount);   <- compiler error
		Runnable credit = () -> b.credit(amount); <- compiler error
		runInSequence(debit, credit);
	}

The `debit` and `credit` both throw a checked exception, so this time, you can see a compiler error. It makes no difference if we add this to the method signature; the exception would happen inside the lambda. Remember I said exceptions in lambdas are propagated to the caller? In our case, this will be the `runInSequence` method and not the point we define the lambda. The two aren't communicating between themselves that there could be an exception raised.

    // still doesn't compile
    public void transfer(BankAccount a, BankAccount b, Integer amount)
            throws InsufficientFundsException {
		Runnable debit = () -> a.debit(amount);
		Runnable credit = () -> b.credit(amount);
		runInSequence(debit, credit);
	}

So if we can't force a checked exception to be _transparent_ between the lambda and the caller, one option is to wrap the checked exception as a runtime exception like this.

    public void transfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = () -> {
			try {
				a.debit(amount);
			} catch (InsufficientFundsException e) {
				throw new RuntimeException(e);
			}
		};
		Runnable credit = () -> {
			try {
				b.credit(amount);
			} catch (InsufficientFundsException e) {
				throw new RuntimeException(e);
			}
		};
        runInSequence(debit, credit);
	}

That gets us out of the compilation error but it's not the full story yet. It's very verbose and we still have to catch and deal with, what's now a runtime exception, around the call to `runInSequence`.

    public void transfer(BankAccount a, BankAccount b, Integer amount) {
        Runnable debit = () -> { ... };
        };
        Runnable credit = () -> { ... };
        try {
            runInSequence(debit, credit);
        } catch (RuntimeException e) {
            // check balances and rollback
        }
    }

There's still one or two niggles though; we're throwing and catching a `RuntimeException` which is perhaps a little loose. We don't really know what other exceptions, if any, might be thrown in the `runInSequence` method. Perhaps it's better to be more explicit. Let's create a new sub-type of `RuntimeException` and use that instead.

	class InsufficientFundsRuntimeException extends RuntimeException {
		public InsufficientFundsRuntimeException(InsufficientFundsException cause) {
			super(cause);
		}
	}

I'll just have to go back and throw the new exception from within the lambdas.

When we go back to modify the original code, we can restrict the final catch to deal with only exceptions we know about; namely the `InsufficientFundsRuntimeException`. I can now implement some kind of balance check and rollback functionality, confident that I know all the scenarios that cause it.

    public void transfer(BankAccount a, BankAccount b, Integer amount) {
        Runnable debit = () -> {
            try {
                a.debit(amount);
            } catch (InsufficientFundsException e) {
                throw new InsufficientFundsRuntimeException(e);
            }
        };
        Runnable credit = () -> {
            try {
                b.credit(amount);
            } catch (InsufficientFundsException e) {
                throw new InsufficientFundsRuntimeException(e);
            }
        };
        try {
            runInSequence(debit, credit);
        } catch (InsufficientFundsRuntimeException e) {
            // check balances and rollback
        }
    }


The trouble with all this, is that the code has more exception handling boilerplate than actual business logic. Lambdas are supposed to make things less verbose but this is just full of noise. We can make things better if we generalise the wrapping of checked exceptions to runtime equivalents. We could create a functional interface that captures an exception type on the signature using generics.

Let's name it `Callable` and its single method; `call`. Don't confuse this with the class of the same name in the JDK; we're creating a new class to illustrate dealing with exceptions.

	@FunctionalInterface
	interface Callable<E extends Exception> {
		void call() throws E;
	}

We'll change the old implementation of `transfer` and create lambdas to match the "shape" of the new functional interface. I'll leave off the type for a moment.

    public void transfer(BankAccount a, BankAccount b, Integer amount) {
		??? debit = () -> a.debit(amount);
		??? credit = () -> b.credit(amount);
	}

Remember from the type inference lecture that Java would be able to see this as a lambda of type `Callable` as it has no parameters and as does `Callable`, it has the same return type (none) and throws an exception of the same type as the interface. We just need to give the compiler a hint, so we can assign this to an instance of a `Callable`.

    public void transfer(BankAccount a, BankAccount b, Integer amount) {
        Callable<InsufficientFundsException> debit = () -> a.debit(amount);
        Callable<InsufficientFundsException> credit = () -> b.credit(amount);
	}

Creating the lambdas like this doesn't cause a compilation error as the functional interface declares that it could be thrown. It doesn't need to warn us at the point we create the lambda, as the signature of the functional method will cause the compiler to error if required when we actually try and call it. Just like a regular method.

If we try and pass them into the `runInSequence` method, we'll get a compiler error though.

	public void transfer(BankAccount a, BankAccount b, Integer amount) {
        Callable<InsufficientFundsException> debit = () -> a.debit(amount);
        Callable<InsufficientFundsException> credit = () -> b.credit(amount);
        runInSequence(debit, credit); <- doesn't compile
	}

The lambdas are of the wrong type. We still need a lambda of type `Runnable`. We'll have to write a method that can convert from a `Callable` to a `Runnable`. At the same time, we'll wrap the checked exception to a runtime one. Something like this.

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

	public void transfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = unchecked(() -> a.debit(amount));
		Runnable credit = unchecked(() -> b.credit(amount));
        runInSequence(debit, credit);
	}

Once we put the exception handling back in we're back to a more concise method body and have dealt with the potential exceptions in the same way as before.

	public void transfer(BankAccount a, BankAccount b, Integer amount) {
		Runnable debit = unchecked(() -> a.debit(amount));
		Runnable credit = unchecked(() -> b.credit(amount));
		try {
			runInSequence(debit, credit);
		} catch (InsufficientFundsRuntimeException e) {
			// check balances and rollback
		}
	}

The downside is this isn't a totally generalised solution; we'd still have to create variations of the `unchecked` method for different functions. We've also just hidden the verbose syntax away. The verbosity is still there its just been moved. Yes, we've got some reuse out of it but if exception handling were transparent or we didn't have checked exceptions, we wouldn't need to brush the issue under the carpet quiet so much.

It's worth pointing out that we'd probably end up doing something similar if we were in Java 7 and using anonymous classes instead of lambdas. A lot of this stuff can still be done pre-Java 8 and you'll end up creating helper methods to push the verbosity to one side.

It's certainly the case that lambdas offer more concise representations for small anonymous pieces of functionality but because of Java's checked exception model, dealing with exceptions in lambdas will often cause all the same verbosity problems we had before.


### As a Caller (Dealing with Exceptions when Calling Lambdas)

We've seen things from the perspective of writing lambdas, now lets have a look at things when calling lambdas.

Let's imagine that now we're writing the library that offers the `runInSequence` method. We have more control this time and aren't limited to using `Runnable` as a lambda type. Because we don't want to force our clients to jump through hoops dealing with exceptions in their lambdas (or wrap them as runtime exceptions), we'll provide a functional interface that declares that a checked exception might be thrown.

We'll call it `FinancialTransfer` with a `transfer` method.

	@FunctionalInterface
	interface FinancialTransfer {
		void transfer() throws InsufficientFundsException;
	}

We're saying that whenever a banking transaction occurs, there's the possibility that insufficient funds are available. Then when we implement our `runInSequence` method, we accept lambdas of this type.

	public static void runInSequence(FinancialTransfer first,
	        FinancialTransfer second) throws InsufficientFundsException {
		first.transfer();
		second.transfer();
	}

This means that when clients use the method, they're not forced to deal with exceptions within their lambdas. For example, writing a method like this.

    // example client usage
	public void transfer(BankAccount a, BankAccount b, Integer amount) {
        FinancialTransfer debit = () -> a.debit(amount);
        FinancialTransfer credit = () -> b.credit(amount);
	}

This time there is no compiler error when creating the lambdas. There's no need to wrap the exceptions from `BankAccount` methods as runtime exceptions; the functional interface has already declared the exception. However, `runInSequence` would now throw a checked exception, so it's explicit that the client has to deal with the possibility and you'll see a compiler error.

	public void transfer(BankAccount a, BankAccount b, Integer amount) {
        FinancialTransfer debit = () -> a.debit(amount);
        FinancialTransfer credit = () -> b.credit(amount);
        runInSequence(debit, credit);  <- compiler error
	}

So we need to wrap the call in a try/catch to make the compiler happy.

	public void transfer(BankAccount a, BankAccount b, Integer amount) {
        FinancialTransfer debit = () -> a.debit(amount);
        FinancialTransfer credit = () -> b.credit(amount);
        try {
            runInSequence(debit, credit);  <- compiler error
        } catch (InsufficientFundsException e) {
            // whatever
        }
	}

The end result is something like we saw previously but without the need for the `unchecked` method. As a library developer, we've made it easier for clients to integrate with our code.


************ got to here, not sure this is right ************

But what about if we try something more exotic? Let's make the `runInSequence` method asynchronous again. There's no need to throw the exception anymore so this version of the `runInSequence` method doesn't include the `throws` clause.

    public static void runInSequence(Runnable first, Runnable second) {
        new Thread(() -> {
            first.transfer();
            second.transfer();
        }).start();
    }



With that gone and compiler errors still in the `runInSequence` method, we need another way to handle the exception. One technique is to pass in a function that will be called in the event of an exception. We can use this lambda to bridge the code running asynchronously back to the caller.

To start with, we'll add the catch block back in and pass in a functional interface to use as the exception handler.

I'll use the `Consumer` interface here, it's new in Java 8 and part of the `java.util.function` package. We then call the interface method in the catch block, passing in the cause. To make that useful, we need to supply a lambda in the `transfer` method.

The parameter of the lambda, i've called it `exception` here, will be whatever is passed into the `accept` method in `runInSequence`. It will be an instance of `InsufficientFundsException` and the client can deal with it however they chose.

Lets pop that in place to finish the example.

There we are. We've provided the client to our library with an alternative exception handling mechanism rather than forcing them to catch exceptions.

We've _internalised_ the exception handling into the library code. It's a good example of deferred execution; should there be an exception, the client doesn't necessarily know when his exception handler would get invoked. For example, as we're running in another thread, the bank accounts themselves may have be altered by the time it executes. Again it highlights that using exceptions to control your program's flow is a flawed approach.

You can't rely on the exception handler being called when it's convenient for you; we've just opened a whole can of concurrency worms.


