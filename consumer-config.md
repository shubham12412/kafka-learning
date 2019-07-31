https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch04.html

FETCH.MIN.BYTES \
FETCH.MAX.WAIT.MS \
MAX.PARTITION.FETCH.BYTES \

As you recall, the consumer must call poll() frequently enough to avoid session timeout and subsequent rebalance. If the amount of data a single poll() returns is very large, it may take the consumer longer to process, which means it will not get to the next iteration of the poll loop in time to avoid a session timeout. If this occurs, the two options are either to lower max. partition.fetch.bytes or to increase the session timeout.

SESSION.TIMEOUT.MS \
heartbeat.interval.ms. heartbeat.interval.ms \


### SESSION.TIMEOUT.MS
The amount of time a consumer can be out of contact with the brokers while still considered alive defaults to 3 seconds. If more than session.timeout.ms passes without the consumer sending a heartbeat to the group coordinator, it is considered dead and the group coordinator will trigger a rebalance of the consumer group to allocate partitions from the dead consumer to the other consumers in the group. This property is closely related to heartbeat.interval.ms. heartbeat.interval.ms controls how frequently the KafkaConsumer poll() method will send a heartbeat to the group coordinator, whereas session.timeout.ms controls how long a consumer can go without sending a heartbeat. Therefore, those two properties are typically modified together—heartbeat.interval.ms must be lower than session.timeout.ms, and is usually set to one-third of the timeout value. So if session.timeout.ms is 3 seconds, heartbeat.interval.ms should be 1 second. Setting session.timeout.ms lower than the default will allow consumer groups to detect and recover from failure sooner, but may also cause unwanted rebalances as a result of consumers taking longer to complete the poll loop or garbage collection. Setting session.timeout.ms higher will reduce the chance of accidental rebalance, but also means it will take longer to detect a real failure.


 

### AUTO.OFFSET.RESET
This property controls the behavior of the consumer when it starts reading a partition for which it doesn’t have a committed offset or if the committed offset it has is invalid (usually because the consumer was down for so long that the record with that offset was already aged out of the broker). The default is “latest,” which means that lacking a valid offset, the consumer will start reading from the newest records (records that were written after the consumer started running). The alternative is “earliest,” which means that lacking a valid offset, the consumer will read all the data in the partition, starting from the very beginning.

### ENABLE.AUTO.COMMIT
We’ll discuss the different options for committing offsets later in this chapter. This parameter controls whether the consumer will commit offsets automatically, and defaults to true. Set it to false if you prefer to control when offsets are committed, which is necessary to minimize duplicates and avoid missing data. If you set enable.auto.commit to true, then you might also want to control how frequently offsets will be committed using auto.commit.interval.ms

### PARTITION.ASSIGNMENT.STRATEGY
We learned that partitions are assigned to consumers in a consumer group. A PartitionAssignor is a class that, given consumers and topics they subscribed to, decides which partitions will be assigned to which consumer. By default, Kafka has two assignment strategies:

### Range
Assigns to each consumer a consecutive subset of partitions from each topic it subscribes to. So if consumers C1 and C2 are subscribed to two topics, T1 and T2, and each of the topics has three partitions, then C1 will be assigned partitions 0 and 1 from topics T1 and T2, while C2 will be assigned partition 2 from those topics. Because each topic has an uneven number of partitions and the assignment is done for each topic independently, the first consumer ends up with more partitions than the second. This happens whenever Range assignment is used and the number of consumers does not divide the number of partitions in each topic neatly.

### RoundRobin
Takes all the partitions from all subscribed topics and assigns them to consumers sequentially, one by one. If C1 and C2 described previously used RoundRobin assignment, C1 would have partitions 0 and 2 from topic T1 and partition 1 from topic T2. C2 would have partition 1 from topic T1 and partitions 0 and 2 from topic T2. In general, if all consumers are subscribed to the same topics (a very common scenario), RoundRobin assignment will end up with all consumers having the same number of partitions (or at most 1 partition difference).

The partition.assignment.strategy allows you to choose a partition-assignment strategy. The default is org.apache.kafka.clients.consumer.RangeAssignor, which implements the Range strategy described above. You can replace it with org.apache.kafka.clients.consumer.RoundRobinAssignor. A more advanced option is to implement your own assignment strategy, in which case partition.assignment.strategy should point to the name of your class.

CLIENT.ID \


### MAX.POLL.RECORDS
This controls the maximum number of records that a single call to poll() will return. This is useful to help control the amount of data your application will need to process in the polling loop.


-------------------------------------------------------------------------------------------------------------------------

### Commits and Offsets
Whenever we call poll(), it returns records written to Kafka that consumers in our group have not read yet. This means that we have a way of tracking which records were read by a consumer of the group. As discussed before, one of Kafka’s unique characteristics is that it does not track acknowledgments from consumers the way many JMS queues do. Instead, it allows consumers to use Kafka to track their position (offset) in each partition.

We call the action of updating the current position in the partition a commit.

How does a consumer commit an offset? It produces a message to Kafka, to a special __consumer_offsets topic, with the committed offset for each partition. As long as all your consumers are up, running, and churning away, this will have no impact. However, if a consumer crashes or a new consumer joins the consumer group, this will trigger a rebalance. After a rebalance, each consumer may be assigned a new set of partitions than the one it processed before. In order to know where to pick up the work, the consumer will read the latest committed offset of each partition and continue from there.

