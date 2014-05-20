## Scoping

The good news with lambdas is that they don't introduce any new scoping. Using variables within a lambda will refer to variables residing in the enclosing environment.

This is what's called lexical scoping. It means that lambdas don't introduce a new level of scoping at all; you can directly access fields, methods and variables from the enclosing scope. It's also the case for the `this` and `super` keywords. So we don't have to worry about the crazy nested class syntax for resolving scope.

Let's take a look at an example. We have an example class here, with a member variable `i` set to the value of `5`.

    public static class Example {
        int i = 5;

        public Integer example() {
            Supplier<Integer> function = () -> i * 2;
            return function.get();
        }
    }

In the `example` method, a lambda uses a variable called `i` and multiplies it by two.

Because lambdas are lexically scoped, `i` simply refers to the enclosing classes' variable. It's value at run-time will be `5`. Using `this` drives home the point; `this` within a lambda is the same as without.

    public static class Example {
        int i = 5;

        public Integer example() {
            Supplier<Integer> function = () -> this.i * 2;
            return function.get();
        }
    }

In the `anotherExample` method below, a method parameter is used which is also called `i`. The usual shadowing rules kick in here and `i` will refer to the method parameter and not the class member variable. The method variable _shadows_ the class variable. It's value will be whatever is passed into the method.

    public static class Example {
        int i = 5;

        public Integer anotherExample(int i) {
            Supplier<Integer> function = () -> i * 2;
            return function.get();
        }
    }

If you wanted to refer to the class variable `i` and not the parameter `i` from within the body, you could make the variable explicit with `this`. For example, `Supplier<Integer> function = () -> i * 2;`.

The following example has a locally scoped variable defined within the `yetAnotherExample` method. Remember that lambdas use their enclosing scope as their own, so in this case, `i` within the lambda refers to the method's variable. `i` will be `15` and not `5`.

If you want to see this for yourself, you could use a method like the following to print out the values.

    public static void main(String... args) {
		System.out.println("class variable scope   = " + new Example().example());
		System.out.println("method parameter scope = " + new Example().anotherExample(10));
		System.out.println("method scope           = " + new Example().yetAnotherExample());
	}

So, the first method prints `10`; `5` from the class variable multiplied by two. The second method prints `20` as the parameter value was `10` and was multiplied by two and the final method prints `30` as the local method variable was set to `15` and again multiplied by two.


Lexical scoping means deferring to the enclosing environment. Each example had a different enclosing environment or scope. You saw a variable defined as a class member, a method parameter and locally from within a method. In all cases, the lambda behaved consistently and referenced the variable from these enclosing scopes.

Lambda scoping should be intuitive if you're already familiar with basic Java scoping, there's really nothing new here.




## Effectively Final

In Java 7, any variable passed into an anonymous class instance would need to be made final.

This is because the compiler actually copies all the context or _environment_ it needs into the instance of the anonymous class. If those values were to change under it, unexpected side affects could happen. So Java insists that the variable be final to ensure it doesn't change and the inner class can operate on them safely. By safely, I mean without race conditions or visibility problems between threads.

Let's have a look at an example. To start with we'll use Java 7 and create a method called `filter` that takes a list of people and a predicate. We'll create a temporary list to contain any matches we find then enumerate each element testing to see if the predicate holds true for each person. If the test is positive, we'll add them to the temporary list before returning all matches.

    // java 7
	private static List<Person> filter(List<Person> people, Predicate<Person> predicate) {
		ArrayList<Person> matches = new ArrayList<>();
		for (Person person : people)
			if (predicate.test(person))
				matches.add(person);
		return matches;
	}


Then we'll create a method that uses this to find all the people in a list that are eligible for retirement. We set a retirement age variable and then call the filter method with an arbitrary list of people and a new anonymous instance of a `Predicate` interface.

We'll implement this to return true if a person's age is greater than or equal to the retirement age variable.

    public void findRetirees() {
		int retirementAge = 55;
		List<Person> retirees = filter(allPeople, new Predicate<Person>() {
			@Override
			public boolean test(Person person) {
				return person.getAge() >= retirementAge; // <-- compilation error
			}
		});
	}

If you try and compile this, you'll get a compiler failure when accessing the variable. This is because the variable isn't final. We'd need to add `final` to make it compile.

    final int retirementAge = 55;


A>
A> Passing the environment into an anonymous inner class like this is an example of a closure. The environment is what a closure "closes" over; it has to capture the variables it needs to do its job. The Java compiler achieves this using the copy trick rather than try and manage multiple changes to the same variable. In the context of closures, this is called _variable capture_.
A>

Java 8 introduces the idea of "effectively final" which means that if the compiler can work out that a particular variable is _never_ changed, it can be used where ever a final variable would have be used. It interprets it as "effectively" final.

In our example, if we switch to Java 8 and drop the `final` keyword. Things still compile. No need to make the variable final. Java knows that the variable doesn't change so it makes it effectively final.

    int retirementAge = 55;

Of course, it still compiles if you were to make it final.


But how about if we try and modify the variable after we've initialised it?

    int retirementAge = 55;
    // ...
    retirementAge = 65;

The compiler spots the change and can no longer treat the variable as effectively final. We get the original compilation error asking us to make it final. Conversely, if adding the `final` keyword to a variable declaration doesn't cause a compiler error, then the variable is effectively final.


I've been demonstrating the point here with an anonymous class examples because the idea of effectively final isn't something specific to lambdas. It is of course applicable to lambdas though. You can convert this anonymous class above into a lambda and nothing changes. There's still no need to make the variable final.


### Circumventing Final

You can still get round the safety net by passing in final objects or arrays and then change their internals in your lambda.

For example, taking our list of people, lets say we want to sum all their ages. I could create a method to loop and sum like this;

    private static int sumAllAges(List<Person> people) {
        int sum = 0;
        for (Person person : people) {
            sum += person.getAge();
        }
        return sum;
    }

where I maintain a sum count as the list is enumerated.

or I could try and abstract the looping behaviour and pass in a function to be applied to each element. Like this.

   public final static Integer forEach(List<Person> people, Function<Integer, Integer> function) {
        Integer result = null;
        for (Person t : people) {
            result = function.apply(t.getAge());
        }
        return result;
    }

and to achieve the summing behaviour, all I'd need to do is create a function that can sum. I could do this using an anonymous class like this;

    private static void badExample() {
        Function<Integer, Integer> sum = new Function<Integer, Integer>() {
            private Integer sum = 0;

            @Override
            public Integer apply(Integer amount) {
                sum += amount;
                return sum;
            }
        };
    }

Where the functions method takes an integer and returns an integer. In the implementation, the `sum` variable is a class member variable and is mutated each time the function is applied. This kind of mutation is generally bad form when it comes to functional programming.

Nether the less, we can pass this into our `forEach` method like this;

    forEach(allPeople, sum);

and we'd get the sum of all peoples ages. This works because we're using the same instance of the function so the `sum` variable is reused and mutated during each iteration.

The bad news is that we can't convert this into a lambda directly; there's no equivalent to a member variable with lambdas, so there's nowhere to put the `sum` variable other than outside of the lambda.

    double sum = 0;
    forEach(allPeople, x -> {
        return sum += x;
    });

but this highlights that the variable isn't effectively final (it's changed in the lambda's body) and so it must be made final.

But if we make it final

    final double sum = 0;
    forEach(allPeople, x -> {
        return sum += x;
    });

we can no longer modify it in the body! It's a chicken and egg situation.


The trick around this is to use a object or an array; it's reference can remain final but it's internals can be modified

    int[] sum = {0};
    forEach(allPeople, x -> sum[0] += x);


The array reference is indeed final here, but we can modify the array contents without reassigning the reference. However, this is generally bad form as it opens up to all the safety issues we talked about earlier. I wanted to mention it for illustration purposes but I don't recommend you do this kind of thing often. It's generally better not to create functions with side affects and you can avoid the issues completely if you use a more functional approach.

The idiomatic way to do this kind of summing is to use what's called a "fold" or in the Java vernacular "reduce". We'll be looking at this in more detail when we look at streams and the 'java.util.function Package'.
