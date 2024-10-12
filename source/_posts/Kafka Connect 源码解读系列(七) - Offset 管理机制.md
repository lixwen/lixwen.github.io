---
title: Kafka Connect源码解读系列(七) - Offset管理机制
date: 2023-04-10 22:05:31
tags: 
    - 技术
    - kafka
    - java
---

Offset 是 Kafka Connect 用于跟踪数据传输进度的关键概念。Source Connector 使用 Offset 记录从外部系统读取数据的位置,而 Sink Connector 则使用 Offset 跟踪写入目标系统的进度。Connect 提供了自动化的 Offset 管理机制,确保在出错或重启时能够从上次的位置继续处理数据。本文将围绕 Offset 的持久化存储、提交策略、故障恢复机制以及管理 API 等方面,深入分析 Offset 管理相关的核心代码实现。

## Offset 持久化存储源码追踪

Connect 将 Offset 信息持久化存储在内部的 Kafka 主题中,这个主题的名称由 `offset.storage.topic` 配置项指定。我们来看一下 `OffsetBackingStore` 类中关于 Offset 持久化的核心代码:

```java
public class OffsetBackingStore {
    // ...

    public void start() {
        // 创建 Kafka Consumer 用于读取 Offset 主题
        consumer = new KafkaBasedLog(offsetsTopicPartitionMap, offsetsTopicReplicationFactor);
        // 加载已存储的 Offset
        loadOffsets();
    }

    public void put(Map<BytesConsumer, OffsetAndMetadata> consumedOffsets) {
        // 将 Offset 写入 Kafka 主题
        consumer.putOffsets(consumedOffsets);
    }

    // ...
}
```

在 `OffsetBackingStore` 启动时,会创建一个 Kafka Consumer 来读取 Offset 主题,并加载已存储的 Offset。当有新的 Offset 需要持久化时,`put()` 方法会将 Offset 写入到 Kafka 主题中。

## Offset 提交策略实现源码解析

Offset 提交策略决定了 Offset 提交的时机和方式,由 `offset.flush.interval.ms` 和 `offset.flush.timeout.ms` 等配置项控制。我们以 `SourceTask` 为例,分析 Offset 提交策略的实现:

```java
public class SourceTask extends WorkerSourceTask {
    // ...

    @Override
    public List<SourceRecord> poll() throws InterruptedException {
        // ... 从外部系统读取数据记录

        // 检查是否需要提交 Offset
        if (shouldFlushOffsets()) {
            flushOffsets();
        }

        return records;
    }

    private boolean shouldFlushOffsets() {
        // 根据时间间隔和记录数量判断是否需要提交 Offset
        long timeElapsed = time.milliseconds() - lastOffsetFlushTimeMs;
        boolean flushOffsets = timeElapsed >= offsetFlushIntervalMs || records.size() >= offsetFlushRecordsCount;
        return flushOffsets;
    }

    private void flushOffsets() {
        // 提交 Offset
        offsetWriter.offset(offset);
        // ...
    }
}
```

在 `poll()` 方法中,Task 会根据配置的时间间隔和记录数量,判断是否需要提交 Offset。如果满足条件,就会调用 `flushOffsets()` 方法将当前 Offset 提交到 Kafka 主题中。

## 故障恢复机制原理源码分析

当 Connect 进程重启或 Task 重新分配时,Offset 管理机制可以确保数据处理从上次提交的位置继续进行,避免数据重复或遗漏。如果找不到 Offset,则需要根据配置的策略(`offset.reset`)来决定从哪里开始处理数据。

我们来看一下 `WorkerSourceTask` 中与故障恢复相关的源码:

```java
public class WorkerSourceTask extends SourceTask {
    // ...

    @Override
    public void start(Map<String, String> props) {
        // ... 初始化配置和资源

        // 获取初始 Offset
        initializeOffsets();
    }

    private void initializeOffsets() {
        // 从 Kafka 主题读取 Offset
        Map<BytesConsumer, OffsetAndMetadata> offsetsToRewind = offsetReader.getOffsets();

        // 如果找不到 Offset,根据配置的策略重置
        if (offsetsToRewind.isEmpty()) {
            rewindOffsets(offsetsToRewind);
        }

        // ... 设置初始 Offset
    }

    private void rewindOffsets(Map<BytesConsumer, OffsetAndMetadata> offsets) {
        // 根据 offset.reset 配置重置 Offset
        if (offsetResetStrategy == OffsetResetStrategy.EARLIEST) {
            // ...
        } else if (offsetResetStrategy == OffsetResetStrategy.LATEST) {
            // ...
        } else {
            // ...
        }
    }
}
```

在 `start()` 方法中,Task 会调用 `initializeOffsets()` 方法来获取初始 Offset。如果找不到 Offset,就会根据 `offset.reset` 配置的策略,调用 `rewindOffsets()` 方法重置 Offset 到最早或最新的位置。

通过上述机制,Kafka Connect 能够在发生故障或重启时,从上次提交的位置或者配置的位置继续处理数据,确保数据的精确一次性语义。

## Offset 管理 API 源码解读

Connect 提供了一组 Offset 管理 API,用于手动获取、提交和重置 Offset。这些 API 通常用于一些特殊场景,如初始化数据复制或手动进行故障恢复等。

我们以 `OffsetManagementClient` 为例,看一下这些 API 的核心实现:

```java
public class OffsetManagementClient {
    // ...

    public Map<BytesConsumer, OffsetAndMetadata> getOffsets(
        String clusterId,
        String groupId,
        Map<BytesConsumer, ConnectorOffsetBackingStore> offsetStore
    ) {
        // 从 Kafka 主题中获取 Offset
        return offsetStore.getOffsets(clusterId, groupId);
    }

    public void commitOffsets(
        String clusterId,
        String groupId,
        Map<BytesConsumer, OffsetAndMetadata> offsetsToCommit,
        Map<BytesConsumer, ConnectorOffsetBackingStore> offsetStore
    ) {
        // 提交 Offset 到 Kafka 主题
        offsetStore.putOffsets(clusterId, groupId, offsetsToCommit);
    }

    // 其他 API 方法...
}
```

`getOffsets()` 方法用于从 Kafka 主题中获取当前的 Offset 信息,而 `commitOffsets()` 方法则用于手动提交指定的 Offset 到 Kafka 主题中。

这些 API 为开发者提供了直接操作 Offset 的能力,可用于实现一些特殊的数据处理需求或故障恢复场景。

通过本文的源码分析,我们深入了解了 Kafka Connect 中 Offset 管理机制的实现细节,包括 Offset 持久化、提交策略、故障恢复以及管理 API 等方面。结合前几篇文章对 Worker、Connector 和 Transform 的分析,我们已经全面解读了 Kafka Connect Core 核心模块的关键源码。