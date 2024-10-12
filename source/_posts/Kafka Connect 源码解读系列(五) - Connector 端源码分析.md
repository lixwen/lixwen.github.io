---
title: Kafka Connect源码解读系列(五) - Connector端源码分析
date: 2023-03-17 22:03:31
tags: 
    - 技术
    - kafka
    - java
---

在上一篇文章中,我们分析了 Worker 端的核心实现,包括 Worker 启动流程、Herder 的职责以及 Rebalance 机制的源码实现。本文将继续剖析 Connector 端的相关代码,重点包括 Connector 接口、Source/Sink Connector 的实现、Task 执行流程以及 Connector 生命周期管理。

## Connector 接口定义源码分析

Connector 的核心接口定义位于 `org.apache.kafka.connect.connector.Connector` 接口中,让我们逐一分析其中定义的方法:

```java
public interface Connector {
    // 返回当前 Connector 的所有活跃 Tasks
    Set<ConnectorTaskId> getTasks();

    // 在 Connector 启动时调用,用于初始化资源和启动组件
    void start(Map<String, String> props);

    // 在 Connector 停止时调用,用于释放资源和停止组件
    void stop();

    // 验证 Connector 配置的合法性
    ConnectorConfigs validate(Map<String, String> connectorConfigs);

    // 返回 Connector 的配置定义
    ConfigDef config();

    // 根据 Connector 配置生成相应数量的 Task 配置
    List<Map<String, String>> taskConfigs(int maxTasks);

    // 判断 Connector 的类型(Source 或 Sink)
    ConnectorType type();

    // 返回 Connector 的版本信息
    String version();
}
```

这组接口定义了 Connector 在生命周期各个阶段所需要实现的基本功能,例如启动、停止、配置验证、Task 管理等。任何自定义的 Connector 实现都需要遵循这个接口的约定。

## Source/Sink Connector 实现源码解读

Kafka Connect 内置了一些常用的 Source 和 Sink Connector 实现,我们以 `FileStreamSourceConnector` 为例,分析其对 Connector 接口的实现:

```java
public class FileStreamSourceConnector extends SourceConnector {
    // 实现 start() 方法,用于启动 Connector
    @Override
    public void start(Map<String, String> props) {
        // ...
    }

    // 实现 taskConfigs() 方法,生成 Task 配置
    @Override
    public List<Map<String, String>> taskConfigs(int maxTasks) {
        // ...
    }

    // 实现 config() 方法,返回配置定义
    @Override
    public ConfigDef config() {
        // ...
    }

    // 其他接口方法的实现
    // ...
}
```

可以看到,`FileStreamSourceConnector` 继承自 `SourceConnector` 抽象类,并分别实现了 `start()`、`taskConfigs()` 和 `config()` 等方法。其中,`taskConfigs()` 方法的实现决定了该 Connector 在运行时会创建多少个 Task 实例。

对于 Sink Connector 的实现,原理类似,只是继承的是 `SinkConnector` 抽象类。

## Task 执行流程核心源码追踪

Task 是 Connector 的实际执行单元,负责与外部系统进行数据传输。以 `FileStreamSourceTask` 为例,我们来看一下它的主要执行流程:

```java
public class FileStreamSourceTask extends SourceTask {
    // 实现 start() 方法,用于初始化资源
    @Override
    public void start(Map<String, String> props) {
        // ...
    }

    // 实现 poll() 方法,用于读取数据记录
    @Override
    public List<SourceRecord> poll() throws InterruptedException {
        // ...
        while (true) {
            // 从文件中读取数据
            ByteBuffer buffer = fs.read();
            if (buffer != null) {
                // 构造 SourceRecord 并返回
                SourceRecord record = new SourceRecord(/* ... */);
                return Collections.singletonList(record);
            }
        }
    }

    // 实现 stop() 方法,用于停止 Task 并释放资源
    @Override
    public void stop() {
        // ...
    }
}
```

`FileStreamSourceTask` 继承自 `SourceTask` 抽象类,实现了 `start()`、`poll()` 和 `stop()` 等方法。在 `poll()` 方法中,Task 会循环读取外部数据源(这里是文件)中的数据,并将读取到的数据封装为 `SourceRecord` 对象返回。

对于 Sink Task,执行流程与 Source Task 类似,只是将逻辑反向,从 Kafka 消费记录并写入到外部系统中。

## Connector 生命周期管理源码解析

Connector 的生命周期管理由 Herder 组件负责,我们以 `Herder` 的 `putConnectorConfig` 方法为例,分析 Connector 生命周期的管理过程:

```java
public AbstractHerder.ConnectorInfo putConnectorConfig(
        Connector connectorConfig, // Connector 配置
        boolean allowRestart, // 是否允许重启
        boolean forwardRequest, // 是否转发请求
        boolean isRebalanceRequest // 是否为 Rebalance 请求
) {
    // 验证 Connector 配置
    connectorConfig = configTransformer.transform(connectorConfig, isRebalanceRequest);
    ConnectorConfigs configs = connectorConfig.validate(connectorConfig.configs());

    // 创建或更新 ConnectorInfo
    ConnectorInfo info = updateConnectorInfo(connectorConfig.connectorClass(), configs);

    // 执行 Connector 生命周期操作
    if (info.state() == ConnectorState.RUNNING && !allowRestart) {
        // ... 处理 Connector 已存在的情况
    } else if (!info.state().isRestartableState()) {
        // ... 处理非法状态
    } else {
        // 停止并重启 Connector
        putConnectorConfig(info, configs, allowRestart, forwardRequest, isRebalanceRequest);
    }

    return info;
}
```

该方法首先会验证 Connector 配置的合法性,然后根据 Connector 的当前状态,决定执行以下操作之一:

1. 如果 Connector 已存在且不允许重启,则直接返回。
2. 如果 Connector 处于非法状态,则抛出异常。
3. 如果 Connector 处于可重启状态,则停止当前 Connector 并重新启动一个新实例。

在 `putConnectorConfig` 内部,还会调用 `Connector` 接口中定义的生命周期方法,如 `start()`、`stop()` 等,以执行相应的操作。

通过上述源码分析,我们深入了解了 Connector 端的核心实现,包括 Connector 接口定义、Source/Sink Connector 的流程等。