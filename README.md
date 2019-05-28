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




