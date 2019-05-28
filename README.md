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
with far less code***. These differences make Kafka enough of its own thing that it
doesn’t really make sense to think of it as “yet another queue.”




