---
layout: post
title: "Frequent Full GCs"
date: 2022-11-17
description: # Frequent Full GCs
img:  frequent_full_gcs.png 
fig-caption: # Add figcaption (optional)
tags: [Troubleshooting, GC]
---

The optimal performance of Java applications can be seriously hampered by frequent Full garbage collection cycles. In this post, we'll discuss a few situations that could lead to repeated Full GCs as well as how to recognize and handle them. 

## Misconfiguration of Memory Spaces
One of the main causes of frequent Full GCs is the misconfiguration of JVM managed memory spaces of the application: the Java heap might be sized smaller than the live-set of Java objects, or the Metaspace may be configured too small.

We should monitor the utilization of Java heap, in particular, the old generation, and the Metaspace to understand our application’s requirements for those spaces. If the occurring Full GCs are not able to claim any space, then it could be a size configuration issue; the Java heap or Metaspace might be configured smaller than the application’s requirements. Increase the Java heap or Metaspace size, respectively, and then rerun your application using the larger size. If the failure persists at the larger heap or metaspace size as well and the memory growth continues, there may be a memory leak.

Another cause of frequent Full GCs could be the premature promotion of objects from the young generation to the old generation. In this case, increasing the size of the young generation and allowing the short-lived instances to get collected with the minor collections can help circumvent the frequent Full GCs situation. 

## Memory Leaks
Memory leaks occur when certain instances or classes are unintentionally retained in memory. Once we confirm that there is a memory leak in our application, the next step is to identify the instances or classes causing that memory leak. 
For diagnosing memory leaks, Heap histograms and Heap dumps are very helpful. Heap histograms provide a quick view of the Java heap and help us understand what is growing in the heap. Heap dumps are the most useful diagnostic data for troubleshooting memory leaks. Collect heap dumps periodically before and after the application starts exhibiting memory growth.

We can use Eclipse MAT or any other heap dump analysis tool to analyze the gathered heap dumps. In order to determine which objects are growing in the Java heap over time, compare the number of instances of different types in the collected heap dumps. Similarly, for frequent GCs occurring to reclaim space in  Metaspace, look for top growing classes, specifically duplicate classes that you suspect should not be loaded multiple times. Then, finding the GC roots of those instances and classes can lead us to the causes of memory leaks, helping us nail down frequent Full GCs.

## Full GCs with ParallelGC
It is important to note that the parallel garbage collector can continuously attempt to free up room in the Java heap by invoking frequent back-to-back Full GCs, even when the gains of that effort are small and the Java heap is almost full. This affects the application's performance and could prevent the system from re-bouncing. This situation can be avoided by tuning the values for the **-XX:GCTimeLimit** and **-XX:GCHeapFreeLimit** JVM options.

GCTimeLimit sets an upper limit on the amount of time that GCs can spend as a percent of the total time. Its default value is 98%. Decreasing this value reduces the amount of allowed time that can be spent in the garbage collections. GCHeapFreeLimit sets a lower limit on the amount of space that should be free after a garbage collection, represented as a percent of the maximum heap. Its default value is 2%. Increasing this value means that more heap space should get reclaimed by the GCs. An OutOfMemoryError is thrown after a Full GC if the previous 5 consecutive GCs (could be minor or full) were not able to keep the GC cost below GCTimeLimit and were not able to free up GCHeapFreeLimit space.

For example, setting GCHeapFreeLimit to 8 percent can help the garbage collector not get stuck in a loop of invoking back-to-back Full GCs when it is not able to reclaim at least 8% of the heap and is exceeding GCTimeLimit for 5 consecutive GCs.

## System.gc()
The appearance of the System.gc() string in Full GC log entries indicates that those GCs were invoked by explicit calls to *java.lang.Runtime.gc()* or *System.gc()*. The JVM option **-XX:+DisableExplicitGC** can be used to disable such Full GCs.
