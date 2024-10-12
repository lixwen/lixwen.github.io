---
title: Kafka Connect源码解读系列(九) - REST Client调用分析
date: 2023-05-12 23:22:10
tags: 
    - 技术
    - kafka
    - java
---

在上一篇文章中,我们分析了 Kafka Connect REST 服务端的实现原理,包括服务器初始化和 REST 资源处理器的注册过程。本文将继续围绕 REST Client 的使用,探讨如何通过客户端代码调用服务端提供的 REST API。

## REST Client 初始化

Kafka Connect 提供了 `ConnectRestClient` 类作为 REST Client 的入口点,我们来看一下它的初始化代码:

```java
public ConnectRestClient(String url) {
    this.url = url;
    this.client = ClientBuilder.newClient();
}
```

`ConnectRestClient` 的构造函数接收一个 URL 参数,表示 Connect 服务端的基础地址。内部使用 Jersey 客户端 API 创建了一个 `Client` 实例,用于发送 HTTP 请求。

## 发送 REST 请求

`ConnectRestClient` 提供了一组方法,用于调用不同的 REST 端点。以创建 Connector 为例,相关代码如下:

```java
public ConnectorInfo createConnector(ConnectorConfig config) {
    String path = "/connectors";
    Response response = client.target(url)
        .path(path)
        .request(MediaType.APPLICATION_JSON)
        .post(Entity.entity(config, MediaType.APPLICATION_JSON));

    // 处理响应
    int status = response.getStatus();
    if (status == HttpStatus.SC_OK) {
        return response.readEntity(ConnectorInfo.class);
    } else {
        throw new ConnectException("Failed to create connector", status);
    }
}
```

1. 构造请求路径 `/connectors`。
2. 使用 `Client` 实例发送 POST 请求,并在请求体中包含 `ConnectorConfig` 对象。
3. 处理服务端的响应,如果状态码为 200 (OK),则将响应体解析为 `ConnectorInfo` 对象返回;否则抛出异常。

对于其他类型的请求,如获取 Connector 列表、更新 Connector 配置等,`ConnectRestClient` 也提供了相应的方法,实现原理类似。

## REST Client 使用示例

下面是一个使用 `ConnectRestClient` 的简单示例:

```java
String connectUrl = "http://localhost:8083";
ConnectRestClient client = new ConnectRestClient(connectUrl);

// 创建 Connector
ConnectorConfig config = new ConnectorConfig(/* ... */);
ConnectorInfo info = client.createConnector(config);

// 获取 Connector 列表
List<ConnectorInfo> connectors = client.connectors();

// 更新 Connector 配置
config.update(/* ... */);
ConnectorInfo updatedInfo = client.updateConnector(config);

// 删除 Connector
client.deleteConnector(info.name());
```

1. 创建 `ConnectRestClient` 实例,指定 Connect 服务端地址。
2. 调用 `createConnector()` 方法创建一个新的 Connector。
3. 调用 `connectors()` 方法获取当前所有 Connector 的列表。
4. 更新 Connector 配置,并调用 `updateConnector()` 方法应用更改。
5. 调用 `deleteConnector()` 方法删除指定的 Connector。

通过上述源码分析和使用示例,我们了解了如何使用 `ConnectRestClient` 来与 Kafka Connect 的 REST 服务端进行交互,执行各种 Connector 管理和配置操作。结合前面对服务端实现的解读,我们已经全面讲解了 Kafka Connect REST API 的源码细节。

