# 简介 Introduction

`Apache Kafka®` 是一个分布式流平台。这到底是什么意思呢？

 流平台具有三个关键功能：

* 发布和订阅记录数据记录流，类似于消息队列或企业消息传递系统
* 以持久的且具有容错能力的方式存储数据记录流
* 处理数据记录流

`Kafka`通常用于两大类应用程序：

* 建立实时的流数据管道，使系统或应用程序之间能够可靠地获取数据
* 建立实时的流应用程序，可以转换或响应数据流

为了理解`Kafka`如何完成这些事，让我们从头开始深入探索`Kafka`的功能。

首先了解几个概念：

* `Kafka`是在一个或多个可以跨越多个数据中心的服务器上以集群方式运行
* `Kafka`集群以`topics`归类储存的数据记录流
* 每条数据记录由一个键、一个值和一个时间戳构成

`Kafka`有四大核心`APIs`：

![1-1 核心APIs示意图](../.gitbook/assets/kafka-apis.png)


* [`Producer API`][producer]（生产者）允许应用程序发布数据记录流到一个或多个`topics`中
* [`Consumer API`][consumer]（消费者）允许应用程序订阅一个或多个`topics`并处理其生成的数据记录流
* [`Streams API`][streams]（流处理）允许应用程序作为流处理器，消费从一个或多个`topics`中获取的输入流并生产输出流到一个或多个`toppics`中，有效地进行输入流和输出流的转换
* [`Connector API`][connector]（连接器）允许建立和允许可复用的生产者和消费者，用来连接`topics`和已存在的应用程序或数据系统。例如关系型数据库的连接器可能会捕捉对表的所有更改

在`Kafka`中，服务端和客户端之间的通信是通过简单、高性能、无关编程语言的`TCP`协议来完成。该协议已经版本化，并维持对旧版本的向下兼容。我们提供`Java`版本的`Kafka`客户端，但客户端也同时支持[多种语言][multi-languages]。

## 主题和日志 Topics and Logs <a id="topics-n-logs"></a>

首先深入了解一个`Kafka`提供的数据记录流的核心抽象——**主题**(**`topic`**)

**主题**是数据记录发布的分类名称或订阅源名称。Kafka中的**主题**总是面向多订阅者的；也就是说一个主题可以拥有`0`个、`1`个或许多消费者来订阅写入该主题的数据。

对于每个主题，`Kafka`集群都会维护一个分区日志，其结构如下图所示：

![1-2 主题分区日志解析图](../.gitbook/assets/log_anatomy.png)

每个分区(**`partition`**)都是有序、不可变更的数据记录序列，数据记录可以持续地追加到结构化的提交日志中。分区中的数据记录都会被分配一个顺序生成的ID号，被称为偏移量(**`offset`**)，是每条数据记录在分区中的唯一识别标记。

`Kafka`集群会持久地保存所有发布的数据记录，不论其是否被消费使用，其保留期限是可以被配置的。例如如果保留策略设置为两天，那么一条数据记录发布后的两天之内是可以被消费的，之后会被丢弃以释放空间。`Kafka`的性能在不同数据大小情况下几乎是恒定的，因此长时间储存数据是没有问题的。

![1-3 分区日志示意图](../.gitbook/assets/log_consumer.png)

事实上，每个消费者底层保留的唯一元数据是其在日志中的偏移量或位置。此偏移量是可以被消费者控制的：通常来说，消费者读取数据记录时会线性的向前移动其偏移量；但事实上，由于位置可以被消费者控制，那么就可以以想要的任意顺序消费这些数据记录。例如消费者可以重置到较早的偏移量以重新处理过往的数据，也可以跳到最新的数据记录位置并从“现在”开始消费记录。

这些功能的组合意味着`Kafka`消费者是非常轻便的，他们来去自如并且不会对集群或其他消费者产生太多影响。若你使用我们的命令行工具来监控任何topic的尾部内容，你并不会看到其他消费者消费数据时产生的变更。

日志的分区提供多种用途。首先，分区允许日志扩展到超出单个服务器所能容纳的大小。每个单独的分区必须适合承载它的服务器，但是一个主题可能有很多分区，因此它可以处理任意数量的数据。其次，分区可以作为并行的处理单元，more on that in a bit（不知道怎么翻）。

## 分布 distribution <a id="distribution"></a>

日志的分区分布在`Kafka`集群的服务器上，每个服务器处理数据并请求共享分区。每个分区都可以跨服务器复制，以实现容错功能。复制服务器的数量是可以配置的。

每个分区都有一个服务器作为`leader`（领导者），`0`个或多个服务器作为`followers`（追随者）。`leader`处理对分区的所有的读写请求，`followers`被动的复制`leader`。当`leader`服务器发生错误，`followers`中的一个会自动成为新的`leader`。每个服务器都可以作为一些分区的`leader`和其他分区的`followers`，因此集群的负载可以得到很好的平衡。

## 地理复制 Geo-Replication <a id="geo-replication"></a>

`Kafka MirrorMaker`为集群提供地理复制的支持。使用`MirrorMaker`，可以在多个数据中心或云区域中复制消息。您可以在主动/被动方案中使用它进行备份和恢复。或在主动/主动方案中将数据放置在离您的用户更近的位置，或支持数据位置要求。

## 生产者 Producers <a id="producers"></a>

生产者将数据发布到他们选择的主题。生产者负责选择将哪个数据记录分配给主题中的哪个分区。如果仅是为了平衡负载，可以以循环方式完成此操作；也可以根据某些语义分区功能（例如基于数据记录中的某些键）进行此操作。

## 消费者 Consumers <a id="consumers"></a>

消费者使用消费者组(`consumer group`)名来标记自己，发布到主题的每条数据记录都会被传递到每个消费者组中的一个实例。消费者实例可以在不同的进程中也可以在不同的机器中。

如果所有的实例标记为相同的消费者组，那么数据记录的传递会在这些消费者实例中有效地平衡负载。

如果所有的实例标记为不同的消费者组，那么每条数据记录都会被广播到所有的消费者进程中。

![1-4 分布式消费者组示意图](../.gitbook/assets/consumer-groups.png)

如图`1-4`一个双服务器的`Kafka`集群托管了含有两个消费者组的四个分区(`P0`-`P3`)。消费者组`A`含有两个消费者实例，消费者组`B`含有四个消费者实例。

然而更常见的是，我们发现主题只有少量的消费者组作为逻辑上的订阅者，每组都由许多消费者实例组成，以实现可伸缩性和容错能力。这无非就是发布-订阅(`publish-subscribe`)语义，其中订阅者是消费者的集群而不是单个进程。

在`Kafka`中实现消费的方式是通过在消费者实例上划分日志中的分区，以使得每个消费者实例在任何时间点都是分区的“公平份额”的独占消费者。`Kafka`协议动态处理了维护组成员身份的过程。如果新实例加入该组，它们将接管该组其他成员的某些分区；如果实例死亡，其负责的分区会被重新分配给其余实例。

`Kafka`仅提供分区中数据记录的总顺序，并不提供主题中不同分区之间的数据记录顺序。对大多数应用程序来说，按分区排序加上以键对数据进行分区的能力就已经足够了。然而如果你需要所有数据记录的总顺序，则可以使用只有一个分区的主题来实现，尽管这意味着每个消费者组只能有一个消费者实例。

## 多租户 Multi-tenancy <a id="multi-tenancy"></a>

你能够用多租户方案来部署`Kafka`。通过配置哪些主题能生产或消费数据来开启多租户模式。配额也会有运营支持。管理员可以在请求上定义和实施配额以控制客户端使用的代理资源。更多信息可以查看[安全][security]章节。

## 保证 Guarantees <a id="guarantees"></a>

在较高级别上，`Kafka`可以提供如下保证：

* 由生产者发往特定主题分区的消息会按照其发送顺序添加。这意味着，如果记录`M1`是由于记录M2相同的生产者发送的，并且`M1`先被发送，那么`M1`将获得比`M2`更低的偏移量，并且在日志中也会更早出现。
* 消费者实例会按照储存在日志中的顺序查看数据记录。
* 对具有复制因子`N`的主题，我们最多可以容忍在`N-1`个服务器故障时仍然不会丢失任何提交给日志的数据记录。

更多信息可以查看[设计][design]章节。

## 作为消息传递系统 Kafka as a Messaging System <a id="messaging-system"></a>

`Kafka`的流概念与传统的企业消息传递系统相比如何？

传统的消息传递有两种模型：队列和发布-订阅。在队列模型中，消费者池可以从服务器读取内容，并且每条记录都会传递到池中的一个消费者；在发布-订阅模型中，记录会被广播给所有的消费者。两种模型都有各自的优缺点。队列的优势在于它允许将数据的处理划分到多个消费者实例中，从而扩展处理量。不幸的是队列并不是多订阅者的模型，一旦进程读取了数据，数据就丢失了。发布-订阅可以广播数据到多个进程，但是因为每条消息都会发送给每个订阅者，所以无法做扩展处理。

`Kafka`中**消费者组**的概念汇总了这两个概念。与队列一样，消费者组允许将处理划分为一组进程（消费者组的成员）。与发布-订阅一样，`Kafka`允许广播消息到多个消费者组。

`Kafka`模型的优势在于每个主题兼具两种属性——可以扩展处理范围，也可以是多订阅者模式——不用必须在两者中选择其一。

与传统的消息传递系统相比，`Kafka`还具有更强的订购保证。

传统队列将数据记录按顺序保留在服务器上，如果有多个消费者从队列中消费，服务器将按照储存的顺序分发数据记录。然而尽管服务器按顺序分发记录，但是这些记录是异步的传递给消费者的，所以他们送达到不同消费者手中的顺序是错乱的。这实际上意味着在并行消费的情况下会丢失记录的顺序。消息传递系统通常通过“独占使用者”的概念来解决此问题，即仅允许一个进程从队列中消费，但是，这当然意味着在处理中没有并行性。

`Kafka`做得更好。通过主题中具有并行性（即分区）的概念，`Kafka`能够在消费者进程池中提供顺序保证和负载均衡。这是通过将主题中的分区分配给消费者组中的消费者来实现的，这使得每个分区会被组中的某个特定消费者完全消费。通过这样的方式确保了消费者仅仅是该分区的唯一读取器，并按照顺序消费数据。由于存在很多分区，因此这种做法也可以平衡许多消费者实例上的负载。但请注意，消费者组中的实例不能超过分区。

## 作为储存系统 Kafka as a Storage System <a id="storage-system"></a>

Any message queue that allows publishing messages decoupled from consuming them is effectively acting as a storage system for the in-flight messages. What is different about Kafka is that it is a very good storage system.

Data written to Kafka is written to disk and replicated for fault-tolerance. Kafka allows producers to wait on acknowledgement so that a write isn't considered complete until it is fully replicated and guaranteed to persist even if the server written to fails.

The disk structures Kafka uses scale well—Kafka will perform the same whether you have 50 KB or 50 TB of persistent data on the server.

As a result of taking storage seriously and allowing the clients to control their read position, you can think of Kafka as a kind of special purpose distributed filesystem dedicated to high-performance, low-latency commit log storage, replication, and propagation.

For details about the Kafka's commit log storage and replication design, please read this page.

Kafka for Stream Processing
It isn't enough to just read, write, and store streams of data, the purpose is to enable real-time processing of streams.

In Kafka a stream processor is anything that takes continual streams of data from input topics, performs some processing on this input, and produces continual streams of data to output topics.

For example, a retail application might take in input streams of sales and shipments, and output a stream of reorders and price adjustments computed off this data.

It is possible to do simple processing directly using the producer and consumer APIs. However for more complex transformations Kafka provides a fully integrated Streams API. This allows building applications that do non-trivial processing that compute aggregations off of streams or join streams together.

This facility helps solve the hard problems this type of application faces: handling out-of-order data, reprocessing input as code changes, performing stateful computations, etc.

The streams API builds on the core primitives Kafka provides: it uses the producer and consumer APIs for input, uses Kafka for stateful storage, and uses the same group mechanism for fault tolerance among the stream processor instances.

Putting the Pieces Together
This combination of messaging, storage, and stream processing may seem unusual but it is essential to Kafka's role as a streaming platform.

A distributed file system like HDFS allows storing static files for batch processing. Effectively a system like this allows storing and processing historical data from the past.

A traditional enterprise messaging system allows processing future messages that will arrive after you subscribe. Applications built in this way process future data as it arrives.

Kafka combines both of these capabilities, and the combination is critical both for Kafka usage as a platform for streaming applications as well as for streaming data pipelines.

By combining storage and low-latency subscriptions, streaming applications can treat both past and future data the same way. That is a single application can process historical, stored data but rather than ending when it reaches the last record it can keep processing as future data arrives. This is a generalized notion of stream processing that subsumes batch processing as well as message-driven applications.

Likewise for streaming data pipelines the combination of subscription to real-time events make it possible to use Kafka for very low-latency pipelines; but the ability to store data reliably make it possible to use it for critical data where the delivery of data must be guaranteed or for integration with offline systems that load data only periodically or may go down for extended periods of time for maintenance. The stream processing facilities make it possible to transform data as it arrives.

For more information on the guarantees, APIs, and capabilities Kafka provides see the rest of the documentation.

[multi-languages]: https://cwiki.apache.org/confluence/display/KAFKA/Clients
[producer]: ../
[consumer]: ../
[streams]: ../
[connector]: ../
[security]: ../security/untitled
[design]: ../design/untitled
