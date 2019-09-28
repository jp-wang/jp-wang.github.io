---
published: false
---
Recently our team has made a new library to wrap our Routing and Navigation API for Android platform, and we have heavily used Generics to export our functions which saves us a lot templates code and make the code more expressive. However, the Java Generics itself is not that straight comparing with other language's implementation, especially the confusion due to the type erasure of java runtime. So the topics following are the summary of what I have shared internally.

> The discussion we are going to talk here is not about the simple introduction or usage of Java Generics but the deep dive into the mechanism of Generics, the position in java type system and the difference of using Generics between Java and other languages due to the type erasure. So it was assumed that you've already learned the basic concepts of Java Generics, and at least used some common Generic type such as `List<T>`, `Set<T>` and etc, as well as the experience of defining your own generic type or method.

## Some terms of Generics

**Generic type** - it is a type that has formal type parameters such as `class ArrayList<T>`, `class Pair<F, S>`.

**Type parameter** - it is the place holder of generic type. For example, `T` is the type parameter of `interface List<T>`.

**Parameterized type** - it is an instantiation of a generic type with actual type argument such as `List<Integer>`, `List<String>`, `Comparable<Person>` and etc.

**Type argument** - it is the reference type that is used for the instantiation of generic type(parameterized type) or method, or the wildcard of instantiation of generic type, such as the `String` is the type argument of parameterized type `List<String>`.

**Wildcard** - it is a syntactic construct that denotes a family of types. `?` represents the family of all the types. `? extends Type` - a wildcard with upper bound represents the family of all the subtypes of Type and the Type itself. `? super Type` - a wildcard with lower bound represents the family of all the super types of Type and the Type itself.

All those terms are the basic concepts of Java Generics. Understanding them can help to deep dive into the mechanism.

## Why generics?

Before talking the Java Generics let's understand why Java introduced this concept and what's the problems it's trying to solve.

There is no generic type before Java 1.5. But there is one kind of problems that not easy to be handled in Java which is the collection of object with same type. For illustration `List` in java is one of data structure in collections that stores one type of datas that can provide the store, query, delete, and update. Defining a structure that handle the datas in a common way is very useful and helpful in modern programming languages. Due to the missing of generic before Java 1.5, there is no way to define a type in `List` except the `Object` to represent one kind of same type, such as `List.add(Object)`, `List.remove(Object)` and `List.get(int)`.

Yes, it brings a good solution to handle one type of objets in common way. However, it also brings one uncertain thing - type safety. Let's thinking about such case that the user actually wants to create a list of animals, however, due to the limitation of *List* API definition(it's always getting or returning *Object* type), it's losing the type information inside it when using the list. So if user wants to treat the element that's fetched from the *List* and cast to plant type, then a runtime of `ClassCastException` will be thrown out.

So the key point here is that the type of object put into *List* is losing the type information during compile time, and this is not we wanted when introducing collection framework. I believe the purpose of introducing Generics by Java is exactly the same purpose as we discussed here - providing more type information during compile or runtime to make sure the safety of your program.

## Type erasure

Java itself is fucusing very much on improving the safety of type during compile and runtime. However, the way it implements Generic has gave up some type safety due to the tradeoff of being compatiable with old java versions. For example, the way of `List<String>` running will be treated as `List` during runtime and the critical information of element type `String` was dismissed and instead being treated as `Object`.

That's why some people complains that Java is not implementing Generics completely and losing a lot of features in Generics such as not supporting `new T()`, `T.class`, and etc.

## `new T()` and `T.class`

The `new` keyword in java is used to create object. And `T` is the type parameter stands for one kind of type. In theory it should be able to create the object of type `T` during runtime since the actual type of `T` should already been confirmed during runtime. Actually it did work in the some languages such as *C#* that you can create the object of `T` during runtime but it's not working in Java. 

Why? The answer is simple - type erasure. Thinking about this - if it is supported by java, you will get the object of `Object` type due to the type erasure. However, it is not the object that you are expecting for your type. So when you use it as your type, you will be 100% getting the `ClassCastException`. So based on the concern of type safety and the logical correctness, the `new T()` is not supported in java.

Same rule is applied to `T.class` which will be equal to `Object.class` that has nothing meanful.

## List<String>.class

The class literal of parameterized type is not supported due to the same reason - type erasure. Not matter it is `List<String>` or `List<Integer>` they are equals to `List` during runtime so you just need `List.class`.

## List<int>

It is illegal. Due to the type erasure the element type will be defined as *Object* type in *List* but *Object* type cannot be casted into primitive type so it is not supported to use primitive type as the type argument of parameterized type.

## Timing of throwing `ClassCastException`

```java
public <T> T cast(Object input) {
    T output = (T)input; //cast input to expected type, and you are expecting the exception will be thrown here.
    return output;
}
```

If it is called by `String s = cast(new Object());`, you will expect that the `ClassCastException` is throwing at the time when the code is runing on the line of `T output = (T)input;`. But the fact is that the exception is throwing outside at the caller which is `String s = cast(new Object());`.

Why is that? It is same reason of type erasure, and the method can be translated like this:

```java
public Object cast(Object input) {
    Object output = input;
    return output;
}
...
String s = (String)cast(input);
```

So the real cast actually does on the caller.

## Parameterized type in exception catch

Java does not allow parameterized type to be in exception catch.
For example,

```java
try {
    ...
} catch (ParameterizedTypeException<String> e) {
    ...
} catch (ParameterizedTypeException<Integer> e) {
    ...
}
```

If it is allowed, then the JVM will be confused during runtime and code above actually equals to

```java
try {
    ...
} catch (ParameterizedTypeException e) {
    ...
} catch (ParameterizedTypeException e) {
    ...
}
```
which doesn't make any sense.

## Type parameter in exception catch

Same as above -

```java
try {
    ...
} catch(T e) {
    ...
} catch(T e) {
    ...
}
```

equals to:

```java
try {
    
} catch(Object e) {
    ...
} catch(Object e) {
    ...
}
```