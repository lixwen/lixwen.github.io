## 1. 问题背景

某个模型检测服务在消费`RocketMQ`的`FIFO`消息的时候出现了重复消费的情况，相同`MessageId`的消息消费了多次。

`RocketMQ`版本：`5.0.0`
`Java SDK`版本：`5.0.4`
`Topic`: 设置了`FIFO` 模式
`ConsumerGroup` :  开启`consumeOrderlyEnable` ，关闭 `consumeBroadcastEnable` 
`consumptionThreadCount` ：设置为1

## 2. 背景源码分析

基于`RocketMQ Java Client 5.0.4 SDK` 的相关代码进行分析

---

### 2.1 `PushConsumer` “`Push`” 消息流程分析

在`RocketMQ 5.0.0` 中 `PushConsumer` 本质上是基于主动`Pull` 的方式来实现消息消费的。

---

**`PushConsumer` 初始化**

在 `com.d5.framework.rocketmq.core.consumer.RocketMqMessageListenerBeanPostProcessor` 中，公司内部封装了 `PushConsumer` 初始化的代码。

```java
public class RocketMqMessageListenerBeanPostProcessor implements BeanPostProcessor, EnvironmentAware, ApplicationContextAware {
    // ...
    private void createRocketMqCustomerBean(String topic, String consumerGroup, String tag, RocketMqListener listener, String consumerName, int consumerThreadCount) {
        FilterExpression filterExpression = new FilterExpression(tag, FilterExpressionType.TAG);

        try {
            ClientServiceProvider provider = (ClientServiceProvider)this.applicationContext.getBean(ClientServiceProvider.class);
            ClientConfiguration clientConfiguration = (ClientConfiguration)this.applicationContext.getBean(ClientConfiguration.class);
            provider.newPushConsumerBuilder().setClientConfiguration(clientConfiguration).setConsumerGroup(consumerGroup).setSubscriptionExpressions(Collections.singletonMap(topic, filterExpression)).setMessageListener((messageView) -> {
                String traceId = IdUtil.fastSimpleUUID();
                String spanId = IdUtil.fastSimpleUUID();
                MDC.put("traceId", traceId);
                MDC.put("spanId", spanId);

                ConsumeResult var6;
                try {
                    Object body = this.doConvertMessage(messageView, listener);
                    log.info("Consume Message Received, messageId: {}, messageBody: {}, deliveryAttempt:{}",
                            messageView.getMessageId(), JSONObject.toJSONString(body), messageView.getDeliveryAttempt());
                    listener.onMessage(body);
                    log.info("Consume Message Successfully, messageId: {}", messageView.getMessageId());
                    return ConsumeResult.SUCCESS;
                } catch (Exception e) {
                    log.error("Consume Message Failed, messageId is {}", messageView.getMessageId(), e);
                    var6 = ConsumeResult.FAILURE;
                } finally {
                    MDC.remove(traceId);
                    MDC.remove(spanId);
                }

                return var6;
            }).setConsumptionThreadCount(consumerThreadCount).build();  // 此处会 build PushConsumer 对象，并启动消费相关线程 
            log.info("================RocketMq customer init success:{}===================", consumerName);
        } catch (ClientException e) {
            throw new RuntimeException("a consumer as " + consumerName + " registerBean fail, " + e.getMessage());
        }
    }
    // ...
}

// PushConsumerImpl 实现类相关代码
public class PushConsumerBuilderImpl implements PushConsumerBuilder {
    /**
     * @see PushConsumerBuilder#build()
     */
    @Override
    public PushConsumer build() throws ClientException {
        checkNotNull(clientConfiguration, "clientConfiguration has not been set yet");
        checkNotNull(consumerGroup, "consumerGroup has not been set yet");
        checkNotNull(messageListener, "messageListener has not been set yet");
        checkArgument(!subscriptionExpressions.isEmpty(), "subscriptionExpressions have not been set yet");
        // 实例初始化
        final PushConsumerImpl pushConsumer = new PushConsumerImpl(clientConfiguration, consumerGroup,
            subscriptionExpressions, messageListener, maxCacheMessageCount, maxCacheMessageSizeInBytes,
            consumptionThreadCount);
            
        // 启动 consumer
        pushConsumer.startAsync().awaitRunning();
        return pushConsumer;
    }
}

class PushConsumerImpl extends ConsumerImpl implements PushConsumer {
		 // 初始化相关代码
     public PushConsumerImpl(ClientConfiguration clientConfiguration, String consumerGroup,
        Map<String, FilterExpression> subscriptionExpressions, MessageListener messageListener,
        int maxCacheMessageCount, int maxCacheMessageSizeInBytes, int consumptionThreadCount) {
        super(clientConfiguration, consumerGroup, subscriptionExpressions.keySet());
        this.clientConfiguration = clientConfiguration;
        Resource groupResource = new Resource(consumerGroup);
        // 初始化订阅相关配置参数对象
        this.pushSubscriptionSettings = new PushSubscriptionSettings(clientId, endpoints, groupResource,
            clientConfiguration.getRequestTimeout(), subscriptionExpressions);
        this.consumerGroup = consumerGroup;
        this.subscriptionExpressions = subscriptionExpressions;
        this.cacheAssignments = new ConcurrentHashMap<>();
        this.messageListener = messageListener;
        this.maxCacheMessageCount = maxCacheMessageCount;
        this.maxCacheMessageSizeInBytes = maxCacheMessageSizeInBytes;

        this.receptionTimes = new AtomicLong(0);
        this.receivedMessagesQuantity = new AtomicLong(0);
        this.consumptionOkQuantity = new AtomicLong(0);
        this.consumptionErrorQuantity = new AtomicLong(0);

        this.processQueueTable = new ConcurrentHashMap<>();
				// 消费线程池初始化，对于FIFO模式，consumptionThreadCount设置的是1
        this.consumptionExecutor = new ThreadPoolExecutor(
            consumptionThreadCount,
            consumptionThreadCount,
            60,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(),
            new ThreadFactoryImpl("MessageConsumption", this.getClientId().getIndex()));
    }

		// PushConsumerImpl 启动相关代码
    @Override
    protected void startUp() throws Exception {
      try {
          log.info("Begin to start the rocketmq push consumer, clientId={}", clientId);
          GaugeObserver gaugeObserver = new ProcessQueueGaugeObserver(processQueueTable, clientId, consumerGroup);
          this.clientMeterManager.setGaugeObserver(gaugeObserver);
          
	        // 调用ClientImpl的startUp方法
	        //     @Override
				  //   protected void startUp() throws Exception {
				  //      log.info("Begin to start the rocketmq client, clientId={}", clientId);
				  //      this.clientManager.startAsync().awaitRunning();  // 初始化 ClientManagerImpl，后续所有和Broker相关的通信都是执行这个类中的方法
				  //      // Fetch topic route from remote.
				  //      log.info("Begin to fetch topic(s) route data from remote during client startup, clientId={}, topics={}",
				  //          clientId, topics);
				  //      for (String topic : topics) {
				  //          final ListenableFuture<TopicRouteData> future = fetchTopicRoute(topic); // 获取topic路由配置
				  //          future.get();
				  //      }
				  //      log.info("Fetch topic route data from remote successfully during startup, clientId={}, topics={}",
				  //          clientId, topics);
				  //      // Update route cache periodically.
				  //      final ScheduledExecutorService scheduler = clientManager.getScheduler();
				  //      this.updateRouteCacheFuture = scheduler.scheduleWithFixedDelay(() -> {
				  //          try {
				  //              updateRouteCache();
				  //          } catch (Throwable t) {
				  //              log.error("Exception raised while updating topic route cache, clientId={}", clientId, t);
				  //          }
				  //      }, 10, 30, TimeUnit.SECONDS);
				  //     log.info("The rocketmq client starts successfully, clientId={}", clientId);
			    //  }
			    super.startUp();
          final ScheduledExecutorService scheduler = this.getClientManager().getScheduler();
          // 创建 consumerService，是使用 FIFO Consumer
          //  private ConsumeService createConsumeService() {
				  //      final ScheduledExecutorService scheduler = this.getClientManager().getScheduler();
			    //      // 判断当前消费模式是否是 FIFO，是从Broker端返回的配置来进行判断
			    //      if (pushSubscriptionSettings.isFifo()) {
			    //        log.info("Create FIFO consume service, consumerGroup={}, clientId={}", consumerGroup, clientId);
			    //        return new FifoConsumeService(clientId, messageListener, consumptionExecutor, this, scheduler);
			    //      }
			    //      log.info("Create standard consume service, consumerGroup={}, clientId={}", consumerGroup, clientId);
			    //      return new StandardConsumeService(clientId, messageListener, consumptionExecutor, this, scheduler);
			    //  }
          this.consumeService = createConsumeService();
          // Scan assignments periodically. 如果出现了 Rebalance，会重新生成 Assignments
          scanAssignmentsFuture = scheduler.scheduleWithFixedDelay(() -> {
              try {
                  scanAssignments(); // 获取Assignments
              } catch (Throwable t) {
                  log.error("Exception raised while scanning the load assignments, clientId={}", clientId, t);
              }
          }, 1, 5, TimeUnit.SECONDS);
          log.info("The rocketmq push consumer starts successfully, clientId={}", clientId);
      } catch (Throwable t) {
          log.error("Exception raised while starting the rocketmq push consumer, clientId={}", clientId, t);
          shutDown();
          throw t;
      }
    }
}
```

---

**`PushConsumer` 拉取消息相关代码**

继续看`PushConsumerImpl#scanAssignments` 的相关代码

```java
class PushConsumerImpl extends ConsumerImpl implements PushConsumer {
		// ...
		void scanAssignments() {
		    try {
		        log.debug("Start to scan assignments periodically, clientId={}", clientId);
		        for (Map.Entry<String, FilterExpression> entry : subscriptionExpressions.entrySet()) {
		            final String topic = entry.getKey();
		            final FilterExpression filterExpression = entry.getValue();
		            final Assignments existed = cacheAssignments.get(topic);
		            // 异步获取Assignments
		            final ListenableFuture<Assignments> future = queryAssignment(topic);
		            Futures.addCallback(future, new FutureCallback<Assignments>() {
		                @Override
		                public void onSuccess(Assignments latest) {
		                    if (latest.getAssignmentList().isEmpty()) {
		                        if (null == existed || existed.getAssignmentList().isEmpty()) {
		                            log.info("Acquired empty assignments from remote, would scan later, topic={}, "
		                                + "clientId={}", topic, clientId);
		                            return;
		                        }
		                        log.info("Attention!!! acquired empty assignments from remote, but existed assignments"
		                            + " is not empty, topic={}, clientId={}", topic, clientId);
		                    }
												// 假设新增了一个消费者，Assignments发生变更，会执行这个逻辑
		                    if (!latest.equals(existed)) {
		                        log.info("Assignments of topic={} has changed, {} => {}, clientId={}", topic, existed,
		                            latest, clientId);
		                        syncProcessQueue(topic, latest, filterExpression);
		                        cacheAssignments.put(topic, latest);
		                        return;
		                    }
		                    log.debug("Assignments of topic={} remains the same, assignments={}, clientId={}", topic,
		                        existed, clientId);
		                    // Process queue may be dropped, need to be synchronized anyway.
		                    // syncProcessQueue
		                    // 同步MessageQueue，收取消息的代码在这个流程中
		                    syncProcessQueue(topic, latest, filterExpression);
		                }
		
		                @Override
		                public void onFailure(Throwable t) {
		                    log.error("Exception raised while scanning the assignments, topic={}, clientId={}", topic,
		                        clientId, t);
		                }
		            }, MoreExecutors.directExecutor());
		        }
		    } catch (Throwable t) {
		        log.error("Exception raised while scanning the assignments for all topics, clientId={}", clientId, t);
		    }
		}
		
		// ...
    @VisibleForTesting
    void syncProcessQueue(String topic, Assignments assignments, FilterExpression filterExpression) {
        Set<MessageQueueImpl> latest = new HashSet<>();

        final List<Assignment> assignmentList = assignments.getAssignmentList();
        for (Assignment assignment : assignmentList) {
            latest.add(assignment.getMessageQueue());
        }
				// 获取有效队列
        Set<MessageQueueImpl> activeMqs = new HashSet<>();

        for (Map.Entry<MessageQueueImpl, ProcessQueue> entry : processQueueTable.entrySet()) {
            final MessageQueueImpl mq = entry.getKey();
            final ProcessQueue pq = entry.getValue();
            if (!topic.equals(mq.getTopic())) {
                continue;
            }

            if (!latest.contains(mq)) {
                log.info("Drop message queue according to the latest assignmentList, mq={}, clientId={}", mq,
                    clientId);
                dropProcessQueue(mq);
                continue;
            }

            if (pq.expired()) {
                log.warn("Drop message queue because it is expired, mq={}, clientId={}", mq, clientId);
                dropProcessQueue(mq);
                continue;
            }
            activeMqs.add(mq);
        }

        for (MessageQueueImpl mq : latest) {
            if (activeMqs.contains(mq)) {
                continue;
            }
            // 创建ProcessQueue
            // Process queue is a cache to store fetched messages from remote for {@link PushConsumer}
            final Optional<ProcessQueue> optionalProcessQueue = createProcessQueue(mq, filterExpression);
            if (optionalProcessQueue.isPresent()) {
                log.info("Start to fetch message from remote, mq={}, clientId={}", mq, clientId);
                // ProcessQueue 开始拉取消息
                optionalProcessQueue.get().fetchMessageImmediately();
            }
        }
    }
    
    // ...
}
```

接下来继续查看 `ProcessQueueImpl` 的`fetchMessageImmediately` 相关代码。

```java
class ProcessQueueImpl implements ProcessQueue {

		@Override
    public void fetchMessageImmediately() {
        receiveMessageImmediately();
    }

    private void receiveMessageImmediately() {
        final ClientId clientId = consumer.getClientId(); // 获取client id
        if (!consumer.isRunning()) {
            log.info("Stop to receive message because consumer is not running, mq={}, clientId={}", mq, clientId);
            return;
        }
        try {
            final Endpoints endpoints = mq.getBroker().getEndpoints();
            final int batchSize = this.getReceptionBatchSize(); //获取每次拉取消息的数量大小，目前D5的sdk不可以调整
            final ReceiveMessageRequest request = consumer.wrapReceiveMessageRequest(batchSize, mq, filterExpression);
            activityNanoTime = System.nanoTime();

            // Intercept before message reception.
            final MessageInterceptorContextImpl context = new MessageInterceptorContextImpl(MessageHookPoints.RECEIVE);
            consumer.doBefore(context, Collections.emptyList()); // 执行前置拦截器，DEBUG 发现只有一个MessageMeterInceptor，默认未开启

            final ListenableFuture<ReceiveMessageResult> future = consumer.receiveMessage(request, mq,
                consumer.getPushConsumerSettings().getLongPollingTimeout()); // PushConsumerImpl 对象开始从 Broker 拉消息
            Futures.addCallback(future, new FutureCallback<ReceiveMessageResult>() {
                @Override
                public void onSuccess(ReceiveMessageResult result) {
                    // Intercept after message reception.
                    // 从 MessageViewImpl 转为 GeneralMessage 对象
                    final List<GeneralMessage> generalMessages = result.getMessageViewImpls().stream()
                        .map((Function<MessageView, GeneralMessage>) GeneralMessageImpl::new)
                        .collect(Collectors.toList());
                    final MessageInterceptorContextImpl context0 =
                        new MessageInterceptorContextImpl(context, MessageHookPointsStatus.OK);
                    // 执行消息接受的后置拦截器
                    consumer.doAfter(context0, generalMessages);

                    try {
                        onReceiveMessageResult(result);
                    } catch (Throwable t) {
                        // Should never reach here.
                        log.error("[Bug] Exception raised while handling receive result, mq={}, endpoints={}, "
                            + "clientId={}", mq, endpoints, clientId, t);
                        onReceiveMessageException(t);
                    }
                }

                @Override
                public void onFailure(Throwable t) {
                    // Intercept after message reception.
                    final MessageInterceptorContextImpl context0 =
                        new MessageInterceptorContextImpl(context, MessageHookPointsStatus.ERROR);
                    consumer.doAfter(context0, Collections.emptyList());

                    log.error("Exception raised during message reception, mq={}, endpoints={}, clientId={}", mq,
                        endpoints, clientId, t);
                    onReceiveMessageException(t);
                }
            }, MoreExecutors.directExecutor());
            receptionTimes.getAndIncrement();
            consumer.getReceptionTimes().getAndIncrement();
        } catch (Throwable t) {
            log.error("Exception raised during message reception, mq={}, clientId={}", mq, clientId, t);
            onReceiveMessageException(t);
        }
    }

}
```

接下来开始关注: `consumer.receiveMessage` 方法

```java
	
abstract class ConsumerImpl extends ClientImpl {

	
	// ReceiveMessageRequest 对象包含 group_、filterExpression、messageQueue_、batchSize_、bitField0_等信息处理
	// 详情可以参考 apache.rocketmq.v2.ReceiveMessageRequest 对象
	// awaitDuration = 30s
	protected ListenableFuture<ReceiveMessageResult> receiveMessage(ReceiveMessageRequest request,
	    MessageQueueImpl mq, Duration awaitDuration) {
	    List<MessageViewImpl> messages = new ArrayList<>();
	    try {
	        final Endpoints endpoints = mq.getBroker().getEndpoints();
	        final Duration tolerance = clientConfiguration.getRequestTimeout(); // D5 在封装SDK中设置了 30s
	        final Duration timeout = awaitDuration.plus(tolerance);
	        // rocketmq 封装的 client 与 broker 通信的对象
	        // public abstract class ClientImpl extends AbstractIdleService implements Client, ClientSessionHandler, MessageInterceptor
	        final ClientManager clientManager = this.getClientManager(); 
	        // 从 broker 拉取消息 timeout = 60s，虽然此处是 60s，但是 broker 端设置的 longPollTimeout 时间是 30s，所以现象是大约 30s 会拉一次消息
	        final RpcFuture<ReceiveMessageRequest, List<ReceiveMessageResponse>> future =
	            clientManager.receiveMessage(endpoints, request, timeout);
	            
	        // 执行 transform
	        return Futures.transformAsync(future, responses -> {
	            
	            Status status = Status.newBuilder().setCode(Code.INTERNAL_SERVER_ERROR)
	                .setMessage("status was not set by server")
	                .build();
	            Long transportDeliveryTimestamp = null;
	            List<Message> messageList = new ArrayList<>();
	            for (ReceiveMessageResponse response : responses) {
	                switch (response.getContentCase()) {
	                    case STATUS:
	                        status = response.getStatus();
	                        break;
	                    case MESSAGE:
	                        // 类型是消息，放入消息集合
	                        messageList.add(response.getMessage());
	                        break;
	                    case DELIVERY_TIMESTAMP:
	                        final Timestamp deliveryTimestamp = response.getDeliveryTimestamp();
	                        transportDeliveryTimestamp = Timestamps.toMillis(deliveryTimestamp);
	                        break;
	                    default:
	                        log.warn("[Bug] Not recognized content for receive message response, mq={}, " +
	                            "clientId={}, response={}", mq, clientId, response);
	                }
	            }
	            for (Message message : messageList) {
	                // 从protobuf 转换为 MessageViewImpl
	                final MessageViewImpl view = MessageViewImpl.fromProtobuf(message, mq, transportDeliveryTimestamp);
	                messages.add(view);
	            }
	            StatusChecker.check(status, future);
	            final ReceiveMessageResult receiveMessageResult = new ReceiveMessageResult(endpoints, messages);
	            return Futures.immediateFuture(receiveMessageResult);
	        }, MoreExecutors.directExecutor());
	    } catch (Throwable t) {
	        // Should never reach here.
	        log.error("[Bug] Exception raised during message receiving, mq={}, clientId={}", mq, clientId, t);
	        return Futures.immediateFailedFuture(t);
	    }
	}
}
```

继续查看 `clientManager.receiveMessage` 相关代码

```java
public class ClientManagerImpl extends ClientManager {
    // ...    
    public RpcFuture<ReceiveMessageRequest, List<ReceiveMessageResponse>> receiveMessage(Endpoints endpoints,
        ReceiveMessageRequest request, Duration duration) {
        try {
            final Metadata metadata = client.sign();
            final Context context = new Context(endpoints, metadata);
            
            // 从 rpcClientTable 中根据 endpoints 获取对应的 rpcClient，此处使用了读写锁来处理并发问题
            final RpcClient rpcClient = getRpcClient(endpoints);
            final ListenableFuture<List<ReceiveMessageResponse>> future =
                rpcClient.receiveMessage(metadata, request, asyncWorker, duration);
            return new RpcFuture<>(context, request, future);
        } catch (Throwable t) {
            return new RpcFuture<>(t);
        }
    }
}

// rpc 客户端初始化逻辑如下，感兴趣的可以研究 rocketmq 封装的 MessagingServiceGrpc 对象
public RpcClientImpl(Endpoints endpoints) throws SSLException {
    final SslContextBuilder builder = GrpcSslContexts.forClient();
    builder.trustManager(InsecureTrustManagerFactory.INSTANCE);
    SslContext sslContext = builder.build();
		// 创建 netty 的 channel
    final NettyChannelBuilder channelBuilder =
        NettyChannelBuilder.forTarget(endpoints.getGrpcTarget())
            .withOption(ChannelOption.CONNECT_TIMEOUT_MILLIS, CONNECT_TIMEOUT_MILLIS)
            .maxInboundMessageSize(GRPC_MAX_MESSAGE_SIZE)
            .intercept(LoggingInterceptor.getInstance())
            .sslContext(sslContext);
    // Disable grpc's auto-retry here.
    channelBuilder.disableRetry();
    final List<InetSocketAddress> socketAddresses = endpoints.toSocketAddresses();
    if (null != socketAddresses) {
        final IpNameResolverFactory ipNameResolverFactory = new IpNameResolverFactory(socketAddresses);
        channelBuilder.nameResolverFactory(ipNameResolverFactory);
    }
    this.channel = channelBuilder.build();
    this.futureStub = MessagingServiceGrpc.newFutureStub(channel);
    this.stub = MessagingServiceGrpc.newStub(channel);
    this.activityNanoTime = System.nanoTime();
}

```

继续查看`onReceiveMessageResult` 相关代码，里面是接收到消息之后的处理逻辑

```java
private void onReceiveMessageResult(ReceiveMessageResult result) {
    final List<MessageViewImpl> messages = result.getMessageViewImpls();
    if (!messages.isEmpty()) {
        // 消息不为空时
        cacheMessages(messages);
        receivedMessagesQuantity.getAndAdd(messages.size());
        consumer.getReceivedMessagesQuantity().getAndAdd(messages.size());
        // consumer service 开始消费消息，对于 FIFO 队列，进入  FifoConsumeService 执行消费逻辑
        consumer.getConsumeService().consume(this, messages);
    }
    // 继续收取新的消息
    receiveMessage();
}

// FIFO 消费逻辑
class FifoConsumeService extends ConsumeService {
  private static final Logger log = LoggerFactory.getLogger(FifoConsumeService.class);

  public FifoConsumeService(ClientId clientId, MessageListener messageListener,
      ThreadPoolExecutor consumptionExecutor, MessageInterceptor messageInterceptor,
      ScheduledExecutorService scheduler) {
      super(clientId, messageListener, consumptionExecutor, messageInterceptor, scheduler);
  }

  @Override
  public void consume(ProcessQueue pq, List<MessageViewImpl> messageViews) {
      consumeIteratively(pq, messageViews.iterator());
  }

  public void consumeIteratively(ProcessQueue pq, Iterator<MessageViewImpl> iterator) {
      if (!iterator.hasNext()) {
          return;
      }
      final MessageViewImpl messageView = iterator.next();
      if (messageView.isCorrupted()) {
          // Discard corrupted message.
          log.error("Message is corrupted for FIFO consumption, prepare to discard it, mq={}, messageId={}, "
              + "clientId={}", pq.getMessageQueue(), messageView.getMessageId(), clientId);
          pq.discardFifoMessage(messageView);
          consumeIteratively(pq, iterator);
          return;
      }
      // 执行消费逻辑，创建ConsumeTask，并且 submit task，返回 ConsumeResult
      final ListenableFuture<ConsumeResult> future0 = consume(messageView);
      
      // eraseFifoMessage 逻辑如下： 根据消息消费结果（成功/失败）和重试次数，决定：
      // 是否重试（在本地队列中重试）
      // 是否确认 Ack（成功）
      // 是否丢入死信队列（超过最大重试次数）
      // 并在完成后将消息从缓存队列中驱逐（evictCache）
      ListenableFuture<Void> future = Futures.transformAsync(future0, result -> pq.eraseFifoMessage(messageView,
          result), MoreExecutors.directExecutor());
          
      // 继续消费下一条消息
      future.addListener(() -> consumeIteratively(pq, iterator), MoreExecutors.directExecutor());
  }
}

class ProcessQueueImpl implements ProcessQueue {
    public ListenableFuture<Void> eraseFifoMessage(MessageViewImpl messageView, ConsumeResult consumeResult) {
        // 判断消费结果，记录 consumer.consumptionOkQuantity 或者 consumer.consumptionErrorQuantity
        statsConsumptionResult(consumeResult);
        final RetryPolicy retryPolicy = consumer.getRetryPolicy();
        // 获取最大重试次数和当前消息的重试次数
        final int maxAttempts = retryPolicy.getMaxAttempts();
        int attempt = messageView.getDeliveryAttempt();
        final MessageId messageId = messageView.getMessageId();
        final ConsumeService service = consumer.getConsumeService();
        final ClientId clientId = consumer.getClientId();
        if (ConsumeResult.FAILURE.equals(consumeResult) && attempt < maxAttempts) {
            // 消费失败，并且还没有达到最大重试次数
            // 设置下次重试延迟
            final Duration nextAttemptDelay = retryPolicy.getNextAttemptDelay(attempt);
            // 增加 attempt 的数值，++deliveryAttempt;
            attempt = messageView.incrementAndGetDeliveryAttempt();
            log.debug("Prepare to redeliver the fifo message because of the consumption failure, maxAttempt={}," +
                    " attempt={}, mq={}, messageId={}, nextAttemptDelay={}, clientId={}", maxAttempts, attempt, mq,
                messageId, nextAttemptDelay, clientId);
            // 再次消费
            final ListenableFuture<ConsumeResult> future = service.consume(messageView, nextAttemptDelay);
            // 递归进去当前方法，判断消息是否需要继续重试
            return Futures.transformAsync(future, result -> eraseFifoMessage(messageView, result),
                MoreExecutors.directExecutor());
        }
        
        // 消费成功 or 超出最大重试次数
        boolean ok = ConsumeResult.SUCCESS.equals(consumeResult);
        if (!ok) {
            log.info("Failed to consume fifo message finally, run out of attempt times, maxAttempts={}, "
                + "attempt={}, mq={}, messageId={}, clientId={}", maxAttempts, attempt, mq, messageId, clientId);
        }
        // 根据结果来判断是否是提交消息还是发送到死信队列
        ListenableFuture<Void> future = ok ? ackMessage(messageView) : forwardToDeadLetterQueue(messageView);
        // 清理消息缓存
        future.addListener(() -> evictCache(messageView), consumer.getConsumptionExecutor());
        return future;
    }
    
    // 消息提交确认逻辑
    private void ackMessage(final MessageViewImpl messageView, final int attempt, final SettableFuture<Void> future0) {
      final ClientId clientId = consumer.getClientId();
      final String consumerGroup = consumer.getConsumerGroup();
      final MessageId messageId = messageView.getMessageId();
      final Endpoints endpoints = messageView.getEndpoints();
      final RpcFuture<AckMessageRequest, AckMessageResponse> future =
          consumer.ackMessage(messageView);
      Futures.addCallback(future, new FutureCallback<AckMessageResponse>() {
          @Override
          public void onSuccess(AckMessageResponse response) {
              final String requestId = future.getContext().getRequestId();
              final Status status = response.getStatus();
              final Code code = status.getCode();
              if (Code.INVALID_RECEIPT_HANDLE.equals(code)) {
                  log.error("Failed to ack message due to the invalid receipt handle, forgive to retry, "
                          + "clientId={}, consumerGroup={}, messageId={}, attempt={}, mq={}, endpoints={}, "
                          + "requestId={}, status message=[{}]", clientId, consumerGroup, messageId, attempt, mq,
                      endpoints, requestId, status.getMessage());
                  future0.setException(new BadRequestException(code.getNumber(), requestId, status.getMessage()));
                  return;
              }
              // Log failure and retry later.
              if (!Code.OK.equals(code)) {
                  log.error("Failed to ack message, would attempt to re-ack later, clientId={}, "
                          + "consumerGroup={}, attempt={}, messageId={}, mq={}, code={}, requestId={}, endpoints={}, "
                          + "status message=[{}]", clientId, consumerGroup, attempt, messageId, mq, code, requestId,
                      endpoints, status.getMessage());
                  ackMessageLater(messageView, 1 + attempt, future0);
                  return;
              }
              // Set result if FIFO message is acknowledged successfully.
              future0.setFuture(Futures.immediateVoidFuture());
              // Log retries.
              if (1 < attempt) {
                  log.info("Finally, ack message successfully, clientId={}, consumerGroup={}, attempt={}, "
                          + "messageId={}, mq={}, endpoints={}, requestId={}", clientId, consumerGroup, attempt,
                      messageId, mq, endpoints, requestId);
                  return;
              }
              log.debug("Ack message successfully, clientId={}, consumerGroup={}, messageId={}, mq={}, "
                  + "endpoints={}, requestId={}", clientId, consumerGroup, messageId, mq, endpoints, requestId);
          }

          @Override
          public void onFailure(Throwable t) {
              // Log failure and retry later.
              log.error("Exception raised while acknowledging message, clientId={}, consumerGroup={}, "
                      + "would attempt to re-ack later, attempt={}, messageId={}, mq={}, endpoints={}", clientId,
                  consumerGroup, attempt, messageId, mq, endpoints, t);
              ackMessageLater(messageView, 1 + attempt, future0);
          }
      }, MoreExecutors.directExecutor());
  }
}

```

### 2.2 Client 代码相关分析总结

`consumer.getPushConsumerSettings().getLongPollingTimeout()`  默认是30s，超过30s之后，`client`会重新去拉消息。而此时 `PushConsumer` 还没有消费完成，导致重复消费了。

```java
public class PushSubscriptionSettings extends Settings {
    private static final Logger log = LoggerFactory.getLogger(PushSubscriptionSettings.class);

    private final Resource group;
    private final Map<String, FilterExpression> subscriptionExpressions;
    private volatile Boolean fifo = false;
    private volatile int receiveBatchSize = 32;
    private volatile Duration longPollingTimeout = Duration.ofSeconds(30);
    ... 
}
```

---

### 2.3 Broker 关联源码分析

Broker 源码中相关配置代码如下：

```java
public class ProxyConfig implements ConfigFile {
	  
	  // ...
    private long grpcClientConsumerLongPollingTimeoutMillis = Duration.ofSeconds(30).toMillis();
		// ...
}
```

配置组装代码如下：

```java
public class GrpcClientSettingsManager {
    // ...
    protected static Settings mergeSubscriptionData(Settings settings, SubscriptionGroupConfig groupConfig) {
        Settings.Builder resultSettingsBuilder = settings.toBuilder();
        ProxyConfig config = ConfigurationManager.getProxyConfig();

        resultSettingsBuilder.getSubscriptionBuilder()
            .setReceiveBatchSize(config.getGrpcClientConsumerLongPollingBatchSize())
            // 设置LongPollingTimeout的值
            .setLongPollingTimeout(Durations.fromMillis(config.getGrpcClientConsumerLongPollingTimeoutMillis()))
            .setFifo(groupConfig.isConsumeMessageOrderly());

        resultSettingsBuilder.getBackoffPolicyBuilder().setMaxAttempts(groupConfig.getRetryMaxTimes() + 1);

        GroupRetryPolicy groupRetryPolicy = groupConfig.getGroupRetryPolicy();
        if (groupRetryPolicy.getType().equals(GroupRetryPolicyType.EXPONENTIAL)) {
            ExponentialRetryPolicy exponentialRetryPolicy = groupRetryPolicy.getExponentialRetryPolicy();
            if (exponentialRetryPolicy == null) {
                exponentialRetryPolicy = new ExponentialRetryPolicy();
            }
            resultSettingsBuilder.getBackoffPolicyBuilder().setExponentialBackoff(convertToExponentialBackoff(exponentialRetryPolicy));
        } else {
            CustomizedRetryPolicy customizedRetryPolicy = groupRetryPolicy.getCustomizedRetryPolicy();
            if (customizedRetryPolicy == null) {
                customizedRetryPolicy = new CustomizedRetryPolicy();
            }
            resultSettingsBuilder.getBackoffPolicyBuilder().setCustomizedBackoff(convertToCustomizedRetryPolicy(customizedRetryPolicy));
        }

        return resultSettingsBuilder.build();
    }
    // ...
}
```

## 3. 分析总结

### 3.1 场景分析

- `longPollingTimeout` 是 `broker` 端等待客户端拉取请求的最长挂起时间（30s）
- 客户端收到消息后，**消费处理过程未在“`ack` 超时时间”内完成并确认消息**
- `broker` 会认为客户端挂了或消费失败
- 于是该消息会重新出队，触发 **重复消费**

AI给出相关机制，目前还没有做源码分析进行验证：

`RocketMQ 5` 引入了类似 `SQS` 的机制，**消息被“隐藏”一定时间（`Invisible Duration`）后会重新变为可消费状态**：

- 如果 `client` 在 `invisibleDuration` 内 **`ack`** 消息，`broker` 标记为成功消费
- 如果 `client` **未 `ack`**（超时、异常、宕机），`broker` 会将消息重新入队
- 所以，**`InvisibleDuration` < 客户端消费耗时**时就会出现重复消费！

### 3.2 疑问点：

为什么有些消息会出发重试，而有些消息没有进行重试？线上环境出现重复消费很少，测试环境单条消息消费出现重试概率高。

需要继续研究 `Broker` 的代码。

## 4. 解决方案

当前程序做了基础的幂等处理，即使重复消费也不影响。后续继续研究`Broker`相关代码，找到更好的解决方案。
