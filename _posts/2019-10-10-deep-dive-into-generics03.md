---
layout: post
title: Deep dive into Java Generics(0x03)
published: true
---

**[Part 1: Deep dive into Java Generics(0x01)](https://jp-wang.github.io/deep-dive-into-generics01/)**

**[Part 2: Deep dive into Java Fenerics(0x02)](https://jp-wang.github.io/deep-dive-into-generics02/)**

In last two chapters we have discussed the urgent requirement from Java Community to bring in Generics Type and how it was implemneted finally with the boundary of couple of restrictions due to the compatibility demand.

Of course it met the purpose of bringing in Generics with couple of trickys by the type erasure. Especially, it did bring some waves on where the Java type system originally was not considered *Generics*. 

So this post will be deep diving into the impact of Java type system by introducing Generics type and how it works with existing Java types.

## Generics <> Array

Both Generics and Array are still belonging to Java Type system, so they should be able to use as the base type to construct each other(Generics-Generics, Array-Array, Generics-Array, Array-Generics), such as `List<List<String>>`, `String[][]`, `List<String>[]`, `List<String[]>`.

However, using parameterized type to construct array(which is `List<String>[]`) is not allowed due to the type safety concern(except the unbounded wildcard type). But how about the `List<String[]>` which was using the array to construct the parameterized type? Yes, it is allowed. Actually most of the reference types in Java can be used as the actual type argument except the primitive types.

The `String[][]` of Array-Array is also doable which we called two-dimensions array, that means we can construct a new array type with another array. Ideally, you can construct a new array with N-dimensions, but Java 

## Variance

In last post we talked that *Java Array* was one of the type that handling variance which was extending the existing Java regular types relationship into more complicated types. *Java Generics* was exactly the same one that using existing types to construct more complicate type.

Since the Java Array is covariant type you will think that Generics will be of course designed as covariant. The answer is Yes and No. It works with Variance but different than Java Array.

## Generics <> Covariance

Java designed Array as covariant type with some trade-off of type safety by delaying the type checking during runtime. 

```java
Integer[] ints = new Integer[10];
Number[] nums = ints; //covariant
nums[0] = new Long(100L); // Will throw runtime expcetion: ArrayStoreException
```

However, it is possible to gurantee the type safety of array during compile time. Let's see if Java can introduce a new keyword called `covariant` which can be added in front of type definition and describe that the array is read-only.

```java
Integer[] ints = new Integer[10];
covariant Number[] nums = ints; //covariant
nums[0] = new Long(100L); // Will get compile error, such as `Number[]` is a covariant type that supports read-only.
```

Why didn't Java handle it like this? I think the major reason is the trade-off they have chosen to make array real useful for most of cases. For example, you have a sequence of items that you want to swap their order. Keep in mind that you didn't have the collection framework to server it at that old age, the only structure you had was array. To make the method really ususful for all different arrays, you have to define the array type to be `object[]`. So if you add read-only restriction on it, then it won't be useful.

```java
void swap(object[] items) {
    ...
}
```

However, it is very easy to achieve it by using Java Generic Method and it's not necessary to bring in covariant.

```java
<T> void swap(T[] itmes) {
    ...
}
...
Integer[] ints = new Integer[] {1, 2, 3};
String[] ss = new String[] {"a", "b", "c"};
swap(ints);
swap(ss);
```

How did Java implement the super-typing and sub-typing relationship among Generics? Can it be directly designed as covariant like array?

Ok, let's assume the Generics can be directively covariant. Let's write some code like this, 

```java
ArrayList<Integer> intList = new ArrayList<>();
ArrayList<Number> numList = intList; //assume it is covariant.
numList.add(new Long(20)); //it is expected to have some exception such as CollectionStoreException like array did, but it won't be due to the type erasure.
```

As a result of type erasure there is neither runtime type safety check nor compile-time type safety check since we allow the covariant between `ArrayList<Integer>` and `ArrayList<Number>`. This will bring in huge problem into Java type system and against the rules of type safety.

So same thought as we discussed earlier to make the Array type safety if allowing covariant - it can be defined in such a way that it allows read-only to gurantee the type safety. Java introduces the synctax called `upper bounded wildcard` like this `ArrayList<? extends Number>` to make the `ArrayList<? extends Number>` and `ArrayList<Integer>` being covariant. Meanwhile, the Compiler will gurantee the read-only when you access the operations in `ArrayList<? extends Number>`. For example, the `add()` method will not be able to called anymore.

```java
ArrayList<Integer> intList = new ArrayList<>();
ArrayList<? extends Number> numList = intList; //assume it is covariant.
numList.add(new Long(20)); //it is forbidded during compile.
```

## Generics <> Contravariance

We have talked the **Covariance** which will preserve the ordering of types(<=) from more specific to more generic. And **Contravariance** is the other side which will reverse the ordering.

What does it mean? How is it useful?

To explain it more detail we should come always back to the foundmental of subtying system in modern programming languanges.

> In programming language theory, subtyping (also subtype polymorphism or inclusion polymorphism) is a form of type polymorphism in which a subtype is a datatype that is related to another datatype (the supertype) by some notion of substitutability, meaning that program elements, typically subroutines or functions, written to operate on elements of the supertype can also operate on elements of the subtype.                             - wikipedia.org

Basically by leveraging the subtyping concept the programming languange can easily allow you to create a function that takes an object of a certain type `T`, but also work correctly, if passed an object that belongs to a type `S` that is a subtype of `T`.

So both **Covariance** and **Contravariance** are serving the same purpose which is making the subtype relationship and leveraging language's object oriented features.

Let's take one example. Assume you want to create a function that copying all the items from one list into another list. You can define it like this,

```java
<T> void copy(List<T> source, List<T> dest) {
    ...
}

List<String> source = ...;
List<String> dest = ...;

copy(source, dest);
    
```

However, you can only copy the lists that have exact same type arguments. For instance, you cannot copy `List<Integer>` into `List<Number>` even it is safe for `List<Number>` to accept item whose type is `Integer`.

To achieve that the parameter of `dest` thats defined in method `copy` has to be variant so that it can accept its subtypes. There is one way we just discussed which is `Covariant`. By applying covariance to `List<T> dest` you will get `List<? extends T> dest`. However, it is not we expected, and we actually want the reversed way which the `dest` parameter can accept the list that its type argument is actually the super type of `T`. E.g., A `List<Integer>` is going to be copied into a dest whose type could be `List<Number>`. And meanwhile the compiler won't allow the insert operation on the `dest` which doesn't meet our requirement.

This is the exactly case that `Contravariance` saves the world. By applying `Contravariance` on the `dest` you will get the type like this `List<? super T> dest` which seems allow you to pass `List<Number>` as a subtype of `List<Integer>`.

```java
<T> void copy(List<T> source, List<? super T> dest) {
    ...
}

List<Integer> source = ...;
List<Number> dest = ...;

copy(source, dest); // the method signature equals to void copy(List<Integer> source, List<? super Integer> dest). Integer is the subtype of Number, but List<? super Integer> is the supertype of List<Number>.
```

As you may notice it is safe to add item whose type is `T` or subtype of `T` into contravariant collection `List<? super T>` but not for read. That's the same reason as covariance - the detail component type information is lost after contravariance except the component type is the supertype of `T`, so it is safe to add `T` or subtype of `T` into it but not the read.
