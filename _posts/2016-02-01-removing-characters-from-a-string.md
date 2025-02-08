---
layout: post
title: Removing Characters From a String
date: 2016-02-01 23:20
author: matejoseph
comments: true
categories: [Uncategorized]
---
<a href="http://www.amazon.com/Programming-Interviews-Exposed-Secrets-Landing/dp/1118261364/ref=sr_1_1?ie=UTF8&qid=1454104897&sr=8-1&keywords=programming+interviews+exposed">Programming Interviews Exposed</a> asks us to remove characters from an input string, looking for the characters in a second string. I am unhappy with solution they provide, particularly because of the reasons they use to invalidate what I believe to the better solution. You can check out my solution to the problem on <a href="https://github.com/josephmate/coding_interview_practice/tree/master/remove_chars">my github repository</a>.

In this problem we are given the following C# signature:

```java
string removeChars( string str, string remove );
```

'str' is the original string we are removing characters from. 'remove' is the string that contains that characters that we need to remove from 'str'. For example if
str="hello world" and
remove=" wo"
the result should be "hellwrld" with all spaces, w's, and o's removed.
<h3>Synchronized</h3>
My first issue, is they recommend not using the StringBuffer for the wrong reason. The reason you do not want to use StringBuffer, is because it's synchronized in both Java and C#. Below is the Java source code from <a href="http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/lang/StringBuffer.java#StringBuffer.append%28char%29">grepcode</a>:

```java
    public synchronized StringBuffer append(char c) {
        super.append(c);
        return this;
    }
```

&nbsp;

The <a href="https://msdn.microsoft.com/en-us/library/aa245204%28v=vs.60%29.aspx">Microsoft documentation on StringBuffer says</a>,
<blockquote>The methods are synchronized where necessary so that all the operations on any particular instance behave as if they occur in some serial order that is consistent with the order of the method calls made by each of the individual threads involved."</blockquote>
Since the buffer does not exceed the scope of our method, and we know that only one thread is modifying the buffer, we don't need the expensive overhead of protecting against concurrent access. We can go with the cheaper solution of using a StringBuilder (in both Java and C#).
<h3>Extra Memory</h3>
Here the authors recommend building the string in the original character array and using two pointers to the array. One for the next non-deleted character and a second for where the next character is coming from. Ideally this is an excellent solution, if implementing correctly. I have annotated their solution with the issues.

```java
    // str has N characters
    // remove has M characters
    string removeChars( string str, string remove ){
      //NOTE: We use N extra memory because .toCharArray() copies the
      //array in string because strings are immutable
      char[] s = str.toCharArray();
      //Note: Again, we use M extra memory because .toCharArray copies
      char[] r = remove.toCharArray();
      bool[] flags = new bool[128]; // assumes ASCII!
      int len = s.Length;
      int src, dst;
      // Set flags for characters to be removed
      for( src = 0; src < len; ++src ){
        flags[r[src]] = true;
      }
      src = 0;
      dst = 0;
      // Now loop through all the characters,
      // copying only if they aren’t flagged
      while( src < len ){
        if( !flags[ (int)s[src] ] ){
          s[dst++] = s[src];
        }
        ++src;
      }
      // NOTE: Again we use N extra memory because the string constrcutor
      // copies the characters of s, into the new string
      return new string( s, 0, dst );
    }
```

Because Strings are immutable in Java and C#, <a href="http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/lang/String.java#String.toCharArray%28%29">toCharArray() in Java</a> and <a href="http://referencesource.microsoft.com/#mscorlib/system/string.cs,709">toCharArray() in C#</a> both have to allocate a new character array to prevent the caller from modifying the internal character array. Secondly you cannot force the <a href="http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/lang/String.java#String.%3Cinit%3E%28char[]%2Cint%2Cint%29">Java String(char[], int, int)</a> or the C# string(char[],int,int) [<a href="http://referencesource.microsoft.com/#mscorlib/system/string.cs,6">String</a>-><a href="https://github.com/dotnet/coreclr/blob/master/src/classlibnative/bcltype/stringnative.cpp#L111">StringNative</a>-><a href="https://github.com/dotnet/coreclr/blob/43b39a73cbf832ec13ec29ed356cb75834e7a8d7/src/vm/object.cpp#L2078">StringObject</a>] methods to use your char[] array. It has to copy it because the caller has a reference to the original character array and potentially modify it.

Overall, we used 2N + M memory, when the best possible solution in Java or C# uses 2N extra memory. We need N for the buffer and N for returning the result as a String. So we might as well have kept a StringBuilder anyways as it tremendously simplifies the code and uses potentially less extra memory because potentially a majority of the characters in the original string might be removed. I think this is an excellent example of what Knuth meant in "premature optimization is the root of all evil" because in this solution we didn't make it more efficient <span style="text-decoration:underline;"><strong>and</strong></span> we made it more complex.

If you really wanted to use no extra memory, this should have been implemented in C/C++ with the following signature:

```java
    void removeChars( char * str, char * rem );
```

Keeping the same code as above, but modifying str instead, allows us to incur no extra memory by modifying the contents of char*str in place.
<h3>Summary</h3>
Given the solution provided in the book and the limitations of immutable Strings in Java and C#, we might as well have used a StringBuilder, incurring potentially less excess memory and a simpler solution. StringBuffer should not be used as it's synchronized and we're not using Threads. Finally, we were really wanted to achieve the author's intended solution, we need to use C/C++ and change the method signature.

