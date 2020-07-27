---
layout: post
title: "Javascript Does Not Need a StringBuilder"
date: 2020-02-24 18:40
author: joseph.mate
comments: true
categories: [Java, Javascript, StringBuilder]
---

Today I learned that Javascript does not need a StringBuilder for accumulating a large number of concatinations.
As a Java programmer, that came as a shock to me.

# Effective Java
A StringBuilder is grilled into Java programmers. Effective Java says, "Item 51: Beware the performance of string concatenation."
Many books, articles, blog posts and stack overflow questions have extensively covered this in detail 
[2](http://www.thefinestartist.com/effective-java/51)
[3](http://jtechies.blogspot.com/2012/07/item-51-beware-performance-of-string.html)
[4](https://www.corejavaguru.com/effective-java/items/51)
[5](https://dzone.com/articles/string-concatenation-performacne-improvement-in-ja)
[6](https://stackoverflow.com/questions/4645020/when-to-use-stringbuilder-in-java)
[7](https://stackoverflow.com/questions/10078912/best-practices-performance-mixing-stringbuilder-append-with-string-concat)
[8](https://stackoverflow.com/questions/1532461/stringbuilder-vs-string-concatenation-in-tostring-in-java).

Now you can see why it came as such a shock to me.
I assumed javascript would do the same thing as java to maintain the immutability of strings.
Java allocates a new array containing the result of each concatenation resulting in the O(N^2) runtime complexity.

# Experiments
When I first learned this from a co-worker, I couldn't believe it!
I put together some experiments to confirm.
I measured how long it would take to do different powers of 10 of concatenations in Java and javascript.


|           |             |                    | Chrome          | Chrome          | Firefox         | Firefox         |
|           | Java        | Java               | Javascript      | Javascript      | Javascript      | Javascript      |
| # concats | String (ms) | StringBuilder (ms) | concat (ms)     | Array.join (ms) | concat (ms)     | Array.join (ms) |
|-----------|-------------|--------------------|-----------------|-----------------|-----------------|-----------------|
|      2^10 |           4 |                  0 |               0 |               0 |               0 |               0 |
|      2^11 |          12 |                  0 |               0 |               0 |               0 |               0 |
|      2^12 |          28 |                  0 |               1 |               0 |               0 |               0 |
|      2^13 |          99 |                  0 |               1 |               1 |               1 |               0 |
|      2^14 |         374 |                  1 |               3 |               2 |               1 |               0 |
|      2^15 |        1197 |                  2 |               2 |               2 |               2 |               1 |
|      2^16 |        2961 |                  1 |               8 |               4 |               3 |               3 |
|      2^17 |        4505 |                  3 |              15 |               8 |               6 |               4 |
|      2^18 |       21945 |                  5 |              30 |              18 |              14 |              11 |
|      2^19 |       94077 |                  6 |              79 |              37 |              12 |              18 |
|      2^20 |      412675 |                 11 |             144 |              71 |              42 |              44 |
|      2^21 |           0 |                  0 |             408 |             156 |              68 |              70 |
|      2^22 |           0 |                  0 |             906 |             318 |             172 |             149 |
|      2^23 |           0 |                  0 |            1650 |             621 |             272 |             384 |
|      2^24 |           0 |                  0 |            3418 |            1416 |             518 |             766 |
|      2^25 |           0 |                  0 |            7411 |            3275 |            1836 |            1846 |
|      2^26 |           0 |                  0 |           14548 |            5385 |            5075 |            3655 |
|      2^27 |           0 |                  0 |                 |                 |            9850 |           10912 |

<details>
<summary>Click to see the java source code of the experiment</summary>

```java
blah
```
</details>

You can try out the javascript experiment in your browser at my [git hub page](https://josephmate.github.io/JavaVsJavascriptStringBuilder/).
<details>
<summary>Click to see the javascript source code of the experiment</summary>

```javascript
blah
```
</details>
If you would like to checkout a copy of the source code take a look at [my repo.](https://github.com/josephmate/JavaVsJavascriptStringBuilder).

# Why Javascript Does not Need a StringBuilder

Following the experiments, I wanted to know why.
My research let me to [this email chain](https://www.mail-archive.com/es-discuss@mozilla.org/msg10071.html) where Dr. Axel Rauschmayer
proposes adding StringBuilder to the next version of ECMAScript,
> Is this worthy of ES.next support? Or does it belong into a library?
> 
> The two concatenation approaches I know of are:
> 1. via +=
> 2. push() into an array, join() it after the last push()
> 
> (1) can’t possibly be efficient, but if (2) is OK on all(!) platforms, then a 
> library would be OK. However, given how frequently this is used, I would like 
> this to become part of the standard library.

[Tom Schuster responds](https://www.mail-archive.com/es-discuss@mozilla.org/msg10129.html) that it's no necessary.
> (1) is in  fact really good optimized in modern engines.  (In case you
are interested search for "Ropes: an alternative to strings")
> 
> I think today it's not a very good idea to propose methods on probably
existing performance problems.

I did more digging and learned the ropes are a datastructure the allow immutable String concatenations in O(lgN) time.

https://hg.mozilla.org/mozilla-central/file/tip/js/src/vm/StringType.h#l73

>    To avoid O(n^2) char buffer copying, a "rope" node (JSRope) can be created
>    to represent a delayed string concatenation. Concatenation (called
>    flattening) is performed if and when a linear char array is requested. In
>    general, ropes form a binary dag whose internal nodes are JSRope string
>    headers with no associated char array and whose leaf nodes are linear
>    strings.

# Question to the Reader
Could Java implement a similar optimization to the JVM?