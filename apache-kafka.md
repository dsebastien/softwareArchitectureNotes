# Apache Kafka

Using a system like Kafka and its transactions frees us from worries of failures and retries in the distributed world.

Kafka is built for big data, it has production workloads with over a hundred TB :\)

## Recommendations

### Ordering guarantees

Strong ordering guarantees are of critical importance.

Couple of things to consider to ensure strong ordering guarantees:

* messages that require relative ordering **need to be sent on the same partition**
  * supply the same key for all messages that require a relative order
* sometimes key-based ordering is not enough and global ordering is required
  * use a single partition topic: throughput will be limited to a single machine but it's often enough for those use cases
* retries should in almost all cases be enabled in the producer
  * messages are sent in batches so we should be careful to send these batches one at a time, per destination broker, so there is no potential for a reordering of events when failures happen and whole batches are retried
    * configured via max.inflight.requests.per.connection

### Durability guarantees

To ensure durability, you can

* set acknowledgements to "all"
* min.insync.replicas to 2
  * ensures that data is always written to at least two replicas, but Kafka will continue to accept messages even if a machine is lost
* configure retries in the producer

Sensitive use cases require data to be flushed to disk synchronously

* set log.flush.interval.messages = 1
  * should be used sparingly: significant impact on throughput, particularly in highly concurrent environments
  * if you do this, increase the producer batch size, to increase the effectiveness of each disk flush on the broker \(batches of messages are flushed together\)

### Throughput control

Kafka has a thoughput control feature which allows a defined amount of bandwidth to be allocated to specific services. Greedy services are aggressively throttled so that a single cluster can be shared by any number of services without the fear of unexpected network contention. This feature can be applied to either individual service instances or load balanced groups.

### Hold key-based datasets in a compacted form

Topics in Kafka are retention-based by default: messages are retained for some configurable amount of time. Kafka also has a specific topic type called "compacted topics", that retain only the most recent events, with any old events, for a certain key, being removed. They also support deletes.

Compacted topics are scanned periodically and old messages are removed if they have been superseded \(based on their key\). This process is asynchronous, so compacted topics may contain some superseded messages waiting to be compacted away.

Compacted topics allow for optimizations:

* reduce how quickly a dataset grows in a data-specific way
* makes datasets smaller, making it easy to move them from machine to machine

Compacted topics are useful for snapshots for example.

### Long-term storage in Kafka

Kafka can be used as a storage layer without any issue.

### Segregate public and private topics

When using Kafka for Event Sourcing or Stream Processing, in the same cluster through which different services communicate, we usually want to segregate private, internal topics from shared, business topics.

A strict segregation can be applied using the authorization interface: you essentially assign read/write permissions.

Can be implemented through simple runtime validation or fully secured via TLS or SASL.

### Data evolution over time

Data types evolve over time. During the lifecycle of a system, the evolution of data types \(e.g., PersonCreated event design evolves\) will have an impact on the system.

Schemas can be defined and used to have a contract stating what messages should look like. Those schemas can help ensuring backwards compatibility and interoperability. Examples: Apache Avro, Google Protocol Buffers.

If there are breaking changes to some data types, a solution may be to use different topics for different versions \(e.g., orders-v1, orders-v2, ...\). That gives options for processing:

* publishers may dual-publish in both schemas at the same time
* we can add a process that down-converts from v2 to v1 \(e.g., KStreams job\)
  * this can also be done in the other direction \(v1 to v2\)

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

## Network

Choose an option that keeps inter-broker network traffic on the private subnet and allows clients to connect to the brokers. Inter-broker and client communication use the same network interface and port.

When a broker is started, it registers its hostname with ZooKeeper

* depending on how the OS's hostname and network are configured, brokers on EC2 instances may register hostnames with ZooKeeper that aren't reachable by clients
  * the purpose of advertised listeners is to address exactly this problem: the configured protocol, hostname and port in advertised.listeners is registered with ZooKeeper instead of the operating system's hostname

MirrorMaker

* consumer and producer joined together
* if it is configured to consume from an Elastic IP \(EIP\), the single broker tied to the EIP will be reachable, but the other brokers won't be
* MirrorMaker needs access to all brokers in the source and destination region, which in most cases is best implemented with a VPN between regions
* check out "Connecting Multiple VPCs with EC2 instances in the AWS documentation

Client service discovery can be implemented in different ways

* HAProxy on each client machine, proxying localhost requests to an available broker; Synapse works well for this
* use Route 53 for DNS; in this setup the TTL must be set in the client JVM to get around DNS caching
  * see setting the JVM TTL in the AWS documentation
* use an Elastic Load Balancer \(ELB\): ensure that the ELB is not public to the internet
  * sessions and stickiness do not need to be configured because Kafka clients only make a request to the load balancer at startup
  * a health check can be a ping or a telnet

Distribution over multiple availability zones

* Kafka was designed to run in a single data center
  * distributing brokers in a single cluster across different availability zones is NOT recommended
  * stretching brokers in a single cluster across availability zones within the same region is OK
* a multi-availability zone cluster offers stronger fault tolerance because a failed availability zone won't cause Kafka downtime
  * ! cross-availability zones data transfer fees will apply though

Distribution of ZooKeeper nodes across multiple availability zones

* ZooKeeper should be distributed across multiple availability zones to increase fault tolerance
* to tolerate an availability zone failure, ZooKeeper must be running in at least three different availability zones \(to still have quorum\)

Monitor broker performance and terminate poorly performing brokers.

Reference video: [https://www.youtube.com/watch?v=e1VEEtAvQ9E](https://www.youtube.com/watch?v=e1VEEtAvQ9E)

## Security

* Require TLS authentication for client/broker and inter-broker communication
* Encrypt network traffic via TLS
* Perform authorization via access control lists \(ACLs\)
* Network segmentation should be used to restrict access to ZooKeeper

## Links

* [https://kafka.apache.org/uses](https://kafka.apache.org/uses)
* [https://www.confluent.io](https://www.confluent.io)



