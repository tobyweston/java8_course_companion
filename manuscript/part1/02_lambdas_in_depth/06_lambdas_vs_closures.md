
## Lambdas vs Closures {#lambdas_vs_closures}

The terms _closure_ and _lambda_ are often used interchangeably but they are actually distinct. In this section we'll take a look at the differences so you can be clear about which is which.

Below is a table showing the release dates for each major version of Java. Java 5.0 came along in 2004 and included the first major language changes including things like generics support.

![](images/timeline.png)

Around 2008 to 2010 there was a lot of work going on to introduce closures to Java. It was due to go in to Java 7 but didn't quite make it in time. Instead it evolved into lambda support in Java 8. Unfortunately, around that time, people used the term 'closures' and 'lambdas' interchangeably and so it's been a little confusing for the Java community since. In fact, there's still a project page on the OpenJDK site for '[closures](http://openjdk.java.net/projects/closures/)' and one for '[lambdas](http://openjdk.java.net/projects/lambda/)'.

From the OpenJDK project's perspective, they really should have been using 'lambda' consistently from the start. In fact, the OpenJDK got it so wrong, they ignored the fact that Java _has had_ closure support since version 1.1!

I'm being slightly pedantic here as although there are technical differences between closures and lambdas, the goals of the two projects were to achieve the same thing, even if they used the terminology inconsistently.

So what is the difference between lambdas and closures? Basically, a closure _is a_ type of lambda but a lambda isn't necessarily a closure.


### Basic Differences

Just like a lambda, a closure is effectively an anonymous block of functionality, but there are some important distinctions. A closure depends on external values (not just it's arguments) whereas a lambda depends only on it's arguments. A closure is said to "close over" the environment it requires.

For example, the following

    (server) -> server.isRunning();

is a lambda, but this

    () -> server.isRunning();

is a closure.

They both return a boolean indicating if some server is up but one uses it's argument and the other must get the variable from somewhere else. Both are lambdas; in the general sense, they are both anonymous blocks of functionality and in the Java language sense, they both use the new lambda syntax.

The first example refers to a server variable passed into the lambda as an argument whereas the second example (the closure) gets the server variable from somewhere else; i.e. the environment. To get the instance of the variable, the lambda has to "close over" the environment or capture the value of `server`. We've seen this in action when we talked about [effectively final](#effectively_final) before.

Let's expand the example to see things more clearly. Firstly, we'll create a method in a static class to perform a naive poll and wait. It'll check a functional interface on each poll to see if some condition has been met.

    class WaitFor {
        static <T> void waitFor(T input, Predicate<T> predicate)
                throws InterruptedException {
            while (!predicate.test(input))
                Thread.sleep(250);
        }
    }

We use `Predicate` (another new `java.util` interface) as our functional interface and test it, pausing for a short while if the condition is not satisfied. We can call this method with a simple lambda that checks if some HTTP server is running.

    void lambda() throws InterruptedException {
        waitFor(new HttpServer(), (server) -> !server.isRunning());
    }

The `server` parameter is supplied by our `waitFor` method and will be the instance of `HttpServer` we've just defined. It's a lambda as the compiler doesn't need to capture the `server` variable as we supply it manually at runtime.


A>
A> Incidentally, we might have been able to use a method reference...
A>
A> ~~~~~~~~~~~~~~~
A>    waitFor(new HttpServer(), HttpServer::isRunning); // nowhere to put the !
A> ~~~~~~~~~~~~~~~
A>
A> but unfortunately we can't negate method references. You'd get a compile error.
A>


Now, if we re-implement this as a closure, it would look like this. Firstly, we have to add another `waitFor` method.

    static void waitFor(Condition condition) throws InterruptedException {
        while (!condition.isSatisfied())
            Thread.sleep(250);
    }

This time, with a simpler signature. We pass in a functional interface that requires no parameters. The `Condition` interface has a simple `isSatisfied` method with no argument which implies that we have to supply any values an implementation might need. It's already hinting that usages of it may result in closures.

Using it, we'd write something like this.

    void closure() throws InterruptedException {
        Server server = new HttpServer();
        waitFor(() -> !server.isRunning());
    }

The server instance is not passed as a parameter to the lambda here but accessed from the enclosing scope. We've defined the variable and the lambda uses it directly. This variable has to be captured, or copied by the compiler. The lambda "closes over" the `server` variable.

This expression to "close over" comes from the idea that a lambda expression with open bindings (or free variables) have been closed by (or bound in) the lexical environment or scope. The result is a _closed expression_. There are no unbound variables. To be more precise, closures close over _values_ not _variables_.

We've seen a closure being used to provide an anonymous block of functionality and the difference between an equivalent lambda but, there are still more useful distinctions we can make.


### Other Differences

An anonymous function, is a function literal without a name, whilst a closure is an instance of a function. By definition, a lambda has no instance variables; it's not an instance. It's variables are supplied as arguments. A closure however, has instance's variables which are provided when the instance is created.

With this in mind, a lambda will generally be more efficient that a closure as it only needs to evaluated once. Once you have the function, you can re-use it. As a closure closes over something not in it’s local environment, it has to be evaluated every time it’s called. An instance has to be newed up each time it's used.

All the issues we looked at in the functions vs classes section are relevant here too. There may be memory considerations to using closures over lambdas.



### Summary


We've talked about a lot here so lets summarise the differences briefly.

Lambdas are just anonymous functions, similar to static methods in Java. Just like static methods, they can't reference variables outside their scope except for their arguments. A special type of lambda, called a closure, can capture variables outside their scope (or close over them) so they can use external variables or their arguments. So the simple rule is if a lambda uses a variable from outside it's scope, it's also a closure.

Closures can be seen as instances of functions. Which is kind of an odd concept for Java developers.

A great example is the conventional anonymous class that we would pass around if we didn't have the new lambda syntax. These can "close over" variables and so are themselves closures. So we've had closure support in Java since 1.1. 

Take a look at this example. The `server` variable has to be closed over by the compiler to be used in the anonymous instance of the `Condition` interface. This is both an anonymous class instance and a closure.

    @since Java 1.1!
    void anonymousClassClosure() {
        Server server = new HttpServer();
        waitFor(new Condition() {
            @Override
            public Boolean isSatisfied() {
                return !server.isRunning();
            }
        });
    }


Lambda's aren't always closures, but closures are always lambdas.
