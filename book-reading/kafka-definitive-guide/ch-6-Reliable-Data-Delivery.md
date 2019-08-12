https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch06.html

***More so, reliability is a property of a system—not of a single component—so even when we are talking about the reliability guarantees of Apache Kafka, you will need to keep the entire system and its use cases in mind. When it comes to reliability, the systems that integrate with Kafka are as important as Kafka itself. And because reliability is a system concern, it cannot be the responsibility of just one person. Everyone—Kafka administrators, Linux administrators, network and storage administrators, and the application developers—must work together to build a reliable system.***

Apache Kafka is very flexible about reliable data delivery. We understand that Kafka has many use cases, from tracking clicks in a website to credit card payments. ***Some of the use cases require utmost reliability while others prioritize speed and simplicity over reliability. Kafka was written to be configurable enough and its client API flexible enough to allow all kinds of reliability trade-offs.***

---------------------------------------------------------------------------------------------------------------------------

### Reliability Guarantees
When we talk about reliability, we usually talk in terms of guarantees, which are the behaviors a system is guaranteed to preserve under different circumstances.

Understanding the guarantees Kafka provides is critical for those seeking to build reliable applications. This understanding allows the developers of the system to figure out how it will behave under different failure conditions. So, what does Apache Kafka guarantee?

1) ***Kafka provides order guarantee of messages in a partition. If message B was written after message A, using the same producer in the same partition, then Kafka guarantees that the offset of message B will be higher than message A, and that consumers will read message B after message A.

2) ***Produced messages are considered “committed” when they were written to the partition on all its in-sync replicas (but not necessarily flushed to disk). Producers can choose to receive acknowledgments of sent messages when the message was fully committed, when it was written to the leader, or when it was sent over the network.

3) ***Messages that are committed will not be lost as long as at least one replica remains alive.

4) ***Consumers can only read messages that are committed.***


These basic guarantees can be used while building a reliable system, but in themselves, don’t make the system fully reliable. ***There are trade-offs involved in building a reliable system, and Kafka was built to allow administrators and developers to decide how much reliability they need by providing configuration parameters that allow controlling these trade-offs. The trade-offs usually involve how important it is to reliably and consistently store messages versus other important considerations such as availability, high throughput, low latency, and hardware costs. 


------------------------------------------------------------------------------------------------------------------------

### Replication
Kafka’s replication mechanism, with its multiple replicas per partition, is at the core of all of Kafka’s reliability guarantees. ***Having a message written in multiple replicas is how Kafka provides durability of messages in the event of a crash.***

Each Kafka topic is broken down into partitions, which are the basic data building blocks. A partition is stored on a single disk. Kafka guarantees order of events within a partition and a partition can be either online (available) or offline (unavailable). Each partition can have multiple replicas, one of which is a designated leader. All events are produced to and consumed from the leader replica. Other replicas just need to stay in sync with the leader and replicate all the recent events on time. If the leader becomes unavailable, one of the in-sync replicas becomes the new leader.


***A replica is considered in-sync if it is the leader for a partition, or if it is a follower that:

1) Has an active session with Zookeeper—meaning, it sent a heartbeat to Zookeeper in the last 6 seconds (configurable).

2) Fetched messages from the leader in the last 10 seconds (configurable).

3) Fetched the most recent messages from the leader in the last 10 seconds. That is, it isn’t enough that the follower is still getting messages from the leader; it must have almost no lag.



If a replica loses connection to Zookeeper, stops fetching new messages, or falls behind and can’t catch up within 10 seconds, the replica is considered out-of-sync. An out-of-sync replica gets back into sync when it connects to Zookeeper again and catches up to the most recent message written to the leader. This usually happens quickly after a temporary network glitch is healed but can take a while if the broker the replica is stored on was down for a longer period of time.

#### OUT-OF-SYNC REPLICAS
Seeing one or more replicas rapidly flip between in-sync and out-of-sync status is a sure sign that something is wrong with the cluster. The cause is often a misconfiguration of Java’s garbage collection on a broker. Misconfigured garbage collection can cause the broker to pause for a few seconds, during which it will lose connectivity to Zookeeper. When a broker loses connectivity to Zookeeper, it is considered out-of-sync with the cluster, which causes the flipping behavior.

An in-sync replica that is slightly behind can slow down producers and consumers—since they wait for all the in-sync replicas to get the message before it is committed. Once a replica falls out of sync, we no longer wait for it to get messages. It is still behind, but now there is no performance impact. The catch is that with fewer in-sync replicas, the effective replication factor of the partition is lower and therefore there is a higher risk for downtime or data loss.

------------------------------------------------------------------------------------------------------------------

### Broker Configuration

There are three configuration parameters in the broker that change Kafka’s behavior regarding reliable message storage. Like many broker configuration variables, these can apply at the broker level, controlling configuration for all topics in the system, and at the topic level, controlling behavior for a specific topic.

Being able to control reliability trade-offs at the topic level means that the same Kafka cluster can be used to host reliable and nonreliable topics. For example, at a bank, the administrator will probably want to set very reliable defaults for the entire cluster but make an exception to the topic that stores customer complaints where some data loss is acceptable.

Let’s look at these configuration parameters one by one and see how they affect reliability of message storage in Kafka and the trade-offs involved.

-------------------------------------------------------------------------------------------------------------------

### Replication Factor
The topic-level configuration is replication.factor. At the broker level, you control the default.replication.factor for automatically created topics.


Until this point, throughout the book, we always assumed that topics had a replication factor of three, meaning that each partition is replicated three times on three different brokers. This was a reasonable assumption, as this is Kafka’s default, but this is also a configuration that users can modify. ***Even after a topic exists, you can choose to add or remove replicas and thereby modify the replication factor.***

***A replication factor of N allows you to lose N-1 brokers while still being able to read and write data to the topic reliably. So a higher replication factor leads to higher availability, higher reliability, and fewer disasters. On the flip side, for a replication factor of N, you will need at least N brokers and you will store N copies of the data, meaning you will need N times as much disk space. We are basically trading availability for hardware.***


Placement of replicas is also very important. By default, Kafka will make sure each replica for a partition is on a separate broker. However, in some cases, this is not safe enough. If all replicas for a partition are placed on brokers that are on the same rack and the top-of-rack switch misbehaves, you will lose availability of the partition regardless of the replication factor. To protect against rack-level misfortune, we recommend placing brokers in multiple racks and using the broker.rack broker configuration parameter to configure the rack name for each broker. If rack names are configured, Kafka will make sure replicas for a partition are spread across multiple racks in order to guarantee even higher availability.

------------------------------------------------------------------------------------------------------------------

## Unclean Leader Election
This configuration is only available at the broker (and in practice, cluster-wide) level. The parameter name is unclean.leader.election.enable and by default it is set to true.

As explained earlier, ***when the leader for a partition is no longer available, one of the in-sync replicas will be chosen as the new leader. This leader election is “clean” in the sense that it guarantees no loss of committed data—by definition, committed data exists on all in-sync replicas.***


***But what do we do when no in-sync replica exists except for the leader that just became unavailable?

This situation can happen in one of two scenarios:

1) The partition had three replicas, and the two followers became unavailable (let’s say two brokers crashed). In this situation, as producers continue writing to the leader, all the messages are acknowledged and committed (since the leader is the one and only in-sync replica). Now let’s say that the leader becomes unavailable (oops, another broker crash). In this scenario, if one of the out-of-sync followers starts first, we have an out-of-sync replica as the only available replica for the partition.

2) The partition had three replicas and, due to network issues, the two followers fell behind so that even though they are up and replicating, they are no longer in sync. The leader keeps accepting messages as the only in-sync replica. Now if the leader becomes unavailable, the two available replicas are no longer in-sync.

If we don’t allow the out-of-sync replica to become the new leader, the partition will remain offline until we bring the old leader (and the last in-sync replica) back online. In some cases (e.g., memory chip needs replacement), this can take many hours.

***If we do allow the out-of-sync replica to become the new leader, we are going to lose all messages that were written to the old leader while that replica was out of sync and also cause some inconsistencies in consumers. Why? Imagine that while replicas 0 and 1 were not available, we wrote messages with offsets 100-200 to replica 2 (then the leader). Now replica 2 is unavailable and replica 0 is back online. Replica 0 only has messages 0-100 but not 100-200. If we allow replica 0 to become the new leader, it will allow producers to write new messages and allow consumers to read them. So, now the new leader has completely new messages 100-200. First, let’s note that some consumers may have read the old messages 100-200, some consumers got the new 100-200, and some got a mix of both. This can lead to pretty bad consequences when looking at things like downstream reports. In addition, replica 2 will come back online and become a follower of the new leader. At that point, it will delete any messages it got that are ahead of the current leader. Those messages will not be available to any consumer in the future.




***In summary, if we allow out-of-sync replicas to become leaders, we risk data loss and data inconsistencies. If we don’t allow them to become leaders, we face lower availability as we must wait for the original leader to become available before the partition is back online.

***Setting unclean.leader.election.enable to true means we allow out-of-sync replicas to become leaders (knowns as unclean election), knowing that we will lose messages when this occurs. If we set it to false, we choose to wait for the original leader to come back online, resulting in lower availability. We typically see unclean leader election disabled (configuration set to false) in systems where data quality and consistency are critical—banking systems are a good example (most banks would rather be unable to process credit card payments for few minutes or even hours than risk processing a payment incorrectly). In systems where availability is more important, such as real-time clickstream analysis, unclean leader election is often enabled.



----------------------------------------------------------------------------------------------------------------------

### Minimum In-Sync Replicas
Both the topic and the broker-level configuration are called min.insync.replicas.

As we’ve seen, there are cases where even though we configured a topic to have three replicas, we may be left with a single in-sync replica. If this replica becomes unavailable, we may have to choose between availability and consistency. This is never an easy choice. Note that part of the problem is that, per Kafka reliability guarantees, data is considered committed when it is written to all in-sync replicas, even when all means just one replica and the data could be lost if that replica is unavailable.

If you would like to be sure that committed data is written to more than one replica, you need to set the minimum number of in-sync replicas to a higher value. If a topic has three replicas and you set min.insync.replicas to 2, then you can only write to a partition in the topic if at least two out of the three replicas are in-sync.

When all three replicas are in-sync, everything proceeds normally. This is also true if one of the replicas becomes unavailable. However, if two out of three replicas are not available, the brokers will no longer accept produce requests. Instead, producers that attempt to send data will receive NotEnoughReplicasException. Consumers can continue reading existing data. In effect, with this configuation, a single in-sync replica becomes read-only. This prevents the undesirable situation where data is produced and consumed, only to disappear when unclean election occurs. In order to recover from this read-only situation, we must make one of the two unavailable partitions available again (maybe restart the broker) and wait for it to catch up and get in-sync.

--------------------------------------------------------------------------------------------------------------------

### Using Producers in a Reliable System
Even if we configure the brokers in the most reliable configuration possible, the system as a whole can still accidentally lose data if we don’t configure the producers to be reliable as well.

Here are two example scenarios to demonstrate this:

1) We configured the brokers with three replicas, and unclean leader election is disabled. So we should never lose a single message that was committed to the Kafka cluster. However, we configured the producer to send messages with acks=1. We send a message from the producer and it was written to the leader, but not yet to the in-sync replicas. The leader sent back a response to the producer saying “Message was written successfully” and immediately crashes before the data was replicated to the other replicas. The other replicas are still considered in-sync (remember that it takes a while before we declare a replica out of sync) and one of them will become the leader. Since the message was not written to the replicas, it will be lost. But the producing application thinks it was written successfully. The system is consistent because no consumer saw the message (it was never committed because the replicas never got it), but from the producer perspective, a message was lost.

2) We configured the brokers with three replicas, and unclean leader election is disabled. We learned from our mistakes and started producing messages with acks=all. Suppose that we are attempting to write a message to Kafka, but the leader for the partition we are writing to just crashed and a new one is still getting elected. Kafka will respond with “Leader not Available.” At this point, if the producer doesn’t handle the error correctly and doesn’t retry until the write is successful, the message may be lost. Once again, this is not a broker reliability issue because the broker never got the message; and it is not a consistency issue because the consumers never got the message either. But if producers don’t handle errors correctly, they may cause message loss.

-------------------------------------------------------------------------------------------------------------------

### Send Acknowledgments
Producers can choose between three different acknowledgment modes:

1) acks=0 means that a message is considered to be written successfully to Kafka if the producer managed to send it over the network. You will still get errors if the object you are sending cannot be serialized or if the network card failed, but you won’t get any error if the partition is offline or if the entire Kafka cluster decided to take a long vacation. This means that even in the expected case of a clean leader election, your producer will lose messages because it won’t know that the leader is unavailable while a new leader is being elected. Running with acks=0 is very fast (which is why you see a lot of benchmarks with this configuration). You can get amazing throughput and utilize most of your bandwidth, but you are guaranteed to lose some messages if you choose this route.


2) acks=1 means that the leader will send either an acknowledgment or an error the moment it got the message and wrote it to the partition data file (but not necessarily synced to disk). This means that under normal circumstances of leader election, your producer will get LeaderNotAvailableException while a leader is getting elected, and if the producer handles this error correctly (see next section), it will retry sending the message and the message will arrive safely to the new leader. You can lose data if the leader crashes and some messages that were successfully written to the leader and acknowledged were not replicated to the followers before the crash.

3) acks=all means that the leader will wait until all in-sync replicas got the message before sending back an acknowledgment or an error. In conjunction with the min.insync.replicas configuration on the broker, this lets you control how many replicas get the message before it is acknowledged. This is the safest option—the producer won’t stop trying to send the message before it is fully committed. This is also the slowest option—the producer waits for all replicas to get all the messages before it can mark the message batch as “done” and carry on. The effects can be mitigated by using async mode for the producer and by sending larger batches, but this option will typically get you lower throughput.

-------------------------------------------------------------------------------------------------------------------------

### Configuring Producer Retries
There are two parts to handling errors in the producer: the errors that the producers handle automatically for you and the errors that you as the developer using the producer library must handle.

The producer can handle retriable errors that are returned by the broker for you. When the producer sends messages to a broker, the broker can return either a success or an error code. Those error codes belong to two categories—errors that can be resolved after retrying and errors that won’t be resolved. For example, if the broker returns the error code LEADER_NOT_AVAILABLE, the producer can try sending the message again—maybe a new broker was elected and the second attempt will succeed. This means that LEADER_NOT_AVAILABLE is a retriable error. On the other hand, if a broker returns an INVALID_CONFIG exception, trying the same message again will not change the configuration. This is an example of a nonretriable error.

***In general, if your goal is to never lose a message, your best approach is to configure the producer to keep trying to send the messages when it encounters a retriable error. Why? Because things like lack of leader or network connectivity issues often take a few seconds to resolve—and if you just let the producer keep trying until it succeeds, you don’t need to handle these issues yourself. I frequently get asked “how many times should I configure the producer to retry?” and the answer really depends on what you are planning on doing after the producer throws an exception that it retried N times and gave up. If your answer is “I’ll catch the exception and retry some more,” then you definitely need to set the number of retries higher and let the producer continue trying. You want to stop retrying when the answer is either “I’ll just drop the message; there’s no point to continue retrying” or “I’ll just write it somewhere else and handle it later.”

Note that Kafka’s cross-DC replication tool (MirrorMaker, which we’ll discuss in Chapter 8) is configured by default to retry endlessly (i.e., retries = MAX_INT)—because as a highly reliable replication tool, it should never just drop messages.

***Note that retrying to send a failed message often includes a small risk that both messages were successfully written to the broker, leading to duplicates. For example, if network issues prevented the broker acknowledgment from reaching the producer, but the message was successfully written and replicated, the producer will treat the lack of acknowledgment as a temporary network issue and will retry sending the message (since it can’t know that it was received). In that case, the broker will end up having the same message twice. Retries and careful error handling can guarantee that each message will be stored at least once, but in the current version of Apache Kafka (0.10.0), we can’t guarantee it will be stored exactly once. Many real-world applications add a unique identifier to each message to allow detecting duplicates and cleaning them when consuming the messages. Other applications make the messages idempotent—meaning that even if the same message is sent twice, it has no negative impact on correctness***. For example, the message “Account value is 110$” is idempotent, since sending it several times doesn’t change the result. The message “Add $10 to the account” is not idempotent, since it changes the result every time you send it.

-----------------------------------------------------------------------------------------------------------------------


### Additional Error Handling
Using the built-in producer retries is an easy way to correctly handle a large variety of errors without loss of messages, but as a developer, you must still be able to handle other types of errors. 

These include:

1) Nonretriable broker errors such as errors regarding message size, authorization errors, etc.

2) Errors that occur before the message was sent to the broker—for example, serialization errors

3) Errors that occur when the producer exhausted all retry attempts or when the available memory used by the producer is filled to the limit due to using all of it to store messages while retrying


In Chapter 3 we discussed how to write error handlers for both sync and async message-sending methods. The content of these error handlers is specific to the application and its goals—do you throw away “bad messages”? Log errors? Store these messages in a directory on the local disk? Trigger a callback to another application? These decisions are specific to your architecture. Just note that if all your error handler is doing is retrying to send the message, you are better off relying on the producer’s retry functionality.


------------------------------------------------------------------------------------------------------------------------


### Using Consumers in a Reliable System
Now that we have learned how to produce data while taking Kafka’s reliability guarantees into account, it is time to see how to consume data.

As we saw in the first part of this chapter, ***data is only available to consumers after it has been committed to Kafka—meaning it was written to all in-sync replicas. This means that consumers get data that is guaranteed to be consistent***. The only thing consumers are left to do is make sure they keep track of which messages they’ve read and which messages they haven’t. This is key to not losing messages while consuming them.


When reading data from a partition, a consumer is fetching a batch of events, checking the last offset in the batch, and then requesting another batch of events starting from the last offset received. This guarantees that a Kafka consumer will always get new data in correct order without missing any events.

***When a consumer stops, another consumer needs to know where to pick up the work—what was the last offset that the previous consumer processed before it stopped? The “other” consumer can even be the original one after a restart. It doesn’t really matter—some consumer is going to pick up consuming from that partition, and it needs to know in which offset to start. This is why consumers need to “commit” their offsets. For each partition it is consuming, the consumer stores its current location, so they or another consumer will know where to continue after a restart. The main way consumers can lose messages is when committing offsets for events they’ve read but didn’t completely process yet. This way, when another consumer picks up the work, it will skip those events and they will never get processed. This is why paying careful attention to when and how offsets get committed is critical.***


#### COMMITTED MESSAGES VERSUS COMMITED OFFSETS
This is different from a committed message, which, as discussed previously, is a message that was written to all in-sync replicas and is available to consumers. Committed offsets are offsets the consumer sent to Kafka to acknowledge that it received and processed all the messages in a partition up to this specific offset.

------------------------------------------------------------------------------------------------------------------

### Important Consumer Configuration Properties for Reliable Processing
There are four consumer configuration properties that are important to understand in order to configure your consumer for a desired reliability behavior.

The first is group.id, as explained in great detail in Chapter 4. The basic idea is that if two consumers have the same group ID and subscribe to the same topic, each will be assigned a subset of the partitions in the topic and will therefore only read a subset of the messages individually (but all the messages will be read by the group as a whole). If you need a consumer to see, on its own, every single message in the topics it is subscribed to—it will need a unique group.id.

The second relevant configuration is auto.offset.reset. This parameter controls what the consumer will do when no offsets were committed (e.g., when the consumer first starts) or when the consumer asks for offsets that don’t exist in the broker (Chapter 4 explains how this can happen). There are only two options here. If you choose earliest, the consumer will start from the beginning of the partition whenever it doesn’t have a valid offset. This can lead to the consumer processing a lot of messages twice, but it guarantees to minimize data loss. If you choose latest, the consumer will start at the end of the partition. This minimizes duplicate processing by the consumer but almost certainly leads to some messages getting missed by the consumer.

The third relevant configuration is enable.auto.commit. This is a big decision: are you going to let the consumer commit offsets for you based on schedule, or are you planning on committing offsets manually in your code? The main benefit of automatic offset commits is that it’s one less thing to worry about when implementing your consumers. If you do all the processing of consumed records within the consumer poll loop, then the automatic offset commit guarantees you will never commit an offset that you didn’t process. (If you are not sure what the consumer poll loop is, refer back to Chapter 4.) The main drawbacks of automatic offset commits is that you have no control over the number of duplicate records you may need to process (because your consumer stopped after processing some records but before the automated commit kicked in). If you do anything fancy like pass records to another thread to process in the background, the automatic commit may commit offsets for records the consumer has read but perhaps did not process yet.

The fourth relevant configuration is tied to the third, and is auto.commit.interval.ms. If you choose to commit offsets automatically, this configuration lets you configure how frequently they will be committed. The default is every five seconds. In general, committing more frequently adds some overhead but reduces the number of duplicates that can occur when a consumer stops.

------------------------------------------------------------------------------------------------------------------------






