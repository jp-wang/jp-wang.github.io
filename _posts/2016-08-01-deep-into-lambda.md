---
layout: post
title: Deep dive into Java Lambda(0x01)
published: true
---

### What's lambda

(by [@jpwang](https://github.com/jp-wang) and original post is [here](https://github.com/jp-wang/jp-wang.github.io/blob/master/_posts/2016-08-01-deep-into-lambda.md))

**Lambda** is one of most fashion and useful features in [Java 8](http://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.27). A lambda expression is like a method: it provides a list of formal parameters and a body - an expression or block - expressed in terms of those parameters. It is a compile-time error if a lambda expression occurs in a program in someplace other than an assignment context, an invocation context, or a casting context.

### Why Java needs Lambda expressions?

In fact, Java is always comitted to be an object-oriented languange and contributing a lot of classic object-oriented design. With the exception of primitive types, everything in Java is object! Even an array is object. Before the Lambda in Java, there is no way of just defining a function or method which exists in Java all by itself, and also there is no way to pass method as parameter. More or less, what we are currently using annonymous class in Java is the thinking of functional programming although we are still using everythings as object. 

The world is changing everyday! But Java hasn't envolved too much since its beginning if you ignore some of the features like Generic types, Annotations, Auto Boxing and such things. Mostly, Java always remains object as first language. But as a modern programming language, no one can get rid of supporting lambda expression just because it was an object-oriented language! Neither of Java!

If you treat Java as a country with multiple class citizens, Lambda has been discarded as alien for a long time! But now with lambda expressions supported in Java 8, it becomes the first class citizen and adds the missing link of functional programming. That's why I loved Java so much! Its always thinking to change, not just standing there like a puppet.

### How to use?

Usually, writting using syntax like `(argument) -> (body)`. For instance,

`(arg1, arg2) -> {body}` or
`(type arg1, type arg2) -> {body}`.

E.g.,

`(String s) -> System.out.println(s)`

`(int a, int b) -> { return a + b; }`

`() -> 0`

Lets check the structure of Lambda expressions:

* A lambda expression can have zero, one or more parameters.
* The type of the parameters can be explicitly declared or it can be inferred from the context. e.g. (int a) is same as just (a)
* Parameters are enclosed in parentheses and separated by commas. e.g. (a, b) or (int a, int b) or (String a, int b, float c)
* Empty parentheses are used to represent an empty set of parameters. e.g. () -> 42
* When there is a single parameter, if its type is inferred, it is not mandatory to use parentheses. e.g. a -> return a*a
* The body of the lambda expressions can contain zero, one or more statements.
* If body of lambda expression has single statement brackets are not mandatory and the return type of the anonymous function is the same as that of the body expression.
* When there is more than one statement in body than these must be enclosed in brackets (a code block) and the return type of the anonymous function is the same as the type of the value returned within the code block, or void if nothing is returned.

### Functional interface

Java8 provides a lot of functional interfaces like Comparable, Runnable which we are using Lambda expressions based on them. You may notice that functional interface is one special type of interface in Java. In the history of java, there are also some special interfaces and one of the example is `Marker Interface` that we called. For special instance of Serializable and Clonable such interfaces which has empty body and no methods inside.

And Functional interface is another special type of interface which has only one abstract method declared in it. Previously, we use annoymous inner classes to instantiate objects of functional interface. Now with Lambda expressions, this one can be simplified.

[@FunctionalInterface](https://docs.oracle.com/javase/8/docs/api/java/lang/FunctionalInterface.html) is a new Annotation added in Java8 used to indicate that an interface type declaration is intended to be a functional interface. You will get compile error if the interface annotated is not a valid functional interface.

Following is an example of valid customized functional interface.

{% gist jp-wang/3737915838eebbf8168229d7a5092a23 FunctionalInterface.java %}

As we talked before, Functional Interface can only have one abstract method. If you do like below, you will get compile error:

{% gist jp-wang/3737915838eebbf8168229d7a5092a23 FunctionalInterfaceWrong.java %}

After you defined the correct interface, you could use it and take advantage of Lambda Expression like below,

{% gist jp-wang/3737915838eebbf8168229d7a5092a23 MyInterfaceTest.java %}

Output:

> Using annoymous class

> Using lambda expression

Here we created our own Functional interface and used to with lambda expressions. execute() method can now take lambda expressions as argument.
