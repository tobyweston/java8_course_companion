
## λs in Functional Programming

Before we look at things in a more depth, I want to give you some general background to lambdas.

If you haven't seen it before, the Greek letter `λ` ("lambda") is often used as shorthand when talking about lambdas.

In computer science, lambdas go back to the lambda-calculus. A mathematical notation for functions introduced by Alonzo Church in the 1930s. It was a way to explore mathematics using functions and was later re-discovered as a useful tool in computer science.

It formalised the notion of "lambda terms" and rules to transform those terms. These rules or "functions" map directly into modern computer science ideas. All functions in the lambda-calculus are anonymous which again has been taken literally in computer science.

Here's an example of a lambda-calculus expression.

{title="A lambda-calculus expression", lang="text", line-numbers="off"}
~~~~~~~
λx.x+1
~~~~~~~

We define an anonymous function or "lambda" with an argument `x`. The body follows the dot and adds one to that argument.

Then in the 1950s, John McCarthy invented LISP whilst at MIT. This was a programming language designed to model mathematical problems and was heavily influenced by the lambda-calculus.

It used the word `lambda` as an operator to define an anonymous function.

Here's an example.

{title="A LISP expression", lang="text", line-numbers="off"}
~~~~~~~
(lambda (arg) (+ arg 1))
~~~~~~~

This LISP expression evaluates to a function, that when applied will take a single argument, bind it to `arg` and then add `1` to it.

The two expressions produce the same thing, a function to increment a number. You can see the two are very similar.

The lambda-calculus and LISP have had a huge influence on functional programming. The ideas of applying functions and reasoning about problems using functions has moved directly into programming languages. Hence the use of the term in our field. A lambda in the calculus is the same thing as in modern programming languages and is used in the same way.


### What is a Lambda

In simple terms then, a lambda is just an anonymous function. That's it. Nothing special, it's just a compact way to define a function. Anonymous functions are useful when you want to pass around functionality. For example, passing functionality as a argument to a method.

Many main stream languages already support lambdas including Scala, C#, Objective-C, Ruby, C++(11), Python and many others.

{pagebreak}

