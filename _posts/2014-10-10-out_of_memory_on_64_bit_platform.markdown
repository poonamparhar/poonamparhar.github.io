---
layout: post
title: "Running on a 64-bit platform and still running out of memory?"
date: 2014-10-10
description: 
img:  compressed_oops_native_oom.jpg 
fig-caption: # Add figcaption (optional)
tags: [Troubleshooting, 64-bit, OOM, Compressed Oops]
---

It sounds strange that we might run out of native memory even while running on a 64-bit platform. On a 64-bit machine and while running with a 64-bit JVM, we get almost unlimited memory, then how is it possible to run out of memory? Well, it may happen in a certain situation as described in this post.

The HotSpot JVM has a feature called **CompressedOops** wherein it uses 32-bit offsets (compressed oops) on 64-bit platforms in order to have smaller memory footprints and better performance for Java applications. With CompressedOops enabled, 64-bit address values of Java objects are represented as 32-bit offsets using an encoding, and those 64-bit addresses are retrieved back using a corresponding decoding.

The CompressedOops implementation tries to allocate Java heap using different strategies based on the heap size and the platform that an application runs on. If the heap size is less than 4GB, then it tries to allocate it into the lower virtual address space (below 4GB) so that the compressed oops can be used without any encoding or decoding (**Unscaled**). If that allocation attempt fails or the heap size is greater than 4GB, then it attempts to allocate the Java heap below the 32GB address boundary in order to use the **Zero Based** compressed oops. If that attempt also fails, then it switches to using regular compressed oops having a **Narrow Oop Base**. If you are interested in learning more details of the CompressedOops implementation, please refer to [CompressedOops](https://wiki.openjdk.java.net/display/HotSpot/CompressedOops).

Now, the JVM needs to compute the base where the Java heap gets placed at. The computation of the heap-base depends on the heap size and the value of the JVM option **HeapBaseMinAddress** for the platform that the application is running on. If the **Java heap size+HeapBaseMinAddress** can fit within the first 4GB of the address space, then the heap is based below 4GB. If it doesn't, then it is allocated below the 32GB boundary if it can fit there, otherwise the Java heap is allocated in the address space higher than 32GB. 

In JDK8 source base, the heap-base computation is implemented by the JVM function *Universe::preferred_heap_base()*, available at the following link: [http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/memory/universe.cpp#l672](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/memory/universe.cpp#l672)

Interestingly, for cases where the Java heap gets allocated below 4GB or 32GB address boundaries, a **limited space** gets left for the native heap. And that can lead to native out-of-memory errors for applications that need to make intensive native memory allocations.

This is evident from the following *process map* that shows that the Java heap got based at 8GB (00000001FB000000) heap-base, leaving only 4GB (00000001FB000000-0000000100110000) of space for the native heap.

```console
0000000100000000          8K r-x--  /sw/.es-base/sparc/pkg/jdk-1.7.0_60/bin/sparcv9/java
0000000100100000          8K rwx--  /sw/.es-base/sparc/pkg/jdk-1.7.0_60/bin/sparcv9/java
0000000100102000         56K rwx--    [ heap ]
0000000100110000       2624K rwx--    [ heap ]   <--- native heap
00000001FB000000      24576K rw---    [ anon ]   <--- Java heap starts here
0000000200000000    1396736K rw---    [ anon ]
0000000600000000     700416K rw---    [ anon ]
```

## Workarounds for this situation:

There are two possible workarounds for this situation:

1. Run with **-XX:-UseCompressedOops** (keeping in mind that you would lose the performance benefit that comes with enabling CompressedOops). This will instruct the JVM to run without the CompressedOops feature, and it will not attempt to place the Java heap in the first 4GB or 32GB of the address space.

2. Keep CompressedOops enabled, but configure the base of the Java heap using the JVM option **-XX:HeapBaseMinAddress=n** to specify the address where the Java heap should start from. If your application is running out of native memory due to a capped/limited native heap, setting this option to a higher address would help. If you need the Java heap to be placed beyond 4GB or 32GB then set HeapBaseMinAddress to a value such that `HeapBaseMinAddress + heap size > 4GB or 32GB`. The default value of HeapBaseMinAddress on most platforms is 2GB.

## HeapBaseMinAddress and Java heap placement

Let's run a simple Java program on solaris-sparc 10 and solaris-x64 11 and see where the Java heap gets placed on these platforms when CompressedOops is enabled.

### On Solaris-sparc 10 with JDK 1.7.0_67

```console
$ uname -a
SunOS svr 5.10 Generic_148888-03 sun4v sparc SUNW,Sun-Fire-T200

$ java -version
java version "1.7.0_67"
Java(TM) SE Runtime Environment (build 1.7.0_67-b32)
Java HotSpot(TM) Server VM (build 24.65-b04, mixed mode)

$ java -d64 -XX:+PrintCommandLineFlags -XX:+PrintFlagsFinal TestClass &
[3] 22638
$ -XX:InitialHeapSize=266338304 -XX:MaxHeapSize=4261412864 -XX:+PrintCommandLineFlags -XX:+PrintFlagsFinal -XX:+UseCompressedOops -XX:+UseParallelGC
[Global flags]
    uintx AdaptivePermSizeWeight                    = 20              {product}
    ...<snip>...
    uintx GCTimeRatio                               = 99              {product}
    uintx HeapBaseMinAddress                        = 4294967296      {pd product}
    bool HeapDumpAfterFullGC                        = false           {manageable}
    ....<snip>...
    Heap Parameters:
ParallelScavengeHeap [ PSYoungGen [ eden =  [0xffffffff1e800000,0xffffffff1ebf5d88,0xffffffff22a00000] , from =  [0xffffffff235000
00,0xffffffff23500000,0xffffffff24000000] , to =  [0xffffffff22a00000,0xffffffff22a00000,0xffffffff23500000]  ] PSOldGen [  [0xfffffffe75400000,0xfffffffe75400000,0xfffffffe7fc00000]  ] PSPermGen [  [0xfffffffe70400000,0xfffffffe706843a0,0xfffffffe71c00000]  ]
```

As we can see, the default value of HeapBaseMinAddress is 4GB here, and the Java heap including the PermGen starts at 0xfffffffe70400000, which is in the higher virtual address space much beyond the 32GB boundary.

### On Solaris-x64 11 with JDK 1.7.0_67

```console
bash-4.1$ uname -a
SunOS hsdev-15 5.11 11.0 i86pc i386 i86pc

bash-4.1$ java -d64 -XX:+PrintCommandLineFlags -XX:+PrintFlagsFinal TestClass &
[1] 6049
bash-4.1$ -XX:InitialHeapSize=805015296 -XX:MaxHeapSize=12880244736 -XX:+PrintCommandLineFlags -XX:+PrintFlagsFinal -XX:+UseCompressedOops -XX:+UseParallelGC
[Global flags]
    ....
    uintx GCTimeRatio                              = 99              {product}
    uintx HeapBaseMinAddress                       = 268435456       {pd product}
    bool HeapDumpAfterFullGC                       = false           {manageable}
    ....

bash-4.1$ java -d64 -classpath $JAVA_HOME/lib/sa-jdi.jar sun.jvm.hotspot.CLHSDB 6049
Attaching to process 6049, please wait...
hsdb> universe
Heap Parameters:
ParallelScavengeHeap [ PSYoungGen [ eden =  [0x0000000700000000,0x00000007007ae238,0x000000070c000000] , from =  [0x000000070e0000
00,0x000000070e000000,0x0000000710000000] , to =  [0x000000070c000000,0x000000070c000000,0x000000070e000000]  ] PSOldGen [  [0x0000000500400000,0x0000000500400000,0x0000000520200000]  ] PSPermGen [  [0x00000004fb200000,0x00000004fb483380,0x00000004fc800000]  ]

bash-4.1$ pmap 6049
0000000000400000          4K r-x--  /java/bin/amd64/java
0000000000410000          4K rw---  /java/bin/amd64/java
0000000000411000       2288K rw---    [ heap ]
00000004FB200000      22528K rw---    [ anon ]  <----- Java heap
0000000500400000     522240K rw---    [ anon ]
0000000700000000     262144K rw---    [ anon ]
```

Here, the default value of HeapBaseMinAddress is 256MB, and the Java heap allocation starts at 00000004FB200000 (=19GB); below the 32GB address boundary. The default value of HeapBaseMinAddress on solaris-x64 has been increased to 2GB in JDK8 with [JDK-8013938](https://bugs.openjdk.java.net/browse/JDK-8013938). 

### On Solaris-x64 11 with JDK 1.8.0_20

```console
bash-4.1$ java -version
java version "1.8.0_20"
Java(TM) SE Runtime Environment (build 1.8.0_20-b26)
Java HotSpot(TM) 64-Bit Server VM (build 25.20-b23, mixed mode)

bash-4.1$ java -d64 -XX:+PrintCommandLineFlags -XX:+PrintFlagsFinal TestClass &
[1] 19069

bash-4.1$ -XX:InitialHeapSize=805015296 -XX:MaxHeapSize=12880244736 -XX:+PrintCommandLineFlags -XX:+PrintFlagsFinal -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC
[Global flags]
    uintx AdaptiveSizeDecrementScaleFactor          = 4                                   {product}
    ....
    uintx GCTimeRatio                               = 99                                  {product}
    uintx HeapBaseMinAddress                        = 2147483648                          {pd product}
    bool HeapDumpAfterFullGC                        = false                               {manageable}
    .....

bash-4.1$ java -d64 -classpath $JAVA_HOME/lib/sa-jdi.jar sun.jvm.hotspot.CLHSDB 19069
Attaching to process 19069, please wait...
hsdb> universe
Heap Parameters:
ParallelScavengeHeap [ PSYoungGen [ eden =  [0x00000006c0200000,0x00000006c190a4d8,0x00000006cc200000] , from =  [0x00000006ce2000
00,0x00000006ce200000,0x00000006d0200000] , to =  [0x00000006cc200000,0x00000006cc200000,0x00000006ce200000]  ] PSOldGen [  [0x00000004c0400000,0x00000004c0400000,0x00000004e0400000]  ]  ]

bash-4.1$ pmap 19069
19069:  java -d64 -XX:+PrintCommandLineFlags -XX:+PrintFlagsFinal TestClass
0000000000411000       3696K rw---    [ heap ]
00000004C0400000     524288K rw---    [ anon ]
00000006C0200000     262144K rw---    [ anon ]
00000007C0000000        512K rw---    [ anon ]
```

In this case, the HeapBaseMinAddress value is 2GB by default, and the Java heap including the PermGen starts at 00000004C0400000, which lies in the first 32GB address space.


Hope this helps!

