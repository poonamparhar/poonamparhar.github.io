---
layout: post
title: "Crashes in ZIP_GetEntry"
date: 2016-09-28
description: 
img:  crashes_zip_getentry.png 
fig-caption: # Add figcaption (optional)
tags: [Crash, libzip, ZIP_GetEntry]
---


Recently, we received quite a few reports of application crashes from different customers and product groups with the stack trace looking like the following:

```
Stack: [0xb0c00000,0xb0c80000],  sp=0xb0c7c890,  free space=498k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
C  [libc_psr.so.1+0x700]  memcpy+0x2f8
C  [libzip.so+0xd19c]
C  [libzip.so+0x2380]  ZIP_GetEntry+0xe4
C  [libzip.so+0x2800]  Java_java_util_zip_ZipFile_getEntry+0xc4
j  java.util.zip.ZipFile.getEntry(JLjava/lang/String;Z)J+0
j  java.util.zip.ZipFile.getEntry(JLjava/lang/String;Z)J+0
j  java.util.zip.ZipFile.getEntry(Ljava/lang/String;)Ljava/util/zip/ZipEntry;+31
j  java.util.jar.JarFile.getEntry(Ljava/lang/String;)Ljava/util/zip/ZipEntry;+2
j  java.util.jar.JarFile.getJarEntry(Ljava/lang/String;)Ljava/util/jar/JarEntry;+2
```

Most of the times, the crashes in ZIP_GetEntry occur when the jar file being accessed has been modified/overwritten while the JVM instance was running. For performance reasons, the HotSpot JVM memory-maps each Jar file's central directory structure using **mmap**. This is done so as to avoid reading the central directory structure data from the disk every time it needs to read entries from the Jar file. When a Jar file is modified or overwritten on the disk, the JVM's copy of the data it read before becomes inconsistent with the jar file on the disk, and any attempt to read and load entries from the modified jar can result in an application crash.

Since 1.6.0_23, a property can be used to disable the memory mapping of the central directory structure of Jar files:
```
-Dsun.zip.disableMemoryMapping=true
```

Please note that enabling this property can have some performance impact on the application as the JVM needs to read the central directory structure from the Jar files on the disk again and again whenever it reads a Jar file entry. So, it is best to ensure that the jar files are not modified or overwritten while the JVM has an image of them loaded.

In Java 9, this ZIP crash has been resolved with the following fix:

**[JDK-8142508](https://bugs.openjdk.java.net/browse/JDK-8142508): To bring j.u.z.ZipFile's native implementation to Java to remove the expensive jni cost and mmap crash risk**

The code review thread for this change is [here](http://mail.openjdk.java.net/pipermail/core-libs-dev/2015-November/036495.html)

This fix replaces the ZIP file native implementation that used mmap with the Java implementation, and that removes the risk of application crashes described above.
