---
layout: post
title: "Understanding Metaspace and Class Space GC Log Entries"
date: 2021-11-30
description: # Understanding Metaspace and Class Space GC Log Entries
img:  #metaspace_gc_logs.jpg 
fig-caption: # Add figcaption (optional)
tags: [Metaspace, Classspace, GC Logs]
---

In this post, I want to share some details on Metaspace and Compressed Class Space, and how to read and interpret the relevant GC logging information.

## Metaspace

Metaspace is a native memory region that stores metadata for classes. As a class is loaded by the JVM, its metadata (i.e. its runtime representation in the JVM) is allocated into the Metaspace.

The Metaspace occupancy grows as more and more classes are loaded. And, when a classloader and all its loaded classes become unreachable in the Java Heap, the associated class metadata in the Metaspace becomes eligible for deallocation. The class metadata is cleaned up with a garbage collection cycle. 

Is there a limit on how much the Metaspace occupancy can grow before a GC is run to clean up the not-needed metadata? Yes, there is. The JVM internally maintains a threshold (called High Water Mark) for the Metaspace occupancy, and while allocating into the Metaspace it checks for this threshold. When a particular allocation can not be satisfied within that threshold, a “Metadata GC Threshold” garbage collection cycle is invoked, as can be seen in the following log.

```
2021-11-30T08:50:44.641+0800: [Full GC (Metadata GC Threshold) [PSYoungGen: 1168K->0K(90112K)] [ParOldGen: 1989K->3037K(2104832K)] 3158K->3037K(2194944K), [Metaspace: 1142318K->1142318K(1265664K)], 0.0914589 secs] [Times: user=0.10 sys=0.00, real=0.09 secs] 
```
The JVM options - **MetaspaceSize** and **MaxMetaspaceSize** can be used to control the initial and maximum size of the Metaspace. If the values for these are not explictly provided on the commandline, the JVM option **-XX:+PrintFlagsFinal** can be used to see the values set by the JVM for these options.

```
uintx MaxMetaspaceSize       = 18446744073709547520                   {product}
uintx MetaspaceSize          = 21807104                               {pd product}
```

As we can see, **MaxMetaspaceSize**, if left unconfigured on the commandline is almost unlimited. **MetaspaceSize** sets the initial capacity and threshold for the Metaspace at which a GC is invoked to clean up space, or to expand the capacity if enough space can not be reclaimed.

*Please note that the example logs in this blog are produced with Java 8.*

## Metaspace usage information

Let's look at a few log entries generated using a Java program that loads and unloads classes into the Metaspace. The JVM options I have used for executing the program are:

```
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -XX:-UseCompressedClassPointers
```

Ignore the option **UseCompressedClassPointers** for the moment; we will talk about this in the following section. 

While running, the Java program encounters several "Metadata GC Threshold" GCs. Let's examine one such log record closely.

```
2021-10-07T14:13:10.847+0800: [Full GC (Metadata GC Threshold) [PSYoungGen: 384K->0K(93696K)] [ParOldGen: 982K->1360K(1172480K)] 1366K->1360K(1266176K), [Metaspace: 410659K->410659K(456704K)], 0.0418110 secs] [Times: user=0.04 sys=0.00, real=0.04 secs]
```

Here, we can see that before this particular Full GC, the Metaspace occupancy was 410659K, and after the GC the occupancy didn’t change. The number in the parentheses indicates the new reserved Metaspace size, which is 456704K. 

Since we had added **-XX:+PrintHeapAtGC** option, detailed information before and after GC gets printed, and we can see why this GC was invoked and how the Metaspace size changed with this particular collection.
 
 ```
{Heap before GC invocations=16 (full 1):
 PSYoungGen      total 93696K, used 384K [0x00000007aab00000, 0x00000007b0e80000, 0x0000000800000000)
  eden space 91136K, 0% used [0x00000007aab00000,0x00000007aab00000,0x00000007b0400000)
  from space 2560K, 15% used [0x00000007b0680000,0x00000007b06e0000,0x00000007b0900000)
  to   space 2560K, 0% used [0x00000007b0400000,0x00000007b0400000,0x00000007b0680000)
 ParOldGen       total 919552K, used 982K [0x0000000700000000, 0x0000000738200000, 0x00000007aab00000)
  object space 919552K, 0% used [0x0000000700000000,0x00000007000f5bc8,0x0000000738200000)
 Metaspace       used 410659K, capacity 455416K, committed 455424K, reserved 456704K
```

From the Metaspace details above, before the GC, the JVM had reserved 456704K space and had committed 455424K for metadata allocations. Metaspace usage was at 410659K.  An interesting thing to note here is the capacity, which acts as the internal high-water-mark for the Metaspace. When a particular metadata allocation request can not be satisfied within that threshold, a GC is invoked to clean up and make some room available in the Metaspace. And, if that GC is not able to make enough room available, more memory is committed and capacity is also raised. In this case, this threshold was set at 455416K, out of which 410659K was already in use. A new metadata allocation could not be satisfied within that capacity limit causing this Full GC to occur. 

After the GC, the reserved, committed and capacity boundaries of the Metaspace are adjusted so as to meet the metadata space requirements. We can find the new values in the ‘Before GC’ details logged before the next GC event.

```
{Heap before GC invocations=17 (full 1):
 PSYoungGen      total 93696K, used 1822K [0x00000007aab00000, 0x00000007b0e80000, 0x0000000800000000)
  eden space 91136K, 2% used [0x00000007aab00000,0x00000007aacc7af0,0x00000007b0400000)
  from space 2560K, 0% used [0x00000007b0680000,0x00000007b0680000,0x00000007b0900000)
  to   space 2560K, 0% used [0x00000007b0400000,0x00000007b0400000,0x00000007b0680000)
 ParOldGen       total 1172480K, used 1360K [0x0000000700000000, 0x0000000747900000, 0x00000007aab00000)
  object space 1172480K, 0% used [0x0000000700000000,0x00000007001542b0,0x0000000747900000)
 Metaspace       used 685064K, capacity 759024K, committed 759040K, reserved 759808K

2021-10-07T14:13:11.843+0800: [GC (Metadata GC Threshold) [PSYoungGen: 1822K->704K(93696K)] 3183K->2064K(1266176K), 0.0021419 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
--- <snip> ---
2021-10-07T14:13:11.845+0800: [Full GC (Metadata GC Threshold) [PSYoungGen: 704K->0K(93696K)] [ParOldGen: 1360K->1989K(1497600K)] 2064K->1989K(1591296K), [Metaspace: 685064K->685064K(759808K)], 0.0501529 secs] [Times: user=0.05 sys=0.00, real=0.05 secs]
```

The above Metaspace log entries show that after the GC invocation #16, the reserved and committed space were increased to 759808K and 759040K respectively. The high-water-mark was also raised to 759024K, and the usage increased from 410659K to 685064K since the previous GC.

## Compressed class space

We can also have a separate space as part of the Metaspace to store only the *class part* of the metadata. This separate space is called **Compressed Class Space**, and the *class part* metadata in it is referenced using 32-bit offsets from Java objects. Compressed Class Space is explained very beautifuly in this blog post here: https://stuefe.de/posts/metaspace/what-is-compressed-class-space/

In the previous section, for simplicity of logs, I had disabled the use of a separate Compressed Class Space using the option *-XX:-UseCompressedClassPointers*. On 64-bit platforms, Compressed Class Space is enabled by default with a default reserved space size of 1 Gigabytes (GB). The reserved space for the Compressed Class Space is set up at the launch time of the JVM and its size can not be changed later. The maximum reserved space size that the Hotspot JVM allows for the Compressed Class Space is 3GB, and it can be configured using the JVM option *-XX:CompressedClassSpaceSize=n*.

## 'Compressed class space' usage information

Now, let’s look at a few log records when we have a separate Compressed Class Space as part of the Metaspace. 

```
{Heap before GC invocations=6 (full 3):
--- <snip> ---
 Metaspace       used 20669K, capacity 25846K, committed 35456K, reserved 1077248K
  class space    used 1243K, capacity 2086K, committed 8064K, reserved 1048576K

2021-10-09T07:23:16.585+0800: [Full GC (Metadata GC Threshold) [PSYoungGen: 96K->0K(76288K)] [ParOldGen: 410K->446K(219136K)] 506K->446K(295424K), [Metaspace: 20669K->20669K(1077248K)], 0.0035401 secs] [Times: user=0.01 sys=0.00, real=0.00 secs]
```

The first two log entries above show that the total Metaspace usage is at 20669K, and out of which 1243K is used by the class space. Similarly, the total Metaspace capacity (HWM) is 25846K, out of that the class space capacity (HWM) at this GC invocation is 2086K. To reiterate, the capacity is a threshold, and when the usage gets close to the capacity, a GC is invoked. 

In this case, the Metaspace committed size is 35456K, and from that space, 8064K is committed for the class space. This means that 35456K-8064K=27392K is the committed space for the non-class-part of metadata. The JVM option **MaxMetaspaceSize** can be used for setting the maximum limit on the total committed size.

```
MaxMetaspaceSize = Max limit on the committed size of Metaspace = Max limit on (committed size of non-class-part of metaspace + committed size of class space)
```

For the reserved space, we can see that the class space has 1GB (1048576K) of virtual reserved space (which is default), and the total Metaspace (including the class space) has 1077248K reserved space. 

## java.lang.OutOfMemoryError

It is important to understand the space requirements of class and non-class metadata for our application, and size Metaspace adequately. *MaxMetaspaceSize* JVM option sets an upper limit on the commited size of the Metaspace, and if not configured large enough can cause *java.lang.OutOfMemoryError: Metaspace*. Also, as mentioned before, all of the space configured for the Compressed class space is reserved upfront, and can not grow later at runtime. So, if the reserved space for the Compressed class space is not configured sufficiently large, we might encounter *java.lang.OutOfMemoryError: Compressed class space* failure.

## Fragmentation

Note that we may encounter OutOfMemoryError failures due to fragmentation in the Metaspace (in both non-class-part and class-part of the Metaspace).  Let's take a look at a few examples pointing fragmentation in the Metaspace. 

```
Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
Heap
 PSYoungGen      total 166912K, used 6574K [0x000000076ab00000, 0x0000000775a80000, 0x00000007c0000000)
  eden space 164352K, 4% used [0x000000076ab00000,0x000000076b16ba08,0x0000000774b80000)
  from space 2560K, 0% used [0x0000000774e00000,0x0000000774e00000,0x0000000775080000)
  to   space 2560K, 0% used [0x0000000774b80000,0x0000000774b80000,0x0000000774e00000)
 ParOldGen       total 2522624K, used 4065K [0x00000006c0000000, 0x0000000759f80000, 0x000000076ab00000)
  object space 2522624K, 0% used [0x00000006c0000000,0x00000006c03f8560,0x0000000759f80000)
 Metaspace       used 1751795K, capacity 2097086K, committed 2097132K, reserved 2977792K
  class space    used 94384K, capacity 168958K, committed 168960K, reserved 1048576K
```

The log record above shows that the Java program failed with *java.lang.OutOfMemoryError: Metaspace* error. From the Metaspace and class space usage details, we can see that the committed size and capacity for non-class metaspace is (2097132K-168960K) 1928172K and (2097086K-168958K) 1928128K respectively, while only 1657411K is being used for the non-class metadata. Therefore, there is clearly plenty of free space available in non-class part of Metaspace, but the application still failed with an OutOfMemoryError.

The following logs show that the class space usage is at 24868K while the committed space size is 1048576K, indicating that there is plenty of room available but still the allocation into the class space failed with an OutOfMemoryError.

```
Exception in thread "main" java.lang.OutOfMemoryError: Compressed class space
Heap 
 PSYoungGen      total 894959616, used 0 [0x0000000780000000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 699392K, 0% used [0x0000000780000000,0x0000000780000000,0x00000007aab00000)
  from space 174592K, 0% used [0x00000007aab00000,0x00000007aab00000,0x00000007b5580000)
  to   space 174592K, 0% used [0x00000007b5580000,0x00000007b5580000,0x00000007c0000000)
 ParOldGen       total 3221225472, used 273448976 [0x00000006c0000000, 0x0000000780000000, 0x0000000780000000)
  object space 3145728K, 8% used [0x00000006c0000000,0x00000006d04c8010,0x0000000780000000)
 Metaspace       used 231029K, capacity 248352K, committed 1454592K, reserved 1456128K
  class space    used 24868K, capacity 30973K, committed 1048576K, reserved 1048576K
}
```

The enhancement https://bugs.openjdk.java.net/browse/JDK-8198423 takes care of the fragmentation problem in Metaspace. It is integrated into Java 11. Since the Metaspace barring the separate class space is unlimited, in order to circumvent OutOfMemoryError failures due to fragmentation while running with Java 8, it might be helpful to run without the Compressed class space. Note that this configuration will increase the Java heap space requirements as without the Compressed class space Java objects reference class metadata using 64-bit klass pointers instead of 32-bit offsets.

Hope this helps! More later.







