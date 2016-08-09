---
layout: post
title: Deep dive into Java Lambda(0x02)
published: true
---
# Into the implementation

(by [@jpwang](https://github.com/jp-wang) and original post is [here](https://github.com/jp-wang/jp-wang.github.io/blob/master/_posts/2016-08-02-deep-into-lambda2.md))

In last chapter, we talked a lot about how to use [Lambda Expressions](https://jp-wang.github.io/deep-into-lambda/) and implement custom defined [Functional Interface](https://jp-wang.github.io/deep-into-lambda/). And you may want to know more about it. This is what's the current chapter trying to do! By using a very simple demo and deeping into the Java byte code, let's find out how does JVM finally support it and what kind of magic is playing inside. Let's begin!

### [Annoymous inner class](https://docs.oracle.com/javase/tutorial/java/javaOO/anonymousclasses.html)

Before we start talking about how the Java compiler implements the lambda expressions and how the Java Virtual Machine(JVM) deals with them, if you are asked to use your way to implement it, one of possible solution you may choose is Java's sugar - Anonymous Inner Class. Yes or not! I won't tell if it is correct or wrong. Let's firstly see how to use anonymous class:

{% gist jp-wang/0ba691314d7f452cb1015b7ab9a227fe Lambda.java %}

See the byte code after it gets compiled:

Two classes were generated out - `Lambda.class` and `Lambda$1.class`

`javap -v -p -s -sysinfo -constants Lambda.class `

{% gist jp-wang/28b144e1d166282e92e7f26371fd84ce Lambda.class %}

From the output, you will understand why we call the anonymous class is a sugar syntax provided by Java: because in JVM, it only recorgnize classes and objects(or instances). There is no kind of `inner class` concept in JVM. Even the programmer like you doesn't give any name of that when we get instance by `new Printer<String> {}` in source code, but the JVM will automatically create a class and give a special name for it, such as `Lambda$0`.

### Why the anonymous class unsatisfactory?

First, as you saw above, the Compiler will automatically generates a new class file for each anonymous inner class. So if the lambda expressions were translated into anonymous inner class, you'd have on new class file for each lambda expression. And also when you are trying to use it, each generated anonymous class has to been loaded in, verify, and link before it's used. The whole process takes space, memory and time more and more if you more lambda expressions added in(classes byecode are stored into method area which as perminate memory in Java Memory Mode).

Secondly, most importantly, choosing to implement the lambda expression using anonymous inner class will limit the scope of future lambda expression changes since it has deep coupling with mechanism of anonymous inner class. And it will be tied so deeply with the byte code generation and not easily envolve in future.

### The truth of Lambda

To avoid the concerns explained in previous sessions, the java languange and JVM engineers desice to defer the translation until run time by using special bytecode instruction - invokedynamic introduced in Java 7.

Lets see how it works!
1. Generate an invokedynamic call point, which returns an instance of functional interface to which the lambda is being converted
2. Convert the body of lambda into the inner method which will be invoked by the invokedynamic instruction

To show it more clear, lets change the example above which was using anonymous inner class to using lambda expression.

{% gist jp-wang/29ba00700ce941f4ee60e2158f1b997c Lambda.java %}

You will see No extra class file was generated. And the bytecode is:

{% gist jp-wang/9ebe77982a8b99c419b7d691c4421ee5 Lambda.class %}

Did you see the difference? The major changes the JVM did are:

1. In `void main(..)` method, it runs `invokedynamic #4,  0` instead of `invokespecial #5` before. `invokespecial #5` will construct an object of the anonymous class thats generated out.
2. A new private static method was created inside with a special name format `className$invokeMethod$number` such as `lambda$main$0`, and its parameter is same as the ones defined in lambda expression.

### What's invokedynamic instruction doing?

So now you may see the key point is [invokedynamic instruction](http://docs.oracle.com/javase/7/docs/technotes/guides/vm/multiple-language-support.html#invokedynamic). Lets see the definition from Java offical declaration.

> Each instance of an invokedynamic instruction is called a dynamic call site. A dynamic call site is originally in an unlinked state, which means that there is no method specified for the call site to invoke. As previously mentioned, a dynamic call site is linked to a method by means of a bootstrap method. A dynamic call site's bootstrap method is a method specified by the compiler for the dynamically-typed language that is called once by the JVM to link the site. The object returned from the bootstrap method permanently determines the call site's behavior.

With this ability, the things running here are difered until runtime with bootstrap method.

Lets take a look at the bootstrap methods:

{% gist jp-wang/f399b35e407a4f8c5518fe7320058296 BootstrapMethods.java %}

Actually, when you are using IDE to debug it, it will go into the method from `LambdaMetafactory`: `CallSite metafactory(...)`.

{% gist jp-wang/f7aabd9dc6a7c9cefb8ebbd252c43f22 LambdaMetafactory1.java %}

After going through the source code, you will find that an anonymous class was generated at runtime.

{% gist jp-wang/5046148e3dfe4a1a31cae84c4b7a4222 LambdaMetafactory2.java %}

{% gist jp-wang/c10150e5fdcb8412440fcfe25806707f LambdaMetafactory3.java %}

Most of time, you will be confused by the strategy here: Both of anonymous inner class and lambda expression are creating extra class in memory, what's the difference?

Yes, you are partially right! But there is a big difference between them: Anonymous inner class is creating extra class file in compile time, but lambda did it in run time. And also lambda expression is only creating the anonymous class into memory, not have any class file which could save obvious performance impact by loading class file, verify and link. Its real dynamic! 

You can also print out the anonymous class thats generated by `LambdaMetafactory`:

`java -Djdk.internal.lambda.dumpProxyClasses Lambda` - Add parameter `-Djdk.internal.lambda.dumpProxyClasses` when running it.

Lets see the anonymous class that we print out(the format is `className$$Lambda$number`):

`javap -v -p -s -sysinfo -constants Lambda\$\$Lambda\$1.class`

{% gist jp-wang/2854b967c483b2a03fb95dbd672140eb Lambda$$Lambda$1.class %}

It will implement the functional interface we defined before, and have its own `print(...)` method which will delegate to invoke static method that was automatically generated in `Lambda.class` - `void lambda$main$0(String)`. It's proxy design pattern!

Now you get the full knowledge of how the JVM implements the lambda expression.


```
sequenceDiagram
Lambda caller->>Anonymous class: invokedynamic
Anonymous class->>Static method: execute lambda body
```


### How to handle the outside variables?

One of the characteristics in anonymous inner class is that the body itself can access the outside variables which could be outside local variables or instance fields. How does the Lambda Expressions translation handle these staffs?

So let's separate into two parts:

**Access local variables defined outside of body**

Let's the change the example to:

{% gist jp-wang/0d78af38c0e00ddd0dfcddc082f45d9d LambdaAccessVar.java %}


And the bytecode difference than before is the method thats generated out:

{% gist jp-wang/8479c663bc1b68cce7372ea3e9d2c515 LambdaAccessVar.class %}

It has two parameter instead of one: which means both the outside local variables and method itself parameters are treated as same as method parameters. The value of local variables would be assigned into the generated nonymous class by instance constructor.

{% gist jp-wang/9adf51f31eb6d33d62d1f152c0754c7b Lambda$$Lambda$1AccessVar.class %}

So that means both anonymous inner class and Lambda Expression are taking this case as the same way.


**Access class fields defined outside of body**

Let's the change the example to:

{% gist jp-wang/9594100aaa77c4bd7618cb8007d07867 LambdaAccessLocal.java %}

To access the class field, we cannot directly access it in static method. So I put the lambda expression into non-static method and access the class field `a` in the expression body.

And the bytecode after runtime generation is :

{% gist jp-wang/e7b51f0c7595205073d6d8b201d46596 Lambda.class %}

Instead of static method automatically generated out, an instance method is there which means this method can access any fields in this class. If you take a look at the bytecode of `Lambda$$Lambda$1.class`, the Lambda object itself will be passed into the constructor of `Lambda$$Lambda$1.class`.

Most of time, we call these two cases *non-capturing* (the lambda doesn’t access any variables defined outside its body) or *capturing* (the lambda accesses variables defined outside its body).


### Conclusion

However this translation strategy is not set in stone because the use of the invokedynamic instruction gives the compiler the flexibility to choose different implementation strategies in the future. For instance, the captured values could be boxed in an array or, if the lambda expression reads some fields of the class where it is used, the generated method could be an instance one, instead of being declared static, thus avoiding the need to pass those fields as additional arguments.
