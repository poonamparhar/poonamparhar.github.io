---
layout: post
title: "Why do I get message 'CodeCache is full. Compiler has been disabled'?"
date: 2015-09-10
description: 
img:  codecache_full.jpg 
fig-caption: # Add figcaption (optional)
tags: [CodeCache, JIT, Compiler]
---

The JVM JIT generates compiled code and stores that in a memory area called CodeCache. The default maximum size of the CodeCache on most of the platforms is 48M. If any application needs to compile large number of methods resulting in huge amount of compiled code then the CodeCache may become full. When it becomes full, the compiler is disabled to stop any further compilations of methods, and a message like the following gets logged:

```
Java HotSpot(TM) 64-Bit Server VM warning: CodeCache is full. Compiler has been disabled.
Java HotSpot(TM) 64-Bit Server VM warning: Try increasing the code cache size using -XX:ReservedCodeCacheSize=
Code Cache  [0xffffffff77400000, 0xffffffff7a390000, 0xffffffff7a400000) total_blobs=11659 nmethods=10690 adapters=882 free_code_cache=909Kb largest_free_block=502656
```

When this situation occurs, the JVM may invoke sweeping and flushing of this space to make some room available in the CodeCache. There is a JVM option **UseCodeCacheFlushing** that can be used to control the flushing of the Codeache. With this option enabled JVM invokes an emergency flushing that discards older half of the compiled code(nmethods) to make space available in the CodeCache. In addition, it disables the compiler until the available free space becomes more than the configured **CodeCacheMinimumFreeSpace**. The default value of **CodeCacheMinimumFreeSpace** option is 500K.

**UseCodeCacheFlushing** is set to false by default in JDK6, and is enabled by default since JDK7u4. This essentially means that in jdk6 when the CodeCache becomes full, it is not swept and flushed and further compilations are disabled, and in jdk7u+, an emergency flushing is invoked when the CodeCache becomes full. Enabling this option by default made some issues related to the CodeCache flushing visible in jdk7u4+ releases. The following are two known problems in jdk7u4+ with respect to the CodeCache flushing:

1. The compiler may not get restarted even after the CodeCache occupancy drops down to almost half after the emergency flushing.
2. The emergency flushing may cause high CPU usage by the compiler threads leading to overall performance degradation.

This performance issue, and the problem of the compiler not getting re-enabled again has been addressed in JDK8. To workaround these in JDK7u4+, we can increase the code cache size using ReservedCodeCacheSize option by setting it to a value larger than the compiled-code footprint so that the CodeCache never becomes full. Another solution to this is to disable the CodeCache Flushing using **-XX:-UseCodeCacheFlushing** JVM option.

The above mentioned issues have been fixed in JDK8 and its updates. Here's the list of the bugs that have been fixed in jdk8 and 8u20 that address these problems:
- JDK-8006952: CodeCacheFlushing degenerates VM with excessive codecache freelist iteration (fixed in 8)
- JDK-8012547: Code cache flushing can get stuck without completing reclamation of memory (fixed in 8)
- JDK-8020151: PSR:PERF Large performance regressions when code cache is filled (fixed in 8)
- JDK-8029091 Bug in calculation of code cache sweeping interval (fixed in 8u20)
- JDK-8027593: performance drop with constrained codecache starting with hs25 b111 (fixed in 8)
