---
layout: post
title: Deep dive into Java Default Methods
published: true
---
## Preface

In my last post([Deep dive into Java Lambda](https://jp-wang.github.io/deep-into-lambda/)), we have looked at the new feature added into Java 8 which gives us a good syntax sugar and effecient way to replace the traditional anonymous inner classes. That is really helpful and lovely. 

There is another new feature called **Default Methods** that enable you to add new functionality to the interfaces of your libraries and ensure binary compatibility with code written for older versions of those interfaces. To be honest, it doesn't shock me too much like *Lambda*, either other new features(which I will introduce later), and only give me the feeling like "Oh, that's it?!" The reason I'm not so surprised is due to below two reasons:
1. The most benefit of **Default Methods** is the binary compatibility.
2. The Diamond problem(multiple inheritance) is introduced in which makes the syntax of Java much more complicated than before.

Before we take a detail look at these two reasons, let's give a simple introduction of **Default Methods** and how to use.

## What's Default Methods?

To simplfy the concept, let's use one example to explain it. Assume you have a library which defined a interface `Animal`, and your library is currently used by a lot of third partners day over night again and again(Yeap! I'm an Animal lovers!).

{% gist jp-wang/1d16b9a012df1442f0cf895ef8e142f3 Animal.java %}

However, In some day, you have to change your interface to add a new method `String getColor()`. But you cannot just directly change it, or else the whole world will stop immediately due to the highly dependency on your `Animal` interface.

What can you do?! Yes, thats how the **Default Methods** trying to solve. By using the syntax of Default Methods like below, you don't need to care the binary compatibility anymore and just add what you want.

{% gist jp-wang/cd789f9c2249eda2a065b4499f30713f AnimalNew.java %}

Even more, as the word of `default` means, you can add default behavior in it(Tip: whether you like it or not, that's my hatest one! I will talk more about it right later).

{% gist jp-wang/b506e7b64d40d5fb84157f1974ad8b45 AnimalDefault.java %}

Well, that's the **Default Methods** and really straightforward for understanding. And it's also used in lots of common antique interfaces, such as `Collection`, `List`, `Map` and so on. 

So, what I'm talking about next is why I'm not excited by this new feature.

## Why not to go

### 1. Binary Compatibility

In my personal opinion, there are bunch of ways to support Binary Compatibility like Composition, Delegation, Bridge and etc., and I don't think changing the interfaces silently(which means adding behaviors in the interface without any notificaton and changes to the caller) is a good efficient way.

Interface is defined as a contract. It's a declaration of all your behaviors. If there is anything you must change your behavior or add new behavior, just change it and notify your customer to make the change if they want to have the new things. Changing contract is bad, but changing contract silently without any notification is always worse.

Maybe the guys who are introducing Default Methods into Java Grammar struggled a lot when there were looking for the balance between adding new features(such as Streaming) and introducing Default Methods as negative. However, the new features like Stream is so awesome! Default Methods? Who cares?

### 2. Diamond problem

**Multiple inheritance** is a very common feature in other language like `C++` in which an object or class can inherit characteristics and features from more than one parent object or parent class. It is distinct from single inheritance, where an object or class may only inherit from one particular object or class.

And the **Diamond problem** is an sensitive issue for many years that may be ambiguous as to which parent class a particular feature is inherited from if more than one parent class implements said feature. To be honest, it caused more problems and complexity rather than benefit. 

> For example, if there are two exact same default methods naming but different implementation defined in different interfaces A and B, and a class or another interface implement/extend both A and B, which behavior should it inherits?

The reason I like Java so much is the concision: C++ thinks the programmer is superman, but Java think you are Muggle. Java discards the complex ways which make developer easily be lost and sufferring pains to deal with the complex grammar such as Pointer, Garbage recycle, and especially of the `Multiple inheritance` - it's using multiple interfaces instead of multiple classes inheritance. Most of time, `Inheritance` is used to either changing the parent class's behavior or supporting the polymorphism. So Java uses the interface to support polymorphism in a beautiful way. So you can treat this feature in Java as `multiple inheritance of type`, instead of the concepts `multiple inheritance of state/implementation` in C++.

> The Diamond problem we talked about is majorly `multiple inheritance of state/implementation`

## Before you really to go

If you really have no ways(such as Object Composition, Delegate) to alter your interface except adding new methods in, and don't want to change any sub-classes who implement your interface, then you need to learn and understand the below complexities before making any changes by using `Default Methods`.

> Most of the complexities are related to Diamond problem(multiple inheritance)

* Two interfaces has same default method

{% gist jp-wang/456bf0bb23a927b9914b63b54bcc3473 SameDefaultMethod.java %}

{% gist jp-wang/0a500682f3a1d177fbe7bc2aec9e8842 SameDefaultMethod.java %}

A defines a default method `String say(String words)`, and same signature as in B. So Java8 compiler will force you to override the default method if anyone extends both A and B, or else a compile error will show up.

{% gist jp-wang/5c38cb261a6f14d2d0b91d2a9bfdb89f COverride.java %}

> Method override is totally different than method overload. The case like below will not get any compile error.

{% gist jp-wang/495da1b1429eca9a43597f6dee812258 COverload.java %}
    
* Multiple interfaces are inheriting like a chain

{% gist jp-wang/d601824dd5a199d07eacc1509ee67fb3 InterfaceChainDiagram.java %}

{% gist jp-wang/f08019b69d8d1fbb2fa2248cdf63236d InterfaceChain.java %}

So if multiple interfaces are inherited like chain, then the implementation class of any interfaces in that chain will use the last `default method` defined in child-interface which means: in our case, "B" will be printed out.

* Indirectly inheritance with same interface

 The case is already shown up above: `D extends A,B` and also B is the child-interface of A. Actually, the rule is same as above and it will always use the last `default method` defined in child-interface no matter whether D extends A or not.

* combined the inheritance between interfance and class

{% gist jp-wang/ba10f639cbbe1a66057c38acf9467ebe CombinedDiagram.java %}

{% gist jp-wang/e004c178ab7e0cda44824c347afc06b2 Combined.java %}

We all know the polymophism in Java, but if the class C extends B and also implements interface A which has a exact same method defined in class B, how does the object of class C behave when we called `c.say()`?

The answer is - method defined in parent class will be inherited firstly, which means the `void say()` defined in class B will be inherited by C thats printed out "B". Only when the parent class doesn't have the same method, then the `default method` in the interface will be inherited. So if we change a little bit of our code like below, you will get a different result.

{% gist jp-wang/46a2e3ae3e13fa6e2f664ea10925e3c1 CombinedWithoutOverride.java %}

## Summary

You can follow the below rules to meet on any further complex cases:
1. The class takes precedence over the interface. If a subclass inherits the parent class and the interface has the same method implementation, then the subclass inherits the parent class method
2. Methods in sub-interface take precedence over methods in parent-interface
3. If the above conditions are not met, you must override its methods, or declare it as abstract
