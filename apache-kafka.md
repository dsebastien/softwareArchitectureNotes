# Apache Kafka

Using a system like Kafka and its transactions frees us from worries of failures and retries in the distributed world.

Kafka is built for big data, it has production workloads with over a hundred TB :\)

## Amazon AWS

Advice to host Kafka on AWS:

* [https://www.confluent.io/blog/design-and-deployment-considerations-for-deploying-apache-kafka-on-aws/](https://www.gitbook.com/book/dsebastien/software-architecture-notes/edit#)

### Replication

Most Kafka on AWS deployments use a replication factor of three.

### Instance

Recommended instance type

* d2.xlarge if you're using instance storage
* r3.xlarge or m4.2xlarge if you're using EBS

Isolate Kafka to dedicated disks

* Kafka brokers should be configured with dedicated disks, to limit disk trashing and increase trashing
* logs.dirs should only contain disks \(or EBS volumes\) that are dedicated to deploy Kafka

### logs.dirs

log.dirs configuration specifies where the broker stores its logs \(topic partition replicas\)

* both instance storage and EBS \(Elastic Block Store\) work well
* each has its tradeoffs
* using EBS
  * will decrease network traffic when a broker fails or is replaced
  * the replacement broker will join the cluster more quickly
  * BUT they add cost to your AWS deployment
  * when an EC2 instance running a Kafka broker fails or is terminated, the broker's on-disk partition replaicas remain intact and can be mounted by a new EC2 instance
  * by using EBS, most of the replica data for the replacement broker will already be in the EBS volume and won't need to be transferred over the network
  * only data produced since the original broker failed or was terminated will need to be fetched across the network
  * EBS does its own replication under the cover for fault tolerance
  * EBS can be a single point of failure within an availability zone; if the EBS service within the availability zone has an outage, all brokers in that availability zone will be affected
* using instance storage
  * cheaper
  * recovering from a failed or terminated broker will take longer and require more network traffic
  * instance storage is recommended for larger Kafka clusters

### Replacing a failed broker

The recommended practice is to use the broker ID from the failed broker in the replacement broker.

Doing so is the most operationally simple approach, because the replacement broker will resume the work that the failed broker was doing automatically.

## Links

* [https://kafka.apache.org/uses](https://kafka.apache.org/uses)
* [https://www.confluent.io](https://www.confluent.io)



