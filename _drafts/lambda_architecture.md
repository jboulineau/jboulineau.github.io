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

In the last part of the series we (nearly) completed the design of a conceptual real-time data streaming application, but were left with two remaining issues: **shrinking batch windows** and **processing inefficiency**. 

The Lambda Architecture was first defined in a [2011 blog post by Nathan Marz](http://nathanmarz.com/blog/how-to-beat-the-cap-theorem.html) and later further detailed in his book, [Big Data](https://www.manning.com/books/big-data). A Twitter employee at the time, Nathan's was coming from a perspective that assumes the need for a distributed system. The title of his post, _How to Beat the CAP Theorem_, shows that he was concerned with the fundamental tradeoffs of distributed systems: consistency, availbility, and partition tolerance. A lot has been discussed about the CAP Theorem, much of which exposed the misconceptions and limited usefullness of the theorem as defined (see [Problems with CAP, and Yahoo’s little known NoSQL system](http://dbmsmusings.blogspot.com/2010/04/problems-with-cap-and-yahoos-little.html) by Daniel Abadi and especially [Please stop calling databases CP or AP](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html)] by Martin Kleppmann ). However, Marz was wrestlling with an essential question: given the reality of unreliable systems (and humans), how can we meet the potentially competing requirements of producing accurate data processing results in real-time. 


Query = Function(All Data)

Principle of immutability

The term _Lambda_ refer to

Marz described an architecture of 'layers', the 'speed layer' and the 'batch layer.' The 'speed layer' shares a lot in common with the conceptual design described in previous posts in this series. 


http://jameskinley.tumblr.com/post/37398560534/the-lambda-architecture-principles-for

https://jvns.ca/blog/2016/11/19/a-critique-of-the-cap-theorem/