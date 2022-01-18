---
layout: post
title: "Can young generation size impact the application response times?"
date: 2017-12-11
description: 
img:  young_gen_size_response_time.jpg 
fig-caption: # Add figcaption (optional)
tags: [Young Gen, Sizing, Response Time]
---

I was recently involved in a performance issue where the application's response times were decreasing over time, and I was tasked to determine if GC was playing any role in that.

Well, as we all know that misconfiguration of the Java heap space and its generations can lead to unexpected/frequent/longer garbage collection cycles causing inferior performance and longer response times of the application, and that indeed was the cause of this performance issue.

I want to share some details of this issue, how it was resolved, and also want to emphasize on the importance of proper configuration of the memory pools that are managed by the garbage collector in the JVM.

For this case, the first thing to look at, of course, were GC logs. The application was configured to run with the CMS and ParNew collectors. The GC logs revealed the following:

1. There were too frequent ParNew collections.

    This indicated that the young generation was sized smaller than the allocation rate of the application.

2. Pause times of ParNew collections were increasing over time.

    This meant that the time taken to promote young objects from the Young generation to the CMS generation was increasing over time.

3. There were ‘ParNew promotion’ and ‘CMS concurrent mode’ failures.

    Here’s a log entry pertaining to one of the concurrent mode failures:

```
[GC: [ParNew (promotion failed): 528684K->529315K(613440K), 0.0234550 secs] [CMS: [CMS-concurrent-sweep: 2.807/3.322 secs] [Times: user=5.09 sys=0.15, real=3.32 secs]  (concurrent mode failure): 4425412K->3488043K(7707072K), 10.4466460 secs] 4954097K->3488043K(8320512K), [CMS Perm : 389189K->388749K(653084K)] , 10.4703900 secs] [Times: user=10.61 sys=0.00, real=10.46 secs]
```

The interesting thing to notice here is that there is plenty of room available (~ 3GB) in the CMS generation yet the promotion of around 500MB (at the most) of young objects had failed. This indicated that there was severe fragmentation in the CMS space.

To confirm that the application was suffering from fragmentation in the CMS generation, we collected GC logs with the **-XX:PrintFLSStatistics=1** JVM option. FLS statistics showed that the Maximum and the average chunk sizes of the binary tree dictionaries in the CMS space were decreasing over time, indicating that the fragmentation was increasing.

All the above three observations helped us conclude that the young space was sized smaller than the application’s actual requirement, and the CMS space was suffering from acute fragmentation. The application was tested with increased young generation sizes. Please note that the JVM options **-XX:NewSize=n and -XX:MaxNewSize=m, or -XX:NewRatio=p** can be used to configure the young generation size.

Increasing the young gen size helped address all the above points:

- It helped in reducing the ParNew collections frequency
- It helped in reducing premature promotions of objects into the old gen, and that helped combat CMS fragmentation
- With reduced fragmentation, the rising ParNew GC times also became stable.

This case and the involved exercise showed us again how important it is to size the JVM memory pools in accordance with our application’s requirements.

More later!