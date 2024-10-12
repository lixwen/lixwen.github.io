---
title: Kafka Connect源码解读系列(十一) - Sink Connector源码分析
date: 2023-06-03 21:03:02
tags: 
    - 技术
    - kafka
    - java
---

与 Source Connector 用于从外部系统导入数据到 Kafka 不同,Sink Connector 则是将数据从 Kafka 主题导出到其他系统中。本文将重点分析两种常见的 Sink Connector:JDBC Sink Connector 和 Kafka Sink Connector,深入探讨它们的实现细节和关键源码。

## JDBC Sink Connector 源码分析

JDBC Sink Connector 允许将数据从 Kafka 主题导出到关系型数据库中。我们从以下几个方面来分析它的实现:

### 1. JdbcSinkConnector 实现

`JdbcSinkConnector` 继承自 `SinkConnector` 抽象类,是 JDBC Sink Connector 的入口点。我们来看一下它的核心实现:

```java
public class JdbcSinkConnector extends SinkConnector {
    @Override
    public void start(Map<String, String> props) {
        // 解析配置属性
        initialize(props);
    }

    @Override
    public Class<? extends Task> taskClass() {
        return JdbcSinkTask.class;
    }

    @Override
    public List<Map<String, String>> taskConfigs(int maxTasks) {
        // 生成 Task 配置
        List<Map<String, String>> taskConfigs = new ArrayList<>(maxTasks);
        for (int i = 0; i < maxTasks; i++) {
            Map<String, String> taskProps = new HashMap<>(config.originals());
            taskConfigs.add(taskProps);
        }
        return taskConfigs;
    }

    // ... 其他方法实现
}
```

与 Source Connector 类似,`JdbcSinkConnector` 会在 `start()` 方法中解析配置属性,在 `taskConfigs()` 方法中生成 Task 配置。

### 2. JdbcSinkTask 执行流程

`JdbcSinkTask` 是 JDBC Sink Connector 的实际执行单元,我们来看一下它的 `put()` 方法:

```java
public class JdbcSinkTask extends SinkTask {
    @Override
    public void put(Collection<SinkRecord> records) {
        // 初始化数据库连接
        // ...

        final DatabaseDialect dialect = dialects.findDatabaseDialect(config);
        for (SinkRecord record : records) {
            // 将 SinkRecord 转换为 SQL 语句
            final String sql = dialect.buildInsertStatement(config, record);

            // 执行 SQL 语句
            executeInsertStatement(sql, record);
        }
    }

    // ... 其他方法实现
}
```

在 `put()` 方法中,Task 会执行以下核心逻辑:

1. 初始化数据库连接。
2. 遍历要写入的 `SinkRecord` 列表。
3. 将每条 `SinkRecord` 转换为相应的 SQL 插入语句。
4. 执行 SQL 语句,将数据写入数据库。

通过这个过程,JDBC Sink Connector 能够将 Kafka 主题中的数据持续地导出到关系型数据库中。

## Kafka Sink Connector 源码分析

Kafka Sink Connector 用于将数据从一个 Kafka 主题复制到另一个 Kafka 主题中。它的实现原理与 JDBC Sink Connector 类似,但有一些特殊之处。

### 1. KafkaSinkConnector 实现

`KafkaSinkConnector` 继承自 `SinkConnector`,我们来看一下它的核心实现:

```java
public class KafkaSinkConnector extends SinkConnector {
    @Override
    public void start(Map<String, String> props) {
        // 解析配置属性
        config = new KafkaSinkConnectorConfig(props);
    }

    @Override
    public Class<? extends Task> taskClass() {
        return KafkaSinkTask.class;
    }

    @Override
    public List<Map<String, String>> taskConfigs(int maxTasks) {
        // 生成 Task 配置
        List<Map<String, String>> taskConfigs = new ArrayList<>(maxTasks);
        for (int i = 0; i < maxTasks; i++) {
            Map<String, String> taskProps = new HashMap<>(config.originals());
            taskConfigs.add(taskProps);
        }
        return taskConfigs;
    }

    // ... 其他方法实现
}
```

与 JDBC Sink Connector 类似,`KafkaSinkConnector` 会在 `start()` 方法中解析配置属性,在 `taskConfigs()` 方法中生成 Task 配置。

### 2. KafkaSinkTask 执行流程

`KafkaSinkTask` 是 Kafka Sink Connector 的实际执行单元,我们来看一下它的 `put()` 方法:

```java
public class KafkaSinkTask extends SinkTask {
    @Override
    public void put(Collection<SinkRecord> records) {
        // 初始化 Kafka Producer
        // ...

        for (SinkRecord record : records) {
            // 构建 ProducerRecord
            ProducerRecord<byte[], byte[]> producerRecord = getProducerRecord(record);

            // 发送记录到目标 Kafka 主题
            producer.send(producerRecord);
        }
    }

    // ... 其他方法实现
}
```

在 `put()` 方法中,Task 会执行以下核心逻辑:

1. 初始化 Kafka Producer。
2. 遍历要写入的 `SinkRecord` 列表。
3. 将每条 `SinkRecord` 转换为 `ProducerRecord` 对象。
4. 使用 Kafka Producer 将 `ProducerRecord` 发送到目标 Kafka 主题。

通过这个过程,Kafka Sink Connector 能够将数据从一个 Kafka 主题复制到另一个 Kafka 主题中。

总的来说,无论是 JDBC Sink Connector 还是 Kafka Sink Connector,它们的核心执行流程都遵循类似的模式:初始化连接、遍历 SinkRecord、转换记录格式、写入目标系统。通过对这些常见 Sink Connector 的源码分析,我们深入了解了 Kafka Connect 将数据导出到外部系统的实现细节。