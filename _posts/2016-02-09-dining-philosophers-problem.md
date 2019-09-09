---
layout: post
title: Dining Philosophers Problem
date: 2016-02-09 03:18
author: matejoseph
comments: true
categories: [Computer Science, concurrency, Java, Job Hunt, Programming]
---
The book I'm going through, <a href="http://www.amazon.com/Programming-Interviews-Exposed-Secrets-Landing/dp/1118261364/ref=sr_1_1?ie=UTF8&amp;qid=1454104897&amp;sr=8-1&amp;keywords=programming+interviews+exposed">Programming Interviews Exposed</a>, presents the Dining Philosophers Problem and their solution to the problem. Firstly, their explanation for their solution is weak because they do not describe the underlying principle they applied to prevent deadlocks. Secondly, their implementation of their solution still results in starvation. Finally, with a few modifications, I'll go over <a href="https://github.com/josephmate/coding_interview_practice/tree/master/dining_philosophers">my improved solution</a> and compare the results of the two.
<h3>The Problem</h3>
This question comes from the <a href="http://www.amazon.com/Programming-Interviews-Exposed-Secrets-Landing/dp/1118261364/ref=sr_1_1?ie=UTF8&amp;qid=1454104897&amp;sr=8-1&amp;keywords=programming+interviews+exposed">Programming Interviews Exposed</a> book:
<blockquote>Five introspective and introverted philosophers are sitting at a circular table. In front of each philosopher is a plate of food. A fork (or a chopstick) lies between each philosopher, one by the philosopher’s left hand and one by the right hand. A philosopher cannot eat until he or she has both forks in hand. Forks are picked up one at a time. If a fork is unavailable, the philosopher simply waits for the fork to be freed. When a philosopher has two forks, he or she eats a few bites and then returns both forks to the table. If a philosopher cannot obtain both forks for a long time, he or she will starve. Is there an algorithm that will ensure that no philosophers starve?</blockquote>
<h3>Applying an Order to Resources</h3>
In Programming Interviews Exposed, the authors recommend that all except one philosopher grabs the left fork, then the right. They mention this prevents the dead lock with no explanation or how someone would come up with this solution.

Firstly, you get deadlocks where there is a cycle in your resource allocation graph. In these graphs, you have directed edges from resources to threads when that thread has a lock on that resource. You have an edge from a thread to a resource when that thread is waiting for that resource. A cycle in this graph describes a scenario where no one can make progress because everyone in the cycle is waiting on each other to finish.
<pre>
Chopstick1 -> Philospher1 -> C2 -> P2 -> C3 -> P3 -> C4 -> P4 -> C5 -> P5
    ^                                                                  /
     \________________________________________________________________/</pre>
One way to prevent deadlocks, is to ensure that you never have cycles. If you can create an order to describe your resources and if all threads acquire the resources in that order, then you will never have deadlocks.

To solve the dining philosophers problem, you can assign a number to each fork, giving it an order. You always grab the lowest fork first. For instance philosopher 5 needs forks 1 and 5. Since 1 is the lowest, philosopher 5 will try to grab this fork first. This is how you arrive at the solution where all except one philosopher graphs the left first.

Another real world problem you can see this resource ordering technique applied to is bank accounts. Imagine you're implementing a transfer from a withdrawing account into the depositing account. The naive way would be to first lock the withdrawing account, then lock the depositing account. However, this generates a cycle when the two accounts are trying to transfer to each other at the same time. First both transfers lock the withdrawing account, locking both accounts. Now both transfers try to lock the depositing accounts and waits forever because both accounts are already locked. Instead of locking the withdrawing account, then the depositing account, first lock the account with the lower ID. This will prevent all possible deadlocks.
<h3>Barging</h3>
The idea behind Programming Interviews Exposed's solution is correct as explained in the previous section, but their implementation is not. At a high level, their solution involves monitors and looks like this:

```java
    synchronized(lowerLock) {
        synchronized(higherLock) {
            // philosopher eats the food
        }
    }
```

&nbsp;

Unfortunately, Java monitors do not have a queue for threads waiting for a monitor. As a result, any thread waiting for a monitor even the most recently added one can be the next one allowed into the monitor. This is contrary to a programmer's intuition as we expect the oldest waiter to be allowed in next. Not allowing the longest waiting thread is called <span style="text-decoration:underline;"><strong>barging</strong></span>. When the lives of philosophers are at stake, you want to prevent barging because it allows starvation.

To illustrate, I implemented the books solution of the problem in Java and recorded how many times each philosopher got to eat, with various eating durations:
<pre>When philosopher takes no time to eat:
count   philosopher_id
      1 0
    104 1
    390 3
When philosopher takes 100 ms to eat:
count   philosopher_id
      8 0
     28 1
     76 2
    371 3
     12 4
When philosopher takes 1 second to eat:
count   philosopher_id
     16 0
     48 1
     74 2
    337 3
     20 4</pre>
From these results, we can see that philosopher 3 gets to eat an unfair amount of time and that when it takes no time to eat, some philosophers never get to eat.
<h3>Preventing Barging</h3>
The easiest way to prevent barging in Java is to use fair locks. ReentrantLock's constructor accepts a boolean fairness parameter. By setting this to true, it fulfills the .lock() calls in the order that they come, unlike a monitor. The resulting code would look something like:

```java
    ReentrantLock [] locks = new ReentrantLock[5];
    for(int i = 0 ; i < locks.length; i++) {
        locks[i] = new ReentrantLock(true); // make it fair
    }
    // ...
    // ...

    lowerLock.lock();
    try {
        higherLock.lock();
        try {
            // philosopher eats the food
        } finally {
            higherLock.unlock();
        }
    } finally {
        lowerLock.unlock();
    }
```

&nbsp;

I re-ran the problem with the fair locks and recorded how many times each philosopher ate:
<pre>When philosopher takes no time to eat
count   philosopher_id
    108 0
     38 1
     48 2
    263 3
     38 4
When philosopher takes 100ms to eat
count   philosopher_id
     85 0
     95 1
     99 2
    131 3
     85 4
When philosopher takes 1 second to eat
count   philosopher_id
     89 0
     91 1
    100 2
    127 3
     88 4</pre>
From these results, we see that in all scenarios, no philosopher starved. Secondly, we see a more even distribution of eating compared to the previous barging solution. Unfortunately we are unable to eliminate all the unfairness as the resource ordering we used to solve the problem gives higher priority to the second highest philosopher resulting in an inherent unfairness. From this example we can see that if you have a scenario where you need to prevent starvation, make sure to use fair locks instead of monitors.
