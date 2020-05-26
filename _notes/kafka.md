# kafka

format filesystem with XFS
use /kafka path for zookeeper
modify file handle limits

yahoo kafka manager

KafkaProducer<>.send() is async and returns a Future<RecordMetadata>
`configure` file in /etc/confluent/docker has a lot of info on how to docker configurations

https://jack-vanlightly.com/blog/2018/9/14/how-to-lose-messages-on-a-kafka-cluster-part1

[Maven plugin oddness](https://github.com/confluentinc/schema-registry/issues/868)

[TLS 1.3 support](https://issues.apache.org/jira/browse/KAFKA-7251)

Consumer group offset expiration
https://cwiki.apache.org/confluence/display/KAFKA/KIP-211%3A+Revise+Expiration+Semantics+of+Consumer+Group+Offsets
https://issues.apache.org/jira/browse/KAFKA-4682

https://cwiki.apache.org/confluence/display/KAFKA/KIP-320%3A+Allow+fetchers+to+detect+and+handle+log+truncation

## event sourcing

- partition modification throws off key-based allocation
- multiple message types per topic isn't widely supported
  - https://github.com/confluentinc/ksql/issues/1267
- topic alignment hotspots

## Optimizations

- export KAFKA_HEAP_OPTS="-Xmx4g"
- disable RAM swap
  - sudo sysctl vm.swappiness=1
  - echo 'vm.swappiness=1' | sudo tee --append /etc/sysctl.conf

- monitor GC
- increase file descriptor limits to at least 100k

- set Kafka quotas?
- st1 EBS volumes

https://blog.newrelic.com/engineering/kafka-best-practices/

offsets.retention.minutes
unclean.leader.elections.enable

## message size

https://www.cloudera.com/documentation/kafka/latest/topics/kafka_performance.html#concept_gqw_rcz_yq
https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines
https://www.quora.com/How-do-I-send-Large-messages-80-MB-in-Kafka

### compression
https://blog.cloudflare.com/squeezing-the-firehose/

## ssl broker config

Create the CA --
openssl req -new -newkey rsa:4096 -days 3650 -x509 -subj "/CN=Kafka-CA" -keyout ca-key -out ca-cert -nodes

export PASS=secret

Create broker cert --
keytool -genkey -keystore kafka.server.keystore.jks -validity 3650 -storepass $PASS -keypass $PASS -dname "CN=localhost" -storetype pkcs12

keytool -list -v -keystore kafka.server.keystore.jks

Create the CSR --
keytool -keystore kafka.server.keystore.jks -certreq -file cert-file -storepass $PASS -keypass $PASS

Sign the cert --
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 3650 -CAcreateserial -passin pass:$PASS

keytool -printcert -v -file cert-signed

Create the broker truststore; trust the CA --
keytool -keystore kafka.server.truststore.jks -alias CARoot -import -file ca-cert -storepass $PASS -keypass $PASS -noprompt

Add the CA public key into the broker keystore --
keytool -keystore kafka.server.keystore.jks -alias CARoot -import -file ca-cert -storepass $PASS -keypass $PASS -noprompt

Add the signed cert into the broker keystore --
keytool -keystore kafka.server.keystore.jks -import -file cert-signed -storepass $PASS -keypass $PASS -noprompt

Test connection --
openssl s_client -connect localhost:9093 -cipher ALL

NOTE: https://github.com/openssl/openssl/issues/6289

## ssl client config

Add CA public key to client keystore --
keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file ca-cert -storepass $PASS -keypass $PASS -noprompt

## Scaling

https://engineeringblog.yelp.com/2020/01/streams-and-monk-how-yelp-approaches-kafka-in-2020.html

https://www.waitingforcode.com/apache-kafka/apache-kafka-idempotent-producer/read
http://cloudurable.com/blog/kafka-tutorial-kafka-producer-advanced-java-examples/index.html

## Endpoint identification validation

https://issues.apache.org/jira/browse/KAFKA-7376
