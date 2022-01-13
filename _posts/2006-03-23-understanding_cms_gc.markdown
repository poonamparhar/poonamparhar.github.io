---
layout: post
title: "Understanding CMS Logs"
date: 2006-03-23
description: 
img:  cms_gc_logs.jpg 
fig-caption: # Add figcaption (optional)
tags: [CMS, GC, Logs]
---

The Concurrent Mark Sweep (CMS) collector prints a lot of useful information with the JVM options **-XX:+PrintGCDetails and -XX:+PrintGCTimeStamps**. Understanding this information can help us immensely in fine tuning various parameters of Java applications and CMS to achieve their best performance.

## CMS Cycle

Let's take a look at some of the log records of a normal CMS cycle produced with 1.4.2_10:

```
39.910: [GC 39.910: [ParNew: 261760K->0K(261952K), 0.2314667 secs] 262017K->26386K(1048384K), 0.2318679 secs]
```

**Young generation (ParNew) collection**. Here, the Young generation capacity is 261952K, and after the collection its occupancy drops from 261760K to 0. This collection took 0.2318679 secs.

```
40.146: [GC [1 CMS-initial-mark: 26386K(786432K)] 26404K(1048384K), 0.0074495 secs]
```

**Beginning of tenured generation collection**. This is the initial-marking phase of CMS, where all the objects directly reachable from roots are marked while all the mutator threads are stopped. In this case, the capacity of the tenured generation space is 786432K, and CMS was triggered at an occupancy of 26386K.

```
40.154: [CMS-concurrent-mark-start]
```
**Start of the Concurrent Marking phase**. In the Concurrent Marking phase, threads that were stopped in the first phase are started again, and all the objects transitively reachable from the objects marked in first phase are marked now.

```
40.683: [CMS-concurrent-mark: 0.521/0.529 secs]
```
Concurrent marking took a total of 0.521 seconds of elapsed time, and 0.529 seconds of wall time that includes the yield to other threads.

```
40.683: [CMS-concurrent-preclean-start]
```
**Start of precleaning**. Precleaning is also a concurrent phase. Here, in this phase we look at the objects in the CMS heap that got updated by promotions from young generation, or by new allocations, or got updated by mutators while we were performing concurrent marking in the previous concurrent marking phase. By rescanning those objects concurrently, the precleaning phase helps reduce the work in the next stop-the-world “remark” phase.

```
40.701: [CMS-concurrent-preclean: 0.017/0.018 secs]
```
Concurrent precleaning took 0.017 secs of elapsed time and 0.018 secs of wall time.

```
40.704: [GC40.704: [Rescan (parallel) , 0.1790103 secs]40.883: [weak refs processing, 0.0100966 secs] [1 CMS-remark: 26386K(786432K)] 52644K(1048384K), 0.1897792 secs]
```
**Stop-the-world Remark phase**. This phase rescans any residual updated objects in the CMS heap, retraces from roots, and also processes Reference objects. Here, the rescanning work took 0.1790103 secs, and processing of weak reference objects took 0.0100966 secs. This phase took a total of 0.1897792 secs to complete.

```
40.894: [CMS-concurrent-sweep-start]
```

**Start of Sweeping of the dead/non-marked objects**. Sweeping is a concurrent phase performed while all the application threads are running.

```
41.020: [CMS-concurrent-sweep: 0.126/0.126 secs]
```
Sweeping in this case took 0.126 secs.

```
41.020: [CMS-concurrent-reset-start]
```
**Start of reset phase**.

```
41.147: [CMS-concurrent-reset: 0.127/0.127 secs]
```
In this phase, the CMS data structures are reinitialized so that a new cycle may begin at a later time. In this case, it took 0.127 secs.

## Other CMS log records

So far, we looked at how a normal CMS cycle runs. Now let's take a look at a few other CMS log entries:

```
197.976: [GC 197.976: [ParNew: 260872K->260872K(261952K), 0.0000688 secs]197.976: [CMS197.981: [CMS-concurrent-sweep: 0.516/0.531 secs]
(concurrent mode failure): 402978K->248977K(786432K), 2.3728734 secs] 663850K->248977K(1048384K), 2.3733725 secs]
```
This shows that a ParNew collection was requested, but it was not attempted because it was estimated that there was not enough space in the CMS generation to promote the worst case surviving young generation objects. We name this failure as “full promotion guarantee failure”.

Due to this, Concurrent Mode of CMS is interrupted and a Full GC is invoked at 197.981. This mark-sweep-compact stop-the-world Full GC took 2.3733725 secs, and the CMS generation space occupancy dropped from 402978K to 248977K. 

The concurrent mode failure can either be avoided by increasing the tenured generation siz,e or initiating the CMS collection at a lesser heap occupancy by setting **CMSInitiatingOccupancyFraction** to a lower value and setting **UseCMSInitiatingOccupancyOnly** to true. The value for CMSInitiatingOccupancyFraction should be chosen appropriately because setting it to a very low value will result in too frequent CMS collections.

Sometimes, we see these promotion failures even when the logs show that there is enough free space in the tenured generation. The reason is 'fragmentation' - the free space available in tenured generation is not contiguous, and promotions from young generation require a contiguous free block to be available in the tenured generation. The CMS collector is a non-compacting collector, and can have fragmentation of space for some type of applications. Jon talks in detail on how to deal with this fragmentation problem in his blog post here: http://blogs.sun.com/roller/page/jonthecollector?entry=when_the_sum_of_the

Starting with 1.5, for the CMS collector, the promotion guarantee check is done differently. Instead of assuming that the promotions would be worst case i.e. all of the surviving young generation objects would get promoted into old gen, the expected promotion is estimated based on recent history of promotions. This estimation is usually much smaller than the worst case promotion and hence requires lesser free space to be available in old generation. And if the promotion in a scavenge attempt fails, then the young generation is left in a consistent state and a stop-the-world mark-compact collection is invoked. To get the same functionality with UseSerialGC, we need to explicitly specify the switch **-XX:+HandlePromotionFailure**.

```
283.736: [Full GC 283.736: [ParNew: 261599K->261599K(261952K), 0.0000615 secs] 826554K->826554K(1048384K), 0.0003259 secs]
GC locker: Trying a full collection because scavenge failed
283.736: [Full GC 283.736: [ParNew: 261599K->261599K(261952K), 0.0000288 secs]
```

Stop-the-world GC happening when a JNI Critical section is released. Here again, the young generation collection failed due to “full promotion guarantee failure” and then a Full GC is being invoked.

CMS can also be run in incremental mode (i-cms), enabled with **-XX:+CMSIncrementalMode**. In this mode, the CMS collector does not hold the processor for the entire duration of concurrent phases but periodically stops and yields the processor back to other threads of application. It divides the work to be done in concurrent phases in small chunks(called duty cycle) and schedules them between minor collections. This is very useful for applications that need low pause times and are run on machines having a small number of processors.

Some logs below showing the incremental CMS.

```
2803.125: [GC 2803.125: [ParNew: 408832K->0K(409216K), 0.5371950 secs] 611130K->206985K(1048192K) icms_dc=4 , 0.5373720 secs]
2824.209: [GC 2824.209: [ParNew: 408832K->0K(409216K), 0.6755540 secs] 615806K->211897K(1048192K) icms_dc=4 , 0.6757740 secs]
```

Here, the scavenges took 537 ms and 675 ms respectively. In between these two scavenges, iCMS ran for a brief period as indicated by the icms_dc value, which indicates a duty-cycle. In this case the duty cycle was 4%. A simple calculation shows that the iCMS incremental step lasted for 4/100 \* (2824.209 - 2803.125 - 0.537) = 821 ms, i.e. 4% of the time between the two scavenges.

Starting with 1.5, CMS has one more phase: **Concurrent Abortable Preclean**. Abortable preclean is run between a 'concurrent preclean' and 'remark' until we have the desired occupancy in eden. This phase is added to help schedule the 'remark' phase so as to avoid back-to-back pauses for a scavenge closely followed by a CMS remark pause. In order to maximally separate a scavenge from a CMS remark pause, we attempt to schedule the CMS remark pause roughly mid-way between scavenges.
There is a second reason why we do this. Immediately following a scavenge there are likely a large number of grey objects that need rescanning. The abortable preclean phase tries to deal with such newly grey objects thus reducing a subsequent CMS remark pause.

The scheduling of 'remark' phase can be controlled by two JVM options **CMSScheduleRemarkEdenSizeThreshold** and **CMSScheduleRemarkEdenPenetration**. The defaults for these are 2m and 50% respectively. The first parameter determines the Eden size below which no attempt is made to schedule the CMS remark pause because the pay off is expected to be minuscule. The second parameter indicates the Eden occupancy at which a CMS remark is attempted.

After 'concurrent preclean' if the Eden occupancy is above CMSScheduleRemarkEdenSizeThreshold, we start 'concurrent abortable preclean' and continue precleanig until we have CMSScheduleRemarkEdenPenetration percentage occupancy in eden, otherwise we schedule 'remark' phase immediately.

```
7688.150: [CMS-concurrent-preclean-start]
7688.186: [CMS-concurrent-preclean: 0.034/0.035 secs]
7688.186: [CMS-concurrent-abortable-preclean-start]
7688.465: [GC 7688.465: [ParNew: 1040940K->1464K(1044544K), 0.0165840 secs] 1343593K->304365K(2093120K), 0.0167509 secs]
7690.093: [CMS-concurrent-abortable-preclean: 1.012/1.907 secs]
7690.095: [GC[YG occupancy: 522484 K (1044544 K)]7690.095: [Rescan (parallel) , 0.3665541 secs]7690.462: [weak refs processing, 0.0003850 secs] [1 CMS-remark: 302901K(1048576K)] 825385K(2093120K), 0.3670690 secs]
```

In the above log records, after a preclean, 'abortable preclean' starts. After the young generation collection, the young gen occupancy drops down from 1040940K to 1464K. When the young gen occupancy reaches 522484K which is 50% of the total capacity, precleaning is aborted and 'remark' phase is started.

Note that in 1.5, young generation occupancy also gets printed in the final remark phase.
