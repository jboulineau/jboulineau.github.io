---
layout: post
title: "Improving the Real-Time App"
date: 2018-05-14 21:20:00 -0600
series: "Intro to Data Streaming"
categories: "Architecture"
---

In the [last post]({% post_url 2017-10-17-a-simple-real-time-app %}) we considered an application architecture that would start to achieve real-time ETL requirements, but there were issues remaining with the design. In this post, we will improve the design to further improve upon batch processing and understand the data streaming pattern. <!--more-->

Here's where our system design stands as of the last post:

<pre>
+-------------+ 
|             | 
| OLTP DB     +--------+                
|             |        |         
+-------------+        |         
                       |     
+-------------+  +-------------+     +---------+--+     +-------+
|             |  |             |     |            |     |       |
| OLTP DB     +-->   CDC DB    + ---->  ETL App   +----->  EDW  |
|             |  |             |     |            |     |       |
+-------------+  +-------------+     +--------^---+     +-------+
                                              |
+-------------+  +-------------+              |
|             |  |             |              |
| Micro Svc   +- > Msg Queue   + -------------+
|             |  |             |               
+-------------+  +-------------+
</pre>

With this design we've accomplished quite a lot. Our application deals with:

- Catastrophic failure
- Tool Complexity
- Polyglot Persistence
- Temporal Inaccuracies
- Distributed MicroServices
- Event-Driven Architecture
- Data Loss

However, we still need a solution to some remaining issues:

- Shrinking batch windows
- Processing inefficiency
- Statefull algorithms
- Reprocessing  

One of the primary issues we have is that messages on the queue are ephemeral. That is, whenever they are read they are removed from the queue. But, what if we need to **reprocess** messages? There may have been a failure in processing that left your destination data store in an inconsistent / inaccurate state. Or logic may need to change due to code defects or shifting business requirements. Often ETL developers utilize a staging data store in part to allow data to be reprocessed without going all the way back to the source (which likely no longer has the same data). While effective this does have the downside of, for example, increased data storage costs, the creation of another point-of-failure, and an additional loading process to maintain.

We also have the problem of not being able to implement efficient stateful algorithms. To review, for our purposes a stateful algorithm is any logic that requires knowledge of data outside of the working set, which in this case is a single message. Implementing logic that, for example, requires 'Customer' data from the message queue and call center data from the CDC is immensely inefficient. The EDW *may* have the data required, but that means the ETL app has to perform expensive queries for each and every message it processes. This is not an effective design for that need. In order to resolve these problems we need to add two additional components to the architecture, as shown in the following diagram.

<pre>
+-------------+                      +------------+
|             |                      |            |
| OLTP DB     +--------+             |  Cache     |
|             |        |             |            |
+-------------+        |             +------+-----+
                       |                    |
+-------------+  +-----+-------+     +------+-----+     +-------+
|             |  |             |     |            |     |       |
| OLTP DB     +-->   CDC DB    | +--->  ETL App   +----->  EDW  |
|             |  |             |     |            |     |       |
+-------------+  +-------------+     +--------^---+     +-------+
                                              |
+-------------+  +-------------+              |
|             |  | Persistent  |              |
| Micro Svcs  ++ | Event Store | +------------+
|             |  |             |
+-------------+  +-------------+
</pre>

First, let's replace the message queue with a persistent event store. By simply keeping the messages available we can remove the necessity of a separate staging area in addition to the message queue. Of course, this can't just be a database. The same capabilities of the message queue must exist as a subset of the capabilities of this store. We must be able to utilize the pub/sub pattern and provide the same semantics such as at-most-once message delivery. Persisting data rather than removing them after they have been consumed enables us to simply start from the beginning, or a set point-in-time, in order to reprocess data. 

Second, we need to add a local cache for the ETL app to allow implementing stateful algorithms. Ideally this will be an in-memory store for performance reasons. While it could be a disk-based data store that is on the same hardware as the ETL app, the farther we get away from memory-resident data the worse off we are going to be. As the ETL app processes new data it can store data it needs to have available for processing future messages. This will typically be a small subset of the data that are persisted in the EDW. Now, our app can perform calculations as simple as running averages or as complex as fraud detection. 

Now, let's consider our remaining design challenges, handling **shrinking batch windows** and **processing inefficiency**. While we've expanded our processing window to 24/7/365, which helps handle the shrinking batch window issue, we've actually made these two issues more difficult by introducing RBAR (row-by-agonizing row) processing. We also need to consider carefully the impact of processing failures of all kinds. As the value of analytics solutions grows ever greater, tolerance for outages, and the window for recovery, is in much shorter supply. We'll handle these two issues in the subsequent parts in the series.

You might be thinking at this point, "this is a lot more complicated than the ETL I'm used to." If you aren't now, you will. As you dive deeper into the nuances of the solution space you will realize that there is much more to the matter than is covered in this series. Persistence mechanisms for real-time processing, for instance, is a blog series all its own. Architecting real-time solutions introduces entire categories of problems that do not exist, or are at least more rare and less visible, in batch ETL scenarios. In this, and many situations, I'm reminded of one of my favorite quotes, often attributed to [Einstein](https://quoteinvestigator.com/2011/05/13/einstein-simple/):

> Everything should be made as simple as possible, but not simpler.

ETL is by it's very nature complex due to the variance of data sources, hardware limitations, tool limitations, and the sheer number of things that could happen that are outside of your control, but can still break your code. Adding in real-time challenges and the demands of modern analytic methods adds many different dimensions. So the question 'is it simple' will always yield 'No.' The real question is, 'is it as simple as possible, given the requirements of the system?' Fortunately, reference architectures exist to help us reason about how we might implement an as-simple-as-possible system. The remaining posts in this series will focus on two of these: the _Lambda Architecture_ and the _Kappa Architecture_.
