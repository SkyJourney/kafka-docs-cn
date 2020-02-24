# 简介 Introduction

`Apache Kafka®` 是一个分布式流平台。这到底是什么意思呢？

 流平台具有三个关键功能：

* 发布和订阅记录消息记录流，类似于消息队列或企业消息传递系统
* 以持久的且具有容错能力的方式存储消息记录流
* 处理消息记录流

`Kafka`通常用于两大类应用程序：

* 建立实时的流数据管道，使系统或应用程序之间能够可靠地获取数据
* 建立实时的流应用程序，可以转换或响应数据流

为了理解`Kafka`如何完成这些事，让我们从头开始探索`Kafka`的功能。

首先了解几个概念：

* `Kafka`是在一个或多个可以跨越多个数据中心的服务器上以集群方式运行
* `Kafka`集群以`topics`归类储存的消息记录流
* 每条数据记录由一个键、一个值和一个时间戳构成

`Kafka`有四大核心`APIs`：

![1-1 核心APIs示意图](../.gitbook/assets/kafka-apis.png)


* [`Producer API`][producer]（生产者）允许应用程序发布数据记录流到一个或多个`topics`中
* [`Consumer API`][consumer]（消费者）允许应用程序订阅一个或多个`topics`并处理其生成的数据记录流
* [`Streams API`][streams]（流处理）允许应用程序作为流处理器，消费从一个或多个`topics`中获取的输入流并生产输出流到一个或多个`toppics`中，有效地进行输入流和输出流的转换
* [`Connector API`][connector]（连接器）允许建立和允许可复用的生产者和消费者，用来连接`topics`和已存在的应用程序或数据系统。例如关系型数据库的连接器可能会捕捉对表的所有更改

在`Kafka`中，服务端和客户端之间的通信是通过简单、高性能、无关编程语言的`TCP`协议来完成。该协议已经版本化，并维持对旧版本的向下兼容。我们提供`Java`版本的`Kafka`客户端，但客户端也同时支持[多种语言][multi-languages]。



[multi-languages]: https://cwiki.apache.org/confluence/display/KAFKA/Clients
[producer]: ../
[consumer]: ../
[streams]: ../
[connector]: ../
