---
layout: post
title: "Intro to Data Streaming: The Problem with Batch ETL, Part 1"
date: 2017-06-08 20:00:00 -0700
categories: "Architecture"
excerpt_separator: <!--more-->
---

Chances are, batch ETL is the majority, or perhaps the exclusive, solution for data engineering underlying Business Intelligence in your enterprise. There are good reasons for this. Batch ETL has a legion of engineers trained in its patterns. It is politically non-controversial. There are many established tools backed by major, executive and compliance approved, corporations. While far from simple, it does eliminate some complexities such as isolation from other processes and partially removing contention with other workloads. It is also the approach that most caters to the highest performance write operations of relational database management systems by loading large quantities of data at one time, rather than in separate transactions. Unfortunately, because it has been the default choice for so long, most enterprises have become complacent about its limitations. It is time to take a hard look at this venerable practice. <!--more-->Consider the following problems:

## Catastrophic Failure​

When batch ETL breaks bad it goes all the way bad. Typically, it's an all or nothing proposition. One of the strengths of batch ETL, executing bulk operations for speed, is also one of its major shortcomings. With bulk operations a problem with a single row of data (improper data type is a common issue) can cause long running processes to fail completely. If, in the process of loading 100MM rows of data, a single row creates a problem, the significant work done on the other 99,999,999 rows is simply thrown away as the transaction rolls back. With processing windows shrinking ever smaller, a failure like this in hour 3 of a 4 hour window can mean many difficult conversations come morning. 

## Complexity 

Batch ETL tools were designed in days before lean software development principles captured the hearts of developers everywhere. Test Driven Development, Continuous Integration, Continuous Deployment and other must-haves are largely missing. There have been efforts to implement unit testing, for instance, but in practice it's nowhere near as natural as using the now ubiquitous frameworks for application development. Given the vast productivity improvements seen in software development directly as a result of these practices, the limitations of ETL tools are glaring. 

Another problem is the management overhead posed by thousands of independent ETL packages. Some tools are better than others, but simple techniques such as code reuse, abstractions, and project deployments are often simply not available, or largely ineffectual. In many organizations it is a major undertaking to keep track of the inter-dependencies of thousands of individual packages. Often, due to the sheer complexity of the ETL, maintenance knowledge exists only in the minds of the original developers. After natural turnover, whatever original organization logic is lost. 

This effect is compounded by the often chaotic environment in which ETL is developed. Given the unique aspects of Business Intelligence projects (surely a subject for another post), organizations often plan based on experiences with more common IT projects, which tend to yield unrealistic expectations. As a result, ETL is often rushed as well as plagued by constantly shifting demands with which the tools and 'traditional' practices of ETL are simply unable to effectively cope. 

> **A note about scale**. As data volumes grow it's also worth noting that most ETL tools do not effectively scale in relation to expense. 'Scale up' remains the primary way of increasing ETL throughput, and often requires significant refactoring of code for parallelism to take full advantage. Combined with shrinking batch processing windows, this is a problem that will only get worse over time. Even those tools that are enabling 'scale out' capabilities still require significant design and implementation effort to function.

## Lossy​

Commonly, batch ETL processes run once per day. This means that analytics performed on the results of these processes only take into account a snapshot (often inconsistent snapshots, as we will see later) of a particular moment in time. However, in most organizations there is a significant amount of data that is simply lost to analytics. How often does the data in your organization change throughout the day? How many accounts, contacts, transactions, etc. are added, deleted, and updated during the time that your Business Intelligence simply isn't listening? 

The questions we ask of analytics grow ever more exacting as the competitive landscape demands more sophistication. What insights are you losing by not having access to these daily shifts that occur? How much more effective could your fraud detection algorithms, your marketing campaign tracking, or your customer support satisfaction be if you were able to capture data *as it changes* throughout the day? Consider this: how likely is it that you would own a credit card today that did not have real-time fraud detection?

## Temporal Inaccuracies​

Consider this scenario:  

> Your batch ETL process starts by capturing customer data updates from your CRM system. It then moves on to other data sources before finally collecting your financial transactions. Between the time the ETL process reads data from the CRM system and the time it collects your transactions, 'Acme' company is updated to 'Beta' company, a change of which your Business Intelligence will be unaware. Any transactions that occur could be assigned to 'Acme' company, rather than the correct 'Beta' company.

This example may or may not be a serious issue in your organization, but consider that this is only one example of a pattern endemic to batch ETL. The pattern is likely repeating throughout the data upon which your Business Intelligence is based in proportion to how often your data changes. If your organization is still able to maintain very consistent windows for batch processing during which very little changes, then you may be in the clear. But, as more and more businesses become 24/7 operations, this problem will only grow, threatening trust in your reporting and analytics accuracy. Combined with the 'Lossy' nature of batch ETL, data accuracy could be a non-trivial issue.  


Part 2 of this series will continue to look at the limitations of batch ETL in the context of modern application architectures. 

---
{% include TOC_intro_data_streaming.md %}