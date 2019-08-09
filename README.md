# kafka-learning

#### kafka command cheat sheat https://gist.github.com/ursuad/e5b8542024a15e4db601f34906b30bb5

https://kafka.apache.org/documentation/

https://learning.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch06.html

https://www.confluent.io/blog/event-streaming-platform-2

https://cwiki.apache.org/confluence/display/KAFKA/KIP-186%3A+Increase+offsets+retention+default+to+7+days

https://kafka.apache.org/23/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html



-------------------------------------------------------------------------------------------------------------
#### consumer settings doubts

https://stackoverflow.com/questions/39730126/difference-between-session-timeout-ms-and-max-poll-interval-ms-for-kafka-0-10-0

https://stackoverflow.com/questions/19588090/kafka-long-polling

https://kafka.apache.org/21/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html




### Detecting Consumer Failures
After subscribing to a set of topics, the consumer will automatically join the group when poll(Duration) is invoked. The poll API is designed to ensure consumer liveness. As long as you continue to call poll, the consumer will stay in the group and continue to receive messages from the partitions it was assigned. Underneath the covers, the consumer sends periodic heartbeats to the server. If the consumer crashes or is unable to send heartbeats for a duration of session.timeout.ms, then the consumer will be considered dead and its partitions will be reassigned.
It is also possible that the consumer could encounter a "livelock" situation where it is continuing to send heartbeats, but no progress is being made. To prevent the consumer from holding onto its partitions indefinitely in this case, we provide a liveness detection mechanism using the max.poll.interval.ms setting. Basically if you don't call poll at least as frequently as the configured max interval, then the client will proactively leave the group so that another consumer can take over its partitions. When this happens, you may see an offset commit failure (as indicated by a CommitFailedException thrown from a call to commitSync()). This is a safety mechanism which guarantees that only active members of the group are able to commit offsets. So to stay in the group, you must continue to call poll.

The consumer provides two configuration settings to control the behavior of the poll loop:

### max.poll.interval.ms: 
By increasing the interval between expected polls, you can give the consumer more time to handle a batch of records returned from poll(Duration). The drawback is that increasing this value may delay a group rebalance since the consumer will only join the rebalance inside the call to poll. You can use this setting to bound the time to finish a rebalance, but you risk slower progress if the consumer cannot actually call poll often enough.

### max.poll.records: 
Use this setting to limit the total records returned from a single call to poll. This can make it easier to predict the maximum that must be handled within each poll interval. By tuning this value, you may be able to reduce the poll interval, which will reduce the impact of group rebalancing.

For use cases where message processing time varies unpredictably, neither of these options may be sufficient. The recommended way to handle these cases is to move message processing to another thread, which allows the consumer to continue calling poll while the processor is still working. Some care must be taken to ensure that committed offsets do not get ahead of the actual position. Typically, you must disable automatic commits and manually commit processed offsets for records only after the thread has finished handling them (depending on the delivery semantics you need). Note also that you will need to pause the partition so that no new records are received from poll until after thread has finished handling those previously returned.

--------------------------------------------------------------------------------------------------------------------
