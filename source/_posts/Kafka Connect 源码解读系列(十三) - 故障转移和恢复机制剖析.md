---
title: Kafka Connect源码解读系列(十三) - 故障转移和恢复机制剖析
date: 2023-08-15 22:03:02
tags: 
    - 技术
    - kafka
    - java
---

在分布式环境中,故障是无法完全避免的。为了确保数据传输的可靠性和持续性,Kafka Connect 提供了强大的故障转移和恢复机制。本文将基于 Kafka Connect 2.7.0 版本的源码,深入剖析这一机制的实现原理和关键流程。

## 故障转移概述

在 Kafka Connect 中,故障转移主要发生在以下两种情况:

1. **Worker 实例故障** :当某个 Worker 实例出现故障或重启时,它负责的 Connectors 和 Tasks 需要被迁移到其他健康的 Worker 实例上。
2. **Task 实例故障** :当某个 Task 实例出现故障时,需要在同一个 Worker 上重新创建一个新的 Task 实例来接替它的工作。

无论哪种情况,故障转移的目标都是确保 Connectors 和 Tasks 的连续运行,避免数据丢失或重复。

## Worker 实例故障转移流程源码分析

当 Worker 实例出现故障时,Rebalance 机制会被触发,将该 Worker 上的 Connectors 和 Tasks 重新分配给其他健康的 Worker 实例。我们以 `Coordinator` 的 `poll()` 方法为切入点,分析这一过程的关键源码实现。

```java
@Override
public void poll() {
    try {
        // 检查是否需要重建分配计划
        maybeResolveMissingCoordinators();
        maybeRefreshCoordinatorMetrics();
        maybeRefreshFencedInstances();

        // 如果需要重建分配计划,则执行 Rebalance
        if (needsRebuildPartitionAssignment()) {
            maybeRebuildPartitionAssignment();
        }

        // 执行新的分配计划
        if (needsConnectorAssignment()) {
            performConnectorAssignment();
        }
    } catch (...) {
        // ...
    }
}
```

在这个方法中,`Coordinator` 会首先检查集群状态,如果发现有 Worker 实例出现故障,就会触发 `maybeRebuildPartitionAssignment()` 方法重新计算分配计划。

接下来,我们继续深入 `maybeRebuildPartitionAssignment()` 方法:

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

在这个方法中,`Coordinator` 会首先获取集群中活跃的 Worker 实例列表,然后调用 `WorkerAssignor` 计算新的分配计划。计算过程中,已故障的 Worker 实例会被排除在外,其负责的 Connectors 和 Tasks 将被重新分配给其他健康的 Worker 实例。

最后,`Coordinator` 会遍历新的分配计划,向每个目标 Worker 发送分配请求,完成故障转移过程。

## Task 实例故障恢复流程源码分析

当某个 Task 实例出现故障时,Kafka Connect 会在同一个 Worker 上重新创建一个新的 Task 实例来接替它的工作。这个过程由 `Herder` 组件负责,我们来看一下 `Herder` 的 `updateConnector()` 方法:

```java
private void updateConnector(Set<ConnectorInfo> connectors, String connectorName, Assignment assignment) {
    // 获取当前 Connector 的信息
    ConnectorInfo info = connectors.stream()
        .filter(c -> c.name().equals(connectorName))
        .findFirst()
        .orElseThrow(/* ... */);

    // 停止和恢复故障 Tasks
    List<ConnectorTaskId> taskIds = restartingTasks(info, assignment.taskIds(connectorName));

    // 启动新的 Tasks
    startNewTasks(info, assignment.taskIds(connectorName), taskIds);
}
```

在这个方法中,`Herder` 会首先获取故障 Connector 的信息。然后,它会调用 `restartingTasks()` 方法停止和恢复故障的 Tasks。接下来,`Herder` 会调用 `startNewTasks()` 方法创建和启动新的 Task 实例。

我们继续深入 `restartingTasks()` 方法:

```java
private List<ConnectorTaskId> restartingTasks(ConnectorInfo info, Set<ConnectorTaskId> activeTaskIds) {
    // 获取需要重启的 Task 列表
    List<ConnectorTaskId> restartingTasks = new ArrayList<>();
    for (ConnectorTaskId taskId : info.taskIds()) {
        if (!activeTaskIds.contains(taskId)) {
            info.taskStatus(taskId).state(TaskStatus.State.RESTARTING);
            restartingTasks.add(taskId);
        }
    }

    // 停止故障 Tasks
    restartingTasks.forEach(taskId -> info.taskStatus(taskId).state(TaskStatus.State.RESTARTING));
    restartingTasks.forEach(taskId -> info.taskStatus(taskId).state(TaskStatus.State.UNSTARTED));

    return restartingTasks;
}
```

在这个方法中,`Herder` 会首先获取需要重启的 Task 列表。对于每个故障的 Task,`Herder` 会更新它的状态为 `RESTARTING`。然后,`Herder` 会停止这些故障的 Tasks,并将它们的状态更新为 `UNSTARTED`。

接下来,我们继续看 `startNewTasks()` 方法:

```java
private void startNewTasks(ConnectorInfo info, Set<ConnectorTaskId> taskIds, List<ConnectorTaskId> restartingTasks) {
    // 创建新的 Task 实例
    for (ConnectorTaskId taskId : taskIds) {
        if (!restartingTasks.contains(taskId)) {
            info.taskStatus(taskId).state(TaskStatus.State.STARTING);
            startTask(info, taskId);
        }
    }
}
```

在这个方法中,`Herder` 会遍历所有需要创建的新 Task。对于每个新的 Task,`Herder` 会更新它的状态为 `STARTING`,然后调用 `startTask()` 方法创建和启动新的 Task 实例。

通过上述源码分析,我们深入了解了 Kafka Connect 是如何实现故障转移和恢复机制的。在 Worker 实例故障时,Rebalance 机制会被触发,将该 Worker 上的 Connectors 和 Tasks 重新分配给其他健康的 Worker 实例。而在 Task 实例故障时,`Herder` 会在同一个 Worker 上停止故障的 Task,并重新创建新的 Task 实例来接替它的工作。这些机制确保了数据传输的可靠性和持续性,提高了 Kafka Connect 的可用性和容错能力。

在下一篇文章中,我们将继续探讨 Kafka Connect 中另一个重要的特性 —— Incremental Cooperative Rebalancing。