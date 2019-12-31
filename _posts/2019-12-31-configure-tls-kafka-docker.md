---
layout: post
title: "TLS/SSL for Kafka in Docker Containers"
date: 2019-12-31
categories: Kafka
excerpt: While configuring TLS/SSL for Confluent Kafka is straightforward, there are twists when running in Docker containers. This posts covers what I discovered that isn't (as of time of writing) covered in the official documentation.
---

Confluent provides [generally strong documentation](https://docs.confluent.io/current/kafka/authentication_ssl.html) for configuring TLS/SSL. This post assumes you are familiar with this documentation, especially around key/certificate management. The documentation is for using the standard configuration files. For running in containers, the same settings are configured in environment variables, which are generally translatable nearly 1-1 with the keys in the files. More on that later. Confluent provides various `docker-compose.yml` files to demonstrate how to translate the configuration values. [This](https://github.com/confluentinc/cp-demo/blob/5.3.2-post/docker-compose.yml) is a good, fairly comprehensive, example from a platform demo. Additionally, [a series of examples](https://github.com/confluentinc/cp-docker-images/tree/5.3.1-post/examples) are provided that cover a large number of scenarios.

However, after configuring TLS/SSL for Kafka using the Confluent Docker images, you may run into an error like this one when attempting to connect to the broker.

``` bash
Failed authentication with /172.31.0.1(SSL handshake failed)
(org.apache.kafka.common.network.Selector)
```

After fiddling with `log4j` settings, you may uncover an error message like this one:

``` bash
Inbound closed before receiving peer's close_notify: possible truncation attack?
```

Testing with `openssl s_client` may reveal further information:

``` bash
CONNECTED(00000003)
140218707678080:error:1408F10B:SSL routines:ssl3_get_record:wrong version number:ssl/record/ssl3_record.c:332:
---
no peer certificate available
---
No client certificate CA names sent
---
SSL handshake has read 5 bytes and written 407 bytes
Verification: OK
---
New, (NONE), Cipher is (NONE)
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
```

If you're here, you are no doubt as frustrated as I was. I was _certain_ that I had the correct certificates and configurations in place. It turns out, there is a hidden twist to the configuration. To understand what was going on, a deeper at the container configurations is required.

For configuring TLS/SSL in Confluent containers, there are two methods that may be used.

### Example 1

The first is to set the path to the keystore files and include the passwords in plain text.

``` bash
KAFKA_SSL_KEYSTORE_LOCATION=<path>
KAFKA_SSL_KEYSTORE_PASSWORD=<plain-text-password>
KAFKA_SSL_TRUSTSTORE_LOCATION=<path>
KAFKA_SSL_TRUSTSTORE_PASSWORD=<plain-text-password>
```

Each `LOCATION` variable is the full path to the keystore file wherever you decide to mount them.

### Example 2

The example `docker-compose.yml` files prefer the method of setting keystore filenames and using credential files to store the passwords for the keystores. This is clearly preferable for production as secrets files can be injected at runtime as part of your CI/CD pipeline and you can keep sensitive values out of source control. This method assumes you have mounted the necessary files into the `/etc/kafka/secrets` path of the container.

``` bash
KAFKA_SSL_KEYSTORE_FILENAME: <filename>
KAFKA_SSL_KEYSTORE_CREDENTIALS: <filename>
KAFKA_SSL_KEY_CREDENTIALS: <filename> 
KAFKA_SSL_TRUSTSTORE_FILENAME: <filename>
KAFKA_SSL_TRUSTSTORE_CREDENTIALS: <filename>
```

What I found is that when using the first option the TLS endpoint worked as expected, but the second method resulted in the errors above. This demonstrated, at least, that the keystores and certificates were created properly. But, why wouldn't the second (much preferred) configuration option work?

The answer is to be found in the [configure script](https://github.com/confluentinc/cp-docker-images/blob/5.3.1-post/debian/kafka/include/etc/confluent/docker/configure#L65) for the Confluent Kafka Docker image, which is executed by the [entry point script](https://github.com/confluentinc/cp-docker-images/blob/5.3.1-post/debian/kafka/include/etc/confluent/docker/run). Line 65 of the script looks at the `KAFKA_ADVERTIZED_LISTENERS` environment variable to determine whether or not SSL is configured.

``` bash
if [[ $KAFKA_ADVERTISED_LISTENERS == *"SSL://"* ]]
```

The script requires that the name of the TLS listener **must** have `SSL` as the final three characters. In my case, I was using `SSL_INTERNAL` as the name of my listener, which did not match the pattern. Changing the name to `INTERNAL_SSL` resolved the problem. 

The code section that runs in the conditional translates the environment variables set in **example 2** into the environment variables of **example 1**, which are the ones actually read by Kafka during startup. This means that if you use the method of **example 1**, you can use whatever listener name you want.

There is now an open (internal) ticket with Confluent to update the documentation to make this behavior clear. Perhaps they will also reconsider the implementation of the image so that it is less fragile.

_Thanks to Ryan Alexander of Confluent for his help._
