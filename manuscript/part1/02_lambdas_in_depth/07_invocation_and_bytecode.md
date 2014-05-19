## Invocation & Bytecode

In this section we'll explore how the compiler output differs when you compile anonymous classes to when you compile lambdas.

First we'll remind ourselves about java bytecode and how to read it.

Then we'll look at both anonymous classes and lambdas when they capture variables and when they don't.

We'll compare pre-Java 8 closures with lambdas and explore how lambdas are _not_ just syntactic sugar but produce very different bytecode from the traditional approaches.


### Bytecode Recap

To start with then, let's recap on what we know about bytecode.

To get from source code to machine runnable code. The Java compiler produces bytecode. This is either interpreted by the JVM or re-compiled by the Just-in-time compiler.

When it's interpreted, the bytecode is turned into machine code on the fly and executed. This happens each time the bytecode is encountered but he JVM.

When it's Just-in-time compiled, the JVM compiles it directly into machine code the first time it's encountered and then goes on to execute it.

Both happen at run-time but Just-in-time compilation offer lots of optimisations.

So, Java bytecode is the intermediate representation between source code and machine code.

As a quick side bar: Java's JIT compiler has enjoyed a great reputation over the years. But going back full circle to our introduction, it was John McCarthy that first wrote about JIT compilation way back in 1960. So it's interesting to think that it's not just lambda support that was influenced by LISP. ([Aycock 2003, 2. JIT Compilation Techniques, 2.1 Genesis, p. 98](http://user.it.uu.se/~kostis/Teaching/KT2-04/jit_survey.pdf)).


Getting back to bytecode...

bytecode is the instruction set of the JVM. As it's name suggests, bytecode consists of single-byte instructions (called opcodes) along with associated bytes for parameters. There are therefore a possible 256 opcodes available although only about 200 are actually used.

The JVM uses a [stack based computation model](http://en.wikipedia.org/wiki/Model_of_computation), if we want to increment a number, we have to do it using the stack. All instructions or opcodes work against the stack.

So for example, `5 + 1` becomes `5 1 +`

[slide]

 where `5` is pushed to the stack,

[slide]

`1` is pushed then...

 [slide]

the `+` operator is applied. Plus would pop the top two frames, add the numbers together and push the result back onto the stack.

The result would look like this.



Each opcode works against the stack like this so we can translate our example into a sequence of bytecodes

push 5 becomes `iconst_5` [slide] push 1 becomes `iconst_1` [slide] and `add` becomes `iadd`

`iconst_x` and `iadd` are examples of opcodes. Opcodes often have prefixes and/or suffices to indicate the types they work on, `i` in these examples refers to `integer`.


We can group the opcodes into the following categories.

Instructions concerned with stack manipulation, like we've seen before. Examples being `aload`, `istore` etc.

To control program flow with things like if and while, we use opcodes like `goto` and `if equal`.

Creating objects and accessing methods use codes like `new` and `invokespecial`. We'll be particularly interested in this group when we look at the different opcodes used to invoke lambdas.

The last group is about arithmetic, logic and type conversion and includes codes like `iadd`, float compare long (fcmpl) and integer to byte (`i2b`).


| Group                                       | Examples                              |
|---------------------------------------------|---------------------------------------|
| Stack manipulation                          | `aload_n`, `istore`, `swap`, `dup2`
| Control flow instructions                   | `goto`, `ifeq`, `iflt`
| Object interactions                         | `new`, `invokespecial`, `areturn`
| Arithmetic, logic and type conversion       | `iadd`, `fcmpl`, `i2b`


### Descriptors

Opcodes will often use parameters, these look a little cryptic in the bytecode as they usually referenced via lookup tables. Internally, Java uses what's called "descriptors" to describe these.

They describe types and signatures using a specific grammar you'll see throughout the bytecode. You'll often see the same grammar used in compiler or debug output, so it's useful to recap it here.

Here's an example of a method signature descriptor.

It's describing the constructor of a class called `$1`, which we happen to know is the JVM's name for the first anonymous class instance within another class. In this case `Example`.

So we've got a constructor of an anonymous class that takes two parameters, an instance of the outer class `com.foo.Example` and an instance of `com.foo.Server`.

Being a constructor, the method doesn't return anything. The `V` symbol represents void.


Have a look at breakdown of the descriptor syntax and we'll work through an example.

If you see an uppercase `Z` in a descriptor, it's referring to a `boolean`, an uppercase `B` a `byte` and so on.

A couple of ones to mention;

 * classes are described with an uppercase `L` followed by the fully qualified class name, followed by a semi-colon. The class name is seperated with slashes rather than the dots.
 * and arrays are described using an opening square bracket followed by a type from the list. No closing bracket.


Let's take this method and turn it into a method descriptor.

The method returns a long, so we describe the fact that it is a method with brackets and that it returns a long with a uppercase `J`.

The first argument is of type int, so we use an uppercase `I`.

The next argument is an object, so we use `L` to describe it's an object, fully qualify the name and close it with a semi-colon.

The last argument is an integer array so we drop in the array syntax followed by int type.

and we're done. A JVM method descriptor.







### Static vs Dynamic Typing

Most of the JVM invocation opcodes are statically typed; they check the method signature types at compile time.

Most of the existing JVM instruction set is statically typed - in the sense that method calls have their signatures type-checked at compile time, without a mechanism to defer this decision to run time, or to choose the method dispatch by an alternative approach.[5]

JSR 292 (Supporting Dynamically Typed Languages on the Java™ Platform)[6] added a new invokedynamic instruction at the JVM level, to allow method invocation relying on dynamic type checking (instead of the existing statically type-checked invokevirtual instruction).



## invokedynamic bytecode instruction (optimisation, construct the lambda at runtime)

 * `invokestatic` is used to call static methods of a class
 * `invokespecial` is used to call constructors, private methods and accessible methods of a super class
 * `invokevirtual` is used to cal public, protected an package protected method
`invokeinterface` is used when the method being called belongs to an interface
`invokedynamic` is used ...







## Code Examples

Right, so we've had a refresher on bytecode, now lets have a look at the bytecode produced for some examples.

We're going to look at the bytecode for four distinct blocks of functionality based on the example we looked at in lambdas vs closures section.

We'll explore

1. A simple anonymous class
1. An anonymous class closing over some variable (an old style closure)
1. A lambda with no arguments
1. A lambda with arguments
1. A lambda closing over some variable (a new style closure)


### Example 1

Here's the first example. I've got the Java code in the top half of the window and the generated bytecode in the bottom. Incidentally, I generated this using the `javap` command line tool.

The first example is a simple anonymous class instance passed into our `waitFor` method. If we look at the bytecode, we can see two main blocks, the first is the default constructor that the compiler created. We're not interested in that. The second block is the bytecode for the `example` method. We are interested in that.

The thing to note is that an instance of the anonymous class is new'ed up here. The #2 refers to a lookup, the result of which is shown in the comment. So it uses the `new` opcode with whatever is at #2 in the constant pool, this happens to be the class anonymous class.

Once created, the constructor is called using `invokespecial`. This opcode is used to call constructor methods, private methods and accessible methods of a super class. You might notice the method descriptor includes a reference to `Example1`. All anonymous class instances have this implicit reference to the parent class.

The next step uses 'invokestatic' to call our `waitFor` method passing in the anonymous class. `invokestatic` is used to call static methods and is very fast as it can direct dial a method rather than figure out which to call as would be the case in an object hierarchy.


### Example 2

Example 2 is another anonymous class but this time it closes over the `server` variable. It's an old style closure.

The bytecode is similar to the previous except that an instance of the `Server` class is new'ed up and it's constructor called. The instance of the anonymous class `$1` is still constructed ut this time it takes the instance of `Server` as an argument as well as the instance of the calling class.

To close over the `server` variable, it's passed directly into the anonymous class.


### Example 3

Example 3 uses a Java 8 lambda with our `waitFor` method. The lambda doesn't do anything other than return `true`. It's equivalent to example 1.

The bytecode is super simple this time. It uses the `invokedynamic` opcode to create the lambda which is then passed to the `invokestatic` opcode on the next line.

The descriptor for the `invokedynamic` call is targeting the `isSatisfied` method on the `Condition` interface. We'll talk about the #0 in a moment.

What we're not seeing here is the mechanics of `invokedynamic`. `invokedynamic` is a new opcode to Java 7, it was intended to provide better support for dynamic languages on the JVM. It does this by not linking the types to methods until run-time.

The other invoke opcodes all resolve types at compile time.

For lambdas, this means that placeholder method invocations can be put into the bytecode like we've just seen and working out the implementation can be done on the JVM at runtime.

If we look at a more verbose bytecode that includes the constant pool we can dereference the lookups. For example, if we look up number 2, we can see it references #0 and #26.

The constant 0 is in a special lookup table for bootstrapping methods. It refers to a static method call to the JDK `LambdaMetafactory` to create the lambda. This is where the heavy lifting goes on. All the target type inference to adapt types and any partial argument evaluation goes on here.

The actual lambda is shown as a method handle called `lambda$example$25` with no arguments, returning a boolean. It's invoked using `invokestatic` which shows that it's accessed as a genuine function; there's no object associated with it. There's also no implicit reference to a containing class unlike the anonymous examples before.

It's passed into the `LambdaMetafactory` and we know it's a method handle by looking it up in the constant pool. The number of the lambda is compiler assigned and just increments from zero for each lambda required.


### Example 4

Example 4 is another lambda but this time it takes an instance of `Server` as an argument. It's equivalent in functionality to example 2 but it doesn't close over the variable; it's not a closure.

Just like example 2, the bytecode has to create the instance of server but this time, the `invokedynamic` opcode references the `test` method of type `Predicate`. If we follow the reference (#4) to the boostrap methods table, we see the actual lambda requires an argument of type `HttpServer` and returns a `Z` which is a primitive boolean.

So the call to the lambda is still a static method call like before but this time takes the variable as a parameter when it's invoked.


### Example 4 (with method reference)

Interestingly, if we use a method reference instead, the functionality is exactly the same but we get different bytecode.

Via the call to the `LambdaMetafactory` when the final execution occurs, the method reference results in a call to `invokevirtual` rather than `invokestatic`. `invokevirtual` is used to call public, protected an package protected methods so it implies an instance is required. The instance is supplied to the `metafactory` method and no lambda (or static function) is needed at all; there are no `lambda$` in this bytecode.


### Example 5

Lastly, example 5 uses a lambda but closes over the `server` instance. It's equivalent to example 2 and is a new style closure.

It goes through the basics in the same way as the other lambdas but if we lookup the `metafactory` method in the bootstrap methods table, you'll notice that this time, the lambda's method handle has an argument of type `Server`. It's invoked using `invokestatic` and the variable is passed directly into the lambda at invocation time.

So like the anonymous class in example 2, an argument is added by the compiler to capture the term although this time, it's a method argument rather than a constructor argument.

## Summary

So, we saw how using an anonymous class will create a new instance and call it's constructor with `invokespecial`.

We saw anonymous classes that close over variables have an extra argument on their constructor to capture that variable.

and we saw how Java 8 lambdas use the `invokedynamic` instruction to defer binding of the types and that a special "lambda$" method handle is used to actually represent the lambda. This method handle has no arguments in this case and is invoked using `invokestatic` making it a genuine function.

The lambda was created by the `LambdaMetafactory` class which itself was the target of the `invokedynamic` call.

When a lambda has arguments, we saw how the `LambdaMetafactory` describes the argument to be passed into the lambda. `invokestatic` is used to execute the lambda like before. But we also had a look at a method reference used in-lieu of a lambda. In this case, no `lambda$` method handle was created and `invokevirtual` was used to call the method directly.

Lastly, we looked at a lambda that closes over a variable. This one creates an argument on the `lambda$` method handle and again is called with `invokestatic`.

## Notes

See [Zero Turnaournd](http://zeroturnaround.com/rebellabs/java-8-the-first-taste-of-lambdas/) post.

[Takipi article](http://www.takipiblog.com/2014/01/16/compiling-lambda-expressions-scala-vs-java-8/) talks about dynamically linking the call site to the actual lambda function. Invoke dynamic allows languages (originally intended for dynamic languages) to bind symbols at run-time rather than do all the linkage statically when the code is compiled.

Usually there's an implicit parameter to anonymous functions; `this`. Lambdas are defined as static functions and so avoid the `this` parameter.