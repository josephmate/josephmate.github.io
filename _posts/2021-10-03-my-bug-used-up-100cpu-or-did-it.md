---
layout: post
title: "100% CPU: My Fault?"
date: 2021-10-03 19:22
author: joseph.mate
comments: true
categories: [Java, ExecutorService]
---

# The Sirens are blaring

I received a bug report claiming that I caused 100% cpu util on a VM when it should be idle.
I was suspicious at first because I try my best to avoid the `while(true)` and `for(;;)` patterns.

The support engineer that wrote the initial report included the evidence for the 100% cpu uitl.
```
top
  PID USER      PR  NI    VIRT    RES  %CPU  %MEM     TIME+ S COMMAND
    1 root      20   0 1832.5m  20.6m 100.7   1.0   4:11.04 S java Main
   25 root      20   0    1.6m   1.1m   0.0   0.1   0:00.04 S sh
   38 root      20   0    1.9m   1.2m   0.0   0.1   0:00.00 R  `- top
```
The JVM was using up 100% CPU.
However, how did they track me down?

# How did they find me?

The bug was triaged by a software developer.
First, he got a list of thread ids related to the java process using `top -H -p <pid>`
```
top -H -p 1
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
Thread with id 18 was consuming 100% cpu.

Then he took 18 and converted it to convert to 12 in hex.
```
printf "%x\n" 18
12
```

Then he grabbed the thread dump using `kill -3 <pid>` and searched for the the string `nid=0x12`.
```
kill -3 1
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
Doh! That's definately my thread.
I'm glad I provided a thread name factory to my thread pool to make debugging easier for other developers.
It looks like something I put in the `ScheduledThreadPoolExecutor` was eating up all the cpu.

# Was it my bad?

My first attempt was to look at what I scheduled to run the the thread pool.
I found the code and to my surprise, it did not look like it should take very much CPU:
```
executorService.scheduleWithFixedDelay(
    () -> doSomethingReallyCheap(),
    0, // Initial delay
    10, // delay
    TimeUnit.MINUTES
);
```
This was something that runs only every 10 minutes and on top of that it was not CPU bound.
There were other similar blocks of code scheduling this kind of work on application startup, again all of them were IO bound.
On top of that, I intentionally chose `scheduleWithFixedDelay` just incase for whatever reason it took longer than 10 minutes and would not overload the thread pool by adding more work when the previous instance of that task had not completed.
At this point I was out of ideas, so I asked a co-worker to double check my work.

My co-worker took a look and agreeded with me.
It doesn't look like an issue with `doSomethingReallyCheap()`.
However, he took a closer look at the stack trace and noticed it wasn't stuck in `doSomethingReallyCheap()`;
it was actually stuck in `DelayedWorkQueue.poll(ScheduledThreadPoolExecutor.java:809)`!
He suggested I google a bit and see what comes up.

# YES! Vindicated!

Googling for `ScheduledThreadPoolExecutor 100% CPU` brings up stackoverflow article that describes the exact same symptoms:
[ScheduledExecutorService consumes 100% CPU when corePoolSize = 0](https://stackoverflow.com/questions/53401197/scheduledexecutorservice-consumes-100-cpu-when-corepoolsize-0)

The author had the exact same line of code as me:
> `ScheduledExecutorService executorService = Executors.newScheduledThreadPool(0);`

I intentionally used this because I did not want to continuously consume the resources needed for a thread for something that ran once every 10 minutes!
They even warn you not to set it to 0:
> Additionally, it is almost never a good idea to set corePoolSize to zero or use allowCoreThreadTimeOut because this may leave the pool without threads to handle tasks once they become eligible to run. 
But I was okay with that, because I did not care about waiting for the pool to create a new thread.
The 10 minute delay was not super strict.
>     corePoolSize - the number of threads to keep in the pool, even if they are idle, unless allowCoreThreadTimeOut is set
So I was expecting the pool to only keep thread around temporarily.

At this point, I switched to corePoolSize=1 to workaround this unexecpted behavior.
I was upset that I had to always consume a thread, but that's the best I could do with the time available.

However, I was really bother by that unexpected behavior.
Did I interpret the javadocs incorrectly?
Scrolling down, I was much relieved by the top answer.
> This is a known bug: JDK-8129861 https://bugs.openjdk.java.net/browse/JDK-8129861. It has been fixed in JDK 9. 

YES! Vindicated!

# WHY!?

I still wasn't satified.
How could corePoolSize=0 cause JDK to consume 100% cpu?
The details from the JDK bug didn't give me any hints:

> Looks like yet another zero-core thread bug, fixed in the Great jsr166 jdk9 integration 

All this tells me is that they planned to make some massive changes to ScheduledThreadPoolExecutor in JDK9.
Also this tells me that they will probably never fix this in JDK8.
That's why if you try this right now in the latest JDK8, you will likely reproduce the problem.
Looks like we'll need to dive into the source code.
We'll start our investigation at the top of the stack trace from the thread dump earlier: `ScheduledThreadPoolExecutor$DelayedWorkQueue.poll(ScheduledThreadPoolExecutor.java:809)`.
By running a minimal reproducing example in Intellij and places a break point near `DelayedWorkerQueue.poll` we can find the infinite loop.
Popping a few stack frames we see `ThreadPoolExecutor.getTask` as the infinte loop:

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

1. Note that keepAliveTime is 0 here.
2. worker count (wc) is 1 since there is one active thread
3. corePoolSize is 0
4. that means timed = true
5. next we check if we can cull a worker
6. there are 2 conditions that always prevent us from culling a work
    1. wc > 1, but we only have 1 worker
    2. workQueue.isEmpty() can never be empty since it's scheduled worked. it will always be in the queue.
5. that means we poll the workerQueue with keepAliveTime=timeout=0!
6. that means the poll returns immediately
7. and then we're back to the beginning of the for(;;) loop and none of the
   conditions change since the last iteration.

In constract, if corePoolSize is 1, then time=false and we take from the
workQueue instead, which blocks.
Which explains why setting corePoolSize=0 is a workaround.

This might be fixable if we change the `wc > 1` condition to `wc > 0` and change
the isEmpty() condition to check if something needs to execute soon. However,
this creates a massive design issue. We are using the current worker thread to 
poll for tasks.
We can't just change the continue to `wc > 0`. 
We should have heeded the javadoc's warning and not used corePoolSize=0.

To fix the JDK limitation, maybe we schedule a new thread to be created when the scheduled tasks deadline approaches and exit the current thread!
However, there's a lot of stuff left to figure out.
Like what thread is going to do the work of creating the thread when scheduled task needs to run?

# What could I have done better?
It's impossible for me to have known that corePoolSize=0 would not work.
I could have read the javadocs more closely and headed their warning of avoiding corePoolSize=0.
Even if I read that line, I don't think it would have stopped me because in my mind the usecase made sense.

What's embarrassing is that the problem is reproducible everytime and 100% cpu is pretty easy to spot.
I should have monitored the CPU while testing it.
But I only look at the CPU when testing my code for a performance change.
I did not anticipate this change to have a performance impact.
Should I check the CPU for all of my changes?
Or should there be an automated test that detects significant difference in CPU utilization between code changes?

# Try it yourself

If you want to this this yourself, compile and run the below code using JDK8.
It should reproduce because it was only fixed starting from JDK9.
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

# Final Thoughts

I was relieved that I was not the root cause.
However, I still share some of blame because I could have caught this issue earlier by monitoring the CPU while testing.
Fortunately, I'm grateful we had strong testing engineers that caught it before it was released.
What do you readers think?
Was this my fault?

<script src="https://utteranc.es/client.js"
        repo="josephmate/josephmate.github.io"
        issue-number="41"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
