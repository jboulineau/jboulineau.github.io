---
layout: post
title: "Dealing With Bad Records in Kafka"
date: 2019-06-13
categories: kafka
excerpt: A single bad record (a.k.a poison pill) on a Kafka topic can ruin your day. KafkaConsumer does not deal with these records gracefully. Here I cover strategies on how to address this issue.
---

There is currently a rather serious flaw in the Java `KafkaConsumer` when combined with `KafkaAvroDeserializer`, which is [used to deserialize records](https://docs.confluent.io/current/schema-registry/schema_registry_tutorial.html) when their schemas are stored in Schema Registry. A [critical issue](https://issues.apache.org/jira/browse/KAFKA-4740) has been opened, but it hasn't been updated since December 2018.

In brief, the issue is that when a record is encountered that cannot be deserialized from Avro (a.k.a a poison pill) an exception will be thrown:

> org.apache.kafka.common.errors.SerializationException: Error deserializing key/value for partition topic-0 at offset 2
> If needed, please seek past the record to continue consumption.

This is, of course, how it should behave. However, it fails in an **unrecoverable** manner. Because it cannot be deserialized the record is not added to the collection returned by `poll()`, which means running a `commit()` variant won't do anything. And, despite the instructions in the exception meessage, you can't easily `seek()` past the record because the necessary partition and offset information would be in the `ConsumerRecord` instance, which is never instantiated. This results in a loop of fail, as `poll()` will continue returning the same record repeatedly with no provided way to get past the record.

Needless to say, this is disappointing for such a widely used and essential platform for just about anyone doing data engineering or event driven architecture of any sort these days. Hopefully the [issue](https://issues.apache.org/jira/browse/KAFKA-4740) will be addressed, although it's going to be tricky to do without introducing breaking semantic changes. The alternative presented in the issue of passing partition id and offset through the exception object isn't exactly clean, but may be the best option. Regardless, in the meantime there are options to explore. And here they are!

## 'Correct' the deserializer

Since the heart of the problem is that the `KafkaConsumer` doesn't handle `SerializationException` in a recoverable manner, one solution would be simply not to throw `SerializationException` anymore. By extending `io.confluent.kafka.serializers.AbstractKafkaAvroDeserializer` (a la `KafkaAvroDeserializer`) we can make this happen.

``` java
    @Override
    public Object deserialize(String s, byte[] bytes) {
        try {
            return deserialize(bytes);
        } catch (SerializationException e)
        {
            return null;
        }
    }

    public Object deserialize(String s, byte[] bytes, Schema readerSchema) {
        try {
            return deserialize(bytes, readerSchema);
        } catch (SerializationException e)
        {
            return null;
        }
```

Now, inject the class into `ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG` ...

``` java

props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, <my_deserializer_name>.class);

```

... and catch those null objects.

``` java
while (true) {
    final ConsumerRecords<String, Payment> records = consumer.poll(Duration.ofMillis(100));
    for (final ConsumerRecord<String, Payment> record : records) {
        final String key = record.key();

        if (record.value() == null) {
            System.out.println("A poison pill record was encountered.");
        } else {
            try {
                final Payment value = record.value();
                System.out.printf("key = %s, value = %s%n", key, value);
            } catch (ClassCastException e) {
                System.out.println("This record is not a `Payment` event.");
            }
        }
        consumer.commitSync();
    }

```

It's not great. Converting exceptions into null objects hurts, but, given the alternatives below, it may be the best option. It's the one I'm using.

## Delete the record from the topic

It is possible to delete the record from the topic manually by using the `kafka-delete-records.sh`. I haven't tried this (yet), so YMMV, use at your own use, etc. [Here](http://www.alternatestack.com/development/kafka-tools-kafka-delete-records/) is a blog post on the subject.

Of course, relying on this option means that anytime your consumer(s) encounter(s) a poison pill you're down until your Kafka administrators can fix the issue. And **Bad Things** can always happen when you're munging from the command line.

## Use the Streams API

(Not tested) Interestingly, the problem has been fixed in the Streams API, essentially by the same method above of implementing the 'catch exception and move on' method. There was an [improvement proposal](https://cwiki.apache.org/confluence/display/KAFKA/KIP-161%3A+streams+deserialization+exception+handlers) on the subject that became a [JIRA issue](https://issues.apache.org/jira/browse/KAFKA-5157) and the fix was [merged](https://github.com/apache/kafka/pull/3423).

The downside here is that the Streams API is meant for the very specific usage patterns of stream processing. It may or may not fit your use case.

## Use Spring

(Not tested) For those using the Spring framework, there is a [fix in version 2.2](https://docs.spring.io/spring-kafka/docs/2.2.0.RELEASE/reference/html/_reference.html#error-handling-deserializer). Here's a [Confluent post](https://www.confluent.io/blog/spring-for-apache-kafka-deep-dive-part-1-error-handling-message-conversion-transaction-support) to help.

The downside of this option is Spring.

## Use a libdrkafka client

(Not tested) Colleagues assure me that `libdrkafka` based clients, such as in [Python](https://docs.confluent.io/current/clients/confluent-kafka-python/index.html#consumer), don't have this issue. If I get the chance I'll test this one out and update this post.

## Parse the error text and seek past the bad record

We're well into hack territory here, but it might get you through in a pinch. Let's not dwell. There's some code below, if you must.

Just don't tell anyone I helped you do this.

``` java
import io.confluent.kafka.serializers.AbstractKafkaAvroSerDeConfig;
import io.confluent.kafka.serializers.subject.TopicRecordNameStrategy;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import io.confluent.kafka.serializers.KafkaAvroDeserializer;
import io.confluent.kafka.serializers.KafkaAvroDeserializerConfig;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.regex.Pattern;
import java.util.regex.Matcher;
import java.util.Collections;
import java.util.Properties;

public class SampleConsumer {

    private static final String TOPIC = "payment";

    @SuppressWarnings("InfiniteLoopStatement")
    public void Consume() {
        final Properties props = new Properties();
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-consumer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class);
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 100000);
        props.put(AbstractKafkaAvroSerDeConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");
        props.put(KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG, true);
        props.put(KafkaAvroDeserializerConfig.KEY_SUBJECT_NAME_STRATEGY, TopicRecordNameStrategy.class.getName());
        props.put(KafkaAvroDeserializerConfig.VALUE_SUBJECT_NAME_STRATEGY, TopicRecordNameStrategy.class.getName());

        /* Sadly, we need to do some string parsing to deal with 'poison pill' records (i.e. any message that cannot be
        de-serialized by KafkaAvroDeserializer, most likely because they weren't produced using Schema Registry) so we
        need to set up some regex things
         */
        final Pattern offsetPattern = Pattern.compile("\\w*offset*\\w[ ]\\d+");
        final Pattern partitionPattern = Pattern.compile("\\w*" + TOPIC + "*\\w[-]\\d+");

        KafkaConsumer<String, LoanCreated> consumer =  new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList(TOPIC));

        // Consume messages
        while (true) {
            try {
                final ConsumerRecords<String, LoanCreated> records = consumer.poll(Duration.ofMillis(1));
                for (final ConsumerRecord<String, LoanCreated> record : records) {
                    final String key = record.key();
                    try {
                        /* A record can be successfully de-serialized, but is not coercable into the type we need. In
                        the case of this example, we're looking for LoanCreated records, but we are also producing
                        Payment records. */
                        final LoanCreated value = record.value();
                        System.out.printf("key = %s, value = %s%n", key, value);
                        // do work here
                    } catch (ClassCastException e) {
                        System.out.println("Record is not the specified type ... skipping");
                    }
                }
                consumer.commitSync();
            } catch (SerializationException e) {
                String text = e.getMessage();
                // Parse the error message to get the partition number and offset, in order to `seek` past the poison pill.
                Matcher mPart = partitionPattern.matcher(text);
                Matcher mOff = offsetPattern.matcher(text);

                mPart.find();
                Integer partition = Integer.parseInt(mPart.group().replace(TOPIC + "-", ""));
                mOff.find();
                Long offset = Long.parseLong(mOff.group().replace("offset ", ""));
                System.out.println(String.format(
                        "'Poison pill' found at partition {0}, offset {1} .. skipping", partition, offset));
                consumer.seek(new TopicPartition(TOPIC, partition), offset + 1);
                // Continue on
            }
        }
    }
}
```
