# 1.3 快速开始 Quick Start

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



Here is an explanation of output. The first line gives a summary of all the partitions, each additional line gives information about one partition. Since we have only one partition for this topic there is only one line.

"leader" is the node responsible for all reads and writes for the given partition. Each node will be the leader for a randomly selected portion of the partitions.
"replicas" is the list of nodes that replicate the log for this partition regardless of whether they are the leader or even if they are currently alive.
"isr" is the set of "in-sync" replicas. This is the subset of the replicas list that is currently alive and caught-up to the leader.
Note that in my example node 1 is the leader for the only partition of the topic.

We can run the same command on the original topic we created to see where it is:

1
2
3
> bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic test
Topic:test  PartitionCount:1    ReplicationFactor:1 Configs:
    Topic: test Partition: 0    Leader: 0   Replicas: 0 Isr: 0
So there is no surprise there—the original topic has no replicas and is on server 0, the only server in our cluster when we created it.

Let's publish a few messages to our new topic:

1
2
3
4
5
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
...
my test message 1
my test message 2
^C
Now let's consume these messages:

1
2
3
4
5
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
Now let's test out fault-tolerance. Broker 1 was acting as the leader so let's kill it:

1
2
3
> ps aux | grep server-1.properties
7564 ttys002    0:15.91 /System/Library/Frameworks/JavaVM.framework/Versions/1.8/Home/bin/java...
> kill -9 7564
On Windows use:
1
2
3
4
> wmic process where "caption = 'java.exe' and commandline like '%server-1.properties%'" get processid
ProcessId
6016
> taskkill /pid 6016 /f
Leadership has switched to one of the followers and node 1 is no longer in the in-sync replica set:

1
2
3
> bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic my-replicated-topic
Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
    Topic: my-replicated-topic  Partition: 0    Leader: 2   Replicas: 1,2,0 Isr: 2,0
But the messages are still available for consumption even though the leader that took the writes originally is down:

1
2
3
4
5
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
Step 7: Use Kafka Connect to import/export data
Writing data from the console and writing it back to the console is a convenient place to start, but you'll probably want to use data from other sources or export data from Kafka to other systems. For many systems, instead of writing custom integration code you can use Kafka Connect to import or export data.

Kafka Connect is a tool included with Kafka that imports and exports data to Kafka. It is an extensible tool that runs connectors, which implement the custom logic for interacting with an external system. In this quickstart we'll see how to run Kafka Connect with simple connectors that import data from a file to a Kafka topic and export data from a Kafka topic to a file.

First, we'll start by creating some seed data to test with:

1
> echo -e "foo\nbar" > test.txt
Or on Windows:
1
2
> echo foo> test.txt
> echo bar>> test.txt
Next, we'll start two connectors running in standalone mode, which means they run in a single, local, dedicated process. We provide three configuration files as parameters. The first is always the configuration for the Kafka Connect process, containing common configuration such as the Kafka brokers to connect to and the serialization format for data. The remaining configuration files each specify a connector to create. These files include a unique connector name, the connector class to instantiate, and any other configuration required by the connector.

1
> bin/connect-standalone.sh config/connect-standalone.properties config/connect-file-source.properties config/connect-file-sink.properties
These sample configuration files, included with Kafka, use the default local cluster configuration you started earlier and create two connectors: the first is a source connector that reads lines from an input file and produces each to a Kafka topic and the second is a sink connector that reads messages from a Kafka topic and produces each as a line in an output file.

During startup you'll see a number of log messages, including some indicating that the connectors are being instantiated. Once the Kafka Connect process has started, the source connector should start reading lines from test.txt and producing them to the topic connect-test, and the sink connector should start reading messages from the topic connect-test and write them to the file test.sink.txt. We can verify the data has been delivered through the entire pipeline by examining the contents of the output file:

1
2
3
> more test.sink.txt
foo
bar
Note that the data is being stored in the Kafka topic connect-test, so we can also run a console consumer to see the data in the topic (or use custom consumer code to process it):

1
2
3
4
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic connect-test --from-beginning
{"schema":{"type":"string","optional":false},"payload":"foo"}
{"schema":{"type":"string","optional":false},"payload":"bar"}
...
The connectors continue to process data, so we can add data to the file and see it move through the pipeline:

1
> echo Another line>> test.txt
You should see the line appear in the console consumer output and in the sink file.

Step 8: Use Kafka Streams to process data
Kafka Streams is a client library for building mission-critical real-time applications and microservices, where the input and/or output data is stored in Kafka clusters. Kafka Streams combines the simplicity of writing and deploying standard Java and Scala applications on the client side with the benefits of Kafka's server-side cluster technology to make these applications highly scalable, elastic, fault-tolerant, distributed, and much more. This quickstart example will demonstrate how to run a streaming application coded in this library.