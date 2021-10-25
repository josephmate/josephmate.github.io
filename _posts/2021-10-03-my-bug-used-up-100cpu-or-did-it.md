---
layout: post
title: "100% CPU: My Fault?"
date: 2021-10-03 19:22
author: joseph.mate
comments: true
categories: [Java, ExecutorService]
---

Update: thanks to:

* [Omar Elrefaei](https://github.com/Omar-Elrefaei) for the [PR](https://github.com/josephmate/josephmate.github.io/pull/42) that fixed the
  formatting of this document.
* [/u/thorn-harvestar/](https://www.reddit.com/r/programming/comments/q1w4od/comment/hfhw9a5/?utm_source=share&utm_medium=web2x&context=3), [/u/philipTheDev/](https://www.reddit.com/r/programming/comments/q1w4od/comment/hfhabww/), and [/u/vips7L/](https://www.reddit.com/r/java/comments/q1wlyw/comment/hfhq8pg/): for identifying another root cause. I shouldn't be using JDK8 anymore!
* [/u/SirSagramore/](https://www.reddit.com/r/programming/comments/q1w4od/comment/hfi5vx3/) and [/u/wot-teh-phuck](https://www.reddit.com/r/programming/comments/q1w4od/comment/hfi9478/): for proving that it really was my fault :(. Premature optimization is the root of all evil.

# The Sirens are blaring

A couple years ago I received a bug report claiming that I caused 100% CPU util on a VM when it should have been idle.
I was suspicious at first because I try my best to avoid patterns such as `while(true)` and `for(;;)`.

The support engineer that wrote the initial report included the evidence for the 100% CPU util.
```
$ top
  PID USER      PR  NI    VIRT    RES  %CPU  %MEM     TIME+ S COMMAND
    1 root      20   0 1832.5m  20.6m 100.7   1.0   4:11.04 S java Main
   25 root      20   0    1.6m   1.1m   0.0   0.1   0:00.04 S sh
   38 root      20   0    1.9m   1.2m   0.0   0.1   0:00.00 R  `- top
```
The JVM was indeed using up 100% CPU.
However, how did they track me down?

# How did they find me?

The bug was triaged by a software developer.
First, he got a list of thread IDs related to the Java process using `top -H -p <pid>` with pid from the previous top command:
```
$ top -H -p 1
  PID USER      PR  NI    VIRT    RES  %CPU  %MEM     TIME+ S COMMAND
    1 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.02 S java Main
    7 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.32 S  `- java Main
    8 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.00 S  `- java Main
    9 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.00 S  `- java Main
   10 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.01 S  `- java Main
   11 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.00 S  `- java Main
   12 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.00 S  `- java Main
   13 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.00 S  `- java Main
   14 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.03 S  `- java Main
   15 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.05 S  `- java Main
   16 root      20   0 1832.5m  20.6m   0.0   1.0   0:00.00 S  `- java Main
   17 root      20   0 1832.5m  20.6m   0.7   1.0   0:00.42 S  `- java Main
   18 root      20   0 1832.5m  20.6m  99.9   1.0   3:08.69 R  `- java Main
```
The thread with ID 18 was consuming 100% CPU.

Then he took the thread ID (18) and converted it to hex, resulting in 12.
```
$ printf "%x\n" 18
12
```

Then he grabbed the thread dump using `kill -s QUIT <pid>` to get a thread dump sent to standard out.
Then he searched for the string `nid=0x12` to find the stack trace of the thread consuming 100% CPU.
```
$ kill -s QUIT 1
...
"joseph-mate-pool-1-thread-1" #8 prio=5 os_prio=0 tid=0x0000558708d94000 nid=0x12 runnable [0x00007f23e77fd000]
   java.lang.Thread.State: RUNNABLE
        at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.poll(ScheduledThreadPoolExecutor.java:809)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1073)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
        at java.lang.Thread.run(Thread.java:748)
...
```
Doh! That's definitely my thread.
I'm glad I provided a thread name factory to my thread pool to make debugging easier for other developers.
It looks like something I put in the `ScheduledThreadPoolExecutor` was eating up all the CPU.

# Was it my bad?

My first attempt was to look at what I scheduled to run in the thread pool.
I found the code and, to my surprise, it did not look like it should use very much CPU:
```
executorService.scheduleWithFixedDelay(
    () -> doSomethingReallyCheap(),
    0, // Initial delay
    10, // delay
    TimeUnit.MINUTES
);
```
This was something that runs only every 10 minutes and on top of that it was not CPU bound.
There were other similar blocks of code scheduling this kind of work on application startup; again, all of them were IO bound.
On top of that, I intentionally chose `scheduleWithFixedDelay` in case it unexpectedly took longer than 10 minutes so that it would not overload the thread pool by adding more work when the previous instance of that task has not completed yet.
At this point I was out of ideas, so I asked a coworker to double check my work.

My co-worker took a look and agreed with me.
It doesn't look like an issue with `doSomethingReallyCheap()`.
However, he took a closer look at the stack trace and noticed it wasn't stuck in `doSomethingReallyCheap()`;
it was actually stuck in `DelayedWorkQueue.poll(ScheduledThreadPoolExecutor.java:809)`!
He suggested I Google a bit and see what comes up.

# YES! Vindicated!

Googling for `ScheduledThreadPoolExecutor 100% CPU` brings up a StackOverflow post that describes the same symptoms:
[ScheduledExecutorService consumes 100% CPU when corePoolSize = 0](https://stackoverflow.com/questions/53401197/scheduledexecutorservice-consumes-100-cpu-when-corepoolsize-0)

The author had the exact same line of code as me:
> `ScheduledExecutorService executorService = Executors.newScheduledThreadPool(0);`

I intentionally used this because I did not want to continuously consume the resources needed for a thread for something that ran once every 10 minutes!
They even warn you not to set it to 0 in the documentation:
> Additionally, it is almost never a good idea to set corePoolSize to zero or use allowCoreThreadTimeOut because this may leave the pool without threads to handle tasks once they become eligible to run.

But I was okay with that, because I did not care about waiting for the pool to create a new thread.
The 10 minute delay was not super strict.
> corePoolSize - the number of threads to keep in the pool, even if they are idle, unless allowCoreThreadTimeOut is set
So I was expecting the pool to only keep threads around temporarily.

At this point, I switched to `corePoolSize=1` to work around this unexpected behavior.
I was upset that I had to always consume a thread, but that's the best I could do with the time available.

However, I was really bothered by that unexpected behavior.
Did I interpret the JavaDocs incorrectly?
Scrolling down, I was very relieved by the top answer.
> This is a known bug: JDK-8129861 https://bugs.openjdk.java.net/browse/JDK-8129861. It has been fixed in JDK 9. 

YES! Vindicated!

# WHY!?

I still wasn't satisfied.
How could `corePoolSize=0` cause JDK to consume 100% CPU?
The details from the JDK bug didn't give me any hints:

> Looks like yet another zero-core thread bug, fixed in the Great jsr166 jdk9 integration 

All this tells me is that they planned to make some massive changes to `ScheduledThreadPoolExecutor` in JDK9.
It also tells me that they will probably never fix this bug in JDK8.
That's why if you try this right now in the latest JDK8, you will likely reproduce the problem.
Looks like we'll need to dive into the source code.
We'll start our investigation at the top of the stack trace from the thread dump earlier: `ScheduledThreadPoolExecutor$DelayedWorkQueue.poll(ScheduledThreadPoolExecutor.java:809)`.
By running a minimal reproducible example in IntelliJ and placing a breakpoint near `DelayedWorkerQueue.poll` we can find the infinite loop.
Popping a few stack frames we see `ThreadPoolExecutor.getTask` as the infinite loop:

```
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        ...
        int wc = workerCountOf(c);
        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        ...
    }
}
```

1. Note that `keepAliveTime` is 0 here.
2. Worker count (`wc`) is 1 since there is one active thread
3. `corePoolSize` is 0
4. This means that `timed = true`
5. Next we check if we can cull a worker
6. There are 2 conditions that always prevent us from culling a worker
    1. `wc` > 1; but in this case we only have 1 worker
    2. `workQueue.isEmpty()` can never be empty since it's scheduled work. It will always be in the queue.
5. That means we poll the `workerQueue` with `keepAliveTime`=`timeout`=0!
6. This in turn means the poll returns immediately
7. Then we're back to the beginning of the `for(;;)` loop and none of the
   conditions changed since the last iteration.

In contrast, if `corePoolSize` is 1, then `time=false` and we take from the
`workQueue` instead, which blocks.
This explains why setting `corePoolSize=0` is a workaround.

This might be fixable if we change the `wc > 1` condition to `wc > 0` and change
the `isEmpty()` condition to check if something needs to execute soon. However,
this creates a massive design issue. We are using the current worker thread to 
poll for tasks.
We can't just change the cull worker condition to `wc > 0`. 
We should have heeded the JavaDoc's warning and not used `corePoolSize=0`.

To fix the JDK limitation, maybe we can schedule a new thread to be created when the scheduled task's deadline approaches and exit the current thread!
However, there's a lot of stuff left to figure out.
For example, what thread is going to do the work of creating the thread when the scheduled task needs to run?

# What could I have done better?

**RTFM:** It's impossible for me to have known that `corePoolSize=0` would not work.
I could have read the JavaDocs more closely and heeded their warning of avoiding `corePoolSize=0`.
However, even if I had read that line, I don't think it would have stopped me because in my mind the use case made sense.

**Testing:** What's embarrassing is that the problem is reproducible every time and 100% CPU should be pretty easy to spot.
I should have monitored the CPU while testing it.
But I only look at the CPU when testing my code for a performance change.
I did not anticipate this change to have a performance impact.
Should I check the CPU for all of my changes?
Or should there be an automated test that detects significant differences in CPU utilization between code changes?

**Upgrade:** If I kept the JDK updated, I would have never encountered this problem.
The problem occurred back in 2018 or 2019, which was around the time JDK 11 was released which would have included the fix.

**Unoptimized:** I never should have tried to optimize the thread pool!
Two commenters summed it up nicely, calling me out on my sin.

> You're guilty of one of the biggest crimes that you can commit in all of Computer Science - preemptive optimization. Consuming a single thread in perpetuity uses almost no meaningful resources. Any attempt to optimize this was pointless at best, and dangerous at worst - as you found out. -[/u/SirSagramore/](https://www.reddit.com/r/programming/comments/q1w4od/comment/hfi5vx3/)

> I've found that it's rarely a good idea to optimise prematurely without facts/numbers to back it up. It's just one OS thread and the OS is probably smarter than the developer when it comes to managing resources in the long run. -[/u/wot-teh-phuck](https://www.reddit.com/r/programming/comments/q1w4od/comment/hfi9478/)

# Try it yourself

If you want to try this yourself, compile and run the below code using JDK8.
It reproduced at the time I wrote this article because OpenJDK only fixed the issue in JDK9.
```
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public final class Main {

  public static void main(String [] args) throws Exception {
    ScheduledExecutorService executorService = Executors.newScheduledThreadPool(0);
    executorService.scheduleWithFixedDelay(
        () -> System.out.println("Hello World!"),
        0, // Initial delay
        10, // delay
        TimeUnit.MINUTES
    );
    while (true) {
      Thread.sleep(10000);
    }
  }

}
```

Just to double check, I tried out the above code with JDK9 and CPU util was low.
```
$ top
  PID USER      PR  NI    VIRT    RES  %CPU  %MEM     TIME+ S COMMAND
    1 root      20   0 3008.5m  29.1m   0.0   1.5   0:00.76 S java Main
   27 root      20   0    1.6m   0.0m   0.0   0.0   0:00.02 S sh
   34 root      20   0    7.9m   1.3m   0.0   0.1   0:00.01 R  `- top
```

# Final Thoughts

I was relieved that I was not the culprit.
However, I still share some of blame because I could have caught this issue earlier by monitoring the CPU while testing.
Fortunately, I'm grateful we had strong test engineers that caught it before it was released.
What do you think?
Was this my fault?

Update: Yes it was my fault. Premature optimization is the root of all evil.

<script src="https://utteranc.es/client.js"
        repo="josephmate/josephmate.github.io"
        issue-number="41"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
