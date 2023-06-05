---
layout: post
title: "HotSpot JVM work to address GLIBC memory retention"
date: 2023-05-30
description: # HotSpot JVM work to address glibc fragmentation
img:  glibc_fragmentation_work.jpg # size:1912x461
fig-caption: # Add figcaption (optional)
tags: [JIT, Compiler, memory, fragmentation, glibc]
---

In continuation of my previous post [UseDynamicNumberOfCompilerThreads and Memory Footprint](../dynamic_compiler_threads/), I'd like to share some of the diagnostic and feature enhancement work that the HotSpot JVM team has been doing to combat memory fragmentation and retention that may be caused by the GLIBC memory allocator.


- [JDK-8302508: Add timestamp to the output TraceCompilerThreads](https://bugs.openjdk.org/browse/JDK-8302508)
This diagnostic enhancement adds timestamps to the output of -XX:+TraceCompilerThreads JVM option. That greatly helps in understanding how frequently the dynamic compiler threads are getting added or removed while using the *UseDynamicNumberOfCompilerThreads* feature.

- [JDK-8302264: Improve dynamic compiler threads creation](https://bugs.openjdk.org/browse/JDK-8302264)
This enhancement request aims to improve how the JIT Compiler creates and terminates dynamic compiler threads so as to reduce their unnecessary creation and termination. This is a work in progress.

- [JDK-8303767: Identify and address the likely causes of glibc allocator fragmentation](https://bugs.openjdk.org/browse/JDK-8303767) The goal of this change request is to determine whether there are any allocation patterns in the JVM that can cause GLIBC memory fragmentation and retention, resulting in an increase in Resident Set Size (RSS) for Java applications. This effort is also in progress.

- [JDK-8301749: Tracking malloc pooled memory size](https://bugs.openjdk.org/browse/JDK-8301749)
This enhancement added a new *jcmd* diagnostic command **System.native_heap_info** that prints the output of [malloc_info(3)](https://man7.org/linux/man-pages/man3/malloc_info.3.html) function on Linux systems. *malloc_info()* provides detailed information about memory allocation statistics of a program, and can be extremely helpful in understanding if significant amount of memory is being retained and not returned to the OS by the GLIBC's allocator.

- [JDK-8249666: Improve Native Memory Tracking to report the actual RSS usage](https://bugs.openjdk.org/browse/JDK-8249666)
This request aims to improve the output of [Native Memory Tracking (NMT)](https://docs.oracle.com/en/java/javase/20/vm/native-memory-tracking.html)] tool to include the **RSS** of a Java process along with its *Committed* and *Reserved* sizes. This will help in accounting the actual total number of pages resident in physical memory.

- [JDK-8157023: Integrate NMT with JFR](https://bugs.openjdk.org/browse/JDK-8157023)
This enhancement added two new JFR events - **NativeMemoryUsage** and **NativeMemoryUsageTotal** that help track the native memory usage of a specific memory type in the JVM and the total native memory usage for the Java process respectively. These events are recorded in a JFR file when the process is started with the JVM option *-XX:NativeMemoryTracking=summary/detail*.

- [JDK-8307374: Add a JFR event for tracking RSS](https://bugs.openjdk.org/browse/JDK-8307374)
The enhancement request will add a new JFR event for tracking RSS (Resident Set Size) of a Java process.




All of these enhancements and serviceability improvements will surely aid in better memory management and understanding the memory requirements and usage patterns of our Java applications. More later!

