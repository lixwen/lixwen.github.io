---
title: Kafka Connect 源码解读系列(六) - Transform 代码分析
date: 2023-03-27 21:03:32
tags: 
    - 技术
    - kafka
---

在数据流经 Kafka Connect 的传输过程中,通常需要对数据记录执行一些转换操作,以满足特定的需求。Kafka Connect 提供了灵活的数据转换机制,支持在 Connector 级别和 Task 级别定义转换逻辑。本文将围绕 Transformation 接口、内置转换实现、转换配置和执行时机等方面,深入分析 Transform 相关代码。

## Transformation 接口定义源码分析

Kafka Connect 中的数据转换逻辑是通过实现 `org.apache.kafka.connect.transforms.Transformation` 接口来定义的。让我们看一下这个接口的核心方法:

```java
public interface Transformation<R extends ConnectRecord<R>> {
    // 使用提供的属性配置转换逻辑
    void configure(Map<String, ?> props);

    // 对单条记录执行转换操作
    R apply(R record);

    // 在转换逻辑关闭时执行清理工作
    void close();
}
```

1. `configure()`：用于初始化转换逻辑,通常会解析配置属性。
2. `apply()`：对传入的单条记录执行转换操作,并返回转换后的记录。
3. `close()`：在转换逻辑关闭时执行清理工作,如释放资源等。

任何自定义的转换逻辑实现都需要遵循这个接口的约定。

## 内置 Transformation 实现源码解读

Kafka Connect 内置了一些常用的转换逻辑实现,例如 `InsertField`、`ValueToKey` 等。我们以 `InsertField` 为例,看一下它的核心实现:

```java
public class InsertField<R extends ConnectRecord<R>> implements Transformation<R> {
    // 配置解析和初始化
    @Override
    public void configure(Map<String, ?> props) {
        // ...
    }

    // 执行转换操作
    @Override
    public R apply(R record) {
        if (operatingValue) {
            final R newRecord = record.newRecord(
                record.topic(), record.kafkaPartition(), record.keySchema(), record.key(),
                valueSchema, insertValue(record.valueSchema(), record.value(), fieldsAndSource));
            return newRecord;
        } else {
            // ...
        }
    }

    // 执行清理工作
    @Override
    public void close() {
        // ...
    }
}
```

可以看到,`InsertField` 实现了 `Transformation` 接口中定义的所有方法。在 `apply()` 方法中,它根据配置的属性,将指定的字段和值插入到记录的 Value 中,并返回新的记录对象。这就是 `InsertField` 转换的核心逻辑。

## 转换逻辑配置和执行时机源码分析

转换逻辑可以在 Connector 级别或 Task 级别进行配置,我们以 `SinkConnectorConfig` 为例,看一下转换配置的相关代码:

```java
public class SinkConnectorConfig extends AbstractConfig {
    // ... 

    public static final String TRANSFORMS_SPEC = "transforms";
    public static final String TRANSFORMS_PREFIX = "transforms.";

    private static final String TRANSFORMS_DOC = "List of transformation to be applied for each record";

    order.add(TRANSFORMS_SPEC);
    configDef.define(
        TRANSFORMS_SPEC,
        Type.LIST,
        Collections.emptyList(),
        Importance.LOW,
        TRANSFORMS_DOC,
        group,
        ++orderInGroup,
        Width.LONG,
        TRANSFORMS_PREFIX
    );

    // ...
}
```

在 `SinkConnectorConfig` 中,`TRANSFORMS_SPEC` 属性用于配置要应用的转换列表。在 Connector 实例化时,会解析这个属性并创建相应的转换逻辑实例。

转换逻辑的执行时机取决于它们被应用的位置:

- **Connector 级别** :所有 Tasks 共享同一组转换逻辑,转换在记录被写入之前执行。
- **Task 级别** :每个 Task 负责执行自己的转换逻辑,转换在记录被发送到 Kafka 或从 Kafka 消费之前执行。

以 `SinkTask` 为例,转换逻辑会在 `put` 方法中执行:

```java
public class SinkTask extends WorkerSinkTask {
    @Override
    public void put(Collection<SinkRecord> sinkRecords) {
        // ...

        // 执行转换逻辑
        for (SinkRecord preTransformRecord : sinkRecords) {
            sinkRecord = transformation.apply(preTransformRecord);
            // ...
        }

        // ...
    }
}
```

在 `put()` 方法中,Task 会遍历要写入的记录,并对每条记录依次应用配置好的转换逻辑。转换后的记录会被进一步处理并写入目标系统。

## 自定义 Transformation 实现示例

除了使用内置的转换逻辑,开发者也可以根据需求自定义转换逻辑。以下是一个简单的自定义 `UppercaseValue` 转换的示例:

```java
public class UppercaseValue<R extends ConnectRecord<R>> implements Transformation<R> {
    @Override
    public void configure(Map<String, ?> props) {
        // 解析配置属性
    }

    @Override
    public R apply(R record) {
        // 将记录值转换为大写
        String uppercaseValue = record.value().toString().toUpperCase();
        return record.newRecord(
            record.topic(), record.kafkaPartition(), record.keySchema(), record.key(),
            record.valueSchema(), uppercaseValue.getBytes(StandardCharsets.UTF_8));
    }

    @Override
    public void close() {
        // 执行清理工作
    }
}
```

在 `apply()` 方法中,我们将记录的值转换为大写字母形式,并创建一个新的记录对象返回。通过实现 `Transformation` 接口,自定义的转换逻辑可以seamlessly地集成到 Kafka Connect 中。

总的来说,Kafka Connect 的转换机制为数据处理管道提供了灵活的扩展能力,同时也提供了一些常用的内置实现。通过上述源码分析,我们深入了解了 Transform 相关代码的实现细节。
