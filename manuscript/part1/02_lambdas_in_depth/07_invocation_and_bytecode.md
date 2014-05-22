## Invocation & Bytecode

In this section we'll explore how the compiler output differs when you compile anonymous classes to when you compile lambdas. First we'll remind ourselves about java bytecode and how to read it. Then we'll look at both anonymous classes and lambdas when they capture variables and when they don't. We'll compare pre-Java 8 closures with lambdas and explore how lambdas are _not_ just syntactic sugar but produce very different bytecode from the traditional approaches.


### Bytecode Recap

To start with, let's recap on what we know about bytecode.

To get from source code to machine runnable code. The Java compiler produces bytecode. This is either interpreted by the JVM or re-compiled by the Just-in-time compiler.

When it's interpreted, the bytecode is turned into machine code on the fly and executed. This happens each time the bytecode is encountered but he JVM.

When it's Just-in-time compiled, the JVM compiles it directly into machine code the first time it's encountered and then goes on to execute it.

Both happen at run-time but Just-in-time compilation offer lots of optimisations.

So, Java bytecode is the intermediate representation between source code and machine code.

A>
A> As a quick side bar: Java's JIT compiler has enjoyed a great reputation over the years. But going back full circle to our introduction, it was John McCarthy that first wrote about JIT compilation way back in 1960. So it's interesting to think that it's not just lambda support that was influenced by LISP. ([Aycock 2003, 2. JIT Compilation Techniques, 2.1 Genesis, p. 98](http://user.it.uu.se/~kostis/Teaching/KT2-04/jit_survey.pdf)).
A>

The bytecode is the instruction set of the JVM. As it's name suggests, bytecode consists of single-byte instructions (called opcodes) along with associated bytes for parameters. There are therefore a possible 256 opcodes available although only about 200 are actually used.

The JVM uses a [stack based computation model](http://en.wikipedia.org/wiki/Model_of_computation), if we want to increment a number, we have to do it using the stack. All instructions or opcodes work against the stack.

So for example, `5 + 1` becomes `5 1 +`  where `5` is pushed to the stack,

{lang="text"}
                    +---------+
    push 5  ----->  |    5    |
                    +---------+
                    |         |
                    +---------+
                    |         |
                    +---------+

`1` is pushed then...

{lang="text"}
                    +---------+
    push 1  ----->  |    1    |
                    +---------+
                    |    5    |
                    +---------+
                    |         |
                    +---------+

the `+` operator is applied. Plus would pop the top two frames, add the numbers together and push the result back onto the stack. The result would look like this.

{lang="text"}
                    +---------+
    add     ----->  |    6    |
                    +---------+
                    |         |
                    +---------+
                    |         |
                    +---------+


Each opcode works against the stack like this so we can translate our example into a sequence of bytecodes;

{lang="text"}
                    +---------+
    push 5  ----->  |    5    |
                    +---------+
                    |         |
                    +---------+
                    |         |
                    +---------+

`push` 5 becomes `iconst_5`.

{lang="text"}
                      +---------+
    iconst_5  ----->  |    5    |
                      +---------+
                      |         |
                      +---------+
                      |         |
                      +---------+

push 1 becomes `iconst_1`

{lang="text"}
                      +---------+
    iconst_1  ----->  |    1    |
                      +---------+
                      |    5    |
                      +---------+
                      |         |
                      +---------+

and `add` becomes `iadd`.

{lang="text"}
                      +---------+
    iadd      ----->  |    6    |
                      +---------+
                      |         |
                      +---------+
                      |         |
                      +---------+

`iconst_x` and `iadd` are examples of opcodes. Opcodes often have prefixes and/or suffices to indicate the types they work on, `i` in these examples refers to `integer`.

We can group the opcodes into the following categories.

| Group                                       | Examples                              |
|---------------------------------------------|---------------------------------------|
| Stack manipulation                          | `aload_n`, `istore`, `swap`, `dup2`
| Control flow instructions                   | `goto`, `ifeq`, `iflt`
| Object interactions                         | `new`, `invokespecial`, `areturn`
| Arithmetic, logic and type conversion       | `iadd`, `fcmpl`, `i2b`

Instructions concerned with stack manipulation, like we've seen before. Examples being `aload`, `istore` etc. To control program flow with things like if and while, we use opcodes like `goto` and `if equal`. Creating objects and accessing methods use codes like `new` and `invokespecial`. We'll be particularly interested in this group when we look at the different opcodes used to invoke lambdas. The last group is about arithmetic, logic and type conversion and includes codes like `iadd`, float compare long (fcmpl) and integer to byte (`i2b`).



### Descriptors

Opcodes will often use parameters, these look a little cryptic in the bytecode as they usually referenced via lookup tables. Internally, Java uses what's called "descriptors" to describe these parameters.

They describe types and signatures using a specific grammar you'll see throughout the bytecode. You'll often see the same grammar used in compiler or debug output, so it's useful to recap it here.

Here's an example of a method signature descriptor.

    Example$1."<init>":(Lcom/foo/Example;Lcom/foo/Server;)V

It's describing the constructor of a class called `$1`, which we happen to know is the JVM's name for the first anonymous class instance within another class. In this case `Example`. So we've got a constructor of an anonymous class that takes two parameters, an instance of the outer class `com.foo.Example` and an instance of `com.foo.Server`.

Being a constructor, the method doesn't return anything. The `V` symbol represents void.

Have a look at breakdown of the descriptor syntax below. If you see an uppercase `Z` in a descriptor, it's referring to a `boolean`, an uppercase `B` a `byte` and so on.

![](images/descriptors.png)

A couple of ones to mention;

 * classes are described with an uppercase `L` followed by the fully qualified class name, followed by a semi-colon. The class name is separated with slashes rather than the dots.
 * and arrays are described using an opening square bracket followed by a type from the list. No closing bracket.

{pagebreak}

### Converting a Method Signature

Let's take the following method signature and turn it into a method descriptor.

{lang="java"}
    long f (int n, String s, int[] array);


The method returns a long, so we describe the fact that it is a method with brackets and that it returns a long with a uppercase `J`.

{lang="text"}
    ()J

The first argument is of type int, so we use an uppercase `I`.

{lang="text"}
    (I)J

The next argument is an object, so we use `L` to describe it's an object, fully qualify the name and close it with a semi-colon.

{lang="text"}
    (ILString;)J

The last argument is an integer array so we drop in the array syntax followed by int type.

{lang="text"}
    (ILString;[I)J


and we're done. A JVM method descriptor.


{pagebreak}

### Code Examples

Lets have a look at the bytecode produced for some examples.

We're going to look at the bytecode for four distinct blocks of functionality based on the example we looked at in [lambdas vs closures](#lambdas_vs_closures) section.

We'll explore

1. A simple anonymous class
1. An anonymous class closing over some variable (an old style closure)
1. A lambda with no arguments
1. A lambda with arguments
1. A lambda closing over some variable (a new style closure)

The example bytecode was generated using the `javap` command line tool. Only partial bytecode listings are shown in this section, for full source and bytecode listsings, see [Appendix A](#appendix_a). Also, fully qualified class names have been shortened to better fit on the page.

### Example 1


The first example is a simple anonymous class instance passed into our `waitFor` method.

    public class Example1 {
        // anonymous class
        void example() throws InterruptedException {
            waitFor(new Condition() {
                @Override
                public Boolean isSatisfied() {
                    return true;
                }
            });
        }
    }

If we look at the bytecode, the thing to notice is that an instance of the anonymous class is newed up at line 6. The #2 refers to a lookup, the result of which is shown in the comment. So it uses the `new` opcode with whatever is at #2 in the constant pool, this happens to be the anonymous class `Example$1`.

{lang="java", line-numbers="on"}
    void example() throws java.lang.InterruptedException;
        descriptor: ()V
        flags:
        Code:
          stack=3, locals=1, args_size=1
             0: new           #2  // class Example1$1
             3: dup
             4: aload_0
             5: invokespecial #3  // Method Example1$1."<init>":(LExample1;)V
             8: invokestatic  #4  // Method WaitFor.waitFor:(LCondition;)V
            11: return
          LineNumberTable:
            line 10: 0
            line 16: 11
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                0      12     0  this   LExample1;
        Exceptions:
          throws java.lang.InterruptedException


Once created, the constructor is called using `invokespecial` on line 9. This opcode is used to call constructor methods, private methods and accessible methods of a super class. You might notice the method descriptor includes a reference to `Example1`. All anonymous class instances have this implicit reference to the parent class.

The next step uses 'invokestatic' to call our `waitFor` method passing in the anonymous class on line 10. `invokestatic` is used to call static methods and is very fast as it can direct dial a method rather than figure out which to call as would be the case in an object hierarchy.


### Example 2

Example 2 is another anonymous class but this time it closes over the `server` variable. It's an old style closure.

    public class Example2 {
        // anonymous class (closure)
        void example() throws InterruptedException {
            Server server = new HttpServer();
            waitFor(new Condition() {
                @Override
                public Boolean isSatisfied() {
                    return !server.isRunning();
                }
            });
        }
    }


The bytecode is similar to the previous except that an instance of the `Server` class is newed up (at line 3.) and it's constructor called at line 5. The instance of the anonymous class `$1` is still constructed with `invokespecial` (at line 11.) but this time it takes the instance of `Server` as an argument as well as the instance of the calling class.

To close over the `server` variable, it's passed directly into the anonymous class.

{lang="java", line-numbers="on"}
    void example() throws java.lang.InterruptedException;
        Code:
           0: new           #2  // class Server$HttpServer
           3: dup
           4: invokespecial #3  // Method Server$HttpServer."<init>":()V
           7: astore_1
           8: new           #4  // class Example2$1
          11: dup
          12: aload_0
          13: aload_1
          14: invokespecial #5  // Method Example2$1."<init>":(LExample2;LServer;)V
          17: invokestatic  #6  // Method WaitFor.waitFor:(LCondition;)V
          20: return


### Example 3

Example 3 uses a Java 8 lambda with our `waitFor` method. The lambda doesn't do anything other than return `true`. It's equivalent to example 1.

    public class Example3 {
        // simple lambda
        void example() throws InterruptedException {
            waitFor(() -> true);
        }
    }

The bytecode is super simple this time. It uses the `invokedynamic` opcode to create the lambda at line 3. which is then passed to the `invokestatic` opcode on the next line.

{lang="java", line-numbers="on"}
    void example() throws java.lang.InterruptedException;
        Code:
           0: invokedynamic #2,  0   // InvokeDynamic #0:isSatisfied:()LCondition;
           5: invokestatic  #3       // Method WaitFor.waitFor:(LCondition;)V
           8: return

The descriptor for the `invokedynamic` call is targeting the `isSatisfied` method on the `Condition` interface (line 3.).

What we're not seeing here is the mechanics of `invokedynamic`. `invokedynamic` is a new opcode to Java 7, it was intended to provide better support for dynamic languages on the JVM. It does this by not linking the types to methods until run-time. The other "invoke" opcodes all resolve types at compile time.

For lambdas, this means that placeholder method invocations can be put into the bytecode like we've just seen and working out the implementation can be done on the JVM at runtime.

If we look at a more verbose bytecode that includes the constant pool we can dereference the lookups. For example, if we look up number 2, we can see it references #0 and #26.

{lang="java", line-numbers="on"}
    Constant pool:
       #1 = Methodref          #6.#21    //  Object."<init>":()V
       #2 = InvokeDynamic      #0:#26    //  #0:isSatisfied:()LCondition;
       ...
    BootstrapMethods:
        0: #23 invokestatic LambdaMetafactory.metafactory:
                (LMethodHandles$Lookup;LString;
                LMethodType;LMethodType;
                LMethodHandle;LMethodType;)LCallSite;
          Method arguments:
            #24 ()LBoolean;
            #25 invokestatic Example3.lambda$example$25:()LBoolean;
            #24 ()LBoolean;

The constant 0 is in a special lookup table for bootstrapping methods (line 6.). It refers to a static method call to the JDK `LambdaMetafactory` to create the lambda. This is where the heavy lifting goes on. All the target type inference to adapt types and any partial argument evaluation goes on here.

The actual lambda is shown as a method handle called `lambda$example$25` (line 12.) with no arguments, returning a boolean. It's invoked using `invokestatic` which shows that it's accessed as a genuine function; there's no object associated with it. There's also no implicit reference to a containing class unlike the anonymous examples before.

It's passed into the `LambdaMetafactory` and we know it's a method handle by looking it up in the constant pool. The number of the lambda is compiler assigned and just increments from zero for each lambda required.

{lang="java", line-numbers="on"}
    Constant pool:
        // invokestatic Example3.lambda$example$25:()LBoolean;
        #25 = MethodHandle   #6:#35


{pagebreak}

### Example 4

Example 4 is another lambda but this time it takes an instance of `Server` as an argument. It's equivalent in functionality to example 2 but it doesn't close over the variable; it's not a closure.

    public class Example4 {
        // lambda with arguments
        void example() throws InterruptedException {
            waitFor(new HttpServer(), (server) -> server.isRunning());
        }
    }

Just like example 2, the bytecode has to create the instance of server but this time, the `invokedynamic` opcode references the `test` method of type `Predicate`. If we were to follow the reference (#4) to the boostrap methods table, we would see the actual lambda requires an argument of type `HttpServer` and returns a `Z` which is a primitive boolean.

{lang="java", line-numbers="on"}
    void example() throws java.lang.InterruptedException;
        descriptor: ()V
        flags: 
        Code:
          stack=2, locals=1, args_size=1
             0: new           #2     // class Server$HttpServer
             3: dup           
             4: invokespecial #3     // Method Server$HttpServer."<init>":()V
             7: invokedynamic #4, 0  // InvokeDynamic #0:test:()LPredicate;
            12: invokestatic  #5     // Method WaitFor.waitFor:(LObject;LPredicate;)V
            15: return        
          LineNumberTable:
            line 13: 0
            line 15: 15
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                0      16     0  this   LExample4;
        Exceptions:
          throws java.lang.InterruptedException

So the call to the lambda is still a static method call like before but this time takes the variable as a parameter when it's invoked.


{pagebreak}

### Example 4 (with method reference)

Interestingly, if we use a method reference instead, the functionality is exactly the same but we get different bytecode.

    public class Example4_method_reference {
        // lambda with method reference
        void example() throws InterruptedException {
            waitFor(new HttpServer(), HttpServer::isRunning);
        }
    }

Via the call to the `LambdaMetafactory` when the final execution occurs, the method reference results in a call to `invokevirtual` rather than `invokestatic`. `invokevirtual` is used to call public, protected an package protected methods so it implies an instance is required. The instance is supplied to the `metafactory` method and no lambda (or static function) is needed at all; there are no `lambda$` in this bytecode.

{lang="java", line-numbers="on"}
    void example() throws java.lang.InterruptedException;
        descriptor: ()V
        flags: 
        Code:
          stack=2, locals=1, args_size=1
             0: new           #2     // class Server$HttpServer
             3: dup           
             4: invokespecial #3     // Method Server$HttpServer."<init>":()V
             7: invokedynamic #4, 0  // InvokeDynamic #0:test:()LPredicate;
            12: invokestatic  #5     // Method WaitFor.waitFor:(LObject;LPredicate;)V
            15: return        
          LineNumberTable:
            line 11: 0
            line 12: 15
          LocalVariableTable:
            Start  Length  Slot  Name   Signature
                0      16     0  this   LExample4_method_reference;
        Exceptions:
          throws java.lang.InterruptedException


### Example 5

Lastly, example 5 uses a lambda but closes over the `server` instance. It's equivalent to example 2 and is a new style closure.

    public class Example5 {
        // closure
        void example() throws InterruptedException {
            Server server = new HttpServer();
            waitFor(() -> !server.isRunning());
        }
    }


It goes through the basics in the same way as the other lambdas but if we lookup the `metafactory` method in the bootstrap methods table, you'll notice that this time, the lambda's method handle has an argument of type `Server`. It's invoked using `invokestatic` (line 9.) and the variable is passed directly into the lambda at invocation time.

{lang="java", line-numbers="on"}
    BootstrapMethods:
        0: #34 invokestatic LambdaMetafactory.metafactory:
                (LMethodHandles$Lookup;
                 LString;LMethodType;
                 LMethodType;
                 LMethodHandle;LMethodType;)LCallSite;
          Method arguments:
            #35 ()LBoolean; // <-- SAM method to be implemented by the lambda
            #36 invokestatic Example5.lambda$example$35:(LServer;)LBoolean;
            #35 ()LBoolean; // <-- type to be enforced at invocation time


So like the anonymous class in example 2, an argument is added by the compiler to capture the term although this time, it's a method argument rather than a constructor argument.


{pagebreak}

### Summary

We saw how using an anonymous class will create a new instance and call it's constructor with `invokespecial`.

We saw anonymous classes that close over variables have an extra argument on their constructor to capture that variable.

and we saw how Java 8 lambdas use the `invokedynamic` instruction to defer binding of the types and that a special "lambda$" method handle is used to actually represent the lambda. This method handle has no arguments in this case and is invoked using `invokestatic` making it a genuine function.

The lambda was created by the `LambdaMetafactory` class which itself was the target of the `invokedynamic` call.

When a lambda has arguments, we saw how the `LambdaMetafactory` describes the argument to be passed into the lambda. `invokestatic` is used to execute the lambda like before. But we also had a look at a method reference used in-lieu of a lambda. In this case, no `lambda$` method handle was created and `invokevirtual` was used to call the method directly.

Lastly, we looked at a lambda that closes over a variable. This one creates an argument on the `lambda$` method handle and again is called with `invokestatic`.

