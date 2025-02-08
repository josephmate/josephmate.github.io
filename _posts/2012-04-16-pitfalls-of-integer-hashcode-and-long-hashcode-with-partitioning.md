---
layout: post
title: Pitfalls of Integer.hashCode() and Long.hashCode() With Partitioning
date: 2012-04-16 08:51
author: matejoseph
comments: true
categories: [hashcode, Java, java, partitioning, partitions]
---
<h1>Problem</h1>
So we want to partition our data into nice even chunks. If you use Integer.hashCode(), Long.hashCode(), or modding by the number of your partitions then the size of each partition will depend on the distribution of your data and the number of partitions. Here's a quick example that I wrote up to demonstrate.

Here we have the partition size as 8 and all the numbers are even. I place each number into a bucket by taking the modulus of it. Here are the results:

```java
import java.util.Random;

public class HashCodeIssue {

    private static Random random;

    public static void main(String [] args) {
        int partitions = Integer.parseInt( args[0] );
        long seed = Integer.parseInt( args[1] );
        random = new Random(seed);
        int [] buckets = new int[partitions];

        for(int c = 0; c < 100000; c++) {
            buckets[hash(getNextNumber()) % partitions]++;
        }

        for(int c = 0; c< buckets.length; c++) {
            System.out.println('size of bucket ' + c + ': ' + buckets[c]);
        }

    }

   /**
    * Using a distribution of numbers divided evenly over
    * even numbers.
    */
    private static Integer getNextNumber() {
        return new Integer(random.nextInt(Integer.MAX_VALUE/2) * 2);
    }

   /**
    * Returning the default hashCode() implementation.
    */
    public static int hash(Integer i) {
        return i.hashCode();
    }
}
jmate@jmate-Satellite-C650D:~/Desktop/hashcode-issue$ java HashCodeIssue 8 890453985
size of bucket 0: 24781
size of bucket 1: 0
size of bucket 2: 24959
size of bucket 3: 0
size of bucket 4: 25163
size of bucket 5: 0
size of bucket 6: 25097
size of bucket 7: 0
jmate@jmate-Satellite-C650D:~/Desktop/hashcode-issue$ java HashCodeIssue 8 234985345
size of bucket 0: 25250
size of bucket 1: 0
size of bucket 2: 24735
size of bucket 3: 0
size of bucket 4: 24866
size of bucket 5: 0
size of bucket 6: 25149
size of bucket 7: 0
```

<h1>Root Cause</h1>
Lets take a look at the source code of Integer.hashCode() to find out why the numbers are not being evenly distributed, despite using a hash.

<a title="java/lang/Integer.java" href="http://www.docjar.com/html/api/java/lang/Integer.java.html" target="_blank">java/lang/Integer.java</a>

```java
/**
* Returns a hash code for this {@code Integer}.
*
* @return  a hash code value for this object, equal to the
*          primitive {@code int} value represented by this
*          {@code Integer} object.
*/
public int hashCode() {
    return value;
}
```

<a title="java/lang/Long.java" href="http://www.docjar.com/html/api/java/lang/Long.java.html" target="_blank">java/lang/Long.java</a>

```java
/**
 * Returns a hash code for this {@code Long}. The result is
 * the exclusive OR of the two halves of the primitive
* {@code long} value held by this {@code Long}
* object. That is, the hashcode is the value of the expression:
*
* <blockquote>
*  {@code (int)(this.longValue()^(this.longValue()>>>32))}
* </blockquote>
*
* @return  a hash code value for this object.
*/
public int hashCode() {
    return (int)(value ^ (value >>> 32));
}
```

These hash codes are implemented for speed, as a result they don't do a good job of jumbling up the numbers. Unfortunately, for developers who want partitions of uniform sizes, this is not a good thing. The issue is that all the values share a common divisor with the the number of partitions (4*2 and y=x*2). As a result, all the values end up in the same buckets.
<h1>Solution</h1>
There are a couple of solutions to this problem.
<h2>1. Adjust The Number Of Partitions</h2>
This seems unintuitive at first, but REDUCING the number of partitions to 7 will flatten out the sizes of our partitions because there are a lot less even numbers that share a common divisor with the prime number seven.

```bash
jmate@jmate-Satellite-C650D:~/Desktop/hashcode-issue$ java HashCodeIssue 7 890453985
size of bucket 0: 14153
size of bucket 1: 14335
size of bucket 2: 14377
size of bucket 3: 14265
size of bucket 4: 14364
size of bucket 5: 14179
size of bucket 6: 14327
jmate@jmate-Satellite-C650D:~/Desktop/hashcode-issue$ java HashCodeIssue 7 234985345
size of bucket 0: 14193
size of bucket 1: 14256
size of bucket 2: 14374
size of bucket 3: 14362
size of bucket 4: 14267
size of bucket 5: 14321
size of bucket 6: 14227
```
<h2>2. Use A Better Hash</h2>
Now a lot of times you have no idea what distribution of numbers you have but you still want to be able to distribution the numbers uniformly over your partitions. Or, you need a particular number of partitions that will have a common divisor with a lot of numbers coming in. A safe way to do this is to use a hash function that jumble up the numbers better. In the code above, instead of using the default hash implementation, I use the md5 hash function.

```java
/**
* Hash function that uses MD5.
*/
private static int hash(Integer i, int partitions)
throws java.security.NoSuchAlgorithmException {
    MessageDigest m = MessageDigest.getInstance('MD5');
    m.reset();
    m.update(toBytes(i));
    byte[] digest = m.digest();
    BigInteger bigInt = new BigInteger(1,digest);
    return bigInt.mod(new BigInteger(1,toBytes(partitions))).abs().intValue();
}

private static byte[] toBytes(Integer ii) {
    int i = ii.intValue();
    byte[] result = new byte[4];

    result[0] = (byte) (i >> 24);
    result[1] = (byte) (i >> 16);
    result[2] = (byte) (i >> 8);
    result[3] = (byte) (i /*>> 0*/);

    return result;
}
jmate@jmate-Satellite-C650D:~/Desktop/hashcode-issue$ java HashCodeIssue 8 890453985
size of bucket 0: 12629
size of bucket 1: 12510
size of bucket 2: 12550
size of bucket 3: 12470
size of bucket 4: 12689
size of bucket 5: 12339
size of bucket 6: 12311
size of bucket 7: 12502
jmate@jmate-Satellite-C650D:~/Desktop/hashcode-issue$ java HashCodeIssue 8 234985345
size of bucket 0: 12472
size of bucket 1: 12533
size of bucket 2: 12632
size of bucket 3: 12634
size of bucket 4: 12489
size of bucket 5: 12352
size of bucket 6: 12616
size of bucket 7: 12272
```

<h1>Conclusion</h1>
If your performance depends on the size of your partitions, you should take special care when using Integer.hashCode() or Long.hashCode(). By carefully selecting the number of partitions, you can avoid the problem. If you uncertain about the distribution of numbers you have or you cannot change the number of partitions, you should use a hash function that does a better job of jumbling up the numbers.

Now I realize this example was really contrived because the numbers being partitioned were even. I chose it because it's really simple to understand and illustrate that  if you don't use a good hash function or have the correct number of partitions, this problem is going bite you when your numbers have a special distribution. The last time I saw this problem in the real world is when the numbers we were partitioning on usually had a particular end like XXXXX7831. If you have encountered this problem, post a comment! I am interesting in hearing about it.

