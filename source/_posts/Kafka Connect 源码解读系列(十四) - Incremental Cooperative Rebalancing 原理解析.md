---
title: Kafka Connect源码解读系列(十四) - Incremental Cooperative Rebalancing原理解析
date: 2023-08-22 23:12:23
tags: 
    - 技术
    - kafka
    - java
---

在之前的文章中，我们分析了Kafka Connect的Rebalance机制和故障转移流程。从Kafka 2.4版本开始，Connect引入了一种名为 Incremental Cooperative Rebalancing的新机制，旨在优化 Rebalance 过程，提高集群稳定性和性能。本文将基于Kafka Connect 2.7.0版本的源码，深入剖析这一机制的实现原理和关键流程。

## Incremental Cooperative Rebalancing 概述

传统的Rebalance过程会在集群状态发生变化时,停止所有正在运行的Tasks，重新计算分配计划，然后在各个Worker，上重新启动 Tasks。这种"停止-重新分配-重启"的全量式操作可能会导致短暂的数据丢失或重复，并给集群带来较大的震荡。

Incremental Cooperative Rebalancing机制则采用了一种增量式的方式来进行Rebalance。在这种模式下，只有需要移动的 Tasks会被停止和重新分配，其他Tasks可以继续正常运行，从而最小化集群震荡和数据重复或丢失。

## Incremental Cooperative Rebalancing 流程源码分析

Incremental Cooperative Rebalancing机制的实现主要涉及 `Coordinator` 和 `Herder` 两个核心组件，我们从 `Coordinator` 的 `poll()` 方法入手，分析其关键流程。

```java
@Override
public void poll() {
    try {
        // 检查是否需要执行 Connector 分配
        if (needsConnectorAssignment()) {
            performConnectorAssignment();
        }

        // 执行 Rebalance
        if (needsRebuildPartitionAssignment()) {
            maybeRebuildPartitionAssignment();
        }

        // ... 其他逻辑
    } catch (...) {
        // ...
    }
}
```

在这个方法中,如果发现需要执行Rebalance，就会调用 `maybeRebuildPartitionAssignment()` 方法。我们继续深入这个方法:

```java
private void maybeRebuildPartitionAssignment() {
    // 获取活跃的 Worker 实例列表
    List<String> livingMembers = membership.getMembers();

    // 计算新的分配计划
    Map<String, WorkerAssignor.Assignment> newAssignment =
        workerAssignor.performAssignment(connectors, livingMembers);

    // 执行新的分配计划
    for (Map.Entry<String, WorkerAssignor.Assignment> entry : newAssignment.entrySet()) {
        String workerId = entry.getKey();
        WorkerAssignor.Assignment assignment = entry.getValue();

        // 向目标 Worker 发送分配请求
        sendAssignmentToWorker(workerId, assignment);
    }
}
```

在这个方法中，`Coordinator` 会首先获取集群中活跃的Worker实例列表，然后调用 `WorkerAssignor` 计算新的分配计划。与传统的Rebalance不同，在 Incremental Cooperative Rebalancing模式下，`WorkerAssignor` 会尝试最小化需要移动的Tasks数量，以减少集群震荡。

接下来,`Coordinator` 会遍历新的分配计划,向每个目标 Worker 发送分配请求。这个过程与传统 Rebalance 相同,但是由于需要移动的 Tasks 数量较少,因此对集群的影响也会相对较小。

现在,让我们继续跟踪 `Herder` 是如何响应和执行分配请求的。以 `Herder` 的 `updateAssignment()` 方法为例:

```java
public void updateAssignment(Assignment assignment) {
    // 获取已分配和未分配的 Connectors 和 Tasks
    Set<String> assigned = new HashSet<>(assignment.connectors());
    Set<ConnectorInfo> realConnectors = connectors.stream()
        .filter(info -> assigned.contains(info.name()))
        .collect(Collectors.toSet());

    Set<ConnectorInfo> unassigned = new HashSet<>(connectors);
    unassigned.removeAll(realConnectors);

    // 执行分配操作
    assignment.connectors().forEach(
        connectorName -> updateConnector(realConnectors, connectorName, assignment));
    unassigned.forEach(info -> revokeConnector(info));
}
```

在这个方法中,`Herder` 会首先获取已分配和未分配的 Connectors 和 Tasks 列表。对于已分配的 Connectors,`Herder` 会执行更新或创建操作;对于未分配的 Connectors,`Herder` 会执行撤销操作。

我们继续深入 `updateConnector()` 方法:

```java
private void updateConnector(Set<ConnectorInfo> connectors, String connectorName, Assignment assignment) {
    // 获取当前 Connector 的信息
    ConnectorInfo info = connectors.stream()
        .filter(c -> c.name().equals(connectorName))
        .findFirst()
        .orElseThrow(/* ... */);

    // 停止和恢复需要移动的 Tasks
    List<ConnectorTaskId> taskIds = restartingTasks(info, assignment.taskIds(connectorName));

    // 启动新的 Tasks
    startNewTasks(info, assignment.taskIds(connectorName), taskIds);
}
```

在这个方法中，`Herder` 会首先获取目标 Connector 的信息。然后,它会调用 `restartingTasks()` 方法停止和恢复需要移动的 Tasks。接下来，`Herder` 会调用 `startNewTasks()` 方法创建和启动新的 Task 实例。

通过上述源码分析，我们可以看到 Incremental Cooperative Rebalancing机制的核心思想是: 在执行Rebalance时，仅停止和重新分配那些需要移动的Tasks，而保留其他正常运行的Tasks不受影响。这种增量式的操作方式可以最小化集群震荡，提高数据传输的可靠性和连续性。

总的来说，Incremental Cooperative Rebalancing是Kafka Connect在 2.4版本中引入的一项重要优化，它进一步提升了 Connect在分布式环境下的稳定性和可用性。通过本文的源码分析，我们深入了解了这一机制的实现原理和关键流程。

在下一篇文章中,我们将总结Kafka Connect的整体工作流程，并回顾本系列文章所涵盖的核心内容。