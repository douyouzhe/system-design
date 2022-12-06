# Kafka
* Official definition: Kafka is an open-source **distributed** event streaming platform used by thousands of companies for high-performance data pipelines, streaming analytics, data integration, and mission-critical applications.
* Kafka adopts **Pub/Sub** model where the publishers publish messages of certain categories (topics) and the subscribers can consume according to their interests. 
* Kafka is **much more** than a Message Queue (MQ), compared to ActiveMQ and RabbitMQ....

## Major use cases:
* peak traffic buffering/handling: the traffic varies a lot during peak hours - lunchtime for DoorDash and PrimeDay for Amazon.
* services decoupling: Kafka publishers and subscribers are already decoupled in the sense that they do not communicate with one another directly. It is very useful when you have multiple data sources and streams. *Kakfa Mirror is a special technique used widely to move data from between different datalakes or datazones*
* asynchronous communications: make non-essential steps async for better user experience. 

## Pub/Sub Model
<p align="center">
    <img src="/tech-stacks/kafka/pubsub.jpg" width="500">
    </p>

* Messages/Events of different categories can be published into different Topics by different producers. 
* Messages are not removed after the consumption (Kafka manages the message lifecycle independently based on configs) since Kafka allows multiple consumers to consume from the same Topic.

### Producer
<p align="center">
    <img src="/tech-stacks/kafka/producer.jpg" width="850">
    </p>

* The Producer object calls `.send()` method to send the ProducerRecords which is just a wrapper around the messages.
* The serializer turns the messages into streams of bytes. Serializer can be customized and if you are working with binary like Protobuf or Avro, you can use Schema Registry.
* Partitioner decides where (which partition) the message should be sent to. Partitioning takes the single topic log and breaks it into multiple logs, each of which can live on a separate node in the Kafka cluster. This way, the work of storing messages, writing new messages, and processing existing messages can be split among many nodes in the cluster.
* After that, messages are split into different queues in RecordAccumulator and waiting to be sent over. Note that these queues are all in memory still. We can return the metadata as a successful response back to the Producer if you use `.send(ProducerRecords, Callback)` with a callback method. 
* When do we invoke the Sender to send the messages? This is controlled by two configuration parameters: `BATCH_SIZE_CONFIG` (when the accumulator queue exceeds the batch size) and `LINGER_MS_CONFIG` (when the longest wait time has been reached). 
* The Sender will create a network client to send the messages to Kafka Cluster and wait for an `acks` signal. 

### Consumer
* Kafka adopts **Pull** model where consumers pull data from the topic. This makes the system more decoupled and flexible compared to the Push model which requires the consumer to keep up with the speed of the ingestion.
* More than one individual consumer can pull data from the same topic, but more than one consumer from the same consumer group **cannot** pull data from the same topic since all consumers in a group are treated as a single consumer/subscriber. 

## Partitions & Replicas

<p align="center">
    <img src="/tech-stacks/kafka/partition.jpg" width="600">
    </p>

* To make Kafka *high-performance*, some *parallelism* is added. We can break a topic into several pieces (partitions) and put them on different servers (machines). 
    * DefaultPartitioner: hash(key)%num_of_partitions. 
    * CustomizedPartitioner: implement Partitioner interface and override method `int partition()`

* To make any system reliable and resilient, we use replications to make sure we do not have any single-point failures. Note that Kafka uses the Leader&Follower mechanism. We will cover more in the System Reliability section.

* Every partition has its corresponding append-only log that stores all messages from the Producer. Append-only logs will get big sometimes so they are further divided into Segments (1G). Kafka puts an index record in the `xxxxx.index` file for every 4kb of data for fast retrieval. To prevent logs getting large infinitely, we can set the retention time using `log.retention.hours/minutes/ms` (the default is 7 days). After the retention time, the data will be either deleted or compacted(only keep the latest value of the same key) depends on `log.cleanup.policy`. 


## System Reliability
### acks
The `acks` setting is a client (producer) configuration between the Producer and the Kafka Cluster. It denotes the number of brokers that must receive the record before we consider the write as successful. It supports three values — 0, 1, and -1 (all). It allows the user to have trade-offs between durability guarantees and performance.

* acks=0
    
    With a value of 0, the producer won’t even wait for a response from the broker. It immediately considers the write successful the moment the record is sent out.
* acks=1
    
    With a setting of 1, the producer will consider the writes successful when the leader receives the record. The leader broker will know to immediately respond the moment it receives the record and not wait any longer.

* acks=all

    When set to all, the producer will consider the write successful when all of the in-sync replicas (ISR)receive the record. This is achieved by the leader broker being smart as to when it responds to the request — it’ll send back a response once all the ISRs receive the record themselves. Note that if one ISR is down, the stream will be blocked. This is why Kafka manages ISRs dynamically and removes the unresponsive ISR. However, we have to use `min.insync.replicas=n` to make sure we still have n copies to proceed. 

Therefore, to ensure reliable sending, we need:

* acks = -1
* num_of_partitions > 1
* min.insync.replicas > 1

### Retries
Retries are very important since failure is evitable in any system. It (`RETRIES_CONFIG`) is set to `MAX_INT` for Kafka >= 2.1. Retries create problems of duplication at the same time if we do not manage them.

We can achieve `At-Least-Once` and `At-Most-Once` using `acks=-1` and `acks=0`. `Exactly-Once` is a bit tricky. We will need `At-Least-Once` plus some idempotent settings. Each producer assigns a unique Producer Id (PID) and always includes it every time it sends messages. In addition, each message gets a monotonically increasing sequence number. On the cluster side, on a per-partition basis, it keeps track of the largest PID-Sequence Number combination it has successfully written. So in short, we need <PID, sequence number> to ensure no duplicates are persisted **per partition**. Luckily we do not need to implement this and Kafka supports it via `enable.idempotence=boolean`. 

Another way is using TransactionManager, similar to SQL databases. You can enclose your `.send()` method with `beginTransaction()` and `commitTransaction()`.

### Replicas
* There will be more than one replica (followers) that are in sync with the leader to make the system more reliable. More replicas will result in lower efficiency though (if acks=all) since it takes time to sync.
* Producers only send messages to the leader and followers will fetch data from the leader. 
* ISR(in-sync) is the list of followers that are in sync with the leader. e.g.[0,1,3]. If a node is unresponsive for a while (>replica.lag.time.max.ms), it will be kicked out of ISR to the OSR (out-sync). If the leader itself is unresponsive, a new leader will be elected based on ISR and AR list from ZooKeeper.
* OSR + ISR = Assigned Replicas (AR)
* Each replica maintains an LEO(Log End Offset) number which points to the latest data synced so far. If this replica went offline and came back online, it will also catch up from the LOE to be added back into ISR. 

### High-Speed Read/Write
* The nature of cluster computing enables parallel processing (for both Producers and Consumers) which is more efficient.
* All records are Sparsely Indexed (using .index files)which makes read operation fast.
* The append-only log only requires sequential writes, which is faster compared to random access. 
* Kafka depends on the Operating system's **Page Cache**. Page cache is like files/data written to Disk, its index/meta cached on RAM by the Operating system. Kafka takes the advantage of this. Kafka internal code gets the message from the Producers, then it writes to memory (page cache) and then it writes to disk. Then when there is a read operation, memory will be scanned first, similar to any other caching layers. 
* To read a file from a disk and send it over the network, a traditional data transfer would require four context switches between user mode and kernel mode, making the data be copied four times. If the requested data is larger than the kernel buffer size, there will be even more copies between kernel and user spaces. Zero-copy optimization reduces these redundant data copies. A zero-copy optimization makes the data transferred directly from the read buffer to the socket buffer. In short, we do not need to send the data to the Application Context (Kafka) at all. *Note that since SSL allows us to encrypt data in flight, we no longer send over the same data that is stored on the Broker. This means that when SSL is enabled, zero-copy optimization will be lost since the Broker needs to decrypt and encrypt the data.*

<p align="center">
    <img src="/tech-stacks/kafka/zerocopy.jpg" width="450">
    </p>


## ZooKeeper
ZooKeeper is used in distributed systems for service synchronization and as a naming registry.  When working with Apache Kafka, ZooKeeper is primarily used to track the status of nodes in the Kafka cluster and maintain a list of Kafka topics and messages. 

ZooKeeper has five primary functions:
* Controller Election: picks the first broker in Assigned Replicas (AR) which must be in ISR as well.

    `/kafka/controller`
* Cluster Membership: all alive brokers

    `/kafka/brokers/ids`
* Topic Configuration: full topic information

    `/kafka/brokers/topics/{topic_name}/partitions/{partition_id}/state`
* Access Control Lists
* Quotas: 
    
    ZooKeeper accesses how much data each client is allowed to read/write.



## CLI Example
mostly just CRUD operations using corresponding `.sh` files
* Topic
    ```
    kafka-topics \
   --bootstrap-server `grep "^\s*bootstrap.server" ./test.config | tail -1` \
   --command-config ./test.config \
   --topic test1 \
   --create \
   --replication-factor 3 \
   --partitions 2
    ```

    test.config
    ```
    # keep it simle for now, more configs can be added
    bootstrap.servers=localhost:9092
    ```
* Producer
    ```
    kafka-console-producer \
   --topic test1 \
   --broker-list `grep "^\s*bootstrap.server" ./test.config | tail -1` \
   --property parse.key=true \
   --property key.separator=: \
   --producer.config ./test.config```

* send messages
    ```
    > key:{"val":0}
    > key:{"val":1}
    ```
* Consumer
    ```
    kafka-console-consumer \
   --topic test1 \
   --bootstrap-server `grep "^\s*bootstrap.server" ./test.config | tail -1` \
   --property print.key=true \
   --from-beginning \
   --consumer.config ./test.config
    ```
    you should be able to see the same messages you produced.








## Reference
https://docs.confluent.io/platform/current/tutorials/examples/clients/docs/kafka-commands.html#produce-records

https://dattell.com/data-architecture-blog/what-is-zookeeper-how-does-it-support-kafka/

https://medium.com/@shesh.soft/kafka-idempotent-producer-and-consumer-25c52402ceb9

https://betterprogramming.pub/kafka-acks-explained-c0515b3b707e


https://www.linkedin.com/pulse/kafka-less-explored-facts-part-1-bikash-kumar-sundaray/?trk=pulse-article


https://andriymz.github.io/kafka/kafka-disk-write-performance/#

