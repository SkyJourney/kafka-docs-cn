# 生产者API Producer API

生产者`API`允许应用程序发送数据流到Kafka集群的主题中。

如何使用生产者的示例可以查看[javadocs](http://kafka.apache.org/24/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)。

要使用生产者，可以使用以下`Maven`依赖项：

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.4.0</version>
</dependency>
```

## 附：KafkaProducer javadocs示例翻译

```java
public class KafkaProducer<K,V>
    extends Object
    implements Producer<K,V>
```

用来发布数据记录到`Kafka`集群的客户端。

生产者是线程安全的，跨线程共享单个生产者实例通常比使用多个实例要快。

下面是使用生产者发送字符串的数据记录，其包含顺序数字作为键值对的序号。

```java
 Properties props = new Properties();
 props.put("bootstrap.servers", "localhost:9092");
 props.put("acks", "all");
 props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
 props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

 Producer<String, String> producer = new KafkaProducer<>(props);
 for (int i = 0; i < 100; i++)
     producer.send(new ProducerRecord<String, String>("my-topic", Integer.toString(i), Integer.toString(i)));

 producer.close();
 ```

生产者由一个用来缓存尚未被发送到服务器的数据记录的缓存空间池和一个负责将这些记录转换为请求并将他们传输给集群的后台`I/O`线程组成。若不能在使用后正常关闭生产者，则这些资源会被泄漏。

`send()`方法是异步的。当它被调用时会添加数据记录到待发送记录缓冲并立即返回。这可以使生产者将独立的数据记录合并到一起批量处理以提高效率。

`acks`配置项用来控制确认请求是否完成的条件。我们这里设定为`all`，会使得数据记录的完全提交阻塞，这是最慢但是最持久的设定。

如果请求失败，生产者可以自动重试，当然如果当我们指定重试次数为`0`则不会。开启重试机制也会添加数据重复的可能性（有关详细信息请参阅[邮件传递语义](./../design/message-delivery-semantics.md)章节）。

生产者为每个分区维护未发送数据记录的缓冲区。这些缓冲区的大小由`batch.size`配置项指定。扩大缓冲区的大小可以进行更多的批处理，但同时也会需要更多的内存（因为我们通常会为每个活动的分区从这些缓冲区中分配一个）

默认情况下，即使缓冲区中还有未使用的空间也可以立即发送。然而如果要减少请求数量，就需要设定`linger.ms`大于`0`。该设定表示生产者在发送请求之前等待的毫秒数，以期望有更多记录来填充缓冲区。这种机制跟`TCP`协议中的`Nagie`算法类似。例如在前面的代码段中，由于设置延迟时间为`1`毫秒，所以很可能所有`100`条记录都在同一个请求中发送。然而如果我们没有填满缓冲区，这个设定将为请求增加`1`毫秒的延迟以等待更多的记录送来。注意即使设定`linger.ms=0`，送达时间接近的记录通常也会一起批量处理，因此在高负载下是否进行批处理与延迟设定无关；但是如果设定延迟大于`0`毫秒，则在最大负载情况下会导致更少但是更有效的请求，以少量的延迟为代价。

`buffer.memory`配置项用来控制生产者可用于缓冲区的内存总量。如果数据记录发到生产者的速度超过了记录发送给服务器的速度，则缓冲区空间将会被耗尽，此时后续对生产者的发送调用会被阻塞。阻塞时间的阈值由`max.block.ms`设定。若超过此时间则抛出`TimeoutException`异常。



The key.serializer and value.serializer instruct how to turn the key and value objects the user provides with their ProducerRecord into bytes. You can use the included ByteArraySerializer or StringSerializer for simple string or byte types.

From Kafka 0.11, the KafkaProducer supports two additional modes: the idempotent producer and the transactional producer. The idempotent producer strengthens Kafka's delivery semantics from at least once to exactly once delivery. In particular producer retries will no longer introduce duplicates. The transactional producer allows an application to send messages to multiple partitions (and topics!) atomically.

To enable idempotence, the enable.idempotence configuration must be set to true. If set, the retries config will default to Integer.MAX_VALUE and the acks config will default to all. There are no API changes for the idempotent producer, so existing applications will not need to be modified to take advantage of this feature.

To take advantage of the idempotent producer, it is imperative to avoid application level re-sends since these cannot be de-duplicated. As such, if an application enables idempotence, it is recommended to leave the retries config unset, as it will be defaulted to Integer.MAX_VALUE. Additionally, if a send(ProducerRecord) returns an error even with infinite retries (for instance if the message expires in the buffer before being sent), then it is recommended to shut down the producer and check the contents of the last produced message to ensure that it is not duplicated. Finally, the producer can only guarantee idempotence for messages sent within a single session.

To use the transactional producer and the attendant APIs, you must set the transactional.id configuration property. If the transactional.id is set, idempotence is automatically enabled along with the producer configs which idempotence depends on. Further, topics which are included in transactions should be configured for durability. In particular, the replication.factor should be at least 3, and the min.insync.replicas for these topics should be set to 2. Finally, in order for transactional guarantees to be realized from end-to-end, the consumers must be configured to read only committed messages as well.

The purpose of the transactional.id is to enable transaction recovery across multiple sessions of a single producer instance. It would typically be derived from the shard identifier in a partitioned, stateful, application. As such, it should be unique to each producer instance running within a partitioned application.

All the new transactional APIs are blocking and will throw exceptions on failure. The example below illustrates how the new APIs are meant to be used. It is similar to the example above, except that all 100 messages are part of a single transaction.

```java
 Properties props = new Properties();
 props.put("bootstrap.servers", "localhost:9092");
 props.put("transactional.id", "my-transactional-id");
 Producer<String, String> producer = new KafkaProducer<>(props, new StringSerializer(), new StringSerializer());

 producer.initTransactions();

 try {
     producer.beginTransaction();
     for (int i = 0; i < 100; i++)
         producer.send(new ProducerRecord<>("my-topic", Integer.toString(i), Integer.toString(i)));
     producer.commitTransaction();
 } catch (ProducerFencedException | OutOfOrderSequenceException | AuthorizationException e) {
     // We can't recover from these exceptions, so our only option is to close the producer and exit.
     producer.close();
 } catch (KafkaException e) {
     // For all other exceptions, just abort the transaction and try again.
     producer.abortTransaction();
 }
 producer.close();
```
As is hinted at in the example, there can be only one open transaction per producer. All messages sent between the beginTransaction() and commitTransaction() calls will be part of a single transaction. When the transactional.id is specified, all messages sent by the producer must be part of a transaction.

The transactional producer uses exceptions to communicate error states. In particular, it is not required to specify callbacks for producer.send() or to call .get() on the returned Future: a KafkaException would be thrown if any of the producer.send() or transactional calls hit an irrecoverable error during a transaction. See the send(ProducerRecord) documentation for more details about detecting errors from a transactional send.

By calling producer.abortTransaction() upon receiving a KafkaException we can ensure that any successful writes are marked as aborted, hence keeping the transactional guarantees.
This client can communicate with brokers that are version 0.10.0 or newer. Older or newer brokers may not support certain client features. For instance, the transactional APIs need broker versions 0.11.0 or later. You will receive an UnsupportedVersionException when invoking an API that is not available in the running broker version.

