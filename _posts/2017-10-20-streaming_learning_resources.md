---  
layout: post
title: "Intro to Data Streaming: Learning Resources"
date: 2017-10-20 20:00:00 -0700
categories: "Architecture"
---


During a presentation at the [Nashville BI user group](https://www.meetup.com/NashBI/), I was asked to provide more material on getting started with data streaming. I'll cover some of these in other 'Intro to Data Streaming' posts, but I promised the group a blog post on the subject, and here it is. There is a ton of material out there, but it will get you started. <!--more--> 

Also for the attendees of the meeting, here is the [deck I used](https://rivuletio.files.wordpress.com/2017/10/real_time_etl.pptx).

[The Log: What every software engineer should know about real-time data's unifying abstraction](https://goo.gl/j6mqP6)  

For many, Jay Kreps' article started it all. Jay, of Linkedin at the time, is the driving force behind Kafka and his critical insight that the ordered, immutable, transaction log can form the conceptual foundation of a data architecture has inspired many to rethink how we build systems and integrate applications. Understanding the principles here is, in my opinion, the best place to start. For a more comprehensive view of the subject, you can also read the [I Heart Logs](https://www.confluent.io/ebook/i-heart-logs-event-data-stream-processing-and-data-integration/) e-book. Jay is now the CEO of Confluent, and speaking of Confluent ...  

[Confluent Blog](https://www.confluent.io/blog/)  

Following Confluent's blog is a must. There is a constant stream (ha) of high quality material here from the best in the business. Confluent is the company supporting Kafka, so expect plenty of material about their product. However, Kafka is also the gold-standard implementation of persistent, distributed, immutable transaction logs so it's non-negotiable reading anyway. 

[Making Sense of Stream Processing](https://www.confluent.io/stream-processing/)  

Martin Kleppmann is another name to know. This free e-book is a great way to orient to the subject at-large. For an even broader perspective, his new book [Designing Data-Intensive Applications](https://dataintensive.net/) digs into thorny issues of distributed systems. (Full disclosure, I'm only part way through this one at the time of writing.)  

[How to Beat the CAP Theorem](http://nathanmarz.com/blog/how-to-beat-the-cap-theorem.html)  

Nathan Marz, the creator of [Apache Storm](https://www.amazon.com/Big-Data-Principles-practices-scalable/dp/1617290343), another major player in data streaming, introduces the Lambda Architecture, which was the de-facto standard in achieving real-time capabilities for a while.   

Marz's design is expounded upon in his book with the rather misleading title [Big Data](https://www.amazon.com/Big-Data-Principles-practices-scalable/dp/1617290343).  

[Questioning the Lambda Architecture](http://radar.oreilly.com/2014/07/questioning-the-lambda-architecture.html)  

Jay Kreps' response to Marz is becoming as influential as his 'The Log' post. In it he points out that the Lambda architecture has more complexity than necessary ... if you make Kafka the center of your architecture, of course.   

[The World Beyond Batch: Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101)  
[The World Beyond Batch: Streaming 102](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-102)  

Tyler Akidau works on the Google Millwheel team. In these O'Reilly (another great resource) posts he expounds upon streaming from a different perspective from the above. He writes about computational matters, especially various aspects of dealing with time. This is a very good view into how streaming architecture not only reaches parity with batch, but surpasses it in capabilities. Reading the [Millwheel VLDB paper](https://research.google.com/pubs/pub41378.html), if you're up for it, is a deep dive.

---

{% include TOC_intro_data_streaming.md %}
