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

```java
@FunctionalInterface
interface Print<T> {
    public void print(T x);
}

public class Lambda {
    public static void PrintString(String s, Print<String> print) {
        print.print(s);
    }
    
    public static void main(String[] args) {
        PrintString("test", new Print<String>() {
            @Override
            public void print(String x) {
                System.out.println(x);
            }
        });
    }
}
```

See the byte code after it gets compiled:

Two classes were generated out - `Lambda.class` and `Lambda$1.class`

`javap -v -p -s -sysinfo -constants Lambda.class `

```
Classfile /Users/jpwang/Desktop/Lambda.class
  Last modified Aug 1, 2016; size 562 bytes
  MD5 checksum 9fa9c7b317db05531d938a78e9618095
  Compiled from "Lambda.java"
public class Lambda
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #8.#22         // java/lang/Object."<init>":()V
   #2 = InterfaceMethodref #23.#24        // Print.print:(Ljava/lang/Object;)V
   #3 = String             #25            // test
   #4 = Class              #26            // Lambda$1
   #5 = Methodref          #4.#22         // Lambda$1."<init>":()V
   #6 = Methodref          #7.#27         // Lambda.PrintString:(Ljava/lang/String;LPrint;)V
   #7 = Class              #28            // Lambda
   #8 = Class              #29            // java/lang/Object
   #9 = Utf8               InnerClasses
  #10 = Utf8               <init>
  #11 = Utf8               ()V
  #12 = Utf8               Code
  #13 = Utf8               LineNumberTable
  #14 = Utf8               PrintString
  #15 = Utf8               (Ljava/lang/String;LPrint;)V
  #16 = Utf8               Signature
  #17 = Utf8               (Ljava/lang/String;LPrint<Ljava/lang/String;>;)V
  #18 = Utf8               main
  #19 = Utf8               ([Ljava/lang/String;)V
  #20 = Utf8               SourceFile
  #21 = Utf8               Lambda.java
  #22 = NameAndType        #10:#11        // "<init>":()V
  #23 = Class              #30            // Print
  #24 = NameAndType        #31:#32        // print:(Ljava/lang/Object;)V
  #25 = Utf8               test
  #26 = Utf8               Lambda$1
  #27 = NameAndType        #14:#15        // PrintString:(Ljava/lang/String;LPrint;)V
  #28 = Utf8               Lambda
  #29 = Utf8               java/lang/Object
  #30 = Utf8               Print
  #31 = Utf8               print
  #32 = Utf8               (Ljava/lang/Object;)V
{
  public Lambda();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 6: 0

  public static void PrintString(java.lang.String, Print<java.lang.String>);
    descriptor: (Ljava/lang/String;LPrint;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_1
         1: aload_0
         2: invokeinterface #2,  2            // InterfaceMethod Print.print:(Ljava/lang/Object;)V
         7: return
      LineNumberTable:
        line 8: 0
        line 9: 7
    Signature: #17                          // (Ljava/lang/String;LPrint<Ljava/lang/String;>;)V

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=1, args_size=1
         0: ldc           #3                  // String test
         2: new           #4                  // class Lambda$1
         5: dup
         6: invokespecial #5                  // Method Lambda$1."<init>":()V
         9: invokestatic  #6                  // Method PrintString:(Ljava/lang/String;LPrint;)V
        12: return
      LineNumberTable:
        line 12: 0
        line 17: 12
}
SourceFile: "Lambda.java"
InnerClasses:
     static #4; //class Lambda$1

```

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

```java
//@FunctionalInterface
interface Print<T> {
    public void print(T x);
}
public class Lambda {
    public static void PrintString(String s, Print<String> print) {
        print.print(s);
    }
    
    public static void main(String[] args) {
        PrintString("test", (x) -> System.out.println(x));
    	// PrintString("test", new Print<String>() {
    	// 	public void print(String x) {
    	// 		System.out.println(x);
    	// 	}
    	// });
    }
}
```

You will see No extra class file was generated. And the bytecode is:

```
Classfile /Users/jpwang/Desktop/Lambda.class
  Last modified Aug 1, 2016; size 1176 bytes
  MD5 checksum 40efe9faa71718e7842ef4619a8872b3
  Compiled from "Lambda.java"
public class Lambda
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #9.#24         // java/lang/Object."<init>":()V
   #2 = InterfaceMethodref #25.#26        // Print.print:(Ljava/lang/Object;)V
   #3 = String             #27            // test
   #4 = InvokeDynamic      #0:#33         // #0:print:()LPrint;
   #5 = Methodref          #8.#34         // Lambda.PrintString:(Ljava/lang/String;LPrint;)V
   #6 = Fieldref           #35.#36        // java/lang/System.out:Ljava/io/PrintStream;
   #7 = Methodref          #37.#38        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #8 = Class              #39            // Lambda
   #9 = Class              #40            // java/lang/Object
  #10 = Utf8               <init>
  #11 = Utf8               ()V
  #12 = Utf8               Code
  #13 = Utf8               LineNumberTable
  #14 = Utf8               PrintString
  #15 = Utf8               (Ljava/lang/String;LPrint;)V
  #16 = Utf8               Signature
  #17 = Utf8               (Ljava/lang/String;LPrint<Ljava/lang/String;>;)V
  #18 = Utf8               main
  #19 = Utf8               ([Ljava/lang/String;)V
  #20 = Utf8               lambda$main$0
  #21 = Utf8               (Ljava/lang/String;)V
  #22 = Utf8               SourceFile
  #23 = Utf8               Lambda.java
  #24 = NameAndType        #10:#11        // "<init>":()V
  #25 = Class              #41            // Print
  #26 = NameAndType        #42:#43        // print:(Ljava/lang/Object;)V
  #27 = Utf8               test
  #28 = Utf8               BootstrapMethods
  #29 = MethodHandle       #6:#44         // invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #30 = MethodType         #43            //  (Ljava/lang/Object;)V
  #31 = MethodHandle       #6:#45         // invokestatic Lambda.lambda$main$0:(Ljava/lang/String;)V
  #32 = MethodType         #21            //  (Ljava/lang/String;)V
  #33 = NameAndType        #42:#46        // print:()LPrint;
  #34 = NameAndType        #14:#15        // PrintString:(Ljava/lang/String;LPrint;)V
  #35 = Class              #47            // java/lang/System
  #36 = NameAndType        #48:#49        // out:Ljava/io/PrintStream;
  #37 = Class              #50            // java/io/PrintStream
  #38 = NameAndType        #51:#21        // println:(Ljava/lang/String;)V
  #39 = Utf8               Lambda
  #40 = Utf8               java/lang/Object
  #41 = Utf8               Print
  #42 = Utf8               print
  #43 = Utf8               (Ljava/lang/Object;)V
  #44 = Methodref          #52.#53        // java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #45 = Methodref          #8.#54         // Lambda.lambda$main$0:(Ljava/lang/String;)V
  #46 = Utf8               ()LPrint;
  #47 = Utf8               java/lang/System
  #48 = Utf8               out
  #49 = Utf8               Ljava/io/PrintStream;
  #50 = Utf8               java/io/PrintStream
  #51 = Utf8               println
  #52 = Class              #55            // java/lang/invoke/LambdaMetafactory
  #53 = NameAndType        #56:#60        // metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #54 = NameAndType        #20:#21        // lambda$main$0:(Ljava/lang/String;)V
  #55 = Utf8               java/lang/invoke/LambdaMetafactory
  #56 = Utf8               metafactory
  #57 = Class              #62            // java/lang/invoke/MethodHandles$Lookup
  #58 = Utf8               Lookup
  #59 = Utf8               InnerClasses
  #60 = Utf8               (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #61 = Class              #63            // java/lang/invoke/MethodHandles
  #62 = Utf8               java/lang/invoke/MethodHandles$Lookup
  #63 = Utf8               java/lang/invoke/MethodHandles
{
  public Lambda();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 6: 0

  public static void PrintString(java.lang.String, Print<java.lang.String>);
    descriptor: (Ljava/lang/String;LPrint;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_1
         1: aload_0
         2: invokeinterface #2,  2            // InterfaceMethod Print.print:(Ljava/lang/Object;)V
         7: return
      LineNumberTable:
        line 8: 0
        line 9: 7
    Signature: #17                          // (Ljava/lang/String;LPrint<Ljava/lang/String;>;)V

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: ldc           #3                  // String test
         2: invokedynamic #4,  0              // InvokeDynamic #0:print:()LPrint;
         7: invokestatic  #5                  // Method PrintString:(Ljava/lang/String;LPrint;)V
        10: return
      LineNumberTable:
        line 12: 0
        line 18: 10

  private static void lambda$main$0(java.lang.String);
    descriptor: (Ljava/lang/String;)V
    flags: ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: aload_0
         4: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         7: return
      LineNumberTable:
        line 12: 0
}
SourceFile: "Lambda.java"
InnerClasses:
     public static final #58= #57 of #61; //Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
BootstrapMethods:
  0: #29 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #30 (Ljava/lang/Object;)V
      #31 invokestatic Lambda.lambda$main$0:(Ljava/lang/String;)V
      #32 (Ljava/lang/String;)V

```

Did you see the difference? The major changes the JVM did are:

1. In `void main(..)` method, it runs `invokedynamic #4,  0` instead of `invokespecial #5` before. `invokespecial #5` will construct an object of the anonymous class thats generated out.
2. A new private static method was created inside with a special name format `className$invokeMethod$number` such as `lambda$main$0`, and its parameter is same as the ones defined in lambda expression.

### What's invokedynamic instruction doing?

So now you may see the key point is [invokedynamic instruction](http://docs.oracle.com/javase/7/docs/technotes/guides/vm/multiple-language-support.html#invokedynamic). Lets see the definition from Java offical declaration.

> Each instance of an invokedynamic instruction is called a dynamic call site. A dynamic call site is originally in an unlinked state, which means that there is no method specified for the call site to invoke. As previously mentioned, a dynamic call site is linked to a method by means of a bootstrap method. A dynamic call site's bootstrap method is a method specified by the compiler for the dynamically-typed language that is called once by the JVM to link the site. The object returned from the bootstrap method permanently determines the call site's behavior.

With this ability, the things running here are difered until runtime with bootstrap method.

Lets take a look at the bootstrap methods:

```
BootstrapMethods:
  0: #29 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #30 (Ljava/lang/Object;)V
      #31 invokestatic Lambda.lambda$main$0:(Ljava/lang/String;)V
      #32 (Ljava/lang/String;)V
```

Actually, when you are using IDE to debug it, it will go into the method from `LambdaMetafactory`: `CallSite metafactory(...)`.

```java
 public static CallSite metafactory(MethodHandles.Lookup caller,
                                       String invokedName,
                                       MethodType invokedType,
                                       MethodType samMethodType,
                                       MethodHandle implMethod,
                                       MethodType instantiatedMethodType)
            throws LambdaConversionException {
        AbstractValidatingLambdaMetafactory mf;
        mf = new InnerClassLambdaMetafactory(caller, invokedType,
                                             invokedName, samMethodType,
                                             implMethod, instantiatedMethodType,
                                             false, EMPTY_CLASS_ARRAY, EMPTY_MT_ARRAY);
        mf.validateMetafactoryArgs();
        return mf.buildCallSite();
}
```

After going through the source code, you will find that an anonymous class was generated at runtime.

```java
@Override
    CallSite buildCallSite() throws LambdaConversionException {
        final Class<?> innerClass = spinInnerClass();
        if (invokedType.parameterCount() == 0) {
            final Constructor[] ctrs = AccessController.doPrivileged(
                    new PrivilegedAction<Constructor[]>() {
                @Override
                public Constructor[] run() {
                    Constructor<?>[] ctrs = innerClass.getDeclaredConstructors();
                    if (ctrs.length == 1) {
                        // The lambda implementing inner class constructor is private, set
                        // it accessible (by us) before creating the constant sole instance
                        ctrs[0].setAccessible(true);
                    }
                    return ctrs;
                }
                    });
            if (ctrs.length != 1) {
                throw new LambdaConversionException("Expected one lambda constructor for "
                        + innerClass.getCanonicalName() + ", got " + ctrs.length);
            }

            try {
                Object inst = ctrs[0].newInstance();
                return new ConstantCallSite(MethodHandles.constant(samBase, inst));
            }
            catch (ReflectiveOperationException e) {
                throw new LambdaConversionException("Exception instantiating lambda object", e);
            }
        } else {
            try {
                UNSAFE.ensureClassInitialized(innerClass);
                return new ConstantCallSite(
                        MethodHandles.Lookup.IMPL_LOOKUP
                             .findStatic(innerClass, NAME_FACTORY, invokedType));
            }
            catch (ReflectiveOperationException e) {
                throw new LambdaConversionException("Exception finding constructor", e);
            }
        }
    }

```

```java
private Class<?> spinInnerClass() throws LambdaConversionException {
    ...
    return UNSAFE.defineAnonymousClass(targetClass, classBytes, null);
}
```

Most of time, you will be confused by the strategy here: Both of anonymous inner class and lambda expression are creating extra class in memory, what's the difference?

Yes, you are partially right! But there is a big difference between them: Anonymous inner class is creating extra class file in compile time, but lambda did it in run time. And also lambda expression is only creating the anonymous class into memory, not have any class file which could save obvious performance impact by loading class file, verify and link. Its real dynamic! 

You can also print out the anonymous class thats generated by `LambdaMetafactory`:

`java -Djdk.internal.lambda.dumpProxyClasses Lambda` - Add parameter `-Djdk.internal.lambda.dumpProxyClasses` when running it.

Lets see the anonymous class that we print out(the format is `className$$Lambda$number`):

`javap -v -p -s -sysinfo -constants Lambda\$\$Lambda\$1.class`

```
Classfile /Users/jpwang/Desktop/Lambda$$Lambda$1.class
  Last modified Aug 1, 2016; size 373 bytes
  MD5 checksum ffdfc4be6ac70b331a79d10c4e1ce322
final class Lambda$$Lambda$1 implements Print
  minor version: 0
  major version: 52
  flags: ACC_FINAL, ACC_SUPER, ACC_SYNTHETIC
Constant pool:
   #1 = Utf8               Lambda$$Lambda$1
   #2 = Class              #1             // Lambda$$Lambda$1
   #3 = Utf8               java/lang/Object
   #4 = Class              #3             // java/lang/Object
   #5 = Utf8               Print
   #6 = Class              #5             // Print
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = NameAndType        #7:#8          // "<init>":()V
  #10 = Methodref          #4.#9          // java/lang/Object."<init>":()V
  #11 = Utf8               print
  #12 = Utf8               (Ljava/lang/Object;)V
  #13 = Utf8               Ljava/lang/invoke/LambdaForm$Hidden;
  #14 = Utf8               java/lang/String
  #15 = Class              #14            // java/lang/String
  #16 = Utf8               Lambda
  #17 = Class              #16            // Lambda
  #18 = Utf8               lambda$main$0
  #19 = Utf8               (Ljava/lang/String;)V
  #20 = NameAndType        #18:#19        // lambda$main$0:(Ljava/lang/String;)V
  #21 = Methodref          #17.#20        // Lambda.lambda$main$0:(Ljava/lang/String;)V
  #22 = Utf8               Code
  #23 = Utf8               RuntimeVisibleAnnotations
{
  private Lambda$$Lambda$1();
    descriptor: ()V
    flags: ACC_PRIVATE
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #10                 // Method java/lang/Object."<init>":()V
         4: return

  public void print(java.lang.Object);
    descriptor: (Ljava/lang/Object;)V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=2, args_size=2
         0: aload_1
         1: checkcast     #15                 // class java/lang/String
         4: invokestatic  #21                 // Method Lambda.lambda$main$0:(Ljava/lang/String;)V
         7: return
    RuntimeVisibleAnnotations:
      0: #13()
}
```

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

```java
//@FunctionalInterface
interface Print<T> {
    public void print(T x);
}
public class Lambda {
    public static void PrintString(String s, Print<String> print) {
        print.print(s);
    }
    
    public static void main(String[] args) {
        int i = 1;
        PrintString("test", (x) -> System.out.println(x + i));
    	// PrintString("test", new Print<String>() {
    	// 	public void print(String x) {
    	// 		System.out.println(x);
    	// 	}
    	// });
    }
}

```

And the bytecode difference than before is the method thats generated out:

```
...
private static void lambda$main$0(int, java.lang.String);
```

It has two parameter instead of one: which means both the outside local variables and method itself parameters are treated as same as method parameters. The value of local variables would be assigned into the generated nonymous class by instance constructor.

```
final class Lambda$$Lambda$1 implements Print
{
  private final int arg$1;
    descriptor: I
    flags: ACC_PRIVATE, ACC_FINAL

  private Lambda$$Lambda$1(int);
    descriptor: (I)V
    flags: ACC_PRIVATE
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: invokespecial #13                 // Method java/lang/Object."<init>":()V
         4: aload_0
         5: iload_1
         6: putfield      #15                 // Field arg$1:I
         9: return
...
```

So that means both anonymous inner class and Lambda Expression are taking this case as the same way.


**Access class fields defined outside of body**

Let's the change the example to:

```java
//@FunctionalInterface
interface Print<T> {
    public void print(T x);
}
public class Lambda {
    private String a = "a";

    public static void PrintString(String s, Print<String> print) {
        print.print(s);
    }

    public void tryAccessField() {
        PrintString("test", (x) -> System.out.println(x + a));
    }
    
    public static void main(String[] args) {
        
    	new Lambda().tryAccessField();
    }
}

```

To access the class field, we cannot directly access it in static method. So I put the lambda expression into non-static method and access the class field `a` in the expression body.

And the bytecode after runtime generation is :

```
...
private void lambda$tryAccessField$0(java.lang.String);
    descriptor: (Ljava/lang/String;)V
    flags: ACC_PRIVATE, ACC_SYNTHETIC
    Code:
      stack=3, locals=2, args_size=2
         0: getstatic     #11                 // Field java/lang/System.out:Ljava/io/PrintStream;
         3: new           #12                 // class java/lang/StringBuilder
         6: dup
         7: invokespecial #13                 // Method java/lang/StringBuilder."<init>":()V
        10: aload_1
        11: invokevirtual #14                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        14: aload_0
        15: getfield      #3                  // Field a:Ljava/lang/String;
        18: invokevirtual #14                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        21: invokevirtual #15                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        24: invokevirtual #16                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        27: return
      LineNumberTable:
        line 13: 0
...
```

Instead of static method automatically generated out, an instance method is there which means this method can access any fields in this class. If you take a look at the bytecode of `Lambda$$Lambda$1.class`, the Lambda object itself will be passed into the constructor of `Lambda$$Lambda$1.class`.

Most of time, we call these two cases *non-capturing* (the lambda doesn’t access any variables defined outside its body) or *capturing* (the lambda accesses variables defined outside its body).


### Conclusion

However this translation strategy is not set in stone because the use of the invokedynamic instruction gives the compiler the flexibility to choose different implementation strategies in the future. For instance, the captured values could be boxed in an array or, if the lambda expression reads some fields of the class where it is used, the generated method could be an instance one, instead of being declared static, thus avoiding the need to pass those fields as additional arguments.