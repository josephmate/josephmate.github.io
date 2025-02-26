---
layout: post
title: Preparing For Interviews
date: 2016-01-29 22:03
author: matejoseph
comments: true
categories: [C++, github, Gradle, interviews, java, Job Hunt, Programming]
---
I have completed my Master's Thesis and all that's left is paper work. As a result, it's time to prepare for interviews. I'm going through '<a href="http://www.amazon.com/Programming-Interviews-Exposed-Secrets-Landing/dp/1118261364/ref=sr_1_1?ie=UTF8&amp;qid=1454104897&amp;sr=8-1&amp;keywords=programming+interviews+exposed">Programming Interviews Exposed</a>' as practice. You can follow my progress at my <a href="https://github.com/josephmate/coding_interview_practice/">github repository</a>. Wish me luck in the interview process! I will occasionally make posts if I find something interesting, or can provide deeper insight than the book.

So far I have learned the following things, unrelated to the book, by going through the book:
<ul>
	<li>How to properly use github
<ul>
	<li>For the longest time, I've been committing to the master branch instead of merging into master from a separate branch</li>
</ul>
</li>
	<li>How difficult it is to handle Unicode characters in C++
<ul>
	<li>Take a look at my solution to finding the <a href="https://github.com/josephmate/coding_interview_practice/tree/master/first_non_repeat_char">first non repeating character</a>.</li>
</ul>
</li>
	<li>How to use Gradle for your java builds of small projects
<ul>
	<li>I've found Gradle far more convenient that using ant/maven and ivy</li>
</ul>
</li>
	<li>How to do unit testing in C++
<ul>
	<li>Previously I've done lots of unit testing in Java, but never in C++</li>
	<li>Check out my <a href="https://github.com/josephmate/coding_interview_practice/blob/master/bst/test_bst.cpp">tests for my C binary search tree solution</a></li>
</ul>
</li>
</ul>
Also, by implementing the solutions I've been able to practice many things which I would not get exposure to by working through the problems in my head.

Firstly, I reviewed my C/C++ knowledge which hasn't been used intensely since undergrad. I'm now comfortable with things like manipulating pointers, differences between Java and C++, and the standard library. One interesting caveat is that when you want to used a HashTable data structure, do not used a map&lt;&gt;. That's actually map backed by tree data structure and some interviews will notice this. You'll want to use unordered_map&lt;&gt;, which was added in the C++11 standard.

Secondly, by implementing and testing the solution, you discover edge cases you might not have considered by working the problem through your head. I hit many more edge cases than I expected to with my <a href="https://github.com/josephmate/coding_interview_practice/tree/master/heap">heap implementation</a>.

Finally, during interviews you will be asked to code up the solution. You do not want to waste your precious time recalling C++ or Java libraries. You want to be able to focus as much of your energy on tackling the problem, instead of tackling the language.

Wish me luck!

