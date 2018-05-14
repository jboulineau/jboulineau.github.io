---
layout: post
title: "Intro to Data Streaming: Improving the Application"
categories: "Architecture"
---

In the [last post]({% post_url 2017-10-17-a_simple_real-time_app %}) we designed an application architecture that would start to achieve real-time ETL requirements. <!--more-->

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

+ Catastrophic failure
+ Tool Complexity
+ Polyglot Persistence
+ Temporal Inaccuracies
+ Distributed MicroServices
+ Event-Driven Architecture
+ Lossy

However, we still need a solution to some remaining issues:

+ Shrinking batch windows
+ Processing inefficiency
+ Statefull algorithms
+ Reprocessing  

One of the primary issues we have is that messages on the queue are ephemeral. That is, whenever they are read, they are removed from the queue. But, what if we need to **reprocess** messages? This is a common need in ETL. There may have been a failure in processing that left your destination data store in an inconsistent / inaccurate state. Or, logic needs to change due to code defects or shifting business requirements. Often ETL developers utilize a staging data store in part to allow data to be reprocessed without going all the way back to the source (which likely no longer has the same data). While effective this does have the downside of, for example, increased data storage costs, the existence of another point-of-failure, and an additional loading process to maintain.

We also have the problem of not being able to implement efficient statefull algorithms. To review, for our purposes, a statefull algorithm is any logic that requires knowledge of data outside of the working set, which in this case is a single message. Implementing logic that, for example, requires 'Customer' data from the message queue and call center data from the CDC is immensely inefficient. The EDW *may* have the data required, but that means the ETL app has to perform expensive queries for each and every message it processes. This is not an effective design for that need. In order to resolve these problems we need to add two additional components to the architecture, as shown in the following diagram.

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

First, let's replace the message queue with a persistent event store. By simply keeping the messages available we can remove the necessity of a separate staging area in addition to the message queue. Of course, this can't just be a database. The same capabilities of the message queue must exist as a subset of the capabilities of this store. We must be able to utilize the pub/sub pattern and provide the same semantics such as at-most-once message delivery. Persisting data rather than removing them after they have been consumed enables us to simply start from the beginning, or a set point-in-time, in order to reprocess data. Second, we need to add a local cache for the ETL app, preferably an in-memory store. As the ETL app processes new data it can store data it needs to have available for processing. This will typically be a small subset of the data that are persisted in the EDW.



---

{%- include TOC_intro_data_streaming.md -%}
