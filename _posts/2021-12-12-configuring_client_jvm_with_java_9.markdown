---
layout: post
title: "Configuration for Client Applications with Oracle JDK 9+"
date: 2021-12-12
description: # Understanding Metaspace and Class Space GC Log Entries
img:  client_vm_java_11.png 
fig-caption: # Add figcaption (optional)
tags: [Client VM, Performance, Startup, Memory Footprint]
---

Starting with JDK 9, Oracle no longer ships 32-bit JDK builds. JDK 9 also removed the support of *-client* flag, meaning that the Client VM is no longer available on the [Client-class](https://docs.oracle.com/javase/7/docs/technotes/guides/vm/server-class.html) machines. 

But, what if I have a client-side application and want to have fast startup and low memory footprint, as was possible with a 32-bit JDK running in a Client VM mode? Is it possible to achieve that with JDK 9+? Yes, it is. This post attempts to answer these questions.

There is a JVM option **NeverActAsServerClassMachine**, which when enabled instructs the JVM to treat the host machine as a non-Server-class machine. This option helps emulate Client VM behavior using the Server VM. Please refer to [JDK-8166002](https://bugs.openjdk.java.net/browse/JDK-8166002) to understand how the Client VM behavior is emulated. Enabling this flag helps with reducing startup times and CPU usage for client applications.

Additionally, in order to further improve the startup performance, you can configure the initial Metaspace size (which acts as a threshold for Metaspace GCs) so as to avoid the Metaspace related GCs during startup time. This can be done using the JVM option **-XX:MetaspaceSize=n**. Looking at the GC logs can help understand the Metaspace usage, and help in determining the initial Metaspace size.

Now, for containing the overall memory footprint of client applications, the following two flags can help immensely:

**-XX:MinHeapFreeRatio=0**  
**-XX:MaxHeapFreeRatio=0**  

Setting them both to 0 helps keep the committed memory for the Java heap close to its actual usage, and avoids unnecessary expansion of the heap. 

Specifying these flags individually on the command-line can be tedious, especially if that needs to be done on several deployment machines. In order to configure the JVM easily and seamlessly for client applications, the above mentioned JVM options can be specified through a configuration file as well. Since Java 9, the ‘java’ command-line tool accepts an argument to specify a file containing the command-line options. For more details, please refer to [JDK-8027634](https://bugs.openjdk.java.net/browse/JDK-8027634) enhancement.

So we can add these options to a file, say *client_vm_args_file*:

```
-XX:+NeverActAsServerClassMachine
-XX:MinHeapFreeRatio=0
-XX:MaxHeapFreeRatio=0
```
and then launch our Java program using the **@** argument with the java launcher tool, as shown below.

```
java @client_vm_args_file <java class>
```

Hope this helps in achieving quicker startup times and lower memory footprints for your client-side applications. More later!

