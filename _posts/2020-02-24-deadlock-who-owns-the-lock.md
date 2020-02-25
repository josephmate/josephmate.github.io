---
layout: post
title: "Deadlock: Who Owns the Lock?"
date: 2020-02-24 18:40
author: joseph.mate
comments: true
categories: [Java, Deadlock]
---

# What I Learned
1. Locking a readlock, then locking the write lock on the same lock creates a deadlock. 
2. Deadlocks created using locks instead of monitors does not appear in thread dumps (like those created by kill -3 (linux) or Ctrl+Break (windows)).
3. Keep digging and you'll uncover a nasty incorrect assumption you've been making.

If this interests you read on!

# Every good story starts with a difficult bug
Recently I stumbled up this bug where it appeared like my threads were deadlocked, but triggered a thread dump to stand out using `kill -3 $(pidof java)` or `Ctrl+Break` did not report any deadlocks. Normally when you have a dead lock you would see something like below at the end of the thread dump:
```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x000000001a1c2598 (object 0x00000000d57695f8, a java.lang.Object),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x000000000331b628 (object 0x00000000d5769608, a java.lang.Object),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
        at SimpleMonitorDeadlockMain$DeadlockRunnable.run(SimpleMonitorDeadlockMain.java:25)
        - waiting to lock <0x00000000d57695f8> (a java.lang.Object)
        - locked <0x00000000d5769608> (a java.lang.Object)
        at java.lang.Thread.run(Thread.java:748)
"Thread-0":
        at SimpleMonitorDeadlockMain$DeadlockRunnable.run(SimpleMonitorDeadlockMain.java:25)
        - waiting to lock <0x00000000d5769608> (a java.lang.Object)
        - locked <0x00000000d57695f8> (a java.lang.Object)
        at java.lang.Thread.run(Thread.java:748)

Found 1 deadlock.
```

But in my bug, I was seeing something like
```
"main" #1 prio=5 os_prio=0 tid=0x00000000030c3000 nid=0x4fb0 waiting on condition [0x0000000002f2e000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000d5820098> (a java.util.concurrent.locks.ReentrantReadWriteLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
	at java.util.concurrent.locks.ReentrantReadWriteLock$WriteLock.lock(ReentrantReadWriteLock.java:943)
	at ReadLockThenWriteLockMain.main(ReadLockThenWriteLockMain.java:11)
```
I couldn't find who owned `0x00000000d5820098`.

Fortunately, with a little GoogleFu, I was able to find someone who encountered the same problem and learned a lot.
[StackOverflowError and threads waiting for ReentrantReadWriteLock by Poonam Parhar](https://blogs.oracle.com/poonam/stackoverflowerror-and-threads-waiting-for-reentrantreadwritelock).
Originally, I assumed that the read lock would be promoted to a write lock by stopping future readers and waiting for reader to leave.
So I put together a minimal reproducing program of the Oracle article to double check.
```java
System.out.println("Starting!");
ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
System.out.println("Grabbing the read lock.");
readWriteLock.readLock().lock();
System.out.println("Grabbing the write lock.");
readWriteLock.writeLock().lock();
System.out.println("Releasing the write lock.");
readWriteLock.writeLock().unlock();
System.out.println("Releasing the read lock.");
readWriteLock.readLock().unlock();
System.out.println("Done!");
```

The above program produced the below standard out:
```
Starting!
Grabbing the read lock.
Grabbing the write lock.
```
The program makes no progress to the next println after that. It was deadlocked and the thread dump didn't show who owned the thing the main thread was waiting for. That must have been my bug (it wasn't)!

# The Grim Realization
I wanted to share what I learned by putting together this article. The first thing I needed was a sample deadlock that showed reeantrant locks dead locking and the standard out it produced:

```java
private static class DeadlockRunnable implements Runnable {
    private final Lock lockToGrabFirst;
    private final Lock lockToGrabSecond;
    private final CyclicBarrier ensureBothThreadsGrabTheirLock;

    public DeadlockRunnable(CyclicBarrier ensureBothThreadsGrabTheirLock,
                            Lock lockToGrabFirst,
                            Lock lockToGrabSecond) {
        this.ensureBothThreadsGrabTheirLock = ensureBothThreadsGrabTheirLock;
        this.lockToGrabFirst = lockToGrabFirst;
        this.lockToGrabSecond = lockToGrabSecond;
    }

    @Override
    public void run() {
        try {
            lockToGrabFirst.lock();

            // wait until thread2 reaches barrier before asking for the next lock
            ensureBothThreadsGrabTheirLock.await();

            lockToGrabSecond.lock();
        } catch(Exception e) {
            e.printStackTrace();
        } finally {
            lockToGrabSecond.unlock();
            lockToGrabFirst.unlock();
        }
    }
}

public static void main(String[] args) throws InterruptedException {
    Lock lock1 = new ReentrantLock();
    Lock lock2 = new ReentrantLock();
    CyclicBarrier ensureBothThreadsGrabTheirLock = new CyclicBarrier(2);

    Thread thread1 = new Thread(new DeadlockRunnable(ensureBothThreadsGrabTheirLock, lock1, lock2));
    // reverse the order in which the locks are grabbed
    Thread thread2 = new Thread(new DeadlockRunnable(ensureBothThreadsGrabTheirLock, lock2, lock1));
    thread1.start();
    thread2.start();

    System.out.print("Waiting for thread1 to complete.");
    thread1.join();
    System.out.print("Waiting for thread2 to complete.");
    thread2.join();
    System.out.print("Done!");
}
```

The above program produces the below standard out.
```
Waiting for thread1 to complete.
```
The program makes no progress to the next println after that. It was deadlocked.
But then I sent `kill -3` or `Crtl+Break` to get the thread dump and I saw a familiar issue:
```
"Thread-1" #12 prio=5 os_prio=0 tid=0x0000000019abc800 nid=0x748c waiting on condition [0x000000001a55f000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000d581cbe0> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
	at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)
	at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)
	at SimpleReentrantDeadlockMain$DeadlockRunnable.run(SimpleReentrantDeadlockMain.java:27)
	at java.lang.Thread.run(Thread.java:748)


"Thread-0" #11 prio=5 os_prio=0 tid=0x0000000019aba000 nid=0x19a4 waiting on condition [0x000000001a45e000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000d581cc10> (a java.util.concurrent.locks.ReentrantLock$NonfairSync)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
	at java.util.concurrent.locks.ReentrantLock$NonfairSync.lock(ReentrantLock.java:209)
	at java.util.concurrent.locks.ReentrantLock.lock(ReentrantLock.java:285)
	at SimpleReentrantDeadlockMain$DeadlockRunnable.run(SimpleReentrantDeadlockMain.java:27)
	at java.lang.Thread.run(Thread.java:748)
```
THE DUMP DOES NOT SHOW WHO OWNS WHAT IT'S WAITING FOR!!!
I thought it was because of a read then write lock, similar to the Oracle article, that caused the thread dump to not show who owns the lock.
That was not why.
The real reason was lock ownership is not shown by thread dumps.
"The default deadlock detection works with locks that are obtained using the synchronized keyword, as well as with locks that are obtained using the java.util.concurrent package. If the Java VM flag `-XX:+PrintConcurrentLocks` is set, then the stack trace also shows a list of lock owners." [Troubleshooting Hanging or Looping Processes](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/hangloop.html).
I'll have to try with `-XX:+PrintConcurrentLocks`.

# Sample Code
If you want to try it out, check the sample code I saved to [my JavaDeadlock github repository](https://github.com/josephmate/JavaDeadlocks).
