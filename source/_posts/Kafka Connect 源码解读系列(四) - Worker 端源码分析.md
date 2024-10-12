---
title: Kafka Connect源码解读系列(四) - Worker端源码分析
date: 2023-03-15 21:03:25
tags: 
    - 技术
    - kafka
    - java
---


## **Worker 启动流程源码走查**

Worker 的启动入口位于 `org.apache.kafka.connect.cli.ConnectStandalone` 类的 `main` 方法中。让我们从这里开始追踪 Worker 的启动流程:

```java
public static void main(String[] args) {    
    // ... 解析命令行参数和配置属性       
    // 创建 Herder 实例
    ConnectStandalone ConnectStandalone = new ConnectStandalone(props);        
    // 启动 Herder   
    try {        
        ConnectStandalone.startConnector();    
    } finally {        
        // ...   
    }
}
```

在 `ConnectStandalone` 构造函数中,会初始化一些辅助组件,如 Kafka AdminClient、MetricsReporter 等:

```java
public ConnectStandalone(Map<String, String> props) {
  // ... 初始化配置属性
  
  String remoteHost = ConfigUtils.getRemoteHost(config);
  replicaListener = new Listener(config);
  
  // 创建 AdminClient
  adminClient = AdminClient.create(adminClientConfigOverrides(config));
  
  // 创建 MetricsReporter
  this.metrics = ConfigUtils.getMetrics("Connect");
  this in.forEachRemaining(m -> addMetric(m, overallMetrics));
  String metricReporters = ConfigUtils.getMetricReporters(config);
  reporters = ConfigUtils.initializeMetricReporters(metricReporters);
  this.config = config;

  // 加载 Connector 插件
  plugins = ConfigUtils.plugins(config.getClass().getClassLoader());
}
```

接着在 `startConnector()` 方法中,会进一步初始化 Herder 并启动它:

```java
void startConnector() {    
    String workerId = ConfigUtils.getWorkerId(config);    
    // 创建 Herder 实例              
    herder = new Herder(config, workerId, replicaListener, metrics, plugins);    
    // 加入集群并启动 Herder    
    herder.start();    
    // ... 启动 REST 服务等其他组件*
}
```

总的来说,Worker 启动的核心步骤包括:

1. 解析配置参数
2. 创建 Herder 及其他辅助组件
3. 加载 Connector 插件
4. 启动 Herder 加入集群

接下来,我们将重点分析 Herder 这个 Worker 的大脑组件。

## **Herder 核心类源码分析**

Herder 作为 Worker 的核心管理组件,负责管理和协调 Connectors 和 Tasks 的整个生命周期。我们从以下几个方面来剖析 Herder 的实现:

### **1. Herder 的重要字段**

```java
public Herder(ConfigDef configDef, String workerId, Listener listener, MetricsReporter metrics, PluginClassLoader pluginLoader) {    
    this.time = new WorkerTime();    
    // 创建 ConnectorClientConfigOverridePolicy     
    this.configTransformer = new WorkerConfigTransformer(configDef);    
    this.workerId = workerId;    
    // 创建 DistributedHerderProvider     
    this.federatedConfig = FederatedConfig.create(configDef);    
    this.workerGroup = federatedConfig.getWorkerId();    
    this.herderProvider = new DistributedHerderProvider(federatedConfig, listener, metrics, time);    
    // 创建 Worker Configurator   
    WorkerConfigurator configurator = configurator(configDef);    
    // 创建 ConnectMetrics 实例    
    this.metrics = new ConnectMetrics(workerId, configurator, time);    
    // 加载 Connector 插件    
    this.plugins = plugins(pluginLoader);    
    // 创建 ConnectorClientConfigOverridePolicy   
    this.configTransformer = new WorkerConfigTransformer(configDef);    
    // 创建 Coordinator     
    this.coordinator = herderProvider.getcoordinator(metrics);
}
```

Herder 中有几个关键字段:

- `DistributedHerderProvider`：管理 Worker 集群中的各个 Herder 实例。
- `ConnectMetrics`：收集和报告 Connect 相关的指标数据。
- `Plugins`：存储加载的 Connector 插件。
- `Coordinator`：负责协调 Connectors 和 Tasks 在 Worker 集群中的分布。

### **2. Herder 启动过程**

在 `herder.start()` 方法中,会执行以下核心逻辑:

```java
public void start() {    
    log.info("Herder starting");        
    // 启动 DistributedHerderProvider    
    herderProvider.startProvider();        
    // 启动 Coordinator   
    coordinator.startCoordinator();        
    //...        
    // 加入集群并注册成为 Leader 候选者    
    coordinator.joinGroup();    
    //...
}
```

1. 启动 `DistributedHerderProvider` 和 `Coordinator`。
2. 调用 `Coordinator` 的 `joinGroup()` 方法,将当前 Worker 加入集群,并注册为 Leader 候选者。

在加入集群后,Herder 会循环执行 `poll` 操作,监听并处理集群事件和请求。

### **3. Rebalance 机制**

Rebalance 是 Kafka Connect 实现高可用和负载均衡的关键机制。在 Herder 中,Rebalance 主要由 `Coordinator` 组件驱动,核心实现位于 `Coordinator` 的 `poll` 方法中:

```java
@Override
public void poll() {    
    try {        
        // 检查是否需要 Rebalance        
        maybeResolveMissingCoordinators();        
        maybeRefreshCoordinatorMetrics();        
        maybeRefreshFencedInstances();        
        if (needsRebuildPartitionAssignment()) {            
            // 重新计算 Connectors 和 Tasks 在 Workers 间的分布            
            maybeRebuildPartitionAssignment();        
        } 
        if (needsConnectorAssignment()) {            
            // 基于新的分布计划,执行 Connector 和 Task 的重新分配            
            performConnectorAssignment();        
        }    
	} catch (...) {        
        //...  
    }
}
```

主要步骤包括:

1. 检查是否需要执行 Rebalance。
2. 重新计算 Connectors 和 Tasks 在集群中的分布。
3. 根据分布计划,向相关 Workers 发送分配或回收 Connectors 和 Tasks 的请求。

这个过程由 Herder 持续轮询执行,以确保集群中的工作负载能够及时重新分布。

除了对 Herder 的核心实现进行分析,本文还展示了 Worker 启动流程和 Rebalance 机制的关键源码片段。在下一篇文章中,我们将继续深入探讨 Connector 端的实现原理。