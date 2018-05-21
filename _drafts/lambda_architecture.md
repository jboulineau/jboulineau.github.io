---
layout: post
title: "Intro to Data Streaming: Lambda Architecture"
series: "Intro to Data Streaming"
excerpt: "Through the first parts in this series we have covered problems with batch ETL processes and conceptually designed a real-time data processing system. In this post the series shifts to looking at reference architectures that have been successfully used to implement real-time data streaming solutions. The first of these is known as the _Lambda Architecture_. The next part of the series will cover the _Kappa Architecture_."
---

<!--
TODO
Add twitter links for Kleppman and Abadi
-->

In the last part of the series we (nearly) completed the design of a conceptual real-time data streaming application, but were left with two remaining issues: **shrinking batch windows** and **processing inefficiency**. In large part the IT community has settled upon the solution: fault-tolerant compute clusters and distributed file systems, lead primarily by the Hadoop ecosystem. We can mitigate the two primary challenges here, processing failures and data size, by scaling processing across multiple computers and building in (some types of) fault-tolerance to the system. It is in this context that the _Lambda Architecture_ was conceived. 

The _Lambda<sup>1</sup> Architecture_ was defined in a questionably titled [2011 blog post by Nathan Marz](http://nathanmarz.com/blog/how-to-beat-the-cap-theorem.html) and later further detailed in his book, [Big Data](https://www.manning.com/books/big-data). While this post captures the relevant points for the series, I encourage you to read at least Marz's original blog post. A Twitter employee at the time, Nathan's was coming from a perspective that assumes the need for a distributed system. The title of his post, _How to Beat the CAP Theorem_, shows that he was concerned with the fundamental tradeoffs of distributed systems: consistency, availability, and partition tolerance. A lot has been discussed about the CAP Theorem, much of which exposed the misconceptions and limited usefulness of the theorem as defined (see [Problems with CAP, and Yahoo’s little known NoSQL system](http://dbmsmusings.blogspot.com/2010/04/problems-with-cap-and-yahoos-little.html) by Daniel Abadi and especially [Please stop calling databases CP or AP](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html)] by Martin Kleppmann ). However, Marz was wrestling with an essential question: given the reality of unreliable systems (and humans), how can we meet the potentially competing requirements of producing accurate data processing results in real-time?

For Marz, batch processing is a great solution for data processing. He observed that batch processing has a number of critical benefits. At least in the presence of a system like Hadoop, it is completely scalable. Most importantly, it is as simple as it gets. Apply logic to a set of unchanging data and receive accurate results. Now, Marz is assuming, that traditional batch processing have been solved through modifications to the overall architecture as described in parts [3]({% post_url 2017-10-17-a_simple_real-time_app %}) and [4]({% post_url 2018-05-14-improving_the_application %}) of this series. In that case, Marz is correct the only problem with batch processing is that it does not produce results in real-time. 

Marz described an architecture of 'layers', the 'speed layer,' 'batch layer,' and 'serving layer,' as shown in the following figure: 

<p><center><img src="/assets/images/lambda.png" alt="Lambda Architecture"></center></p>

[Image source](https://dzone.com/articles/lambda-architecture-with-apache-spark)

The batch layer is composed, as you would expect, of a distributed file system such as HDFS and a computation engine like Spark. In order to achieve real-time capabilities, however, another set of systems must be included. A low-latency distributed data store and a built-for-purpose real-time computation engine such as the Marz-authored [Apache Storm](http://storm.apache.org/), which is a popular option, are also required. 

The concept is this: the batch layer provides the foundation of the architecture by producing provably-accurate results at a point-in-time at great scale. In the time between execution of batch processing, the 'speed layer' is to consume data as it arrives and provide approximate calculations. These calculations may suffer from inaccuracies inherent to non-batch processing, such as those caused by out-of-order messages. But, not to worry, these temporary inaccuracies are corrected by the batch layer. The results of applying the batch and real-time functions are separate views of the data that must be unioned at the 'serving layer.' The 'serving layer' is responsible for ensuring that the results produced by the two data processing layers are reconciled and presented in a semantically consistent manner. 

----
1. The term _Lambda_ refers to the computer science concept of anonymous functions, or functions in a system of variable replacement. That is, the architecture is described by applying the same function or functions to the data regardless of the layer of operations. 





http://jameskinley.tumblr.com/post/37398560534/the-lambda-architecture-principles-for

https://jvns.ca/blog/2016/11/19/a-critique-of-the-cap-theorem/