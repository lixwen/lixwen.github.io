---
title: Kafka Connect源码解读系列(十) - Source Connector源码分析
date: 2023-06-02 23:20:01
tags: 
    - 技术
    - kafka
    - java
---


Source Connector 是 Kafka Connect 用于从外部系统导入数据到 Kafka 主题的核心组件。本文将重点分析两种常见的 Source Connector:JDBC Source Connector 和 Kafka Source Connector,深入探讨它们的实现细节和关键源码。

## JDBC Source Connector 源码分析

JDBC Source Connector 允许从关系型数据库中导入数据到 Kafka 主题。我们从以下几个方面来分析它的实现:

### 1. JdbcSourceConnector 实现

`JdbcSourceConnector` 继承自 `SourceConnector` 抽象类,是 JDBC Source Connector 的入口点。我们来看一下它的核心实现:

```java
public class JdbcSourceConnector extends SourceConnector {
    @Override
    public void start(Map<String, String> props) {
        // 解析配置属性
        initialize(props);
    }

    @Override
    public Class<? extends Task> taskClass() {
        return JdbcSourceTask.class;
    }

    @Override
    public List<Map<String, String>> taskConfigs(int maxTasks) {
        // 根据配置生成 Task 配置
        List<Map<String, String>> configs = new ArrayList<>(maxTasks);
        for (int i = 0; i < maxTasks; i++) {
            configs.add(config.originalsStrings());
        }
        return configs;
    }

    // ... 其他方法实现
}
```

在 `start()` 方法中,Connector 会解析配置属性,并在 `taskConfigs()` 方法中根据 `maxTasks` 参数生成相应数量的 Task 配置。每个 Task 配置都是原始配置的副本。

### 2. JdbcSourceTask 执行流程

`JdbcSourceTask` 是 JDBC Source Connector 的实际执行单元,我们来看一下它的 `poll()` 方法:

```java
public class JdbcSourceTask extends SourceTask {
    @Override
    public List<SourceRecord> poll() throws InterruptedException {
        // 初始化数据库连接和游标
        // ...

        List<SourceRecord> records = new ArrayList<>();
        while (remainingTimeMs > timeBetweenQueries) {
            // 查询数据库并构建 SourceRecord
            ResultSet resultSet = executeQuery();
            while (resultSet.next()) {
                SourceRecord record = generateRecord(resultSet);
                records.add(record);
            }
        }

        // 提交 Offset
        commitOffset();

        return records;
    }

    // ... 其他方法实现
}
```

在 `poll()` 方法中,Task 会执行以下核心逻辑:

1. 初始化数据库连接和游标。
2. 循环执行查询,获取结果集。
3. 遍历结果集,为每条记录构建 `SourceRecord` 对象。
4. 提交当前的 Offset。
5. 返回构建好的 `SourceRecord` 列表。

通过这个过程,JDBC Source Connector 能够周期性地从关系型数据库中读取数据,并将数据流式地推送到 Kafka 主题中。

## Kafka Source Connector 源码分析

Kafka Source Connector 用于从一个 Kafka 主题消费数据,并将数据复制到另一个 Kafka 主题中。它的实现原理与 JDBC Source Connector 类似,但有一些特殊之处。

### 1. KafkaSourceConnector 实现

`KafkaSourceConnector` 继承自 `SourceConnector`,我们来看一下它的核心实现:

```java
public class KafkaSourceConnector extends SourceConnector {
    @Override
    public void start(Map<String, String> props) {
        // 解析配置属性
        config = new KafkaSourceConnectorConfig(props);
    }

    @Override
    public Class<? extends Task> taskClass() {
        return KafkaSourceTask.class;
    }

    @Override
    public List<Map<String, String>> taskConfigs(int maxTasks) {
        // 生成 Task 配置
        List<Map<String, String>> taskConfigs = new ArrayList<>(maxTasks);
        for (int i = 0; i < maxTasks; i++) {
            Map<String, String> taskProps = new HashMap<>(config.originalsStrings());
            taskConfigs.add(taskProps);
        }
        return taskConfigs;
    }

    // ... 其他方法实现
}
```

与 JDBC Source Connector 类似,`KafkaSourceConnector` 会在 `start()` 方法中解析配置属性,在 `taskConfigs()` 方法中生成 Task 配置。

### 2. KafkaSourceTask 执行流程

`KafkaSourceTask` 是 Kafka Source Connector 的实际执行单元,我们来看一下它的 `poll()` 方法:

```java
public class KafkaSourceTask extends SourceTask {
    @Override
    public List<SourceRecord> poll() throws InterruptedException {
        // 初始化 Kafka Consumer
        // ...

        List<SourceRecord> records = new ArrayList<>();
        ConsumerRecords<byte[], byte[]> consumerRecords = consumer.poll(pollTimeout);
        for (ConsumerRecord<byte[], byte[]> record : consumerRecords) {
            // 构建 SourceRecord
            SourceRecord sourceRecord = generateRecord(record);
            records.add(sourceRecord);
        }

        // 提交 Offset
        commitOffset();

        return records;
    }

    // ... 其他方法实现
}
```

在 `poll()` 方法中,Task 会执行以下核心逻辑:

1. 初始化 Kafka Consumer。
2. 从源 Kafka 主题消费一批记录。
3. 遍历消费到的记录,为每条记录构建 `SourceRecord` 对象。
4. 提交当前的 Offset。
5. 返回构建好的 `SourceRecord` 列表。

通过这个过程,Kafka Source Connector 能够从一个 Kafka 主题消费数据,并将数据复制到另一个 Kafka 主题中。

总的来说,无论是 JDBC Source Connector 还是 Kafka Source Connector,它们的核心执行流程都遵循类似的模式:初始化连接、消费数据、构建 SourceRecord、提交 Offset。通过对这些常见 Source Connector 的源码分析,我们深入了解了 Kafka Connect 从外部系统导入数据的实现细节。