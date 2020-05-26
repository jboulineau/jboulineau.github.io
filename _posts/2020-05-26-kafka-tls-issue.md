---
layout: post
title: "Unable to connect to Kafka with kafkacat over SSL/TLS"
date: 2020-05-26
categories: Kafka
excerpt: When I was suddenly unable to connect to the TLS listener with kafkacat, after a long period of troubleshooting I was able to track down the issue to changes in the default openssl configuration in Debian docker base images.
---

Occasionally it is helpful to use [kafkacat](https://github.com/edenhill/kafkacat), the [librdkafka](https://github.com/edenhill/librdkafka) based CLI tool, to inspect our Kafka brokers. As our Kafka cluster runs in Docker Swarm, it is isolated from the rest of our network. Only containers that are in the the same Swarm virtual network can connect. As a result, I have an administrative container operating as a jump server to which we can connect to administer the cluster as needed. When kafkacat was suddenly unable to connect to the brokers after working perfectly fine for months, it presented quite a mystery.

After checking the basics such as network connectivity to the brokers, I was able to track down the issue to the TLS listeners, as kafkacat was able to connect via the plaintext listeners which are temporarily running for backwards compatibility. The brokers were also able to communicate over the TLS listeners, which deepened the mystery. I used our docker-compose files to locally create a replica of our cluster and was able to connect to it via kafkacat and the Python client, which also uses librdkafaka. Why, then, was this problem occurring from the administrative container?

Comparing to my MacBook Pro configuration I discovered a difference in librdkafka and kafkacat version that limited the TLS configuration options from the container. Kafkacat was also compiled with an old version of librdkafka while my laptop was on version 1.3.0. Reconciling these was harder than anticipated, as even the official repo for kafkacat had version 1.5.0 built with the older version of librdkafka. Finally, I ended up building both from source as can be seen in this Dockerfile excerpt:

``` Dockerfile
RUN git clone --depth 1 --single-branch --branch v1.3.0 https://github.com/edenhill/librdkafka.git
WORKDIR /librdkafka
RUN ./configure --install-deps && make && make install
WORKDIR /
RUN git clone --depth 1 --single-branch --branch 1.5.0 https://github.com/edenhill/kafkacat.git
WORKDIR /kafkacat
RUN ./configure && make
```

Using the option to disable hostname validation available in the updated version of librdkafka I was able to connect from the container. Strangely, even with identical versions of openssl, librdkafka, and kafkacat I did not have any hostname validation issues running locally.

The openssl CLI has proven to be a very useful tool to troubleshoot TLS issues, so I inspected the endpoint as so:

```bash
openssl s_client -connect <brokername>
```

Finally, this revealed the issue. The output from the container showed an error that did not appear locally or from the brokers:

`Verify return code: 68 (CA signature digest algorithm too weak)`

Connections from the administrative container were finding that the SHA256 signed certificates were not sufficiently secure and were failing validation on connection. The certificates were created strictly following [Confluent's instructions](https://docs.confluent.io/current/security/security_tutorial.html#generating-keys-certs), which uses the default signature strength from the keygen tool. And, of course, this worked perfectly fine for everything else. Luckily, the error message revealed by the openssl CLI lead me to a [Debian mailing list archive](https://www.mail-archive.com/debian-bugs-dist@lists.debian.org/msg1638747.html) that finally solved the mystery. It turns out that in the newer versions of the Debian base image to which we had recently upgraded, the CipherString security level configured in `/etc/ssl/openssl.cnf` had been tightened. The solution presented was not feasible, as the configuration change would have had to be made by all clients throughout the company that used Kafka from Debian images. And we eventually needed to upgrade to SHA512, anyway. The only real option was to generate new CA certificates with the `-sha512` argument, as so:

``` bash
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365 -sha512
```

For good measure, I also regenerated the broker certificates. Doing so required adding the `-sigalg SHA512WithRSA` option argument when running `keytool`, which looks something like this depending on your configuration:

```bash
keytool -genkey -keystore ${NAME} -validity 365 -storepass ${PASS} -keypass ${PASS} -dname "CN=${BROKER_NAME}" -keysize 4096 -keyalg RSA -deststoretype pkcs12 --ext $SAN -sigalg SHA512WithRSA
```

After deploying the new certificates, all was corrected.

Although this was a major configuration change that must have created a lot of problems across the board, not just for Kafka clients, it's hard to fault Debian for doing so given the focus on cryptographic strength at many organizations. I would, however, have thought to find a more official source of documentation on the issue than a mailing list archive. Hopefully this post will help deepen the options available to those googling the error message with hopes and prayers, as I had to do.
