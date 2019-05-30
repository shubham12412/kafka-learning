# kafka-learning

https://kafka.apache.org/documentation/


We’ve come to think of Kafka as a streaming platform: a system that lets you publish
and subscribe to streams of data, store them, and process them, and that is exactly what Apache Kafka is built to be.

Kafka is like a messaging system in that it lets you publish and subscribe to streams of
messages. In this way, it is similar to products like ActiveMQ, RabbitMQ, IBM’s
MQSeries, and other products. 

But even with these similarities, Kafka has a number
of core differences from traditional messaging systems that make it another kind of
animal entirely. 

Here are the big three differences: 

first, it works as ***a modern distributed system that runs as a cluster and can scale to handle all the applications*** in
even the most massive of companies. Rather than running dozens of individual mes‐
saging brokers, hand wired to different apps, this lets you have a central platform that
can scale elastically to handle all the streams of data in a company. 

Secondly, Kafka is a true ***storage system built to store data for as long as you might like***. This has huge
advantages in using it as a connecting layer as it ***provides real delivery guarantees—its
data is replicated, persistent, and can be kept around as long as you like.*** 

Finally, the world of ***stream processing raises the level of abstraction quite significantly***. Messag‐
ing systems mostly just hand out messages. The ***stream processing capabilities in
Kafka let you compute derived streams and datasets dynamically off of your streams
with far less code***. 

These differences make Kafka enough of its own thing that it doesn’t really make sense to think of it as “yet another queue.”




Another view on Kafka—and one of our motivating lenses in designing and building
it—was to think of it as a kind of real-time version of Hadoop. Hadoop lets you store
and periodically process file data at a very large scale. Kafka lets you store and contin‐
uously process streams of data, also at a large scale. At a technical level, there are defi‐
nitely similarities, and many people see the emerging area of stream processing as a
superset of the kind of batch processing people have done with Hadoop and its vari‐
ous processing layers. 

What this comparison misses is that the use cases that continu‐
ous, low-latency processing opens up are quite different from those that naturally fall
on a batch processing system. Whereas Hadoop and big data targeted analytics appli‐
cations, often in the data warehousing space, the low latency nature of Kafka makes it
applicable for the kind of core applications that directly power a business. This makes
sense: events in a business are happening all the time and the ability to react to them
as they occur makes it much easier to build services that directly power the operation
of the business, feed back into customer experiences, and so on.



The final area Kafka gets compared to is ETL or data integration tools. After all, these
tools move data around, and Kafka moves data around. There is some validity to this
as well, but I think the core difference is that Kafka has inverted the problem. Rather
than a tool for scraping data out of one system and inserting it into another, Kafka is
a platform oriented around real-time streams of events. This means that not only can
it connect off-the-shelf applications and data systems, it can power custom applica‐
tions built to trigger off of these same data streams. We think this architecture cen‐
tered around streams of events is a really important thing. In some ways these flows
of data are the most central aspect of a modern digital company, as important as the
cash flows you’d see in a financial statement.


The ability to combine these three areas—to bring all the streams of data together
across all the use cases—is what makes the idea of a streaming platform so appealing
to people.

Still, all of this is a bit different, and learning how to think and build applications ori‐
ented around ***continuous streams of data*** is quite a mindshift if you are coming from
the world of request/response style applications and relational databases.



popular use cases: 
message bus for event-driven microservices,
stream-processing applications, 
and large-scale data pipelines


It is often described as a “distributed commit log” or more recently as a “distrib‐
uting streaming platform.” A filesystem or database commit log is designed to
provide a durable record of all transactions so that they can be replayed to consis‐
tently build the state of a system. Similarly, data within Kafka is stored durably, in
order, and can be read deterministically. In addition, the data can be distributed
within the system to provide additional protections against failures, as well as signifi‐
cant opportunities for scaling performance.

Messages and Batches

The unit of data within Kafka is called a message. If you are approaching Kafka from a
database background, you can think of this as similar to a row or a record. A message
is simply an array of bytes as far as Kafka is concerned, so the data contained within it
does not have a specific format or meaning to Kafka. A message can have an optional
bit of metadata, which is referred to as a key. The key is also a byte array and, as with
the message, has no specific meaning to Kafka. Keys are used when messages are to
be written to partitions in a more controlled manner. The simplest such scheme is to
generate a consistent hash of the key, and then select the partition number for that
message by taking the result of the hash modulo, the total number of partitions in the
topic. This assures that messages with the same key are always written to the same
partition.


For efficiency, messages are written into Kafka in batches. A batch is just a collection
of messages, all of which are being produced to the same topic and partition. An indi‐
vidual roundtrip across the network for each message would result in excessive over‐
head, and collecting messages together into a batch reduces this. Of course, this is a
tradeoff between latency and throughput: the larger the batches, the more messages
that can be handled per unit of time, but the longer it takes an individual message to
propagate. Batches are also typically compressed, providing more efficient data trans‐
fer and storage at the cost of some processing power.

Messages in Kafka are categorized into topics. The closest analogies for a topic are a
database table or a folder in a filesystem. Topics are additionally broken down into a
number of partitions. Going back to the “commit log” description, a partition is a sin‐
gle log. Messages are written to it in an append-only fashion, and are read in order
from beginning to end. Note that as a topic typically has multiple partitions, there is
no guarantee of message time-ordering across the entire topic, just within a single
partition.

Partitions are also the way that Kafka provides redundancy
and scalability. Each partition can be hosted on a different server, which means that a
single topic can be scaled horizontally across multiple servers to provide performance
far beyond the ability of a single server.


The term stream is often used when discussing data within systems like Kafka. Most
often, a stream is considered to be a single topic of data, regardless of the number of
partitions. This represents a single stream of data moving from the producers to the
consumers.

Producers and Consumers

Kafka clients are users of the system, and there are two basic types: producers and
consumers. There are also advanced client APIs—Kafka Connect API for data inte‐
gration and Kafka Streams for stream processing. The advanced clients use producers
and consumers as building blocks and provide higher-level functionality on top.

Producers create new messages. In other publish/subscribe systems, these may be
called publishers or writers. In general, a message will be produced to a specific topic.
By default, the producer does not care what partition a specific message is written to
and will balance messages over all partitions of a topic evenly. In some cases, the pro‐
ducer will direct messages to specific partitions. This is typically done using the mes‐
sage key and a partitioner that will generate a hash of the key and map it to a specific
partition. This assures that all messages produced with a given key will get written to
the same partition. The producer could also use a custom partitioner that follows
other business rules for mapping messages to partitions.

Consumers read messages. In other publish/subscribe systems, these clients may be
called subscribers or readers. The consumer subscribes to one or more topics and
reads the messages in the order in which they were produced. The consumer keeps
track of which messages it has already consumed by keeping track of the offset of
messages. The offset is another bit of metadata—an integer value that continually
increases—that Kafka adds to each message as it is produced. Each message in a given
partition has a unique offset. By storing the offset of the last consumed message for
each partition, either in Zookeeper or in Kafka itself, a consumer can stop and restart
without losing its place.

Consumers work as part of a consumer group, which is one or more consumers that
work together to consume a topic. The group assures that each partition is only con‐
sumed by one member.

In Figure 1-6, there are three consumers in a single group
consuming a topic. Two of the consumers are working from one partition each, while
the third consumer is working from two partitions. The mapping of a consumer to a
partition is often called ownership of the partition by the consumer.

In this way, consumers can horizontally scale to consume topics with a large number
of messages. Additionally, if a single consumer fails, the remaining members of the
group will rebalance the partitions being consumed to take over for the missing
member.

Brokers and Clusters

A single Kafka server is called a broker. The broker receives messages from producers, assigns offsets to them, and commits the messages to storage on disk. It also services consumers, responding to fetch requests for partitions and responding with the messages that have been committed to disk. Depending on the specific hardware and its performance characteristics, a single broker can easily handle thousands of partitions and millions of messages per second.



Kafka brokers are designed to operate as part of a cluster. Within a cluster of brokers, one broker will also function as the cluster controller (elected automatically from the live members of the cluster). The controller is responsible for administrative operations, including assigning partitions to brokers and monitoring for broker failures. A partition is owned by a single broker in the cluster, and that broker is called the leader of the partition. A partition may be assigned to multiple brokers, which will result in the partition being replicated (as seen in Figure 1-7). This provides redundancy of messages in the partition, such that another broker can take over leadership if there is a broker failure. However, all consumers and producers operating on that partition must connect to the leader.

A key feature of Apache Kafka is that of retention, which is the durable storage of messages for some period of time. Kafka brokers are configured with a default retention setting for topics, either retaining messages for some period of time (e.g., 7 days) or until the topic reaches a certain size in bytes (e.g., 1 GB). Once these limits are reached, messages are expired and deleted so that the retention configuration is a minimum amount of data available at any time. Individual topics can also be configured with their own retention settings so that messages are stored for only as long as they are useful. For example, a tracking topic might be retained for several days, whereas application metrics might be retained for only a few hours. Topics can also be configured as log compacted, which means that Kafka will retain only the last message produced with a specific key. This can be useful for changelog-type data, where only the last update is interesting.


The replication mechanisms within the Kafka clusters are designed only to work within a single cluster, not between multiple clusters.

The Kafka project includes a tool called MirrorMaker, used for this purpose. At its core, MirrorMaker is simply a Kafka consumer and producer, linked together with a queue. Messages are consumed from one Kafka cluster and produced for another. 

Figure 1-8 shows an example of an architecture that uses MirrorMaker, aggregating messages from two local clusters into an aggregate cluster, and then copying that cluster to other datacenters. The simple nature of the application belies its power in creating sophisticated data pipelines, which will be detailed further in Chapter 7.


Why Kafka?
There are many choices for publish/subscribe messaging systems, so what makes Apache Kafka a good choice?

Multiple Producers

Multiple Consumers

Disk-Based Retention

Not only can Kafka handle multiple consumers, but durable message retention means that consumers do not always need to work in real time. Messages are committed to disk, and will be stored with configurable retention rules. These options can be selected on a per-topic basis, allowing for different streams of messages to have different amounts of retention depending on the consumer needs. Durable retention means that if a consumer falls behind, either due to slow processing or a burst in traffic, there is no danger of losing data. It also means that maintenance can be performed on consumers, taking applications offline for a short period of time, with no concern about messages backing up on the producer or getting lost. Consumers can be stopped, and the messages will be retained in Kafka. This allows them to restart and pick up processing messages where they left off with no data loss.



Scalable

Kafka’s flexible scalability makes it easy to handle any amount of data. Users can start with a single broker as a proof of concept, expand to a small development cluster of three brokers, and move into production with a larger cluster of tens or even hundreds of brokers that grows over time as the data scales up. Expansions can be performed while the cluster is online, with no impact on the availability of the system as a whole. This also means that a cluster of multiple brokers can handle the failure of an individual broker, and continue servicing clients. Clusters that need to tolerate more simultaneous failures can be configured with higher replication factors. 

High Performance

All of these features come together to make Apache Kafka a publish/subscribe messaging system with excellent performance under high load. Producers, consumers, and brokers can all be scaled out to handle very large message streams with ease. This can be done while still providing subsecond message latency from producing a message to availability to consumers.

Kafka’s Origin
Kafka was created to address the data pipeline problem at LinkedIn. It was designed to provide a high-performance messaging system that can handle many types of data and provide clean, structured data about user activity and system metrics in real time.


The Birth of Kafka

The primary goals were to:

Decouple producers and consumers by using a push-pull model

Provide persistence for message data within the messaging system to allow multiple consumers

Optimize for high throughput of messages

Allow for horizontal scaling of the system to grow as the data streams grew

The result was a publish/subscribe messaging system that had an interface typical of messaging systems but a storage layer more like a log-aggregation system. Combined with the adoption of Apache Avro for message serialization, Kafka was effective for handling both metrics and user-activity tracking at a scale of billions of messages per day. 

Installing Zookeeper

Apache Kafka uses Zookeeper to store metadata about the Kafka cluster, as well as consumer client details, as shown in Figure 2-1. While it is possible to run a Zookeeper server using scripts contained in the Kafka distribution, it is trivial to install a full version of Zookeeper from the distribution.

ZOOKEEPER ENSEMBLE

A Zookeeper cluster is called an ensemble. Due to the algorithm used, it is recommended that ensembles contain an odd number of servers (e.g., 3, 5, etc.) as a majority of ensemble members (a quorum) must be working in order for Zookeeper to respond to requests. This means that in a three-node ensemble, you can run with one node missing. With a five-node ensemble, you can run with two nodes missing.

SIZING YOUR ZOOKEEPER ENSEMBLE
Consider running Zookeeper in a five-node ensemble. In order to make configuration changes to the ensemble, including swapping a node, you will need to reload nodes one at a time. If your ensemble cannot tolerate more than one node being down, doing maintenance work introduces additional risk. It is also not recommended to run more than seven nodes, as performance can start to degrade due to the nature of the consensus protocol.

https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch02.html

Kafka utilizes Zookeeper for storing metadata information about the brokers, topics, and partitions. Writes to Zookeeper are only performed on changes to the membership of consumer groups or on changes to the Kafka cluster itself. This amount of traffic is minimal, and it does not justify the use of a dedicated Zookeeper ensemble for a single Kafka cluster. In fact, many deployments will use a single Zookeeper ensemble for multiple Kafka clusters (using a chroot Zookeeper path for each cluster, as described earlier in this chapter).

KAFKA CONSUMERS AND ZOOKEEPER

Prior to Apache Kafka 0.9.0.0, consumers, in addition to the brokers, utilized Zookeeper to directly store information about the composition of the consumer group, what topics it was consuming, and to periodically commit offsets for each partition being consumed (to enable failover between consumers in the group). With version 0.9.0.0, a new consumer interface was introduced which allows this to be managed directly with the Kafka brokers.


However, there is a concern with consumers and Zookeeper under certain configurations. Consumers have a configurable choice to use either Zookeeper or Kafka for committing offsets, and they can also configure the interval between commits. If the consumer uses Zookeeper for offsets, each consumer will perform a Zookeeper write at every interval for every partition it consumes. A reasonable interval for offset commits is 1 minute, as this is the period of time over which a consumer group will read duplicate messages in the case of a consumer failure. These commits can be a significant amount of Zookeeper traffic, especially in a cluster with many consumers, and will need to be taken into account. It may be neccessary to use a longer commit interval if the Zookeeper ensemble is not able to handle the traffic. However, it is recommended that consumers using the latest Kafka libraries use Kafka for committing offsets, removing the dependency on Zookeeper.


Outside of using a single ensemble for multiple Kafka clusters, it is not recommended to share the ensemble with other applications, if it can be avoided. Kafka is sensitive to Zookeeper latency and timeouts, and an interruption in communications with the ensemble will cause the brokers to behave unpredictably. This can easily cause multiple brokers to go offline at the same time, should they lose Zookeeper connections, which will result in offline partitions. It also puts stress on the cluster controller, which can show up as subtle errors long after the interruption has passed, such as when trying to perform a controlled shutdown of a broker. Other applications that can put stress on the Zookeeper ensemble, either through heavy usage or improper operations, should be segregated to their own ensemble.


------------------------------------------------------------------------------------------------------------------

https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch03.html

Chapter 3. Kafka Producers: Writing Messages to Kafka

There are many reasons an application might need to write messages to Kafka: recording user activities for auditing or analysis, recording metrics, storing log messages, recording information from smart appliances, communicating asynchronously with other applications, buffering information before writing to a database, and much more.

Those diverse use cases also imply diverse requirements: is every message critical, or can we tolerate loss of messages? Are we OK with accidentally duplicating messages? Are there any strict latency or throughput requirements we need to support?

In the credit card transaction processing example we introduced earlier, we can see that it is critical to never lose a single message nor duplicate any messages. Latency should be low but latencies up to 500ms can be tolerated, and throughput should be very high—we expect to process up to a million messages a second.

A different use case might be to store click information from a website. In that case, some message loss or a few duplicates can be tolerated; latency can be high as long as there is no impact on the user experience. In other words, we don’t mind if it takes a few seconds for the message to arrive at Kafka, as long as the next page loads immediately after the user clicked on a link. Throughput will depend on the level of activity we anticipate on our website.

The different requirements will influence the way you use the producer API to write messages to Kafka and the configuration you use.

We start producing messages to Kafka by creating a ProducerRecord, which must include the topic we want to send the record to and a value. Optionally, we can also specify a key and/or a partition. Once we send the ProducerRecord, the first thing the producer will do is serialize the key and value objects to ByteArrays so they can be sent over the network.

Next, the data is sent to a partitioner. If we specified a partition in the ProducerRecord, the partitioner doesn’t do anything and simply returns the partition we specified. If we didn’t, the partitioner will choose a partition for us, usually based on the ProducerRecord key. Once a partition is selected, the producer knows which topic and partition the record will go to. It then adds the record to a batch of records that will also be sent to the same topic and partition. A separate thread is responsible for sending those batches of records to the appropriate Kafka brokers.

When the broker receives the messages, it sends back a response. If the messages were successfully written to Kafka, it will return a RecordMetadata object with the topic, partition, and the offset of the record within the partition. If the broker failed to write the messages, it will return an error. When the producer receives an error, it may retry sending the message a few more times before giving up and returning an error.



Properties kafkaProps = new Properties(); 1
kafkaProps.put("bootstrap.servers", "broker1:9092,broker2:9092");

kafkaProps.put("key.serializer",
    "org.apache.kafka.common.serialization.StringSerializer"); 2
kafkaProps.put("value.serializer",
    "org.apache.kafka.common.serialization.StringSerializer");

producer = new KafkaProducer<String, String>(kafkaProps); 3


Once we instantiate a producer, it is time to start sending messages. There are three primary methods of sending messages:

Fire-and-forget

We send a message to the server and don’t really care if it arrives succesfully or not. Most of the time, it will arrive successfully, since Kafka is highly available and the producer will retry sending messages automatically. However, some messages will get lost using this method.

ProducerRecord<String, String> record =
    new ProducerRecord<>("CustomerCountry", "Precision Products",
        "France"); 1
try {
    producer.send(record); 2
} catch (Exception e) {
    e.printStackTrace(); 3
}


Synchronous send

We send a message, the send() method returns a Future object, and we use get() to wait on the future and see if the send() was successful or not.

ProducerRecord<String, String> record =
    new ProducerRecord<>("CustomerCountry", "Precision Products", "France");
try {
    producer.send(record).get(); 1
} catch (Exception e) {
    e.printStackTrace(); 2
}



Asynchronous send

We call the send() method with a callback function, which gets triggered when it receives a response from the Kafka broker.


private class DemoProducerCallback implements Callback { 1
    @Override
    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
        if (e != null) {
            e.printStackTrace(); 2
        }
    }
}

ProducerRecord<String, String> record =
    new ProducerRecord<>("CustomerCountry", "Biomedical Materials", "USA"); 3
producer.send(record, new DemoProducerCallback()); 4


Configuring Producers

So far we’ve seen very few configuration parameters for the producers—just the mandatory bootstrap.servers URI and serializers.

The producer has a large number of configuration parameters; most are documented in Apache Kafka documentation and many have reasonable defaults so there is no reason to tinker with every single parameter. However, ***some of the parameters have a significant impact on memory use, performance, and reliability of the producers***. We will review those here.


ORDERING GUARANTEES

Apache Kafka preserves the order of messages within a partition. This means that if messages were sent from the producer in a specific order, the broker will write them to a partition in that order and all consumers will read them in that order. For some use cases, order is very important. There is a big difference between depositing $100 in an account and later withdrawing it, and the other way around! However, some use cases are less sensitive.

Setting the retries parameter to nonzero and the max.in.flight.requests.per.connection to more than one means that it is possible that the broker will fail to write the first batch of messages, succeed to write the second (which was already in-flight), and then retry the first batch and succeed, thereby reversing the order.

Usually, setting the number of retries to zero is not an option in a reliable system, so if guaranteeing order is critical, we recommend setting in.flight.requests.per.session=1 to make sure that while a batch of messages is retrying, additional messages will not be sent (because this has the potential to reverse the correct order). This will severely limit the throughput of the producer, so only use this when order is important.




For these reasons, we recommend using existing serializers and deserializers such as JSON, Apache Avro, Thrift, or Protobuf. 


https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch04.html



Kafka consumers are typically part of a consumer group. When multiple consumers are subscribed to a topic and belong to the same consumer group, each consumer in the group will receive messages from a different subset of the partitions in the topic.

The main way we scale data consumption from a Kafka topic is by adding more consumers to a consumer group. It is common for Kafka consumers to do high-latency operations such as write to a database or a time-consuming computation on the data. In these cases, a single consumer can’t possibly keep up with the rate data flows into a topic, and adding more consumers that share the load by having each consumer own just a subset of the partitions and messages is our main method of scaling. This is a good reason to create topics with a large number of partitions—it allows adding more consumers when the load increases. Keep in mind that there is no point in adding more consumers than you have partitions in a topic—some of the consumers will just be idle.


In addition to adding consumers in order to scale a single application, it is very common to have multiple applications that need to read data from the same topic. In fact, one of the main design goals in Kafka was to make the data produced to Kafka topics available for many use cases throughout the organization. In those cases, we want each application to get all of the messages, rather than just a subset. To make sure an application gets all the messages in a topic, ensure the application has its own consumer group. Unlike many traditional messaging systems, Kafka scales to a large number of consumers and consumer groups without reducing performance.


To summarize, you create a new consumer group for each application that needs all the messages from one or more topics. You add consumers to an existing consumer group to scale the reading and processing of messages from the topics, so each additional consumer in a group will only get a subset of the messages.

Consumer Groups and Partition Rebalance

As we saw in the previous section, consumers in a consumer group share ownership of the partitions in the topics they subscribe to. When we add a new consumer to the group, it starts consuming messages from partitions previously consumed by another consumer. The same thing happens when a consumer shuts down or crashes; it leaves the group, and the partitions it used to consume will be consumed by one of the remaining consumers. Reassignment of partitions to consumers also happen when the topics the consumer group is consuming are modified (e.g., if an administrator adds new partitions).

Moving partition ownership from one consumer to another is called a rebalance. Rebalances are important because they provide the consumer group with high availability and scalability (allowing us to easily and safely add and remove consumers), but in the normal course of events they are fairly undesirable. During a rebalance, consumers can’t consume messages, so a rebalance is basically a short window of unavailability of the entire consumer group. In addition, when partitions are moved from one consumer to another, the consumer loses its current state; if it was caching any data, it will need to refresh its caches—slowing down the application until the consumer sets up its state again. Throughout this chapter we will discuss how to safely handle rebalances and how to avoid unnecessary ones.


The way consumers maintain membership in a consumer group and ownership of the partitions assigned to them is by sending heartbeats to a Kafka broker designated as the group coordinator (this broker can be different for different consumer groups). As long as the consumer is sending heartbeats at regular intervals, it is assumed to be alive, well, and processing messages from its partitions. Heartbeats are sent when the consumer polls (i.e., retrieves records) and when it commits records it has consumed.


If the consumer stops sending heartbeats for long enough, its session will time out and the group coordinator will consider it dead and trigger a rebalance. If a consumer crashed and stopped processing messages, it will take the group coordinator a few seconds without heartbeats to decide it is dead and trigger the rebalance. During those seconds, no messages will be processed from the partitions owned by the dead consumer. When closing a consumer cleanly, the consumer will notify the group coordinator that it is leaving, and the group coordinator will trigger a rebalance immediately, reducing the gap in processing. Later in this chapter we will discuss configuration options that control heartbeat frequency and session timeouts and how to set those to match your requirements.




#### HOW DOES THE PROCESS OF ASSIGNING PARTITIONS TO BROKERS WORK?

When a consumer wants to join a group, it sends a JoinGroup request to the group coordinator. The first consumer to join the group becomes the group leader. The leader receives a list of all consumers in the group from the group coordinator (this will include all consumers that sent a heartbeat recently and which are therefore considered alive) and is responsible for assigning a subset of partitions to each consumer. It uses an implementation of PartitionAssignor to decide which partitions should be handled by which consumer.

Kafka has two built-in partition assignment policies, which we will discuss in more depth in the configuration section. After deciding on the partition assignment, the consumer group leader sends the list of assignments to the GroupCoordinator, which sends this information to all the consumers. Each consumer only sees his own assignment—the leader is the only client process that has the full list of consumers in the group and their assignments. This process repeats every time a rebalance happens.


Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("group.id", "CountryCounter");
props.put("key.deserializer",
    "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer",
    "org.apache.kafka.common.serialization.StringDeserializer");

KafkaConsumer<String, String> consumer =
    new KafkaConsumer<String, String>(props);
    
    
    consumer.subscribe(Collections.singletonList("customerCountries")); 
    
    Here we simply create a list with a single element: the topic name customerCountries.
    
    
It is also possible to call subscribe with a regular expression. The expression can match multiple topic names, and if someone creates a new topic with a name that matches, a rebalance will happen almost immediately and the consumers will start consuming from the new topic. This is useful for applications that need to consume from multiple topics and can handle the different types of data the topics will contain. Subscribing to multiple topics using a regular expression is most commonly used in applications that replicate data between Kafka and another system.

To subscribe to all test topics, we can call:

consumer.subscribe(Pattern.compile("test.*"));

#### The Poll Loop

At the heart of the consumer API is a simple loop for polling the server for more data. 

Once the consumer subscribes to topics, ***the poll loop handles all details of coordination, partition rebalances, heartbeats, and data fetching, leaving the developer with a clean API that simply returns available data from the assigned partitions***. The main body of a consumer will look as follows:


try {
    while (true) { 1
        ConsumerRecords<String, String> records = consumer.poll(100); 2
        for (ConsumerRecord<String, String> record : records) 3
        {
            log.debug("topic = %s, partition = %d, offset = %d,"
                customer = %s, country = %s\n",
                record.topic(), record.partition(), record.offset(),
                record.key(), record.value());

            int updatedCount = 1;
            if (custCountryMap.countainsValue(record.value())) {
                updatedCount = custCountryMap.get(record.value()) + 1;
            }
            custCountryMap.put(record.value(), updatedCount)

            JSONObject json = new JSONObject(custCountryMap);
            System.out.println(json.toString(4)) 4
        }
    }
} finally {
    consumer.close(); 5
}


2 This is the most important line in the chapter. The same way that sharks must keep moving or they die, consumers must keep polling Kafka or they will be considered dead and the partitions they are consuming will be handed to another consumer in the group to continue consuming. The parameter we pass, poll(), is a timeout interval and controls how long poll() will block if data is not available in the consumer buffer. If this is set to 0, poll() will return immediately; otherwise, it will wait for the specified number of milliseconds for data to arrive from the broker.



5 Always close() the consumer before exiting. This will close the network connections and sockets. It will also trigger a rebalance immediately rather than wait for the group coordinator to discover that the consumer stopped sending heartbeats and is likely dead, which will take longer and therefore result in a longer period of time in which consumers can’t consume messages from a subset of the partitions.


The poll loop does a lot more than just get data. The first time you call poll() with a new consumer, it is responsible for finding the GroupCoordinator, joining the consumer group, and receiving a partition assignment. If a rebalance is triggered, it will be handled inside the poll loop as well. And of course the heartbeats that keep consumers alive are sent from within the poll loop. For this reason, we try to make sure that whatever processing we do between iterations is fast and efficient.



THREAD SAFETY
You can’t have multiple consumers that belong to the same group in one thread and you can’t have multiple threads safely use the same consumer. One consumer per thread is the rule. To run multiple consumers in the same group in one application, you will need to run each in its own thread. It is useful to wrap the consumer logic in its own object and then use Java’s ExecutorService to start multiple threads each with its own consumer. The Confluent blog has a tutorial that shows how to do just that.



Commits and Offsets

Whenever we call poll(), it returns records written to Kafka that consumers in our group have not read yet. This means that we have a way of tracking which records were read by a consumer of the group. As discussed before, one of Kafka’s unique characteristics is that it does not track acknowledgments from consumers the way many JMS queues do. Instead, it allows consumers to use Kafka to track their position (offset) in each partition.

We call the action of updating the current position in the partition a commit.



How does a consumer commit an offset? It produces a message to Kafka, to a special __consumer_offsets topic, with the committed offset for each partition. As long as all your consumers are up, running, and churning away, this will have no impact. However, if a consumer crashes or a new consumer joins the consumer group, this will trigger a rebalance. After a rebalance, each consumer may be assigned a new set of partitions than the one it processed before. In order to know where to pick up the work, the consumer will read the latest committed offset of each partition and continue from there.


If the committed offset is smaller than the offset of the last message the client processed, the messages between the last processed offset and the committed offset will be processed twice

If the committed offset is larger than the offset of the last message the client actually processed, all messages between the last processed offset and the committed offset will be missed by the consumer group


Clearly, managing offsets has a big impact on the client application. The KafkaConsumer API provides multiple ways of committing offsets:


Automatic Commit
The easiest way to commit offsets is to allow the consumer to do it for you. If you configure enable.auto.commit=true, then every five seconds the consumer will commit the largest offset your client received from poll(). The five-second interval is the default and is controlled by setting auto.commit.interval.ms. Just like everything else in the consumer, the automatic commits are driven by the poll loop. Whenever you poll, the consumer checks if it is time to commit, and if it is, it will commit the offsets it returned in the last poll.

Before using this convenient option, however, it is important to understand the consequences.

Consider that, by default, automatic commits occur every five seconds. Suppose that we are three seconds after the most recent commit and a rebalance is triggered. After the rebalancing, all consumers will start consuming from the last offset committed. In this case, the offset is three seconds old, so all the events that arrived in those three seconds will be processed twice. It is possible to configure the commit interval to commit more frequently and reduce the window in which records will be duplicated, but it is impossible to completely eliminate them.

With autocommit enabled, a call to poll will always commit the last offset returned by the previous poll. It doesn’t know which events were actually processed, so it is critical to always process all the events returned by poll() before calling poll() again. (Just like poll(), close() also commits offsets automatically.) This is usually not an issue, but pay attention when you handle exceptions or exit the poll loop prematurely.

Automatic commits are convenient, but they don’t give developers enough control to avoid duplicate messages.



Standalone Consumer: Why and How to Use a Consumer Without a Group

So far, we have discussed consumer groups, which are where partitions are assigned automatically to consumers and are rebalanced automatically when consumers are added or removed from the group. Typically, this behavior is just what you want, but in some cases you want something much simpler. Sometimes you know you have a single consumer that always needs to read data from all the partitions in a topic, or from a specific partition in a topic. In this case, there is no reason for groups or rebalances—just assign the consumer-specific topic and/or partitions, consume messages, and commit offsets on occasion.

When you know exactly which partitions the consumer should read, you don’t subscribe to a topic—instead, you assign yourself a few partitions. A consumer can either subscribe to topics (and be part of a consumer group), or assign itself partitions, but not both at the same time.

Here is an example of how a consumer can assign itself all partitions of a specific topic and consume from them:



https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch05.html

Since covering every single implementation detail and design decision is beyond the scope of this book, in this chapter we focus on three topics that are especially relevant to Kafka practitioners:

How Kafka replication works

How Kafka handles requests from producers and consumers

How Kafka handles storage such as file format and indexes


Cluster Membership
Kafka uses Apache Zookeeper to maintain the list of brokers that are currently members of a cluster. Every broker has a unique identifier that is either set in the broker configuration file or automatically generated. Every time a broker process starts, it registers itself with its ID in Zookeeper by creating an ephemeral node. Different Kafka components subscribe to the /brokers/ids path in Zookeeper where brokers are registered so that they get notified when brokers are added or removed.


If you try to start another broker with the same ID, you will get an error—the new broker will try to register, but fail because we already have a Zookeeper node for the same broker ID.

When a broker loses connectivity to Zookeeper (usually as a result of the broker stopping, but this can also happen as a result of network partition or a long garbage-collection pause), the ephemeral node that the broker created when starting will be automatically removed from Zookeeper. Kafka components that are watching the list of brokers will be notified that the broker is gone.

The Controller

The controller is one of the Kafka brokers that, in addition to the usual broker functionality, is responsible for electing partition leaders (we’ll discuss partition leaders and what they do in the next section). The first broker that starts in the cluster becomes the controller by creating an ephemeral node in ZooKeeper called /controller. When other brokers start, they also try to create this node, but receive a “node already exists” exception, which causes them to “realize” that the controller node already exists and that the cluster already has a controller. The brokers create a Zookeeper watch on the controller node so they get notified of changes to this node. This way, we guarantee that the cluster will only have one controller at a time.

When the controller broker is stopped or loses connectivity to Zookeeper, the ephemeral node will disappear. Other brokers in the cluster will be notified through the Zookeeper watch that the controller is gone and will attempt to create the controller node in Zookeeper themselves. The first node to create the new controller in Zookeeper is the new controller, while the other nodes will receive a “node already exists” exception and re-create the watch on the new controller node. Each time a controller is elected, it receives a new, higher controller epoch number through a Zookeeper conditional increment operation. The brokers know the current controller epoch and if they receive a message from a controller with an older number, they know to ignore it.


***When the controller notices that a broker left the cluster (by watching the relevant Zookeeper path), it knows that all the partitions that had a leader on that broker will need a new leader. It goes over all the partitions that need a new leader, determines who the new leader should be (simply the next replica in the replica list of that partition), and sends a request to all the brokers that contain either the new leaders or the existing followers for those partitions. The request contains information on the new leader and the followers for the partitions. Each new leader knows that it needs to start serving producer and consumer requests from clients while the followers know that they need to start replicating messages from the new leader.***


When the controller notices that a broker joined the cluster, it uses the broker ID to check if there are replicas that exist on this broker. If there are, the controller notifies both new and existing brokers of the change, and the replicas on the new broker start replicating messages from the existing leaders.




***To summarize, Kafka uses Zookeeper’s ephemeral node feature to elect a controller and to notify the controller when nodes join and leave the cluster. The controller is responsible for electing leaders among the partitions and replicas whenever it notices nodes join and leave the cluster. The controller uses the epoch number to prevent a “split brain” scenario where two nodes believe each is the current controller.***


Replication

Replication is at the heart of Kafka’s architecture. The very first sentence in Kafka’s documentation describes it as ***“a distributed, partitioned, replicated commit log service.”*** Replication is critical because it is the way Kafka guarantees availability and durability when individual nodes inevitably fail.



As we’ve already discussed, data in Kafka is organized by topics. Each topic is partitioned, and each partition can have multiple replicas. Those replicas are stored on brokers, and each broker typically stores hundreds or even thousands of replicas belonging to different topics and partitions.

There are two types of replicas:

