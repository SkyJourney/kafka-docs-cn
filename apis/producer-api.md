# 生产者 API Producer API

生产者`API`允许应用程序发送数据流到Kafka集群的主题中。

如何使用生产者的示例可以查看[javadocs](http://kafka.apache.org/24/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)。

要使用生产者，可以使用以下`Maven`依赖项：

```markup
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.4.0</version>
</dependency>
```

## 附：KafkaProducer javadocs示例翻译 <a id="kafkaproducer-javadocs"></a>

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

如果请求失败，生产者可以自动重试，当然如果当我们指定`retries`为`0`则不会。开启重试机制也会添加数据重复的可能性（有关详细信息请参阅[消息交付语义](../design/message-delivery-semantics.md)章节）。

生产者为每个分区维护未发送数据记录的缓冲区。这些缓冲区的大小由`batch.size`配置项指定。扩大缓冲区的大小可以进行更多的批处理，但同时也会需要更多的内存（因为我们通常会为每个活动的分区从这些缓冲区中分配一个）

默认情况下，即使缓冲区中还有未使用的空间也可以立即发送。然而如果要减少请求数量，就需要设定`linger.ms`大于`0`。该设定表示生产者在发送请求之前等待的毫秒数，以期望有更多记录来填充缓冲区。这种机制跟`TCP`协议中的`Nagie`算法类似。例如在前面的代码段中，由于设置延迟时间为`1`毫秒，所以很可能所有`100`条记录都在同一个请求中发送。然而如果我们没有填满缓冲区，这个设定将为请求增加`1`毫秒的延迟以等待更多的记录送来。注意即使设定`linger.ms=0`，送达时间接近的记录通常也会一起批量处理，因此在高负载下是否进行批处理与延迟设定无关；但是如果设定延迟大于`0`毫秒，则在最大负载情况下会导致更少但是更有效的请求，以少量的延迟为代价。

`buffer.memory`配置项用来控制生产者可用于缓冲区的内存总量。如果数据记录发到生产者的速度超过了记录发送给服务器的速度，则缓冲区空间将会被耗尽，此时后续对生产者的发送调用会被阻塞。阻塞时间的阈值由`max.block.ms`设定。若超过此时间则抛出`TimeoutException`异常。

`key.serializer`和`value.serializer`指定如何将用户通过其ProducerRecord提供的键和值对象序列化为字节数据。您可以将随附的`ByteArraySerializer`或`StringSerializer`用于简单的字节数组或字符串类型。

从`Kafka 0.11`开始，`KafkaProducer`支持两种附加模式：幂等生产者和事务生产者。幂等生产者增强了`Kafka`的交付语义从至少一次交付到精确地一次交付。特别是生产者重试机制将不再引入重复项。事务生产者运行应用程序原子性地发送消息到多个分区（和主题）。

要启用幂等模式，`enable.idempotence`不行设置为`true`。一旦设定，`retries`设定会默认为`Integer.MAX_VALUE`，`acks`设定默认为`all`。对幂等模式不会有任何`API`的改变，所以已有的应用程序无需做任何改变就可以获得该功能的特性。

要利用幂等生产者的优势，米边应用级别的重新发送是必须的，因为这些重复发送无法被去重删除。因此，如果一个应用程序开启了幂等模式，推荐不要设置`retries`，因为它默认会被设定为`Integer.MAX_VALUE`。此外，如果`send（ProducerRecord）`方法即使无线重试（例如，如果消息在发送之前在缓冲区中过期）依然返回错误，则建议关闭生产者并检查最后生产的消息以确保其没有重复。最后，生产者只能保证在单个回话中发送的消息具有幂等性。

要使用事务生产者和其附带的`APIs`，你不许设置`transactional.id`配置项。一旦设定了`transactional.id`，幂等模式会被自动开启且幂等所以来的生产者设置会被自动设定。此外，事务中包含的主题应该被设定为持久性。特别是`replication.factor`应该被设定至少为`3`，并且这些主题的`min.insync.replicas`应该设定为`2`。最后，为了实现端对端的事务保证，消费者同样必须被设定为只读取已经提交的消息。

`transactional.id`配置是为了开启单个生产者在多个会话之间的事务恢复。它通常从分区的有状态的应用程序的分片标识符中派生。这样，它对于分区的应用程序中运行的每个生产者实例都应该是唯一的。

所有新的事务`API`都是阻塞的并且在失败时抛出异常。下面的实例会说明如何使用新的`API`。除了所有`100`条消息都是单个事务的一部分之外，它与上面的示例相似。

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

如示例所示，每个生产者只能有一个打开的事务。所有在`beginTransaction()`和`commitTransaction()`调用之间发送的消息发将成为一个单个的事务。当`transactional.id`被指定时，生产者发送的所有消息都必须是事务的一部分。

事务生产者使用异常来传达错误状态。特别是不需要为`producer.send()`指定回调或在返回的`Future`对象上调用`.get()`：如果在事务过程中任何`producer.send()`或事务调用遇到不可恢复的错误，都将抛出`KafkaException`。有关从事务发送中检测错误的更多详细信息，请参见[`send(ProducerRecord)`](http://kafka.apache.org/24/javadoc/org/apache/kafka/clients/producer/KafkaProducer.html#send-org.apache.kafka.clients.producer.ProducerRecord-)文档。

通过在接收到`KafkaException`之后调用`producer.abortTransaction()`，我们可以确保将任何成功的写操作都标记为已中止，从而保证事务完整性。 该客户端可以与`0.10.0`或更高版本的代理进行通信。较旧或较新的代理可能不支持某些客户端功能。例如，事务性`API`需要代理版本`0.11.0`或更高版本。 当调用正在运行的代理版本中不可用的`API`时，您将收到`UnsupportedVersionException`。

