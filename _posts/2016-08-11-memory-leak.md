---
layout: post
title: Memory leak in Android
published: true
---
**GC**(Garbage collection) is one of most important features in Java, which greatly increases developer's productivity and save a lot away from the most painful things when we are writting programs.

Traditionaly, Java provides a lot of useful mechanisms to improve the GC effeciency such as the Parallel GC, CMS(Concurrent Marker Swap), G1 and so on, which makes the memory recycling more intelligient and fast. And the developer can focus on the program itself instead of be pain with dealing with the pointer, object memory release and such kind of small but trivial things.

There is no perfect in the world! So does Java. There are some ways that memory can be leaked logically within Java, especially in Android. Although Android has been on the way of high speed growing for a long time and the device performance increases a lot, it's still embedded system and limited by some aspects especially with the Battery and Memory. If the memory leaking in your code is not handled carefully, it's really easy make your App runing in a unstable situation and consuming lots of resource within your device, and even worse make the whole system running in unstable.

So today I would like to summarize the common mistakes that we have meet in our projects.

### 1. Static/Global reference

The object in Java thats kept reference by static variable will not be recycled by GC until process died. 
For example,

{% gist jp-wang/747cf96b528bf91202cc8b018d794441 StaticReference.java %}

which is very obvious memory leaking that the object of A keeps alive forever until the static reference is updated by other one or process finishes. This is the easy place to find out memory leak. But in Android, similar problem will be made easily but more obscure. Let's see an instance: You have a static utility class which needs the Android context to inflate some resouces like String or image. However, your utility class keeps a internal static reference to be reused everytime. Ideally, what you want is Android's application context which is global and not to be recycled.

{% gist jp-wang/aa2808f8a82667f6778020970724873e MyUtility.java %}

But the Context in Android has kinds of difference implementation, and one of them is `Activity`. Since you don't have any means to enforce your `init()` method only accepts the Application context instead of `Activity`, then if anyone sets `Context` you want with `Activity`, unluckily the `Activity` will be leaked at the instance time calling `init()`.

Same example like caching `View`. You may keep some heavy `View` in cache to avoid inflating everytime and save the performance. But you may not notice that the `View` itself is created in `Activity`(e.g., `TextView tv = new TextView(this);`), and now its causing the `Activity` leaked unexpectly due to your cache.

The two cases above are indeed happening in my project and make lots of ghost bugs. Keep in mind: The `Activity` in Android is very heavy object which keeps large mounts of memory, and consuming lots of resources if you don't release it properly.

### 2. Inner/Anonymous class

Inner/Anonymous class is another common place to make memory leaking in both Java and Android, especially in Android which uses tons of anonymous class definitions. Let's see a real example in my project: There is a `AlertManager` singleton instance to control all the alert popups in same place.

{% gist jp-wang/a6475c28eacc910545e14bc9061ff1e3 AlertManager.java %}

{% gist jp-wang/74736f15af87d2f32a304115a4d6bb65 MainActivity.java %}

If you have chance to read my last chapter `[Deep Dive into Java Lambda](https://jp-wang.github.io/deep-into-lambda2/)`, you will know how the Java JVM handles the anonymous class. Java Compiler will automatically generate a class to represent the anonymous class with name `MainActivity$1.class` that implements interface `onClickListener`. And if you are looking at the binary code of `MainActivity$1.class`, you will find that a constructor method is automatically generated out with one parameter `MainActivity` which means our code `new OnClickListener(){ ... }` equals to `new MainAcitivity$1(this)`.

So you think you are only creating the dialog in your `AlertManager` which has no bussiness with the caller `MainActivity`, but actually there is a persistent hard reference over here: currentDialog -> onClickListener -> MainActivity.this.

The same as inner class which will also keep a hidden reference to the instance of outside class.

### 3. Thread/Handler/TimerTask

In Android, we often use the Handler/TimerTask to handle the asynchronous jobs such as network I/O or heavy tasks. Most of common ways to create Handler or TimerTask are directly *new* it at the place gets called.

{% gist jp-wang/27755731d7b1962925e4563cd5711ac0 TimerTask.java %}

Since the `TimerTask` is handling jobs in loops, it won't be recycled. And as we talked in #3, the instance of `TimerTask` created here keeps a hidden reference to outside which means any invoker who creates TimerTask will be leaked involuntarily.

The official way to solve this problem is creating inner static class instead of anonymous class, and keep `WeakReference` to outside.

{% gist jp-wang/d8912351fb76ee45aa9fb6e852253229 MyTimerTask.java %}

### 4. Register listener in System

Finally, there are some system services that can be retrived by a `Context` with a call to `getSystemService`. And we are often registering listener into these system services to get the information we need. But be sure to remove the listener yourself from it when you don't need it anymore.

{% gist jp-wang/76b563c91dcd0fedee164283ce0d6b86 SensorManagerListener.java %}