---
published: false
---

There is a very insteresting problem when I have a talking with my friend. When we sit together and bring up the brainstorm, it guides us well to understand the inside nature of the problem. That's where this post was created from!

> Given an array of integers. find the maximum XOR subarray value in given array. Expected time complexity O(n).

Examples:

```
Input: arr[] = {1, 2, 3, 4}
Output: 7
The subarray {3, 4} has maximum XOR value

Input: arr[] = {8, 1, 2, 12, 7, 6}
Output: 15
The subarray {1, 2, 12} has maximum XOR value

Input: arr[] = {4, 6}
Output: 6
The subarray {6} has maximum XOR value
```

It's a good chance to introduce the XOR and what's the real essence behind it. However, this is the classic DP problem, but I don't want to talk too much in this chapter. If you are interested in, pls following and I give have another topic to introduce more about DP.

### What's XOR?

Here is the definiation from [Wikipedia](https://en.wikipedia.org/wiki/Exclusive_or).
> Exclusive or or Exclusive disjunction is a logical operation that outputs true only when inputs differ (one is true, the other is false).

So the result is like below:

^|0|1
--|--|--
0|0|1
1|1|0

which means if the bit at each position is same, then the result is 0, or else it's 1. If you remember Half Adder, they are exactly the same: 0 + 0 = 0, 0 + 1 = 1, 1 + 0 = 1, 1 + 1 = 0.

I think you may understand the logic meaning very easily, but it's not sensitive in your mind most of time. How to use it? What's the inside nature at all?

Let's see classic XOR usage - 
Exchange values of two int variables without introducing in new variable.

E.g., 

```java
int a = 3, b = 4;
a = a ^ b;
b = a ^ b;
a = a ^ b;
```

However, the usual way to do it is like below.

```java
int a = 3, b = 4;
a = a - b;
b = b + a;
a = b - a;
```

The inside nature of this one is very straight forward for your understanding: 
1. get the difference between a and b, then store it into a.
2. now you know the value of b, and the gap between b and a, then you can get the value of a by comparing b and difference between b and a.
3. same as #2, b is now assigned with the value of a, and you also have the difference between a and b, easy to get value a from them, right?

You may notice that the XOR and usual ways are very similar, but still have a little bit difference.

The difference is that the Add/Sub operation has direction, but XOR not.

### But why?

The reason is XOR operation only has two values - 1 or 0 which means true or false. But the normal Add/Sub has lots of values and also direction(positive and negative).

But they have the same nature - difference between two values.

Let's go back to the exchanging values with XOR, and try to read it with the inside nature that we just understand.

```java
int a = 3, b = 4;
a = a ^ b;
b = a ^ b;
a = a ^ b;
```

1. firstly, get the difference between a and b, then store the value into a.
2. now you have b and the gap between a and b, easily get the value of a and assign it to b(a ^ b ^ a - that means the difference between a and the difference of a and b, which means b itself).
3. now you have value a that stores into variable b, and still the gap between a and b, it's also easily get the value of b and assign it to a(a ^ b ^ a - that means the difference between a and the difference of and b, which means a itself).

Magical and fantastic!!

Since the XOR operation has no direction comparing with Add/Sub, it's pretty straight forward to get the expectations like below - 
```java
a ^ b == b ^ a //commutative law
(a ^ b) ^ c == a ^ (b ^ c) //associative law
```

And also
```
0 ^ a = a // difference between 0 and a is a itself
a ^ a = 0 // there is no difference between a and itself

(a ^ b) ^ (a ^ b ^ c) = c
// which means the difference between a ^ b and a ^ b ^ c is c itself
// or you can use associative law to change it `(a ^ a) ^ (b ^ b) ^ c` = `0 ^ 0 ^ c` = `c`
```