---
layout: post
title: "WeakReferences and GC"
date: 2017-02-16
description: 
img:  weakreferences_and_gc.jpg 
fig-caption: # Add figcaption (optional)
tags: [WeakReferences, GC]
---

I was recently working on a memory growth issue involving WeakReferences, and I came across this great blog post on how non-strong references can affect the heap usage and the garbage collection:
https://plumbr.eu/blog/garbage-collection/weak-soft-and-phantom-references-impact-on-gc

It has a demo application that demonstrates an increase in the number of objects getting promoted from young to old generation when WeakReferences are used. That made me deeply interested in investigating the cause of increase in the promotion rate with WeakReferences.

WeakReferences are defined as:

```console
Weak reference objects, which do not prevent their referents from being made finalizable, finalized, and then reclaimed. Weak references are most often used to implement canonicalizing mappings.

Suppose that the garbage collector determines at a certain point in time that an object is weakly reachable. At that time it will atomically clear all weak references to that object and all weak references to any other weakly-reachable objects from which that object is reachable through a chain of strong and soft references. At the same time it will declare all of the formerly weakly-reachable objects to be finalizable. At the same time or at some later time it will enqueue those newly-cleared weak references that are registered with reference queues.
```

I turned to the HotSpot sources to understand how the garbage collector in HotSpot deals with the non-strong references. In simple steps, this is how young garbage collection works and processes the non-strong references.

- scavenge (copy objects from Eden to Survivor or Tenured space)    
    - allocate memory in the destination (say new_object) for the object either in the survivor space or in the tenured gen depending upon its age
    - visit new_object
        - if it is a reference object and its referent is not yet known to be reachable then discover it i.e. add it to its respective - soft, weak or phantom - discovered list (note that soft refs are determined to be discover-able differently than weak refs that itself makes a topic for a separate post)
- process the discovered refs lists prepared during scavenge
    - remove the references from the discovered lists whose referents are still strongly reachable
    - otherwise clear its referent
- enqueue the discovered refs

Following are the comments from the HotSpot sources which I think are worth mentioning here. These comments describe the two policies that the HotSpot can use for discovering Reference objects. HotSpot by default uses ReferenceBasedDiscovery policy. The other policy RefeferentBasedDiscovery is experimental and is not well tested.

```c++
// We mention two of several possible choices here
// #0: if the reference object is not in the "originating generation"
//     (or part of the heap being collected, indicated by our "span"
//     we don't treat it specially (i.e. we scan it as we would
//     a normal oop, treating its references as strong references).
//     This means that references can't be discovered unless their
//     referent is also in the same span. This is the simplest,
//     most "local" and most conservative approach, albeit one
//     that may cause weak references to be enqueued least promptly.
//     We call this choice the "ReferenceBasedDiscovery" policy.
// #1: the reference object may be in any generation (span), but if
//     the referent is in the generation (span) being currently collected
//     then we can discover the reference object, provided
//     the object has not already been discovered by
//     a different concurrently running collector (as may be the
//     case, for instance, if the reference object is in CMS and
//     the referent in DefNewGeneration), and provided the processing
//     of this reference object by the current collector will
//     appear atomic to every other collector in the system.
//     (Thus, for instance, a concurrent collector may not
//     discover references in other generations even if the
//     referent is in its own generation). This policy may,
//     in certain cases, enqueue references somewhat sooner than
//     might Policy #0 above, but at marginally increased cost
//     and complexity in processing these references.
//     We call this choice the "RefeferentBasedDiscovery" policy.
bool ReferenceProcessor::discover_reference(oop obj, ReferenceType rt) {
```

To see things in action, I modified the WeakReferences test program a bit to store objects of class MyObject in the refs[] array so as to be able to identify them easily in the heap dumps. The modified test looks like the following:

```java
public class WeakReferencesPromotion {
    private static final int BUFFER_SIZE = 65536;
    public static volatile MyObject strongRef;

    public static void main(String[] args) throws InterruptedException {
        final Object[] refs = new Object[BUFFER_SIZE];
        for (int index = 0;;) {
            MyObject object = MyObject.createMyObject(); 
            
            // hold a strong reference to this newly created object  
            strongRef = object;   
            // In the next iteration, at this point, this 'object' will only be reachable through 
            // the WeakReference instance at refs[index]
            
            // create WeakReference instace holding 'object' as its referent
            // WeakReference instance itself is atrongly reachable through refs[]
            refs[index++] = new WeakReference<>(object);               
                        
            if (index == BUFFER_SIZE) {
                // make the WeakReference objects in the array unreachable
                Arrays.fill(refs, null);  
                index = 0;
            }
        }
    }    
}

class MyObject {
  byte[] bytes;  
  public static MyObject createMyObject() {
    MyObject obj = new MyObject();
    obj.bytes = new byte[190];
    return obj;
  }
}
```

In the above test program, if a GC (say GC1) occurs when the java thread is at the following statement:
```java
MyObject object = MyObject.createMyObject();
```

At this point, MyObject (say obj1) created in the previous iteration of the 'for' loop is still strongly reachable through 'strongRef'. That MyObject instance (obj1) would become only weakly reachable after 'strongRef' is reassigned with the following:
```java
strongRef = object;
```

So GC1 would see obj1 as strongly reachable but the next GC (say GC2) should see obj1 as reachable only through a weak reference.

Now, to find which objects get promoted and why, I ran this program with **-XX:+TraceReferenceGC -XX:+TraceScavenge** options, collected the heap dumps before Full GC to inspect the objects promoted to the old gen. Please note that -XX:+TraceReferenceGC -XX:+TraceScavenge JVM options are available only in the debug/fastdebug build of the JVM. You can build it yourself from the OpenJDK sources.

```console
java -XX:+HeapDumpBeforeFullGC -Xmx25m -XX:MaxTenuringThreshold=1 -XX:+PrintGCDetails -XX:+TraceReferenceGC -XX:+TraceScavenge WeakReferencesPromotion.java 
```

Note that -XX:MaxTenuringThreshold is set to 1 which means that the objects surviving second GC would get promoted to the tenured gen.

From the heap dumps, I could see that the instances of WeakReference and MyObject were among the top growers in the heap:

```
--------------------------------------------------------------------
Class Name                 | Objects | Shallow Heap | Retained Heap
--------------------------------------------------------------------
java.lang.ref.WeakReference| 473,677 |   15,157,664 |              
byte[]                     |   9,484 |    1,995,992 |              
int[]                      |   2,220 |      542,280 |              
java.lang.Object[]         |     511 |      287,536 |              
MyObject                   |   9,477 |      151,632 |              
--------------------------------------------------------------------
```

Looking at some of the MyObject instances:

```
Class Name                                                      | Shallow Heap | Retained Heap
-----------------------------------------------------------------------------------------------
MyObject @ 0xff627da0                                           |           16 |           224
'- referent java.lang.ref.WeakReference @ 0xff627d80 Unreachable|           32 |           256
MyObject @ 0xff5772e0                                           |           16 |           224
'- referent java.lang.ref.WeakReference @ 0xff575a00 Unreachable|           32 |           256
MyObject @ 0xff4ebce0                                           |           16 |           224
'- referent java.lang.ref.WeakReference @ 0xff4ef060 Unreachable|           32 |           256
MyObject @ 0xff43c120                                           |           16 |           224
'- referent java.lang.ref.WeakReference @ 0xff43c100 Unreachable|           32 |           256
MyObject @ 0xff3c1000                                           |           16 |           224
'- referent java.lang.ref.WeakReference @ 0xff3bf180 Unreachable|           32 |           256
MyObject @ 0xff30e960                                           |           16 |           224
'- referent java.lang.ref.WeakReference @ 0xff30b9a0 Unreachable|           32 |           256
MyObject @ 0xff2a1cc0                                           |           16 |           224
'- referent java.lang.ref.WeakReference @ 0xff2a1ca0 Unreachable|           32 |           256
-----------------------------------------------------------------------------------------------
```

From the GC logs, the address ranges of the young and the old generations are:

```console
 PSYoungGen      total 6144K, used 4064K [0x00000000ff780000, 0x0000000100000000, 0x0000000100000000)
  eden space 3584K, 100% used [0x00000000ff780000,0x00000000ffb00000,0x00000000ffb00000)
  from space 2560K, 18% used [0x00000000ffd80000,0x00000000ffdf8000,0x0000000100000000)
  to   space 2560K, 0% used [0x00000000ffb00000,0x00000000ffb00000,0x00000000ffd80000)
 ParOldGen       total 17920K, used 7208K [0x00000000fe600000, 0x00000000ff780000, 0x00000000ff780000)
```

From these address ranges, we can see that the above sample set of WeakReferences and their referents lie in the tenured gen, and they are unreachable. Looking at the first sample, and searching for the MyObject instance at address 0xff627da0 in the gc logs shows:

```
D:\tests\GC_WeakReferences> grep ff627da0 gc.1.log
{tenuring MyObject 0x00000000ffdba600 -> 0x00000000ff627da0 (2)}
{forwarding MyObject 0x00000000ffdba600 -> 0x00000000ff627da0 (2)}
```

This means that MyObject instance (0x00000000ffdba600) was in one of the survivor spaces and was tenured to the address 0x00000000fe600420 in the old gen.

Searching for ffdba600 in the logs gets us:

```
{copying MyObject 0x00000000ffcffe40 -> 0x00000000ffdba600 (2)}
{forwarding MyObject 0x00000000ffcffe40 -> 0x00000000ffdba600 (2)}
```

This means that this MyObject instance was in Eden at address 0x00000000ffcffe40, it survived and was copied to the survivor space at address 0x00000000ffdba600.

To put things in context, I extracted all the log entries containing WeakReference instance at 0xff627d80 and MyObject instance at 0xffdba600 from the gc logs:

```
//GC #11
Discovered reference (mt) (0x00000000ffd08000: java.lang.ref.WeakReference)
...
{copying java.lang.ref.WeakReference 0x00000000ffcfff20 -> 0x00000000ffd08000 (4)}  //copy reference object from eden to a survivor space
   Process discovered as normal 0x00000000ff4f63b8
{forwarding java.lang.ref.WeakReference 0x00000000ffcfff20 -> 0x00000000ffd08000 (4)}
...
Dropping strongly reachable reference (0x00000000ffd08000: java.lang.ref.WeakReference) //Dropped from discovered list because its referent is strongly reachable at the moment.
{copying MyObject 0x00000000ffcffe40 -> 0x00000000ffdba600 (2)}
{forwarding MyObject 0x00000000ffcffe40 -> 0x00000000ffdba600 (2)}
 Dropped 1 active Refs out of 5630 Refs in discovered list 0x00000000ffd5c040
.....

//GC #12
{tenuring java.lang.ref.WeakReference 0x00000000ffd08000 -> 0x00000000ff627d80 (4)} //Weak reference gets promoted and does not get discovered in this scavenge because it does not lie in the young generation now.
...
{forwarding java.lang.ref.WeakReference 0x00000000ffd08000 -> 0x00000000ff627d80 (4)}
...
{tenuring MyObject 0x00000000ffdba600 -> 0x00000000ff627da0 (2)}  // its referent object gets promoted too
{forwarding MyObject 0x00000000ffdba600 -> 0x00000000ff627da0 (2)}
```

I added comments besides the log entries above to explain what is happening with each log entry. Here's the complete story:

During the scavenge process, GC #11 finds the weak reference (0x00000000ffcfff20) strongly reachable and copies that and its referent to the survivor space, and adds the weak reference to the discovered references list. But this weak reference gets removed from the discovered references list when GC finds its referent still being strongly reachable during the references-processing phase.

The next GC, GC #12 finds that weak reference still strongly reachable (through ref[]) and copies that and its referent to the tenured space during the scavenge phase. At this point, the referent is not strongly reachable and one would expect it to get collected. But since the weak reference had been moved to the tenured gen, with the ReferenceBasedDiscovery discovery policy, the garbage collector does not discover it i.e. does not add it to the discovered references list and does not treat it like a reference object. The weak reference is treated like a normal reachable object and its referent too gets copied over to the tenured generation.

If we increase the tenuring threshold from 1 to 2, the weak references like in the above example would get collected in the young generation itself. The second GC would copy the weak reference object to the survivor space and that would allow the weak reference to get discovered during the scavenge phase. The reference processing phase of this GC would then clear the referent and would enqueue the reference object.
Final Thoughts

Weak references and their referents can get prematurely promoted to the old generation. By increasing -XX:MaxTenuringThreshold to a sufficiently large number, we can avoid this premature promotion.