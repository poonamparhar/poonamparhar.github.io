---
layout: post
title: "StackOverflowError and Threads Waiting for ReentrantReadWriteLock"
date: 2018-02-28
description: 
img:  stackoverflow_stuck_locks.jpg 
fig-caption: # Add figcaption (optional)
tags: [StackOverflowError, ReentrantReadWriteLock]
---

If terms like 'Stuck Threads' or 'Hang' sound familiar and scary to you, please read on. We can sometimes come across situations where the Weblogic Server (WLS) reports some threads as being 'Stuck' or 'not making any progress', but the stack traces of those threads and others in the process look absolutely normal, showing no signs of how and why those threads could get stuck.

I got to analyze such a situation recently. The threads in the application were reported to be *Stuck*. They were waiting for a **ReentrantReadWriteLock** object. All the threads were waiting to acquire a ReadLock on the ReentrantReadWriteLock object except for one thread that was waiting to acquire a WriteLock on it.

This is how the stack traces of the threads waiting for the ReadLock looked like:

```console
"[STUCK] ExecuteThread: '20' for queue: 'weblogic.kernel.Default (self-tuning)'" #127 daemon prio=1 os_prio=0 tid=0x00007f9c01e1c800 nid=0xb92 waiting on condition [0x00007f9b66fe9000]
   java.lang.Thread.State: WAITING (parking)
    at sun.misc.Unsafe.park(Native Method)
    - parking to wait for  <0x00000006c1e34768> (a java.util.concurrent.locks.ReentrantReadWriteLock$FairSync)
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireShared(AbstractQueuedSynchronizer.java:967)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireShared(AbstractQueuedSynchronizer.java:1283)
    at java.util.concurrent.locks.ReentrantReadWriteLock$ReadLock.lock(ReentrantReadWriteLock.java:727)
    at com.sun.jmx.mbeanserver.Repository.retrieve(Repository.java:487)
    at com.sun.jmx.interceptor.DefaultMBeanServerInterceptor.getMBean(DefaultMBeanServerInterceptor.java:1088)
    at com.sun.jmx.interceptor.DefaultMBeanServerInterceptor.getClassLoaderFor(DefaultMBeanServerInterceptor.java:1444)
    at com.sun.jmx.mbeanserver.JmxMBeanServer.getClassLoaderFor(JmxMBeanServer.java:1324)
    .... 

    Locked ownable synchronizers:
    - None  
```
 

Following is the stack trace of the thread waiting for the WriteLock:   

```console
"[STUCK] ExecuteThread: '19' for queue: 'weblogic.kernel.Default (self-tuning)'" #126 daemon prio=1 os_prio=0 tid=0x00007f9c01e1b000 nid=0xb91 waiting on condition [0x00007f9b670ea000]
   java.lang.Thread.State: WAITING (parking)
    at sun.misc.Unsafe.park(Native Method)
    - parking to wait for  <0x00000006c1e34768> (a java.util.concurrent.locks.ReentrantReadWriteLock$FairSync)
    at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:836)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued(AbstractQueuedSynchronizer.java:870)
    at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire(AbstractQueuedSynchronizer.java:1199)
    at java.util.concurrent.locks.ReentrantReadWriteLock$WriteLock.lock(ReentrantReadWriteLock.java:943)
    at com.sun.jmx.mbeanserver.Repository.addMBean(Repository.java:416)
    at com.sun.jmx.interceptor.DefaultMBeanServerInterceptor.registerWithRepository(DefaultMBeanServerInterceptor.java:1898)
    ... 

   Locked ownable synchronizers:
    - None
```

None of the threads in the thread dump were reported as holding this ReentrantReadWriteLock lock (at 0x00000006c1e34768). So what was wrong? Why were all of these threads waiting?

Luckily, a Heap Dump was collected when this situation occurred. From the heap dump, this ReentrantReadWriteLock object looked like this:

```
0x6c1e34768: ReentrantReadWriteLock$FairSync

Type   |     Name      |      Value
-----------------------------------------------
int |firstReaderHoldCount|1
ref |firstReader         |[STUCK] ExecuteThread: '19' for queue: 'weblogic.kernel.Default (self-tuning)'
ref |cachedHoldCounter   |java.util.concurrent.locks.ReentrantReadWriteLock$Sync$HoldCounter @ 0x7bbd32b70
ref |readHolds           |java.util.concurrent.locks.ReentrantReadWriteLock$Sync$ThreadLocalHoldCounter @ 0x6c1e34798
int |state               |65536
ref |tail                |java.util.concurrent.locks.AbstractQueuedSynchronizer$Node @ 0x781c5f9f0
ref |head                |java.util.concurrent.locks.AbstractQueuedSynchronizer$Node @ 0x7bb77d640
ref |exclusiveOwnerThread|null
---------------------------------------------
```

This showed that a ReadLock on this concurrent lock was already held, and the thread holding the ReadLock was actually thread '19' itself, which was also waiting to acquire a WriteLock on it. As per the design of ReentrantReadWriteLock, write lock can not be acquired on a ReentrantReadWriteLock object by a thread if a ReadLock on it is already held by the current or any other thread. And this was the cause of the deadlock. Thread 19 would keep waiting for the WriteLock until the held ReadLock on that ReentrantReadWriteLock object was released.

But how did this happen? If we look at the code of the [com.sun.jmx.mbeanserver.Repository class](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/com/sun/jmx/mbeanserver/Repository.java#l344), all the methods acquiring a ReadLock diligently release it in a finally clause.

So, if thread '19' had made a call to one of those methods of Repository acquiring ReadLock, the acquired ReadLock should have been unlocked at the end of that call, unless of course, the code unlocking the ReadLock in 'finally' clause got skipped somehow. That might happen if an asynchronous exception (ThreadDeath) or a synchronous VirtualMachineError (StackOverflowError or
OutOfMemoryError) was thrown before or during the unlock() call.

There were evidences in the application logs that StackOverflowErrors did occur during the application run, though not in the control path leading to ReadLock.unlock():

```
 <java.lang.StackOverflowError>
 <at java.security.AccessController.doPrivileged(Native Method)>
 <at java.net.URLClassLoader$3.next(URLClassLoader.java:598)>
 <at java.net.URLClassLoader$3.hasMoreElements(URLClassLoader.java:623)>
 [...]
```

Now, since it was the same thread(thread '19') that could have encountered a StackOverflowError (SOE) and was waiting for the WriteLock, it indicated that this thread could have swallowed that SOE. After examining the code for the JDK frames appearing in the stack traces, we could not find any place where we might be catching Error/Throwable and were ignoring or suppressing it. So, it seemed likely that the application could be ignoring/swallowing StackOverflowError somewhere in the call stack of the thread in question.

Thus, we suggested to run the application with an increased threads stack size using -Xss option, and also to examine their code to see if they were swallowing StackOverflowError somewhere. The application was tested for several days with an increased stack size, and it did not encounter this hang anymore.

We learned three important lessons from this exercise:

- Always ensure that a thread does not attempt to acquire WriteLock on a ReentrantReadWriteLock while holding a ReadLock on it. We can call ReentrantReadWriteLock#getReadHoldCount() to get the ReadLock count for the current thread before trying to acquire the WriteLock.
- Never catch and ignore Error/Throwable as you might be supressing StackOverflowError, and that could leave concurrent lock objects in an inconsistent state.
- To workaround the StackOverflowErrors that our application threads might be encountering, we can increase the stack size of threads using the -Xss JVM option.

Hope this helps! More later.