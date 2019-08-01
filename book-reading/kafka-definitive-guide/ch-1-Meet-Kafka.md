https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch01.html

### Publish/Subscribe Messaging
Publish/subscribe messaging is a pattern that is characterized by the sender (publisher) of a piece of data (message) not specifically directing it to a receiver. Instead, the publisher classifies the message somehow, and that receiver (subscriber) subscribes to receive certain classes of messages. Pub/sub systems often have a broker, a central point where messages are published, to facilitate this.



 decouple the publishers of the information from the subscribers to that information 
 
 ---------------------------------------------------------------------------------------------------------------------
 
 ### Enter Kafka
***Apache Kafka is a publish/subscribe messaging system*** . \
It is often described as a “distributed commit log” or more recently as a “distributing streaming platform.” 

***A filesystem or database commit log is designed to provide a durable record of all transactions so that they can be replayed to consistently build the state of a system***. 

Similarly, data within Kafka is stored durably, in order, and can be read deterministically. In addition, the data can be distributed within the system to provide additional protections against failures, as well as significant opportunities for scaling performance.

----------------------------------------------------------------------------------------------------------------------

### Messages and Batches
For efficiency, messages are written into Kafka in batches. A batch is just a collection of messages, all of which are being produced to the same topic and partition. An individual roundtrip across the network for each message would result in excessive overhead, and collecting messages together into a batch reduces this. Of course, this is a tradeoff between latency and throughput: the larger the batches, the more messages that can be handled per unit of time, but the longer it takes an individual message to propagate. Batches are also typically compressed, providing more efficient data transfer and storage at the cost of some processing power.

----------------------------------------------------------------------------------------------------------------------

### Schemas

### Topics and Partitions
Messages in Kafka are categorized into topics. The closest analogies for a topic are a database table or a folder in a filesystem. Topics are additionally broken down into a number of partitions. Going back to the “commit log” description, a partition is a single log. Messages are written to it in an append-only fashion, and are read in order from beginning to end. 

***Partitions are also the way that Kafka provides redundancy and scalability. Each partition can be hosted on a different server, which means that a single topic can be scaled horizontally across multiple servers to provide performance far beyond the ability of a single server.***

----------------------------------------------------------------------------------------------------------------------

### Producers and Consumers
Kafka clients are users of the system, and there are two basic types: producers and consumers. There are also advanced client APIs—Kafka Connect API for data integration and Kafka Streams for stream processing. The advanced clients use producers and consumers as building blocks and provide higher-level functionality on top.

Producers create new messages. In other publish/subscribe systems, these may be called publishers or writers. In general, a message will be produced to a specific topic. By default, the producer does not care what partition a specific message is written to and will balance messages over all partitions of a topic evenly. In some cases, the producer will direct messages to specific partitions. This is typically done using the message key and a partitioner that will generate a hash of the key and map it to a specific partition. This assures that all messages produced with a given key will get written to the same partition. The producer could also use a custom partitioner that follows other business rules for mapping messages to partitions.


Consumers read messages. In other publish/subscribe systems, these clients may be called subscribers or readers. The consumer subscribes to one or more topics and reads the messages in the order in which they were produced.

***The consumer keeps track of which messages it has already consumed by keeping track of the offset of messages. The offset is another bit of metadata—an integer value that continually increases—that Kafka adds to each message as it is produced. Each message in a given partition has a unique offset.*** 

***By storing the offset of the last consumed message for each partition, either in Zookeeper or in Kafka itself, a consumer can stop and restart without losing its place.***


***Consumers work as part of a consumer group, which is one or more consumers that work together to consume a topic. The group assures that each partition is only consumed by one member.***
The mapping of a consumer to a partition is often called ownership of the partition by the consumer.
***In this way, consumers can horizontally scale to consume topics with a large number of messages. Additionally, if a single consumer fails, the remaining members of the group will rebalance the partitions being consumed to take over for the missing member.***

----------------------------------------------------------------------------------------------------------------------

### Brokers and Clusters

A single Kafka server is called a broker. The broker receives messages from producers, assigns offsets to them, and commits the messages to storage on disk. It also services consumers, responding to fetch requests for partitions and responding with the messages that have been committed to disk. Depending on the specific hardware and its performance characteristics, a single broker can easily handle thousands of partitions and millions of messages per second.

***Kafka brokers are designed to operate as part of a cluster.*** 

***Within a cluster of brokers, one broker will also function as the cluster controller (elected automatically from the live members of the cluster).*** 

***The controller is responsible for administrative operations, including assigning partitions to brokers and monitoring for broker failures.*** 

***A partition is owned by a single broker in the cluster, and that broker is called the leader of the partition.*** 

***A partition may be assigned to multiple brokers, which will result in the partition being replicated
This provides redundancy of messages in the partition, such that another broker can take over leadership if there is a broker failure. However, all consumers and producers operating on that partition must connect to the leader.***


***A key feature of Apache Kafka is that of retention, which is the durable storage of messages for some period of time. Kafka brokers are configured with a default retention setting for topics, either retaining messages for some period of time (e.g., 7 days) or until the topic reaches a certain size in bytes (e.g., 1 GB). ***

Once these limits are reached, messages are expired and deleted so that the retention configuration is a minimum amount of data available at any time. 

***Individual topics can also be configured with their own retention settings so that messages are stored for only as long as they are useful.***

For example, a tracking topic might be retained for several days, whereas application metrics might be retained for only a few hours. 

***Topics can also be configured as log compacted, which means that Kafka will retain only the last message produced with a specific key. This can be useful for changelog-type data, where only the last update is interesting.***


------------------------------------------------------------------------------------------------------------------------

### Multiple Clusters
As Kafka deployments grow, it is often advantageous to have multiple clusters. There are several reasons why this can be useful:

1) Segregation of types of data

2) Isolation for security requirements

3) Multiple datacenters (disaster recovery)


















 

