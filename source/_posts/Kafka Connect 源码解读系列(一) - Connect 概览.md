---
title: Kafka Connect源码解读系列(一) - Connect概览
date: 2023-03-12 19:30:38
tags: 
    - 技术
    - kafka
    - java
---

Apache Kafka 是一种高吞吐量的分布式发布订阅消息队列系统,广泛应用于大数据领域的数据传输和处理。Kafka Connect 作为 Kafka 生态中的一员,扮演着将外部数据集成到 Kafka 的重要角色。本文将对 Kafka Connect 的架构设计和核心概念进行概览,为后续的源码分析做好铺垫。

## **Kafka Connect 架构**

Kafka Connect 提供了一个可扩展的数据流平台,用于持续地从各种数据源(Source)中获取数据,并将数据推送到目标数据存储系统(Sink)。Connect 自身作为单个进程运行,但也支持通过工作集群的方式进行扩展部署。

Connect 架构中有两个核心组件:

1. **Connect Worker**

Connect Worker 是运行连接器(Connectors)的进程。它会维护一组连接器的生命周期,接收配置请求,并将相应的连接器任务(Tasks)调度和执行。

2. **Connectors**

Connectors 即连接器,用于实现将数据从特定数据源获取或写入到目标系统中。Connectors 被划分为 Source Connectors 和 Sink Connectors。每个 Connector 在运行时可以具有一个或多个 Tasks。

Connect 架构的工作流程大致如下:

2.1 Connect Worker 进程启动时会加载指定的 Connectors。
2.2 Worker 根据 Connector 的配置,创建相应数量的 Tasks。
2.3 Source Connector 的 Tasks 从数据源读取数据记录(Records),并写入 Kafka 主题(Topics)。
2.4 Sink Connector 的 Tasks 从 Kafka 消费数据记录,并将数据写入目标系统。
2.5 Worker 负责协调 Connectors 和 Tasks 之间的工作。

## **Kafka Connect 核心概念**

了解 Kafka Connect 架构后,我们来看一下其中的核心概念:

**Connector**

Connector 是 Kafka Connect 用于集成外部系统的组件。Connector 有两种类型:Source 和 Sink。Source Connector 用于从外部系统导入数据到 Kafka 主题,而 Sink Connector 则是从 Kafka 主题导出数据到外部系统。

**Connector Configuration**

每个 Connector 都需要进行配置,配置内容包括连接数据源所需的连接信息、转换规则、数据格式等。通过 Kafka Connect REST API 或配置文件的方式来配置 Connectors。

**Task**

Task 是 Connector 的执行单元,由于需要处理大量的数据,所以一个 Connector 通常具有多个 Tasks。Tasks 并行运行,每个 Task 负责处理一部分数据。

**Worker**

Worker 是运行 Connectors 的进程实例。它负责加载配置的 Connectors,创建和管理 Tasks 的生命周期。Worker 会定期重新平衡 Connectors 并协调 Tasks 的执行。

**Transformation**

在数据流转过程中,数据记录可能需要进行转换,如数据清洗、格式化等。Connect 提供了内置的单条记录转换(Single Message Transformation,SMT),也支持定制转换逻辑。

**Offset**

Offset 跟踪了 Source 和 Sink Connector 处理数据的位置。Source 使用 Offset 记录读取位置,Sink 则使用 Offset 来跟踪写入位置。Connect 会自动管理和持久化 Offset。

以上就是对 Kafka Connect 架构和核心概念的概述。下一篇文章将重点分析 Connect 中的 Worker 和 Connector 组件的实现原理。