https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch02.html

This chapter describes how to get started with the Apache Kafka broker, including how to set up ***Apache Zookeeper, which is used by Kafka for storing metadata for the brokers.***
we cover how to install multiple Kafka brokers as part of a single cluster and some specific concerns when using Kafka in a production environment.

### First Things First
There are a few things that need to happen before using Apache Kafka. The following sections tell you what those things are.

Choosing an Operating System 

Installing Java 

------------------------------------------------------------------------------------------------------------------------

#### Installing Zookeeper 
Apache Kafka uses Zookeeper to store metadata about the Kafka cluster, as well as consumer client details.

![Kafka-and-Zookeeper.png](./img/Kafka-and-Zookeeper.png)


### ZOOKEEPER ENSEMBLE
A Zookeeper cluster is called an ensemble. ***Due to the algorithm used, it is recommended that ensembles contain an odd number of servers (e.g., 3, 5, etc.) as a majority of ensemble members (a quorum) must be working in order for Zookeeper to respond to requests***. This means that in a three-node ensemble, you can run with one node missing. With a five-node ensemble, you can run with two nodes missing.


### SIZING YOUR ZOOKEEPER ENSEMBLE
Consider running Zookeeper in a five-node ensemble. In order to make configuration changes to the ensemble, including swapping a node, you will need to reload nodes one at a time. If your ensemble cannot tolerate more than one node being down, doing maintenance work introduces additional risk. ***It is also not recommended to run more than seven nodes, as performance can start to degrade due to the nature of the consensus protocol.***

To configure Zookeeper servers in an ensemble, they must have a common configuration that lists all servers, and each server needs a myid file in the data directory that specifies the ID number of the server. If the hostnames of the servers in the ensemble are zoo1.example.com, zoo2.example.com, and zoo3.example.com, the configuration file might look like this:


`
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=20
syncLimit=5
server.1=zoo1.example.com:2888:3888
server.2=zoo2.example.com:2888:3888
server.3=zoo3.example.com:2888:3888
`

In this configuration, the initLimit is the amount of time to allow followers to connect with a leader. The syncLimit value limits how out-of-sync followers can be with the leader. Both values are a number of tickTime units, which makes the initLimit 20 * 2000 ms, or 40 seconds. The configuration also lists each server in the ensemble. 
The servers are specified in the format server.X=hostname:peerPort:leaderPort, with the following parameters:

X \
The ID number of the server. This must be an integer, but it does not need to be zero-based or sequential.

hostname \
The hostname or IP address of the server.

#### peerPort 
***The TCP port over which servers in the ensemble communicate with each other.***

#### leaderPort 
***The TCP port over which leader election is performed.***


***Clients only need to be able to connect to the ensemble over the clientPort, but the members of the ensemble must be able to communicate with each other over all three ports.***

In addition to the shared configuration file, each server must have a file in the dataDir directory with the name myid. This file must contain the ID number of the server, which must match the configuration file. Once these steps are complete, the servers will start up and communicate with each other in an ensemble.


-----------------------------------------------------------------------------------------------------------------------

### Installing a Kafka Broker

### Broker Configuration

There are numerous configuration options for Kafka that control all aspects of setup and tuning. Many options can be left to the default settings, as they deal with tuning aspects of the Kafka broker that will not be applicable until you have a specific use case to work with and a specific use case that requires adjusting these settings.


BROKER.ID \
PORT \
LOG.DIRS \
NUM.RECOVERY.THREADS.PER.DATA.DIR \
AUTO.CREATE.TOPICS.ENABLE \

#### Topic Defaults

***NUM.PARTITIONS *** 

***The num.partitions parameter determines how many partitions a new topic is created with, primarily when automatic topic creation is enabled (which is the default setting)***. This parameter defaults to one partition. ***Keep in mind that the number of partitions for a topic can only be increased, never decreased***. This means that if a topic needs to have fewer partitions than num.partitions, care will need to be taken to manually create the topic

partitions are the way a topic is scaled within a Kafka cluster, which makes it important to use partition counts that will balance the message load across the entire cluster as brokers are added. Many users will have the partition count for a topic be equal to, or a multiple of, the number of brokers in the cluster. This allows the partitions to be evenly distributed to the brokers, which will evenly distribute the message load. This is not a requirement, however, as you can also balance message load by having multiple topics.

--------------------------------------------------------------------------------------------------------------------------

### HOW TO CHOOSE THE NUMBER OF PARTITIONS
There are several factors to consider when choosing the number of partitions:


1) ***What is the throughput you expect to achieve for the topic?*** For example, do you expect to write 100 KB per second or 1 GB per second?

2) ***What is the maximum throughput you expect to achieve when consuming from a single partition? A partition will always be consumed completely by a single consumer*** (as even when not using consumer groups, the consumer must read all messages in the partition). ***If you know that your slower consumer writes the data to a database and this database never handles more than 50 MB per second from each thread writing to it, then you know you are limited to 50 MB/sec throughput when consuming from a partition.***

3) You can go through the same exercise to estimate the maximum throughput per producer for a single partition, ***but since producers are typically much faster than consumers, it is usually safe to skip this.***

4) If you are sending messages to partitions based on keys, adding partitions later can be very challenging, so calculate throughput based on your expected future usage, not the current usage.

5) Consider the number of partitions you will place on each broker and available diskspace and network bandwidth per broker.

6) ***Avoid overestimating, as each partition uses memory and other resources on the broker and will increase the time for leader elections.***

With all this in mind, it’s clear that you want many partitions but not too many. 

***If you have some estimate regarding the target throughput of the topic and the expected throughput of the consumers, you can divide the target throughput by the expected consumer throughput and derive the number of partitions this way***. 

So if I want to be able to write and read 1 GB/sec from a topic, and I know each consumer can only process 50 MB/s, then I know I need at least 20 partitions. This way, I can have 20 consumers reading from the topic and achieve 1 GB/sec.

If you don’t have this detailed information, our experience suggests that limiting the size of the partition on the disk to less than 6 GB per day of retention often gives satisfactory results.


-----------------------------------------------------------------------------------------------------------------------

### LOG.RETENTION.MS

***The most common configuration for how long Kafka will retain messages is by time***. 

The default is specified in the configuration file using the log.retention.hours parameter, and it is set to 168 hours, or one week. However, there are two other parameters allowed, log.retention.minutes and log.retention.ms. All three of these specify the same configuration—the amount of time after which messages may be deleted—but the recommended parameter to use is log.retention.ms, as the smaller unit size will take precedence if more than one is specified. This will make sure that the value set for log.retention.ms is always the one used. If more than one is specified, the smaller unit size will take precedence.


### RETENTION BY TIME AND LAST MODIFIED TIMES

***Retention by time is performed by examining the last modified time (mtime) on each log segment file on disk. Under normal cluster operations, this is the time that the log segment was closed, and represents the timestamp of the last message in the file***. 

However, when using administrative tools to move partitions between brokers, this time is not accurate and will result in excess retention for these partitions

### LOG.RETENTION.BYTES
***Another way to expire messages is based on the total number of bytes of messages retained. This value is set using the log.retention.bytes parameter, and it is applied per-partition***. This means that if you have a topic with 8 partitions, and log.retention.bytes is set to 1 GB, the amount of data retained for the topic will be 8 GB at most. Note that all retention is performed for individual partitions, not the topic. This means that should the number of partitions for a topic be expanded, the retention will also increase if log.retention.bytes is used.

### CONFIGURING RETENTION BY SIZE AND TIME
If you have specified a value for both log.retention.bytes and log.retention.ms (or another parameter for retention by time), messages may be removed when either criteria is met. For example, if log.retention.ms is set to 86400000 (1 day) and log.retention.bytes is set to 1000000000 (1 GB), it is possible for messages that are less than 1 day old to get deleted if the total volume of messages over the course of the day is greater than 1 GB. Conversely, if the volume is less than 1 GB, messages can be deleted after 1 day even if the total size of the partition is less than 1 GB.

--------------------------------------------------------------------------------------------------------------------------

### LOG.SEGMENT.BYTES
The log-retention settings previously mentioned operate on log segments, not individual messages. As messages are produced to the Kafka broker, they are appended to the current log segment for the partition. Once the log segment has reached the size specified by the log.segment.bytes parameter, which defaults to 1 GB, the log segment is closed and a new one is opened. Once a log segment has been closed, it can be considered for expiration. A smaller log-segment size means that files must be closed and allocated more often, which reduces the overall efficiency of disk writes.

Adjusting the size of the log segments can be important if topics have a low produce rate. For example, if a topic receives only 100 megabytes per day of messages, and log.segment.bytes is set to the default, it will take 10 days to fill one segment. As messages cannot be expired until the log segment is closed, if log.retention.ms is set to 604800000 (1 week), there will actually be up to 17 days of messages retained until the closed log segment expires. This is because once the log segment is closed with the current 10 days of messages, that log segment must be retained for 7 days before it expires based on the time policy (***as the segment cannot be removed until the last message in the segment can be expired***).

--------------------------------------------------------------------------------------------------------------------------


### RETRIEVING OFFSETS BY TIMESTAMP
The size of the log segment also affects the behavior of fetching offsets by timestamp. When requesting offsets for a partition at a specific timestamp, Kafka finds the log segment file that was being written at that time. It does this by using the creation and last modified time of the file, and looking for a file that was created before the timestamp specified and last modified after the timestamp. The offset at the beginning of that log segment (which is also the filename) is returned in the response.

### LOG.SEGMENT.MS
Another way to control when log segments are closed is by using the log.segment.ms parameter, which specifies the amount of time after which a log segment should be closed. As with the log.retention.bytes and log.retention.ms parameters, log.segment.bytes and log.segment.ms are not mutually exclusive properties. Kafka will close a log segment either when the size limit is reached or when the time limit is reached, whichever comes first. By default, there is no setting for log.segment.ms, which results in only closing log segments by size.

### DISK PERFORMANCE WHEN USING TIME-BASED SEGMENTS
When using a time-based log segment limit, it is important to consider the impact on disk performance when multiple log segments are closed simultaneously. This can happen when there are many partitions that never reach the size limit for log segments, as the clock for the time limit will start when the broker starts and will always execute at the same time for these low-volume partitions.

----------------------------------------------------------------------------------------------------------------------------

### MESSAGE.MAX.BYTES
The Kafka broker limits the maximum size of a message that can be produced, configured by the message.max.bytes parameter, which defaults to 1000000, or 1 MB. A producer that tries to send a message larger than this will receive an error back from the broker, and the message will not be accepted. As with all byte sizes specified on the broker, this configuration deals with compressed message size, which means that producers can send messages that are much larger than this value uncompressed, provided they compress to under the configured message.max.bytes size.

***There are noticeable performance impacts from increasing the allowable message size. Larger messages will mean that the broker threads that deal with processing network connections and requests will be working longer on each request. Larger messages also increase the size of disk writes, which will impact I/O throughput.***


### COORDINATING MESSAGE SIZE CONFIGURATIONS
The message size configured on the Kafka broker must be coordinated with the ***fetch.message.max.bytes configuration on consumer clients***. If this value is smaller than ***message.max.bytes***, then consumers that encounter larger messages will fail to fetch those messages, resulting in a situation where the consumer gets stuck and cannot proceed. The same rule applies to the ***replica.fetch.max.bytes*** configuration on the brokers when configured in a cluster.

----------------------------------------------------------------------------------------------------------------------------

### Hardware Selection

Selecting an appropriate hardware configuration for a Kafka broker can be more art than science. Kafka itself has no strict requirement on a specific hardware configuration, and will run without issue on any system. 

***Once performance becomes a concern, however, there are several factors that will contribute to the overall performance: disk throughput and capacity, memory, networking, and CPU***. Once you have determined which types of performance are the most critical for your environment, you will be able to select an optimized hardware configuration that fits within your budget.


1) Disk Throughput
2) Disk Capacity
3) Memory
4) Networking \
The available network throughput will specify the maximum amount of traffic that Kafka can handle. This is often the governing factor, combined with disk storage, for cluster sizing. This is complicated by the inherent imbalance between inbound and outbound network usage that is created by Kafka’s support for multiple consumers. A producer may write 1 MB per second for a given topic, but there could be any number of consumers that create a multiplier on the outbound network usage. Other operations such as cluster replication (covered in Chapter 6) and mirroring (discussed in Chapter 8) will also increase requirements. Should the network interface become saturated, it is not uncommon for cluster replication to fall behind, which can leave the cluster in a vulnerable state.


5) CPU
-----------------------------------------------------------------------------------------------------------------------


### Kafka Clusters

How Many Brokers?

### Broker Configuration
***There are only two requirements in the broker configuration to allow multiple Kafka brokers to join a single cluster. The first is that all brokers must have the same configuration for the zookeeper.connect parameter. This specifies the Zookeeper ensemble and path where the cluster stores metadata. The second requirement is that all brokers in the cluster must have a unique value for the broker.id parameter***. If two brokers attempt to join the same cluster with the same broker.id, the second broker will log an error and fail to start. There are other configuration parameters used when running a cluster—specifically, parameters that control replication, which are covered in later chapters.


### Colocating Applications on Zookeeper
Kafka utilizes Zookeeper for storing metadata information about the brokers, topics, and partitions. ***Writes to Zookeeper are only performed on changes to the membership of consumer groups or on changes to the Kafka cluster itself***. This amount of traffic is minimal, and it does not justify the use of a dedicated Zookeeper ensemble for a single Kafka cluster. In fact, many deployments will use a single Zookeeper ensemble for multiple Kafka clusters (using a chroot Zookeeper path for each cluster, as described earlier in this chapter).


### KAFKA CONSUMERS AND ZOOKEEPER
***Prior to Apache Kafka 0.9.0.0, consumers, in addition to the brokers, utilized Zookeeper to directly store information about the composition of the consumer group, what topics it was consuming, and to periodically commit offsets for each partition being consumed (to enable failover between consumers in the group). With version 0.9.0.0, a new consumer interface was introduced which allows this to be managed directly with the Kafka brokers.***

However, there is a concern with consumers and Zookeeper under certain configurations. Consumers have a configurable choice to use either Zookeeper or Kafka for committing offsets, and they can also configure the interval between commits. If the consumer uses Zookeeper for offsets, each consumer will perform a Zookeeper write at every interval for every partition it consumes. A reasonable interval for offset commits is 1 minute, as this is the period of time over which a consumer group will read duplicate messages in the case of a consumer failure. These commits can be a significant amount of Zookeeper traffic, especially in a cluster with many consumers, and will need to be taken into account. It may be neccessary to use a longer commit interval if the Zookeeper ensemble is not able to handle the traffic. However, it is recommended that consumers using the latest Kafka libraries use Kafka for committing offsets, removing the dependency on Zookeeper.

***Outside of using a single ensemble for multiple Kafka clusters, it is not recommended to share the ensemble with other applications, if it can be avoided. Kafka is sensitive to Zookeeper latency and timeouts, and an interruption in communications with the ensemble will cause the brokers to behave unpredictably. This can easily cause multiple brokers to go offline at the same time, should they lose Zookeeper connections, which will result in offline partitions***. It also puts stress on the cluster controller, which can show up as subtle errors long after the interruption has passed, such as when trying to perform a controlled shutdown of a broker. Other applications that can put stress on the Zookeeper ensemble, either through heavy usage or improper operations, should be segregated to their own ensemble.







 


