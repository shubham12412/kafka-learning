https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch03.html

### Producer Overview
There are many reasons an application might need to write messages to Kafka: recording user activities for auditing or analysis, recording metrics, storing log messages, recording information from smart appliances, ***communicating asynchronously with other applications, buffering information before writing to a database***, and much more.



***Those diverse use cases also imply diverse requirements: is every message critical, or can we tolerate loss of messages? Are we OK with accidentally duplicating messages? Are there any strict latency or throughput requirements we need to support?***


In the credit card transaction processing example we introduced earlier, we can see that it is critical to never lose a single message nor duplicate any messages. Latency should be low but latencies up to 500ms can be tolerated, and throughput should be very high—we expect to process up to a million messages a second.

A different use case might be to store click information from a website. In that case, some message loss or a few duplicates can be tolerated; latency can be high as long as there is no impact on the user experience. In other words, we don’t mind if it takes a few seconds for the message to arrive at Kafka, as long as the next page loads immediately after the user clicked on a link. Throughput will depend on the level of activity we anticipate on our website.

***The different requirements will influence the way you use the producer API to write messages to Kafka and the configuration you use.***

---------------------------------------------------------------------------------------------------------------------------------

### Constructing a Kafka Producer

Once we instantiate a producer, it is time to start sending messages. There are three primary methods of sending messages:

1) ***Fire-and-forget***
We send a message to the server and don’t really care if it arrives succesfully or not. Most of the time, it will arrive successfully, since Kafka is highly available and the producer will retry sending messages automatically. However, some messages will get lost using this method.

2) ***Synchronous send***
We send a message, the send() method returns a Future object, and we use get() to wait on the future and see if the send() was successful or not.

3) ***Asynchronous send***
We call the send() method with a callback function, which gets triggered when it receives a response from the Kafka broker.



***KafkaProducer has two types of errors. Retriable errors are those that can be resolved by sending the message again. For example, a connection error can be resolved because the connection may get reestablished. A “no leader” error can be resolved when a new leader is elected for the partition. KafkaProducer can be configured to retry those errors automatically, so the application code will get retriable exceptions only when the number of retries was exhausted and the error was not resolved. Some errors will not be resolved by retrying. For example, “message size too large.” In those cases, KafkaProducer will not attempt a retry and will return the exception immediately.***


-------------------------------------------------------------------------------------------------------------------------------

### Configuring Producers

### ACKS
***The acks parameter controls how many partition replicas must receive the record before the producer can consider the write successful***. This option has a significant impact on how likely messages are to be lost. There are three allowed values for the acks parameter:


1) ***If acks=0,*** the producer will not wait for a reply from the broker before assuming the message was sent successfully. This means that if something went wrong and the broker did not receive the message, the producer will not know about it and the message will be lost. However, because the producer is not waiting for any response from the server, it can send messages as fast as the network will support, so this setting can be used to achieve very high throughput.

2) ***If acks=1,*** the producer will receive a success response from the broker the moment the leader replica received the message. If the message can’t be written to the leader (e.g., if the leader crashed and a new leader was not elected yet), the producer will receive an error response and can retry sending the message, avoiding potential loss of data. The message can still get lost if the leader crashes and a replica without this message gets elected as the new leader (via unclean leader election). In this case, throughput depends on whether we send messages synchronously or asynchronously. If our client code waits for a reply from the server (by calling the get() method of the Future object returned when sending a message) it will obviously increase latency significantly (at least by a network roundtrip). If the client uses callbacks, latency will be hidden, but throughput will be limited by the number of in-flight messages (i.e., how many messages the producer will send before receiving replies from the server).

3) ***If acks=all,*** the producer will receive a success response from the broker once all in-sync replicas received the message. This is the safest mode since you can make sure more than one broker has the message and that the message will survive even in the case of crash (more information on this in Chapter 5). However, the latency we discussed in the acks=1 case will be even higher, since we will be waiting for more than just one broker to receive the message.

----------------------------------------------------------------------------------------------------------------------

### BUFFER.MEMORY

### COMPRESSION.TYPE
By default, messages are sent uncompressed. This parameter can be set to **snappy, gzip, or lz4,*** in which case the corresponding compression algorithms will be used to compress the data before sending it to the brokers. ***Snappy compression was invented by Google to provide decent compression ratios with low CPU overhead and good performance, so it is recommended in cases where both performance and bandwidth are a concern.***


### RETRIES
When the producer receives an error message from the server, the error could be transient (e.g., a lack of leader for a partition). In this case, the value of the retries parameter will control how many times the producer will retry sending the message before giving up and notifying the client of an issue. By default, the producer will wait 100ms between retries, but you can control this using the retry.backoff.ms parameter. We recommend testing how long it takes to recover from a crashed broker (i.e., how long until all partitions get new leaders) and setting the number of retries and delay between them such that the total amount of time spent retrying will be longer than the time it takes the Kafka cluster to recover from the crash—otherwise, the producer will give up too soon. Not all errors will be retried by the producer. Some errors are not transient and will not cause retries (e.g., “message too large” error). In general, because the producer handles retries for you, there is no point in handling retries within your own application logic. You will want to focus your efforts on handling nonretriable errors or cases where retry attempts were exhausted.



### BATCH.SIZE
When multiple records are sent to the same partition, the producer will batch them together. ***This parameter controls the amount of memory in bytes (not messages!) that will be used for each batch***. When the batch is full, all the messages in the batch will be sent. ***However, this does not mean that the producer will wait for the batch to become full. The producer will send half-full batches and even batches with just a single message in them. Therefore, setting the batch size too large will not cause delays in sending messages; it will just use more memory for the batches***. Setting the batch size too small will add some overhead because the producer will need to send messages more frequently.


### LINGER.MS
linger.ms controls the amount of time to wait for additional messages before sending the current batch. KafkaProducer sends a batch of messages either when the current batch is full or when the linger.ms limit is reached. By default, the producer will send messages as soon as there is a sender thread available to send them, even if there’s just one message in the batch. ***By setting linger.ms higher than 0, we instruct the producer to wait a few milliseconds to add additional messages to the batch before sending it to the brokers. This increases latency but also increases throughput (because we send more messages at once, there is less overhead per message).***

### CLIENT.ID
This can be any string, and will be used by the brokers to identify messages sent from the client. It is used in logging and metrics, and for quotas.

--------------------------------------------------------------------------------------------------------------------------

### MAX.IN.FLIGHT.REQUESTS.PER.CONNECTION
This controls how many messages the producer will send to the server without receiving responses. Setting this high can increase memory usage while improving throughput, but setting it too high can reduce throughput as batching becomes less efficient. ***Setting this to 1 will guarantee that messages will be written to the broker in the order in which they were sent, even when retries occur.***

---------------------------------------------------------------------------------------------------------------------------

### TIMEOUT.MS, REQUEST.TIMEOUT.MS, AND METADATA.FETCH.TIMEOUT.MS
These parameters control how long the producer will wait for a reply from the server when sending data (request.timeout.ms) and when requesting metadata such as the current leaders for the partitions we are writing to (metadata.fetch.timeout.ms). If the timeout is reached without reply, the producer will either retry sending or respond with an error (either through exception or the send callback). timeout.ms controls the time the broker will wait for in-sync replicas to acknowledge the message in order to meet the acks configuration—the broker will return an error if the time elapses without the necessary acknowledgments.

---------------------------------------------------------------------------------------------------------------------------

### MAX.BLOCK.MS
This parameter controls how long the producer will block when calling send() and when explicitly requesting metadata via partitionsFor(). Those methods block when the producer’s send buffer is full or when metadata is not available. When max.block.ms is reached, a timeout exception is thrown.

### MAX.REQUEST.SIZE
This setting controls the size of a produce request sent by the producer. It caps both the size of the largest message that can be sent and the number of messages that the producer can send in one request. For example, with a default maximum request size of 1 MB, the largest message you can send is 1 MB or the producer can batch 1,024 messages of size 1 KB each into one request. In addition, the broker has its own limit on the size of the largest message it will accept (message.max.bytes). It is usually a good idea to have these configurations match, so the producer will not attempt to send messages of a size that will be rejected by the broker.

----------------------------------------------------------------------------------------------------------------------------

### ORDERING GUARANTEES
Apache Kafka preserves the order of messages within a partition. This means that if messages were sent from the producer in a specific order, the broker will write them to a partition in that order and all consumers will read them in that order. For some use cases, order is very important. There is a big difference between depositing $100 in an account and later withdrawing it, and the other way around! However, some use cases are less sensitive.

Setting the retries parameter to nonzero and the max.in.flight.requests.per.connection to more than one means that it is possible that the broker will fail to write the first batch of messages, succeed to write the second (which was already in-flight), and then retry the first batch and succeed, thereby reversing the order.

Usually, setting the number of retries to zero is not an option in a reliable system, so if guaranteeing order is critical, we recommend setting in.flight.requests.per.session=1 to make sure that while a batch of messages is retrying, additional messages will not be sent (because this has the potential to reverse the correct order). This will severely limit the throughput of the producer, so only use this when order is important.

--------------------------------------------------------------------------------------------------------------------------

### Serializers

### Serializing Using Apache Avro
***Apache Avro is a language-neutral data serialization format***. The project was created by Doug Cutting to provide a way to share data files with a large audience.

Avro data is described in a language-independent schema. The schema is usually described in JSON and the serialization is usually to binary files, although serializing to JSON is also supported. Avro assumes that the schema is present when reading and writing files, usually by embedding the schema in the files themselves.

***One of the most interesting features of Avro, and what makes it a good fit for use in a messaging system like Kafka, is that when the application that is writing messages switches to a new schema, the applications reading the data can continue processing messages without requiring any change or update.***


http://avro.apache.org/docs/current/gettingstartedjava.html













