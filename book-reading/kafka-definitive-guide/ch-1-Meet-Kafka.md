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



When working with multiple datacenters in particular, it is often required that messages be copied between them. In this way, online applications can have access to user activity at both sites. For example, if a user changes public information in their profile, that change will need to be visible regardless of the datacenter in which search results are displayed. Or, monitoring data can be collected from many sites into a single central location where the analysis and alerting systems are hosted. ***The replication mechanisms within the Kafka clusters are designed only to work within a single cluster, not between multiple clusters.***

***The Kafka project includes a tool called MirrorMaker, used for this purpose. At its core, MirrorMaker is simply a Kafka consumer and producer, linked together with a queue. Messages are consumed from one Kafka cluster and produced for another.***

-------------------------------------------------------------------------------------------------------------------

### Why Kafka?
There are many choices for publish/subscribe messaging systems, so what makes Apache Kafka a good choice?

### Multiple Producers
Kafka is able to seamlessly handle multiple producers, whether those clients are using many topics or the same topic. This makes the system ideal for aggregating data from many frontend systems and making it consistent. 

For example, a site that serves content to users via a number of microservices can have a single topic for page views that all services can write to using a common format. Consumer applications can then receive a single stream of page views for all applications on the site without having to coordinate consuming from multiple topics, one for each application.


### Multiple Consumers
In addition to multiple producers, Kafka is designed for multiple consumers to read any single stream of messages without interfering with each other. This is in contrast to many queuing systems where once a message is consumed by one client, it is not available to any other. Multiple Kafka consumers can choose to operate as part of a group and share a stream, assuring that the entire group processes a given message only once.


### Disk-Based Retention
Not only can Kafka handle multiple consumers, but durable message retention means that consumers do not always need to work in real time. 

Messages are committed to disk, and will be stored with configurable retention rules. These options can be selected on a per-topic basis, allowing for different streams of messages to have different amounts of retention depending on the consumer needs. 

Durable retention means that if a consumer falls behind, either due to slow processing or a burst in traffic, there is no danger of losing data. It also means that maintenance can be performed on consumers, taking applications offline for a short period of time, with no concern about messages backing up on the producer or getting lost. Consumers can be stopped, and the messages will be retained in Kafka. This allows them to restart and pick up processing messages where they left off with no data loss.



### Scalable
Kafka’s flexible scalability makes it easy to handle any amount of data. Users can start with a single broker as a proof of concept, expand to a small development cluster of three brokers, and move into production with a larger cluster of tens or even hundreds of brokers that grows over time as the data scales up. 

Expansions can be performed while the cluster is online, with no impact on the availability of the system as a whole. This also means that a cluster of multiple brokers can handle the failure of an individual broker, and continue servicing clients. Clusters that need to tolerate more simultaneous failures can be configured with higher replication factors.


### High Performance
***All of these features come together to make Apache Kafka a publish/subscribe messaging system with excellent performance under high load. Producers, consumers, and brokers can all be scaled out to handle very large message streams with ease. This can be done while still providing subsecond message latency from producing a message to availability to consumers.***

----------------------------------------------------------------------------------------------------------------------


producers do not need to be concerned about who is using the data or the number of consuming applications.

### Use Cases

### ACTIVITY TRACKING
***The original use case for Kafka, as it was designed at LinkedIn, is that of user activity tracking. A website’s users interact with frontend applications, which generate messages regarding actions the user is taking. This can be passive information, such as page views and click tracking, or it can be more complex actions, such as information that a user adds to their profile***. The messages are published to one or more topics, which are then consumed by applications on the backend. These applications may be generating reports, feeding machine learning systems, updating search results, or performing other operations that are necessary to provide a rich user experience.


### MESSAGING
Kafka is also used for messaging, where applications need to send notifications (such as emails) to users. Those applications can produce messages without needing to be concerned about formatting or how the messages will actually be sent. A single application can then read all the messages to be sent and handle them consistently, including:

Formatting the messages (also known as decorating) using a common look and feel \
Collecting multiple messages into a single notification to be sent \
Applying a user’s preferences for how they want to receive messages

Using a single application for this avoids the need to duplicate functionality in multiple applications, as well as allows operations like aggregation which would not otherwise be possible.

### METRICS AND LOGGING
Kafka is also ideal for collecting application and system metrics and logs. This is a use case in which the ability to have multiple applications producing the same type of message shines. ***Applications publish metrics on a regular basis to a Kafka topic, and those metrics can be consumed by systems for monitoring and alerting. They can also be used in an offline system like Hadoop to perform longer-term analysis, such as growth projections. Log messages can be published in the same way, and can be routed to dedicated log search systems like Elasticsearch or security analysis applications***. Another added benefit of Kafka is that when the destination system needs to change (e.g., it’s time to update the log storage system), there is no need to alter the frontend applications or the means of aggregation.


### COMMIT LOG
***Since Kafka is based on the concept of a commit log, database changes can be published to Kafka and applications can easily monitor this stream to receive live updates as they happen***. This changelog stream can also be used for replicating database updates to a remote system, or for consolidating changes from multiple applications into a single database view. Durable retention is useful here for providing a buffer for the changelog, meaning it can be replayed in the event of a failure of the consuming applications. ***Alternately, log-compacted topics can be used to provide longer retention by only retaining a single change per key.***

### STREAM PROCESSING
Another area that provides numerous types of applications is stream processing. ***While almost all usage of Kafka can be thought of as stream processing, the term is typically used to refer to applications that provide similar functionality to map/reduce processing in Hadoop. Hadoop usually relies on aggregation of data over a long time frame, either hours or days. Stream processing operates on data in real time, as quickly as messages are produced***. 

Stream frameworks allow users to write small applications to operate on Kafka messages, performing tasks such as counting metrics, partitioning messages for efficient processing by other applications, or transforming messages using data from multiple sources.




---------------------------------------------------------------------------------------------------------------------

### Kafka’s Origin
Kafka was created to address the data pipeline problem at LinkedIn. It was designed to provide a high-performance messaging system that can handle many types of data and provide clean, structured data about user activity and system metrics in real time.


### The Birth of Kafka
create a messaging system that could meet the needs of both the monitoring and tracking systems, and scale for the future. The primary goals were to:

1) Decouple producers and consumers by using a push-pull model

2) Provide persistence for message data within the messaging system to allow multiple consumers

3) Optimize for high throughput of messages

4) Allow for horizontal scaling of the system to grow as the data streams grew

The result was a publish/subscribe messaging system that had an interface typical of messaging systems but a storage layer more like a log-aggregation system. Combined with the adoption of Apache Avro for message serialization, Kafka was effective for handling both metrics and user-activity tracking at a scale of billions of messages per day. The scalability of Kafka has helped LinkedIn’s usage grow in excess of one trillion messages produced (as of August 2015) and over a petabyte of data consumed daily.


















 

