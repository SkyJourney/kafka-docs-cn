# 快速入门 Quick Start

该教程假定读者是从头开始，并且没有现有的`Kafka`或`ZooKeeper`数据。`Kafka`控制台脚本在基于`Unix`的平台和`Windows`平台有所出入，因此在`Windows`平台使用`bin\windows\`而不是`bin/`，并且脚本扩展名改为`.bat`。

## 步骤1：下载软件 Download the code <a id="download-code"></a>

[下载](https://www.apache.org/dyn/closer.cgi?path=/kafka/2.4.0/kafka_2.12-2.4.0.tgz) `2.4.0`发行版并解压：

```bash
> tar -xzf kafka_2.12-2.4.0.tgz
> cd kafka_2.12-2.4.0
```

## 步骤2：启动服务器 Start the server <a id="start-server"></a>

`Kafka`需要使用`ZooKeeper`，因此如果你还没有，请先启动`ZooKeeper`服务器。你也可以使用`Kafka`软件包中的便携脚本获取一个应急的单节点`ZooKeeper`实例：

```bash
> bin/zookeeper-server-start.sh config/zookeeper.properties
[2013-04-22 15:01:37,495] INFO Reading configuration from: config/zookeeper.properties (org.apache.zookeeper.server.quorum.QuorumPeerConfig)
...
```

现在启动`Kafka`服务器:

```bash
> bin/kafka-server-start.sh config/server.properties
[2013-04-22 15:01:47,028] INFO Verifying properties (kafka.utils.VerifiableProperties)
[2013-04-22 15:01:47,051] INFO Property socket.send.buffer.bytes is overridden to 1048576 (kafka.utils.VerifiableProperties)
...
```

## 步骤3：创建主题 Create a topic <a id="create-topic"></a>

创建一个只有一个分区和一个副本的主题叫`test`：

```bash
> bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
```

现在，如果我们运行列出主题命令，我们可以看到该主题：

```bash
> bin/kafka-topics.sh --list --bootstrap-server localhost:9092
test
```

除了手动创建主题外，也可以将消息代理配置为当一个不存在的主题被发布时自动创建该主题。

## 步骤4：发送消息 Send some messages <a id="send-messages"></a>

`Kafka`自带命令行客户端，它将从一个文件或者通过标准输入获取输入，并将其作为消息发送至`Kafka`集群。默认情况下，每一行命令都被作为一条单独的消息发送。

运行生产者然后输入一些消息到控制台，然后发送到服务器：

```bash
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
This is a message
This is another message
```

## 步骤5：开启消费者 Start a consumer <a id="start-consumer"></a>

`Kafka`还有一个命令行消费者，它能够将消息转储到标准输出：

```bash
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
This is a message
This is another message
```

如果你在不同的终端中运行上述命令行工具，那么你现在应该可以在生产者终端输入消息，然后在消费者终端看到它们。

所有的命令行工具都有附加选项；当无参数运行时将显示详细的使用文档信息。

## 步骤6: 设定多代理集群 Setting up a multi-broker cluster <a id="setting-up-multi-broker-cluster"></a>

到目前为止，我们一直运行在单代理模式下，但这并不有趣。对`Kafka`来说，单代理仅意味大小为`1`着集群，因此设定多代理集群除了启动更多的代理实例外，并没有什么太大变化。但是只为了感受一下集群的魅力，让我们将集群扩展到三个节点（全部在本地机器上）。

首先我们为每个代理设置配置文件（Windows下只需复制命令）：

```bash
> cp config/server.properties config/server-1.properties
> cp config/server.properties config/server-2.properties
```

现在编辑这些新文件，并设定如下属性：

```bash
config/server-1.properties:
    broker.id=1
    listeners=PLAINTEXT://:9093
    log.dirs=/tmp/kafka-logs-1

config/server-2.properties:
    broker.id=2
    listeners=PLAINTEXT://:9094
    log.dirs=/tmp/kafka-logs-2
```

`broker.id`属性是集群里每个节点的唯一且永久的名称。我们只需要重写端口和日志目录，因为我们要将这些节点全部运行在同一个机器，并且希望所有代理都不要尝试在统一端口上注册或覆盖彼此的数据。

我们已经运行了ZooKeeper和我们的单节点，所有我们只需要启动两个新节点：

```bash
> bin/kafka-server-start.sh config/server-1.properties &
...
> bin/kafka-server-start.sh config/server-2.properties &
...
```

现在我们创建一个复制系数为`3`的主题：

```bash
> bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 3 --partitions 1 --topic my-replicated-topic
```

好了，现在有了集群，但是如何知道哪个代理在做什么？我们运行描述主题命令来查看：

```bash
> bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic my-replicated-topic
Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
    Topic: my-replicated-topic  Partition: 0    Leader: 1   Replicas: 1,2,0 Isr: 1,2,0
```

解释一下输出信息。第一行给出所有分区的摘要，附加的每一行提供一个分区的详细信息。由于我们的主题只有一个分区，所有只有一行附加。

* `leader`是负责给定分区的所有读写操作的节点。每个节点都会成为随机选择的部分分区的`leader`。
* `replicas`是为该分区复制日志的节点列表，不管它们是否是`leader`或是否当前处于活动状态。
* `isr`是`in-sync`（同步中）副本的集合。这是副本列表的子集，列出当前仍处于活动状态且与`leader`保持同步的节点。

注意在本例当中节点`1`是主题的唯一分区的`leader`。

我们可以在创建的原始主题上运行相同的命令来查看其位置：

```bash
> bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic test
Topic:test  PartitionCount:1    ReplicationFactor:1 Configs:
    Topic: test Partition: 0    Leader: 0   Replicas: 0 Isr: 0
```

因此没有什么意外，原始主题没有副本，只位于集群中唯一创建的服务器`0`。

我们发布一些消息到新主题中：

```bash
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

现在我们来消费这些消息： Now let's consume these messages:

```bash
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

此时，我们来测试一下容错能力。代理`1`正充当`leader`角色，因此我们关掉它：

```bash
> ps aux | grep server-1.properties
7564 ttys002    0:15.91 /System/Library/Frameworks/JavaVM.framework/Versions/1.8/Home/bin/java...
> kill -9 7564
```

`Windows`中操作：

```bash
> wmic process where "caption = 'java.exe' and commandline like '%server-1.properties%'" get processid
ProcessId
6016
> taskkill /pid 6016 /f
```

`leader`已切换到`floowers`之一，且节点`1`不再显示在同步副本的列表中：

```bash
> bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic my-replicated-topic
Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
    Topic: my-replicated-topic  Partition: 0    Leader: 2   Replicas: 1,2,0 Isr: 2,0
```

但是即使最初写入的leader已经下线，消息任何可以使用：

```bash
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

## 步骤7：用 Kafka Connect 导入/导出数据 Use Kafka Connect to import/export data <a id="connect-import-export-data"></a>

从控制台输入数据再写回到控制台是非常方便的起点，但是你可能要从其他来源使用数据或从Kafka导出的数据到其他系统。对于许多系统来说，可以用`Kafka Connect`导入或导出数据，无需编写自定义的整合代码。

`Kafka Connect`是`Kafka`自带的用于将数据导入和导出到`Kafka`的工具。它是运行连接器（`connector`）的扩展工具，其可以实现于外部系统交互的自定义逻辑。在`快速入门`即本篇文章中，我们将看到如何使用简单的连接器运行`Kafka Connect`，并用其从文件导入数据到`Kafka`主题中，并从`Kafka`主题中导出数据到文件。

首先，我们将从创建一些种子数据开始进行测试：

```bash
> echo -e "foo\nbar" > test.txt
```

在Windows中操作：

```bash
> echo foo> test.txt
> echo bar>> test.txt
```

紧接着，我们将启动两个独立模式运行的连接器，也就是说他们各自运行在本地独立专用的进程中。我们提供三个配置文件作为参数。第一个始终是`Kafka Connect`进程的配置，包括通用配置，如要连接的`Kafka`代理和数据的序列化格式。其余的配置文件均指定要创建的连接器。 这些文件包括唯一的连接器名称、要实例化的连接器类以及连接器所需的任何其他配置。

```bash
> bin/connect-standalone.sh config/connect-standalone.properties config/connect-file-source.properties config/connect-file-sink.properties
```

这些随`Kafka`附赠的样本配置文件使用之前启动的默认本地集群配置并创建两个连接器：第一个是源连机器，能够从输入文件读取行数据并将每一行生成为`Kafka`主题中的信息；第二个是接受连接器，能够从`Kafka`主题中读取信息并将每条信息在输出文件中生成一行数据。

在启动过程中，你将看到许多日志信息，其中包括一些标明正在实例化连接器的日志。一旦`Kafka Connect`进程启动完毕，源连接器就应该开始从`test.txt`文件中读取行数据，并将他们转换生成到主题`connect-test`中。然后接收连接器应该开始从主题`connect-test`读取信息，并写入他们到文件`test.sink.txt`中。我们可以我们可以通过检查输出文件的内容来验证数据是否已通过整个管道传递：

```bash
> more test.sink.txt
foo
bar
```

注意数据是被存储在主题`connect-test`中，因此我们也能启动控制台消费者来查看在主题中的数据（或者使用自定义的消费者代码去处理它）：

```bash
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic connect-test --from-beginning
{"schema":{"type":"string","optional":false},"payload":"foo"}
{"schema":{"type":"string","optional":false},"payload":"bar"}
...
```

连接器会继续处理数据，因此我们可以将数据添加到文件中，并查看它在管道中的移动情况：

The connectors continue to process data, so we can add data to the file and see it move through the pipeline:

```bash
> echo Another line>> test.txt
```

您应该看到该行出现在控制台使用者输出和接收器文件中。

## 步骤8：用 Kafka Streams 处理数据 Use Kafka Streams to process data <a id="streams-process-data"></a>

`Kafka Streams`是用于构建关键任务实时应用程序和微服务的客户端库，其中输入和/或输出数据存储在`Kafka`集群中。 `Kafka Streams`结合了在客户端编写和部署标准Java和Scala应用程序的简便性以及`Kafka`服务器端集群技术的优势，使这些应用程序具有高度可伸缩性，弹性，容错性，分布式等特性。此[快速入门示例](https://github.com/SkyJourney/kafka-docs-cn/tree/265103ae9192e2ccdca46e897ef3a6f8b3d37f37/kafka-streams/quick-start.md)将演示如何运行此库中编码的流应用程序。

