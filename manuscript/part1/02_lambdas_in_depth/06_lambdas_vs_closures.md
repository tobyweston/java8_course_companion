
## Lambdas vs Closures

The terms closure and lambda are often used interchangeably but they are actually distinct. In this section we'll take a look at the differences so you can be clear about which is which.

Here's a table showing the release dates for each major version of Java. Java 5.0 came along in 2004 and included the first major language changes including things like generics support.

Around 2008 to 2010 there was a lot of work going on to introduce closures to Java. It was due to go in to Java 7 but didn't quite make it in time. Instead it evolved into lambda support in Java 8. Unfortunately, around that time people used the term "closures" and "lambdas" interchangeably and so it's been a little confusing for the Java community since. In fact, there's still a project page on the OpenJDK site for "closures" and one for "lambdas".

From the OpenJDK project's perspective, they really should have been using "lambda" consistently from the start. In fact, the OpenJDK got it so wrong, they ignored the fact that Java _has had_ closure support since 1.1!

I'm being slightly pedantic here as although there are technical differences between closures and lambdas, the goals of the two projects were to achieve the same thing, even if they used the terminology inconsistently.

So what is the difference between lambdas and closures?

Basically, a closure _is a_ type of lambda.

but a lambda isn't necessarily a closure.


For that to make sense, we need to have a closer look at the differences.

So...

Just like a lambda, a closure is effectively an anonymous block of functionality. But, and it's a big but... there are some important distinctions.

A closure depends on external values (not just it's arguments) whereas a lambda depends only on it's arguments. A closure is said to "close over" the environment it requires.

For example, the following

    (server) -> server.isRunning();

is a lambda, but this

    () -> server.isRunning();

is a closure.

They both return a boolean indicating if some server is up but one uses it's argument and the other must get the variable from somewhere else.

Both are lambdas; in the general sense, they are both anonymous blocks of functionality and in the Java language sense, they both use the new lambda syntax.

The first example refers to a server variable passed into the lambda as an argument whereas the second example (the closure) gets the server variable from somewhere else; ie the environment.

To get the instance of the variable, the lambda has to "close over" the environment or capture the value of `server`. We've seen this in action when we talked about effectively final before.

Let's expand the example so you can see things more clearly.


[demo](lambdas_vs_closures.demo)

Firstly, we'll create a method in a static class to perform a naive poll and wait. It'll check a functional interface on each poll to see if some condition has been met.

    class WaitFor {
        static <T> void waitFor(T input, Predicate<T> predicate) throws InterruptedException {
            while (!predicate.test(input))
                Thread.sleep(250);
        }
    }

We use `Predicate` as our functional interface and test it, pausing for a short while if the condition is not satisfied.

We can call this method with a simple lambda that checks if some HTTP server is running.

    void lambda() throws InterruptedException {
        waitFor(new HttpServer(), (server) -> !server.isRunning());
    }

The `server` parameter is supplied by our `waitFor` method and will be the instance of `HttpServer` we've just defined. It's a lambda as the compiler doesn't need to capture the `server` variable as we supply it manually at runtime.

Incidentally, we might have been able to use a method reference...

    waitFor(new HttpServer(), HttpServer::isRunning); // nowhere do you put the !

but we can't as we can't negate method references. You get a compile error.

Let me just put that back again.


Now, if we re-implement this as a closure, it would look like this.

Firstly, we have to add another `waitFor` method.

	static void waitFor(Condition condition) throws InterruptedException {
		while (!condition.isSatisfied())
			Thread.sleep(250);
	}

This time, with a simpler signature. We pass in a functional interface that requires no parameters. The `Condition` interface has a simple `isSatisfied` method with no argument which implies we'll have to supply any values an implementation might need.

It's already hinting that usages of it may result in closures.

Using it, we'd write something like this.

	void closure() throws InterruptedException {
		Server server = new HttpServer();
		waitFor(() -> !server.isRunning());
	}

The server instance is not passed as a parameter to the lambda here but accessed from the enclosing scope. We've defined the variable and the lambda uses it directly. This variable has to be captured, or copied by the compiler. The lambda "closes over" the `server` variable.

This expression to "close over" comes from the idea that a lambda expression with open bindings (or free variables) have been closed by (or bound in) the lexical environment or scope. The result is a _closed expression_. There are no unbound variables.

[end-demo]


[back to slides]

We've seen a closure being used to provide an anonymous block of functionality and the difference between an equivalent lambda _but_, there are still more useful distinctions we can make.


### Other Differences

For example, an anonymous function, is a function literal without a name, whilst a closure is an instance of a function. By definition, a lambda has no instance variables; it's not an instance. It's variables are supplied as arguments. A closure however, has instance's variables which are provided when the instance is created.

With this in mind, a lambda will generally be more efficient that a closure as it only needs to evaluated once. Once you have the function, you can re-use it. As a closure closes over something not in it’s local environment, it has to be evaluated every time it’s called. An instance has to be newed up each time it's used.

All the issues we looked at in the functions vs classes section are relevant here too. There may be memory considerations to using closures over lambdas.



### Summary


We've talked about a lot here so lets summarise the differences briefly before finishing.

Lambdas are just anonymous functions, similar to static methods in Java. Just like static methods, they can't reference variables outside their scope except for their arguments. A special type of lambda, called a closure, can capture variables outside their scope (or closes over them) so they can use external variables or their arguments.

Closures can be seen as instances of functions. Which is kind of an odd concept for Java developers.

So the simple rule is if a lambda uses a variable from outside it's scope, it's also a closure.

Lambda's aren't always closures, but closures are always lambdas.

A great example is the conventional anonymous class that we would pass around if we didn't have the new lambda syntax. These can "close over" variables and so are themselves closures.

So we've had closure support in Java since 1.1!

Take a look at this example. The `server` variable has to be closed over by the compiler to be used in the anonymous instance of the `Condition` interface. This is both an anonymous class instance and a closure.




### Notes

Break conditions?

Capture semantics?

Effectively final etc. Lambda expressions close over _values_ not _variables_.
