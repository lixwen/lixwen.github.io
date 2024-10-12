---
title: Kafka Connect源码解读系列(八) - REST服务端实现分析
date: 2023-05-12 22:05:31
tags: 
    - 技术
    - kafka
    - java
---

Kafka Connect 提供了 REST 接口,用于外部客户端对 Connectors、Tasks 等资源进行管理和配置。这些 REST 接口由 Connect 进程内置的 HTTP 服务器提供,本文将围绕服务端的实现原理展开源码分析。

## REST 服务器初始化

REST 服务器的初始化发生在 `ConnectStandalone` 类的 `startConnector()` 方法中,代码如下:

```java
AdvertisedListener listener = createAdvertisedListener();
server = new ConnectWebServer(config, listener);
server.initializeServer();
server.initializeResources(herder, advertisedUrl);
```

1. 创建 `AdvertisedListener` 对象,用于监听外部请求。
2. 创建 `ConnectWebServer` 实例,作为 REST 服务器的入口点。
3. 调用 `initializeServer()` 方法初始化服务器。
4. 调用 `initializeResources()` 方法注册 REST 资源处理器。

接下来,我们重点分析 `ConnectWebServer` 中服务器初始化的相关实现。

## Jetty 服务器初始化

`ConnectWebServer` 内部使用 Jetty 作为嵌入式 Web 服务器,其初始化代码如下:

```java
public void initializeServer() {
    server = new Server();

    // 配置 Jetty 连接器
    ServerConnector connector = new ServerConnector(server);
    connector.setPort(port);
    server.setConnectors(new Connector[]{connector});

    servletContextHandler = new ServletContextHandler(ServletContextHandler.SESSIONS);
    servletContextHandler.setContextPath("/");
    server.setHandler(servletContextHandler);

    // 创建 Servlet 容器
    servletContainer = new ServletContainer();
    servletContainer.setServer(server);

    // 注册 Jersey 服务
    servletContainer.addServlet(new ServletHolder(new ResourceConfigProvider()), "/*");
    servletContextHandler.addServlet(servletContainer, "/*");
}
```

该方法执行以下关键步骤:

1. 创建 Jetty `Server` 实例。
2. 配置 Jetty `ServerConnector`。
3. 创建 `ServletContextHandler` 和 `ServletContainer`。
4. 注册 Jersey 服务,用于处理 REST 请求。

通过这些步骤,Jetty 服务器的基本框架就建立起来了。接下来,需要注册具体的 REST 资源处理器。

## REST 资源处理器注册

REST 资源处理器的注册发生在 `initializeResources()` 方法中,代码如下:

```java
public void initializeResources(Herder herder, String advertisedUrl) {
    // 创建 REST 扩展点
    ConnectRestExtension restExtension = new ConnectRestExtension(herder, advertisedUrl);

    // 注册 REST 资源处理器
    resourceProvider.registerResources(
        restExtension.getResourceClasses(),
        restExtension.getContextParams()
    );

    // 启动服务器
    server.start();
}
```

1. 创建 `ConnectRestExtension` 实例,作为 REST 扩展点。
2. 调用 `resourceProvider.registerResources()` 方法,注册 REST 资源处理器类。
3. 启动 Jetty 服务器。

`ConnectRestExtension` 负责提供所有需要注册的 REST 资源处理器类,这些类通过实现 `@Path` 注解来定义 REST 端点路径。例如:

```java
@Path("/connectors")
@ConnectRestExtension.ResourceCounter
public class ConnectorsResource {
    // ...
    @POST
    @Path("")
    public ConnectorInfo createConnector(
        @Suspended AsyncResponse asyncResponse,
        ConnectorConfig connectorConfig
    ) {
        // 处理创建 Connector 的请求
    }
}
```

`ConnectorsResource` 类中的 `createConnector()` 方法对应 `/connectors` 端点的 POST 请求,用于创建新的 Connector。

通过上述源码分析,我们了解了 Kafka Connect 是如何初始化内置的 REST 服务器,以及如何注册处理不同 REST 端点的资源处理器。在下一篇文章中,我们将继续探讨如何通过 REST Client 调用这些服务端 API。
