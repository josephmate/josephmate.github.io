---
layout: post
title: "Bug Story 1: 100% CPU: My Fault?"
date: 2021-10-01 20:14
author: joseph.mate
comments: true
categories: [Java, ExecutorService]
---

I received a bug report claiming that I was at fault for 100% cpu util on a VM when it should have been idle.
I was suspicious at first because I try my best to avoid while(true) {} loops.

The support engineering that wrote the initial report included the evidence for the 100% cpu uitl.
```
top
  PID USER      PR  NI    VIRT    RES  %CPU  %MEM     TIME+ S COMMAND
    1 root      20   0 1832.5m  20.6m 100.7   1.0   4:11.04 S java Main
   25 root      20   0    1.6m   1.1m   0.0   0.1   0:00.04 S sh
   38 root      20   0    1.9m   1.2m   0.0   0.1   0:00.00 R  `- top
```
Up, the JVM was using up 100% CPU.
However, how did they track me down as the cause?

Afterwards, the bug was triaged by a software developer.
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

The engineer took 18 and converted it to convert to 12 in hex.
```
printf "%x\n" 18
12
```

Then he grabbed the thread dump using `kill -3 <pid>` and search for the the string `nid=0x12`.
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
Doh! That's definately my thead.
I'm glad I provided a name factory to my thread pool to make debugging easier.
It looks like something I put in the `ScheduledThreadPoolExecutor` was eating up all the cpu.

My first attempt was to look at what I scheduled to run the the thread pool.
I found the code and to my surprise, it did not look at like it should take very much CPU:
```
executorService.scheduleWithFixedDelay(
    () -> doSomethingReallyCheap(),
    0, // Initial delay
    10, // delay
    TimeUnit.MINUTES
);
```
This was something that runs only 10 minutes which was not CPU bound.
There were other similar blocks of code scheduling this work on application startup, but all of them were IO bound.
On top of that, I intentionally chose `scheduleWithFixDelay` just incase for whatever reason it took longer than 10 minutes and would not overload the thread pool.
At this point I was out of ideas, so I asked a co-worker to double check my work.

My co-worker took a look and agreeded with me, it doesn't look like an issue with `doSomethingReallyCheap()`.
However, he took a closer look at the stack trace and noticed it wasn't stuck in `doSomethingReallyCheap()`,
it was actually stuck in `DelayedWorkQueue.poll(ScheduledThreadPoolExecutor.java:809)`!
He suggested I google a bit and see what comes up.

Googling for `ScheduledThreadPoolExecutor 100% CPU` brings up stackoverflow article that describes the exact same symptoms.

ScheduledExecutorService consumes 100% CPU when corePoolSize = 0
https://stackoverflow.com/questions/53401197/scheduledexecutorservice-consumes-100-cpu-when-corepoolsize-0

> ScheduledExecutorService executorService = Executors.newScheduledThreadPool(0);

Was the exact same line of code I used!
I intentionally used this because I did not want to continuously consume the resources needed for a stack for something that ran once every 10 minutes!
They even warn you not to set it to 0:
> Additionally, it is almost never a good idea to set corePoolSize to zero or use allowCoreThreadTimeOut because this may leave the pool without threads to handle tasks once they become eligible to run. 
But I was okay with that, because I did not care about waiting for the pool to create a new thread.
>     corePoolSize - the number of threads to keep in the pool, even if they are idle, unless allowCoreThreadTimeOut is set
So I was expecting the pool to only keep thread around temporarily.

At this point, I switched to corePoolSize=1 to workaround this unexecpted behavior.
I was upset that I had to always consume a thread, but that's the best I could do with the time available.

However, I really bother by that unexpected behavior.
Did I interpret the javadocs incorrectly?
Scrolling down, I was much relieved by the top answer.
> This is a known bug: JDK-8129861 https://bugs.openjdk.java.net/browse/JDK-8129861. It has been fixed in JDK 9. 

I still wasn't satifies though.
How could corePoolSize=0 cause JDK to consume 100% cpu?
The details from the JDK bug didn't give me any hints:

> Looks like yet another zero-core thread bug, fixed in the Great jsr166 jdk9 integration 



7. Was it really my fault? I'll let you be the judge of that.
8. What I could have done better
