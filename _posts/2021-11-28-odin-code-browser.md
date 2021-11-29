---
layout: post
title: "OdinCodeBrowser: Navigate Code Like in your IDE but on a Static Webpage"
date: 2021-11-28 11:59
author: joseph.mate
comments: true
categories: [Java, ExecutorService]
---

For the last couple of months I've been working on a [Java code navigator that fits into your browser](https://github.com/josephmate/OdinCodeBrowser/#readme).

This won't replace your IDE.
Code discussions are handled better by github or gitlab.
However, if you want to share links to your code with others that let them navigate all the way down to the JDK, then give this a try.

If you don't really know what to do with this, then have some fun.
I challenge you to start from [JDK's JarFile](https://josephmate.github.io/OdinCodeBrowser/jdk17/java/util/jar/JarFile.html) and see if you can reach my favourite class: HashMap.

Before I fell in love with Intellij,
I used [grepcode](https://web.archive.org/web/20150318024036/http://www.grepcode.com/) to jump into the JDK's source code to figure out what was going on with my code.
That website died a long time ago.
This is my attempt at a successor.
 You can jump around the repositories by clicking on types, method calls, and variables.
It's nowhere near Intellij or grepcode yet.
You can look at the massive laundry list in the Future Work section of the readme.
The two things that annoy me the most Odin lacks are chained method calls
(like: thisMethod().someVisibleField.thenAnotherMethod())
and method overloading
(String.valueOf(int) vs String.valueOf(long)).
The coolest things I want to see in the project is linking from Java to native code and generalizing to other languages.

Let me know what you think!

<script src="https://utteranc.es/client.js"
        repo="josephmate/josephmate.github.io"
        issue-number="43"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
