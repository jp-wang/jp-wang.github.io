---
layout: post
title: Deep dive into Java Generics(0x02)
published: true
---

**(Part 1: Deep dive into Java Generics(0x01))[https://jp-wang.github.io/deep-dive-into-generics01/] **

We have talked Java Generics and its limitations after type erasure in previous chapeter. In this post I'm going to discuss another interesting part of generic type while working with array. 

## Array

Java was designed to be a Object Oriented Programming language since it was invented. So the type system it implemented was surrounding the object oriented concept which was the subtyping. And one of detail implementation of super-subtype relationship is the inheritance between superclass and its derived subclass or super-interface and its sub-interfaces, or super-interface and its implementation classes. 

For illustration, `class Dog` is the subclass of `class Animal` if you have the definition like this `class Dog extends Animal`. However, in Java the explicit inheritance between superclass and its derived subclass is not the only way to describe the super-subtype relationship. There are couple of more options in Java that you may notice but not thinking through such as **Variance**, and the `Array` in Java is one of the usage of **Variance**.

So what is the Variance? Variance refers to how subtyping between more complex types relates to subtyping between their components. It could be like - if the component type A is the subtype of type B, the constructed complex type A+ will also be the subtype of type B+, then this kind of variance will be called **Covariance**. Otherwise, if the complex type A+ is the supertype of type B+ then it will be called **Contravariance**. Of course if there is no relationship between A+ and B+, then it will be neither **Covariance** nor **Contravariance**.

Let's go back and take a look at Java array. The way to construct a Java array is very simple - combine the reference *Type* and *[]* as `Type[]`. For example, using `Dog` and `[]` to create a array of dogs will be `Dog[]`. And the `Dog[]` is the subtype of `Animal[]` implicitly due to the type relationship between its component type `Dog` and `Animal`. That is the Variance happening and more explicitly it is **Covariance**.

**Variance** is a very useful way to extend the type relationship into more complex types based on its component type. But it does bring the cost which is the impact to type safety. Let's take one example, 

```java
Dog[] dogs = new Dog[10]
Animal[] animals = dogs; //supertype-subtype convention
animals[0] = new Cat();
```

The code here is totally fine and fully compilable into bytecode. However, it's not compile-time-safe because when you try to operate the array by adding a new element it's throwing a runtime exception `ClassCastException`.

Usually when doing *widening reference convention* such as assign subtype object to supertype reference(like assign `Dog` to `Animal` reference) the compiler will get some kind of safety that only the operations on `Animal` type can be visiable and accessable during compile time.

```java
Dog dog = new Dog();
Animal animal = dog;
animal.move(); //move() is defined in Animal and common for all the animals
//animal.run(); //but run() is only defined in Dog and it's not available for all the animals.
```

So the side effect of losing type safety is not what we want when introducing `Variance`. Theoretically if Java wants to keep the compile-time-safety it should disable the `insert` operation and support read-only. However, it will be huge inconvinient if the `insert` operation is disabled for array especially at the time when Collection framework is not in place yet.

> (p.s.: In some new modern languages such as Kotlin the array type is not supported any more. It is replaced by Generic Type such as Array<T>.)

Although the Java array is giving up some compile time safety but it still has runtime type safety such as `ClassCastException` which will prevent your program running in wrong situation. 

The way Java did it was keeping all the type information during runtime. If you have chance to look at the bytecode of Java program, you will find something like `[I`, `[Ljava/lang/String;` which are the class symbols that represent the array type of the corresponding component.

Java creates the runtime classes to describe the array type. For example,

```java
int[] ints = new int[2];
Integer[] integers = new Integer[2];
long[] longs = new long[2];
Long[] Longs = new Long[2];
...
String[] strs = new String[2];

System.out.println(ints.getClass());
System.out.println(integers.getClass());
System.out.println(longs.getClass());
System.out.println(Longs.getClass());
System.out.println(strs.getClass());
```

```java
class [I
class [Ljava.lang.Integer;
class [J
class [Ljava.lang.Long;
class [Ljava.lang.String;
```

With such class information in runtime the JVM can easily check the type safety by looking up the component type.

## Array of parameterized type

However, thing becomes different when we try to create array for parameterized type. As we discussed earlier the runtime class information of parameterized type is still the raw type. For example, the runtime class of `List<String>` is still `class List`. So if it is allowed to create array like below the JVM  has no way to check the type safety during the runtime since the `intListArray` is represented by `class [Ljava.util.ArrayList` due to the type erasure of parameterized type. And the below code will be able to compile and run which is not we wanted.

```java
ArrayList<Integer>[] intListArray = new ArrayList<Integer>[2];
Object[] stringListArray = intListArray;
stringListArray[0] = new ArrayList<String>();
...
ArrayList<Integer> intList = intListArray[0];
//You ar expecting the `ClassCastException` due to the unmatched types 
//`ArrayList<Integer>` and `ArrayList<String>`, but it does not due to the type erasure.
```

That's why Java does not allow generic array creation for concrete parameterized type.

> (p.s.: But Java does allow to create unbounded wildcard parameterized type array. You can think about it.)

## Reifiable type

So it's pretty obvious that the type information during runtime is very important to gurantee the type safety. 

A type whose type information is fully available at runtime is called **reifiable type**, that is, who does not lose information in the course of type erasure.

Obviously Java array needs the component type being reifiable to make sure the type safety during runtime. And parameterized type is of course not the reifiable type due to the lost of type information by type erasure.

The following types are reifiable:
* primitive types
* non-generic(non-parameterized) reference types
* unbounded wildcard parameterized type
* raw types
* array of any of the above

Another usage of reifiable type is the `instanceof` operation. The `instanceof` is checking the object's runtime type so that it can be safely casted to, which means it needs the full type information to gurantee the type safety.
