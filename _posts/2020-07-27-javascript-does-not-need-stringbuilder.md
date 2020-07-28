---
layout: post
title: "Javascript Does Not Need a StringBuilder"
date: 2020-07-27 20:14
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
Java allocates a new array containing the result of each successive concatenation resulting in the O(N^2) runtime complexity.

# Experiments
When I first learned this from a co-worker, I couldn't believe it!
I put together some experiments to confirm.
I measured how long it would take to do different powers of 10 of concatenations in Java and javascript.

If you would like to checkout a copy of the source code take a look at the last two sections (Java source and Javascript source) of this page or check [my repo](https://github.com/josephmate/JavaVsJavascriptStringBuilder).

You can try out the javascript experiment in your browser at my [git hub page](https://josephmate.github.io/JavaVsJavascriptStringBuilder/).


I've summarized how long each number of concats took in the table below.
I looked at using naive += String concatenation, Java's StringBuilder, and javascript's Array.join.

|           |               |                    | Chrome          | Chrome          | Firefox         | Firefox         | NodeJS          | NodeJS          |
|           | Java          | Java               | Javascript      | Javascript      | Javascript      | Javascript      | Javascript      | Javascript      |
| # concats | naive += (ms) | StringBuilder (ms) | naive += (ms)   | Array.join (ms) | naive += (ms)   | Array.join (ms) | naive += (ms)   | Array.join (ms) |
|-----------|---------------|--------------------|-----------------|-----------------|-----------------|-----------------|-----------------|-----------------|
|      2^10 |             4 |                  0 |               0 |               0 |               0 |               0 |               1 |               0 |
|      2^11 |            12 |                  0 |               0 |               0 |               0 |               0 |               1 |               0 |
|      2^12 |            28 |                  0 |               1 |               0 |               0 |               0 |               1 |               1 |
|      2^13 |            99 |                  0 |               1 |               1 |               1 |               0 |               1 |               0 |
|      2^14 |           374 |                  1 |               3 |               2 |               1 |               0 |               1 |               2 |
|      2^15 |          1197 |                  2 |               2 |               2 |               2 |               1 |               4 |               4 |
|      2^16 |          2961 |                  1 |               8 |               4 |               3 |               3 |              10 |               6 |
|      2^17 |          4505 |                  3 |              15 |               8 |               6 |               4 |              29 |              12 |
|      2^18 |         21945 |                  5 |              30 |              18 |              14 |              11 |              29 |              22 |
|      2^19 |         94077 |                  6 |              79 |              37 |              12 |              18 |             112 |              54 |
|      2^20 |        412675 |                 11 |             144 |              71 |              42 |              44 |             206 |              93 |
|      2^21 |       1886317 |                 39 |             408 |             156 |              68 |              70 |             460 |             245 |
|      2^22 |       8529431 |                 70 |             906 |             318 |             172 |             149 |             813 |             425 |
|      2^23 |      TOO LONG |                119 |            1650 |             621 |             272 |             384 |            2233 |             753 |
|      2^24 |      TOO LONG |                209 |            3418 |            1416 |             518 |             766 |            3599 |            1920 |
|      2^25 |      TOO LONG |                492 |            7411 |            3275 |            1836 |            1846 |            6384 |            4048 |
|      2^26 |      TOO LONG |                898 |           14548 |            5385 |            5075 |            3655 |           13266 |           20333 |
|      2^27 |      TOO LONG |               1738 |             OOM |             OOM |            9850 |           10912 |           33434 |             OOM |

At 2^23, the naive Java String concatenation took long to run.
I gave up at 2^22 when it took over 2 hours!
Each doubling of the input size results in more than double the 
However, with javascript it's hard to tell.
The growth looks kind of linear but maybe log linear.

# Why Javascript Does not Need a StringBuilder

Following the experiments, I wanted to know why.
My research led me to [this email chain](https://www.mail-archive.com/es-discuss@mozilla.org/msg10071.html) where Dr. Axel Rauschmayer
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

# Discussion
Could Java implement a similar optimization to the JVM so that people who overlook this mistake aren't hit with an O(N^2) but a O(NlgN) algorithm instead?

Are ropes good enough? At 2^24 to 2^27 we're measuring the concatinations in seconds! You could argue that's good enough because that's the memory limit of the browser. What about javascript that does not run the in browser but in the backend? Can we do better?

# Java source

```java
public static void main(String [] args) {
    final int base;
    final int startPower;
    final int concatPowerLimit;
    final int powerLimit;

    if(args.length >= 4) {
        base = Integer.parseInt(args[0]);
        startPower = Integer.parseInt(args[1]);
        concatPowerLimit = Integer.parseInt(args[2]);
        powerLimit = Integer.parseInt(args[3]);
    } else {
        base = 2;
        startPower = 1;
        concatPowerLimit = 21; // at least 2 hours to compute the 22nd one
        powerLimit = 27;
    }

    for(int power = startPower; power <= 27; power++) {
        int size = (int)Math.pow(2, power);
        runStringBuilderExperiment(base, power, size);
        if (power <= concatPowerLimit) {
            runConcatExperiment(base, power, size);
        }
    }
}

public static void runConcatExperiment(int base, int power, int size) {
    String result = "";
    long start = System.currentTimeMillis();
    for (int i = 0; i < size; i++) {
        result += (i%10);
    }
    long end = System.currentTimeMillis();
    long duration = end - start;
    System.out.println(String.format("concat %d^%d %d %d %d", base, power, size, result.length(), duration));
}

public static void runStringBuilderExperiment(int base, int power, int size) {
    StringBuilder builder = new StringBuilder();
    long start = System.currentTimeMillis();
    for (int i = 0; i < size; i++) {
        builder.append(i%10);
    }
    String result = builder.toString();
    long end = System.currentTimeMillis();
    long duration = end - start;
    System.out.println(String.format("builder %d^%d %d %d %d", base, power, size, result.length(), duration));
}
```

# Javascript source

```javascript
function runConcatExperiment(size) {
    var result = "";
    var start = new Date().getTime();
    for (var i = 0; i < size; i++) {
        result += (i%10);
    }
    var end = new Date().getTime();
    var duration = end - start;
    console.log("concat %d %d %d", size, result.length, duration);
    return duration;
}

function runArrayJoinExperiment(size) {
    var builder = [];
    var start = new Date().getTime();
    for (var i = 0; i < size; i++) {
        builder.push(i%10);
    }
    var result = builder.join();
    var end = new Date().getTime();
    var duration = end - start;
    console.log("concat %d %d %d", size, result.length, duration);
    return duration;
}

function addRow(base, power, size, concatDuration, arrayJoinDuration) {
    var resultsTable = document.getElementById("results");
    var tableRow = document.createElement("tr");
    var powerCell = document.createElement("td");
    powerCell.innerText = base + "^" + power;
    tableRow.append(powerCell);
    var sizeCell = document.createElement("td");
    sizeCell.innerText = size;
    tableRow.append(sizeCell);
    var concatCell  = document.createElement("td");
    concatCell.innerText = concatDuration;
    tableRow.append(concatCell);
    var joinCell  = document.createElement("td");
    joinCell.innerText = arrayJoinDuration;
    tableRow.append(joinCell);
    resultsTable.appendChild(tableRow);
}

function runExperiment() {
    var baseInput = document.getElementById("base");
    var base = baseInput.value;
    var powerLimitInput = document.getElementById("powerLimit");
    var powerLimit = powerLimitInput.value;
    for(var power = 1; power <= powerLimit; power++) {
        var size = Math.pow(2, power);
        var concatDuration = runConcatExperiment(size);
        var arrayJoinDuration = runArrayJoinExperiment(size);
        addRow(base, power, size, concatDuration, arrayJoinDuration);
    }
}
```