---
layout: post
tite: "Intro to Data Streaming: A Simple Real-Time App"
date:  2017-12-17 20:00:00 
categories: "Data Architecture"
permalink: "/blog/:title"
---

>The previous posts in this series covered the challenges to traditional batch ETL as a result of innate problems and changes in application architecture. In the next two posts a 'from scratch' real-time ETL application will be reviewed, starting simple and incrementally adding sophistication to demonstrate the principles underlying emerging approaches to data engineering. This post covers the most basic real-time (or near-real-time, application).

---

1. [The Problem with Batch ETL, Part 1](/blog/the_problem_with_batch_etl_part_1/)
2. [The Problem with Batch ETL, Part 2](/blog/the_problem_with_batch_etl_part_2/)
3. [A Basic Real-Time ETL Application](/blog/a-simple-real-time-app/)
4. [Improving the Real-Time ETL Application]()
5. [Lambda Architecture]()
6. [Kappa Architecture]()

To refresh from parts [1](/blog/the_problem_with_batch_etl_part_1/) and [2](/blog/the_problem_with_batch_etl_part_2/), the shortcomings of batch ETL include:

+ **Catastrophic failure**: Because we are dealing with many hours worth of data in batch loads, a failure due to a single record out of millions can cause long-running jobs to fail completely, resulting in long recovery times.
+ **Tool Complexity**: Limitations of traditional ETL tools often lead to hundreds or thousands of packages and stored procedures with little attention to standard development principles like code reuse.
+ **Lossy**: Once-daily batch processes take snapshots of data that miss changes that occur throughout the day. We lose data that could be used for analytics.
+ **Temporal Inaccuracies**: ETL and applications are often temporally ignorant, which results in data being related by processing time, rather than event time (the time the data actually becomes valid). This means transactional data can be related to versions of dimensional data that is either older or newer than the transaction itself.
+ **Distributed Microservices**: Monolithic applications (and databases) are being broken into many smaller services, which can dramatically increase ETL integration points.
+ **Polyglot Persistence**: We can no longer count on data being stored only in an RDBMS. NoSQL data stores present challenges in accessing and understanding the data.
+ **Event-Driven Architecture**: Applications or services communicating through the generation of events is increasing in popularity. Traditional ETL tools are not the best option for consuming these events. It has also set the expectation of real-time integration.
+ **Shrinking batch windows**: Businesses are increasingly 24/7 operations. The luxury of having a multi-hour window where server and network resources are minimally used, and application data is changing infrequently, is no longer reality. 

As a first step towards understanding a new architectural approach, consider the following diagram of a very simple ETL application written in, say, C#. This application polls each application database at short intervals (say, every 5 minutes) and writes the output to the data warehouse.    

<pre>
+----------+ 
|          | 
| OLTP DB  +---------------+
|          |               |
+----------+               |
                           ^
+----------+     +---------+--+     +-------+
|          |     |            |     |       |
| OLTP DB  +----->  ETL App   +----->  EDW  |
|          |     |            |     |       |
+----------+     +---------^--+     +-------+
                           |
+----------+               |
|          |               |
| OLTP DB  +---------------+
|          | 
+----------+ 
</pre>

We've immediately gained some benefits over batch ETL. We've significantly reduced the **lossy** shortcoming simply by moving to micro-batching. More frequent runs with delta detection are able to capture more, though not all, changes. **Catastrophic failure** is no longer an issue since failures will impact only 5 minutes of data changes rather than many hours' worth. It will also allow the detection of (and recovery from) failure to occur more quickly. The impact of **shrinking batch windows** has also been reduced because we are making use of more time. Rather than concentrating ETL work into a few hours, we now have 24/7/365 with which to work. We're taking bites rather than eating the elephant whole. However, we really can't call this *fully* solved, yet. We are still limited to vertical scaling. We've also (assuming implementation by a skilled software engineer) eliminated the **complexity** problem inherit in ETL tools. (For more on this, I recommend reading [The Rise of the Data Engineer](https://medium.freecodecamp.org/the-rise-of-the-data-engineer-91be18f1e603)). What do you think about the **temporal inaccuracy** shortcoming? Will this help or hurt?

Of course, we've done nothing to address the issues of **distributed Microservices** and **Event-driven architecture** (although, it's probably easier to deal with API calls and message queues using C# instead of, say, SSIS). **Polyglot persistence** remains as a (potential) impediment. 

We must also now contend with the introduction of new problems: 

### Reprocessing ###
Often it is necessary to change the implementation of transform logic due to modification of business rules, redefinition of calculated metrics, or simply coding errors. In traditional ETL we are typically able to take this capability for granted. We can simply remove the erroneous or out-dated data from our data warehouse and/or marts, modify our code, and reprocess from staging. This is, of course, assuming all data is retained in your staging area. If not, as is more common given the ever growing scale of our data warehouses, the problem is an onerous one. In this case, however, we do not have a staging area and have not replaced the capability.

### Statefull algorithms ###
Here I'm defining 'statefull algorithm' to mean any logic that requires knowledge of data outside of the working set. For example, keeping a running sum of transactions will require not only knowledge of the transactions being processed in the micro-batch, but all transactions that have been processed before. We typically solve this in batch ETL as we solve everything else ... massively expensive queries that pound our staging areas until they produce results. (Given the horsepower of our ETL and staging needs relative to the amount of time in use, I could have included 'waste of hardware resources' as one of the downsides of batch ETL.) However, we are retrieving only the last 5 minutes of changes in our source systems and writing them out, so we don't have this option. Even if we did have a staging areas, unless we install our application on the same server as the staging database we're going to end up with expensive network I/O. One way or another, unless you are one of the lucky few with stateless needs, we need a solution for this deficiency.

### Processing inefficiencies ###
One of the major benefits of batch ETL is that RDBMS implementation architecture favors large transactions for pure efficiency. It is, for instance, a much more efficient use of resources to insert 1M rows in a single transaction than to insert 1M rows in a series of 1,000 row batches. There is overhead related to each transaction such as query optimization, resource allocation, and establishing connection. By moving away from batch ETL we are also increasing the amount of processing overhead that we incur for the same unit of work. 

At this point, our problem set is:  

**Resolved**  
+ Catastrophic failure
+ Tool Complexity

**Problematic**
+ Shrinking batch windows
+ Lossy

**Unresolved**  
+ Temporal Inaccuracies (answer to previous question: hurt! <sup>1</sup>)
+ Distributed MicroServices
+ Polyglot Persistence
+ Event-Driven Architecture
+ Processing inefficiency
+ Statefull algorithms
+ Reprocessing  

So, let's begin to improve this solution to solve some of these problems. Introducing [Change Data Capture](https://en.wikipedia.org/wiki/Change_data_capture) into the architecture is a good place to start. This will make all of the changes that occur in our sources available for use. Let's say we modify our solution to look like this:  

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

Having one or more change-data-capture databases is one way to introduce the capability. [SQL Server](http://bit.ly/2l5ACNl) provides this functionality as a standard feature and it is available for other RDBMS's. RDBMS CDC automagically stores a snapshot of all changes to tables, along with a timestamp of when the change occurred. This is an excellent way of adding this capability for legacy or COTS/ISV applications that cannot, or will not, be modified to publish messages. 

For applications developed for modern architectures, a message queue allows for applications to generate messages or events for consumption by other applications. This is an increasingly heavily used application integration pattern for multiple reasons, but primarily because it allows for decoupling. Because applications now communicate through an intermediary you can more easily change, or even replace, applications without impacting others. The changing application now only has to ensure it maintains the contract of the message queue and all consuming applications remain functional. 

With this architectural change, we've accomplished quite a bit:

+ **Temporal Inaccuracies**: We now have access to event time since CDC timestamps changes as do message queues. 
+ **Distributed MicroServices, Polyglot Persistence, Event-Driven Architecture**: In this architecture we are now down a revolutionary path. We are, as data engineers, moving towards no longer being in the position of cobbling together data as an afterthought (the classic challenge of ETL and BI), but rather participate as first class citizens in *application architecure.* Reading data from a message queue to which applications are publishing their data, just like any other application, allows us to avoid the pain of trying to extract all this data from variant data stores and dozens of services. It also means that our analytics temporal capabilities are on par with application capabilities because we are able to consume events at the same time as those applications. We are closing the divide. We don't care which persistency store is used, nor how many services there are, because we simply have to interact with a well-defined set of queues, a single protocol, and a single format (most commonly JSON).

We're part of the way there, but we're not done yet. In the next post we'll try to solve the thorny issues of **statefull algorithms** and **reprocessing**.

---

1 - If you're here, you're really interested in the answer to the temporal inaccuracies question. The reason why temporal inaccuracies are worse in a microbatch scenario is that you're now running ETL during times of day when transactional applications are most volatile. Because we're usually working with processing time and not event time (unless you're lucky enough to have data sources that track event time), we're still relating data based on the time it is extracted, not the time it is inserted/updated (which is still not necessarily accurate, but that's a story for another post).