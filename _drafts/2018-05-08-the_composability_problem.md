---
title: "The Composability Problem"
---

https://www.confluent.io/blog/handling-gdpr-log-forget/

If the key of the topic is something other than the CustomerId, then you need some process to map the two. For example, if you have a topic of Orders, then you need a mapping of Customer to OrderId held somewhere. Then, to ‘forget’ a customer, simply lookup their Orders and either explicitly delete them from Kafka, or alternatively redact any customer information they contain. You might roll this into a process of your own, or you might do it using Kafka Streams if you are so inclined.

Data modeling material from MongoDB