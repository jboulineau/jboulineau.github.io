# kafka

format filesystem with XFS
use /kafka path for zookeeper
modify file handle limits

yahoo kafka manager

https://github.com/confluentinc/schema-registry/issues/868

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

offsets.retention.minutes
unclean.leader.elections.enable
