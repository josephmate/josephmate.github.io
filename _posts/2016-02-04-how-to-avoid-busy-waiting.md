---
layout: post
title: How To Avoid Busy Waiting
date: 2016-02-04 20:54
author: matejoseph
comments: true
categories: [C++, Computer Science, concurrency, Java, java, Job Hunt, Programming]
---

<a href="http://www.amazon.com/Programming-Interviews-Exposed-Secrets-Landing/dp/1118261364/ref=sr_1_1?ie=UTF8&qid=1454104897&sr=8-1&keywords=programming+interviews+exposed">Programming Interviews Exposed</a> asks us to describe busy waiting and how to avoid it. The book provides a decent solution, but not the best solution. I'll first introduce the problem and then build upon the solution until we have our best solution.
<h2>Busy Waiting</h2>
You have busy waiting When one thread, waits for a result from another thread and you use and NOOP/empty loop to wait for that result. We will explore waiting for a result from two sub tasks to demonstrate and solve this problem.

```java
    Thread thread1 = new Thread(new Subtask1Runnable());
    Thread thread2 = new Thread(new Subtask2Runnable());
    thread1.start();
    thread2.start();
    while( waitinigForThread1() && waitingForThread2() ) {
        // empty loop
    }
    // do something with result from thread1 and thread2
```

You want to avoid this pattern because it wastes CPU cycles:
<ol>
	<li>Other threads you have running could of made use of those cycles</li>
	<li>Other processes sharing the same machine could have also used those cycles</li>
	<li>If the machine only has one CPU, thread2 could have made use of those cycles</li>
</ol>
&nbsp;
<h2>Poor Solution: Sleep</h2>
So the issue is that we're wasting CPU cycles by repeatedly checking the condition. So what if we made the thread sleep for a bit.

```java
    while( waitinigForThread1() && waitingForThread2() ) {
        Thread.sleep(1000); // Sleep for 1 second
    }
```

This is an improvement over busy waiting as we're now wasting less cycles. However
there are still some draw backs to this pattern:
<ol>
	<li>You're still wasting cycles. It may seem like you're wasting one cycle every
second. However, every time the thread wakes up it has to reload the thread's stack, registers, and it has to recompute if the other thread is done. There's still significant wastage.</li>
	<li>How do you set the sleep duration?
<ol>
	<li>Set it too long, and there is a long delay between the threads finishing and
the main thread recognizing that it's done</li>
	<li>Set it too short, and you increase the amount of wasted cycles</li>
</ol>
</li>
</ol>
<h2>Better But Still Poor Solution: Exponential Backoff</h2>
To take away a bit of the tuning, we can add an exponential backoff. To describe the process, imagine that your plane gets delayed for some unknown period of time. What do you do? Well first you might go to the washroom for a few minutes. Then you might get something to eat for half an hour. Then you might play a game on your cellphone for an hour. Finally, you'll read a book for the next couple of hours. Your waiting tasks become progressively larger as you wait more. That's the idea behind exponential backoff. You sleep for progressively longer durations as you wait for longer periods of time.

```java
    private static final int MAX_BACKOFF=1000*60*60; // 1 hour
    private static final int INITIAL_BACKOFF=100; // 100ms
    int backOff=INITIAL_BACKOFF;
    while( waitinigForThread1() && waitingForThread2() ) {
        Thread.sleep(backOff); // Sleep for 1 second
        backOff=backOff<<2; //bit shift to the left to multiply by 2
        if(backOff > MAX_BACKOFF) {
            backOff = MAX_BACKOFF;
        }
    }
```

&nbsp;

Now this solution isn't the best possible solution yet for a couple of reasons:
<ol>
	<li>Ideally we want the the main thread to wake up only once or twice
<ol>
	<li>Once when the first sub task completes</li>
	<li>A second time when the second sub tasks completes</li>
</ol>
</li>
	<li>We still have to play with the starting back off, and max back off parameters to get decent performance</li>
</ol>
<h2>Decent Solution: Locking or Monitors (Based on Programming Interviews Exposed)</h2>
As I hinted in the previous section, is there a way you can just wake up the sleeping thread once each task finishes? This is the solution that Programming Interview Exposed proposes. You can setup up some a lock and have the main thread wait on the lock. Then the two threads 'notify' when they have finished. Each 'notify' wakes the main thread and the main thread check to see if both threads finished.

This method uses the least possible cycles, but it is error prone. If not coded properly, you can end up with race conditions or deadlocks. Consider our two sub task example:

```java
    Object lock = new Object(); // shared object to synchronize on
    int doneCount = 0;
    Thread thread1 = new Thread(new Runnable() {
        public void run() {
            // ...
            // do sub task 1
            // ...
            synchronized(lock) {
                doneCount++;
                lock.notify();
            }
        }
    });
    Thread thread2 = new Thread(new Runnable() {
        public void run() {
            // ...
            // do sub task 2
            // ...
            synchronized(lock) {
                doneCount++;
                lock.notify();
            }
        }
    });

    // this is the main thread again
    synchronized(lock) {
        thread1.start();
        thread2.start();

        while(doneCount < 2) {
            lock.wait();
        }
    }
```

This solution is great as the main thread only wakes up twice. Once when sub task 1 finishes, and a second time when sub task 2 finishes.

However, in this example we can mess up in many ways:
<ol>
	<li>If I started the threads outside the synchronized block, the two threads
could finish before the main thread calls 'wait()'. This would leave the main thread sleeping forever and no future notifies are called.</li>
	<li>If I put the count outside of the synchronized block, the two threads might
update them at the same time and I'll get doneCount==1 instead of 2!</li>
	<li>If I forget to call notify() on one of the threads, then when the main thread
might never wake up because it didn't get one of the notifications.</li>
	<li>Placing the entire sub task in the synchronized block instead of just the
shared data accesses would of only let one sub task run at a time resulting in
no parallelism.</li>
  <li>In a previous version of the above pseudocode I wrote notify() and wait() instead of lock.notify() and lock.wait()! If you use a shared Object lock monitor, you have to remember to call notify() and wait() on that lock instead of this.</li>
</ol>
As a result, I do not recommend the solution presented in Programming Interviews Exposed.
<h2>Good Solution: Futures and Other Concurrency Libraries</h2>
Instead of trying to carefully code this pattern every time, you can use libraries that have already correctly implemented these patterns. In our example where we're waiting for two results from two threads so we can apply Futures.

```java
    ExecutorService executor = ...
    Future<String> future1 = executor.submit( new Callable<String>() {
        public String call() {
            /* do work for first result here */
        }
    });
    Future<String> future2 = executor.submit( new Callable<String>() {
        public String call() {
            /* do work for second result here */
        }
    } );
    String result1 = future1.get();
    String result2 = future2.get();
```

From the example we can immediately see that it's the most elegant solution, and less bug prone. But more importantly, we still use the least possible cycles as execution of the main thread only resumes when each of the futures complete.

Here we were looking at an example waiting on two sub tasks to complete. However, there are many different waiting scenarios and the standard libraries of C# and <a href="https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/package-summary.html">Java</a> provide many concurrency facilities to make developers' lives easier. Before trying to code up your own monitor or locking, see if you problem has been solved before.
<h2>Summary</h2>
If you want to avoid busy waiting you can apply the write concurrent library for your problem. If the library doesn't exist, you can try using monitors and locks, but be warned that there are many opportunities to mess up. Finally, don't use busy loops or sleep to wait on tasks.

Update 2021-04-08: Viliam, a reader on my wordpress spotted an error.
Thank you again!
I called notify() and wait() on the implicit `this` rather than the lock!
This adds to my argument that the best way to wait for something is to use a Future. There are so many ways to get it wrong.

