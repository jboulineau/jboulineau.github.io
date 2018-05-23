---
layout: post
title: "Kappa Architecture"
categories: "Architecture"
series: "Intro to Data Streaming"
excerpt: ""
---

The previous post in this series covered the Lambda Architecture, which conceives of a solution with three layers: the batch layer, the speed layer, and the serving layer. Implemented successfully many times, the Lambda Architecture provides a complete solution that ensures (eventual) accuracy of processing as well as real-time processing results. However, this achievement does not come without costs. 

Jay Kreps outlined these costs in an influential blog post. In [Questioning the Lambda Architecture](http://radar.oreilly.com/2014/07/questioning-the-lambda-architecture.html), Kreps focused on the Lambda Architecture's biggest weakness: complexity. While it is great, Kreps argues, that the Lambda Architecture can maintain conceptual consistency across it's layers, the fact is that they are incongruent implementations. Each layer it's own set of technologies that must be supported. Three codebases must be implemented and maintained. And the serving layer is tasked with the extremely difficult challenge of reconciling results that may be overlapping or otherwise incongruent. While clearly achievable, the Lambda Architecture is far from simple.

The problem, Kreps proposed, is not so much conceptual as technical. Marz's basic observation batch processing is the simplest way to overcome issues related to the nature of distributed systems, human error, and inevitable change is sound. 