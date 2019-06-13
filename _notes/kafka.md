# kafka

https://kafka.apache.org/20/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html

## Avro error
https://stackoverflow.com/questions/53715265/kafka-avro-serializer-and-deserializer-exception-avro-supported-types

Also, has to implement SingleRecord or GenericRecord.

## Dealing with bad records

https://issues.apache.org/jira/browse/KAFKA-4740

https://stackoverflow.com/questions/49297926/how-to-best-handle-serializationexception-from-kafkaconsumer-poll-method

https://msayag.github.io/Kafka/

If you use GenericRecord, you have to worry about handling deserialization yourself.

