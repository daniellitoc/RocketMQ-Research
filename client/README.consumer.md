# com.alibaba.rocketmq.client.consumer.DefaultMQPushConsumer
    public void setInstanceName(String instanceName)                                        // 默认值: System.getProperty("rocketmq.client.name", "DEFAULT")
    public void setMessageModel(MessageModel messageModel)                                  // 默认值: MessageModel.CLUSTERING
    public void setConsumeFromWhere(ConsumeFromWhere consumeFromWhere)                      // 默认值: ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET
    public void setConsumeTimestamp(String consumeTimestamp)                                // 默认值: UtilAll.timeMillisToHumanString3(System.currentTimeMillis() - (1000 * 60 * 30))
    public void setConsumeThreadMin(int consumeThreadMin)                                   // 默认值: 20
    public void setConsumeThreadMax(int consumeThreadMax)                                   // 默认值: 64
    public void setConsumeConcurrentlyMaxSpan(int consumeConcurrentlyMaxSpan)               // 默认值: 2000，针对并行消费场景，processQueue中第一条消息和最后一条消息的offset差，超过阀值后，会延迟拉取操作，直到offset差小于此阀值
    public void setPullThresholdForQueue(int pullThresholdForQueue)                         // 默认值: 1000，队列阀值，当processQueue中消息个数大于此阀值后，会延迟拉取操作，直到消息个数小于此阀值
    public void setPullInterval(long pullInterval)                                          // 默认值: 0，拉取间隔，为0表示每次拉取成功转交出去后立即拉取，否则等待pullInterval毫秒后，在进行拉取
    public void setConsumeMessageBatchMaxSize(int consumeMessageBatchMaxSize)               // 默认值: 1，每次消费多少条，拉取后，对于并行消费，按照consumeMessageBatchMaxSize拆分消息列表并并行消费，对于顺序消费，相同messageQueue加锁，每次从processQueue中取consumeMessageBatchMaxSize条消息，进行消费
    public void setPullBatchSize(int pullBatchSize)                                         // 默认值: 32，每次拉取多少条消息

    public void setOffsetStore(OffsetStore offsetStore)
    public void setAdjustThreadPoolNumsThreshold(long adjustThreadPoolNumsThreshold)        // 默认值: 100000，堆积消息的最后一条和拉取时brokerServer中最大Offset的总值，如果超过阀值，增加消费线程，小于阀值，减少消费线程
    public void setSubscription(Map<String, String> subscription)                           // 默认值: new HashMap<String, String>()，订阅主题和配置该主题的过滤表达式       Key: Topic  Value: subExpression
    public void changeInstanceNameToPID()                                                   // 如果是DEFAULT，设置instanceName为进程PID，后边需要根据ip + instanceName确定MQClientManager
        如果instanceName属性等于DEFAULT
            设置instanceName属性等于当前进程PID

    public DefaultMQPushConsumer(String consumerGroup, RPCHook rpcHook, AllocateMessageQueueStrategy allocateMessageQueueStrategy)
        设置consumerGroup属性等于consumerGroup参数
        设置allocateMessageQueueStrategy属性等于allocateMessageQueueStrategy参数
        设置defaultMQPushConsumerImpl属性等于new DefaultMQPushConsumerImpl(this, rpcHook)实例
    public void subscribe(String topic, String subExpression) throws MQClientException
        执行defaultMQPushConsumerImpl.subscribe(topic, subExpression)
    public void unsubscribe(String topic)
        执行defaultMQPushConsumerImpl.unsubscribe(topic)
    public void registerMessageListener(MessageListenerConcurrently messageListener)
        设置messageListener属性等于messageListener参数
        执行defaultMQPushConsumerImpl.registerMessageListener(messageListener)
    public void registerMessageListener(MessageListenerOrderly messageListener)
        设置messageListener属性等于messageListener参数
        执行defaultMQPushConsumerImpl.registerMessageListener(messageListener)
    public void start() throws MQClientException
        执行defaultMQPushConsumerImpl.start()
    public void shutdown()
        执行defaultMQPushConsumerImpl.shutdown()

## com.alibaba.rocketmq.client.consumer.listener.MessageListenerConcurrently

    ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context)

## com.alibaba.rocketmq.client.consumer.listener.MessageListenerOrderly

    ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context)

## com.alibaba.rocketmq.client.consumer.rebalance.AllocateMessageQueueAveragely

    // 求出当前clientId消费某个topic的哪些消息队列
    // currentCID: 当前客户端ID
    // mqAll: 某个topic的所有消息队列
    // cidAll: 某个topic的所有客户端ID
    public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll, List<String> cidAll)
        如果currentCID参数等于null或者长度等于0
            抛异常IllegalArgumentException("currentCID is empty")
        如果mqAll参数等于null或者元素个数等于0
            抛异常IllegalArgumentException("mqAll is null or mqAll empty")
        如果cidAll参数等于null或者元素个数等于0
            抛异常IllegalArgumentException("cidAll is null or cidAll empty")
        设置result变量等于new ArrayList<MessageQueue>()实例
        如果cidAll参数中不存在currentCID
            返回result变量
        int index = cidAll.indexOf(currentCID);
        int size = (mqAll.size() + cidAll.size() - 1) / cidAll.size();
        List<List<MessageQueue>> partition = Lists.partition(mqAll, size);
        if (index < partition.size()) {
            return partition.get(index);
        }
        return result

# com.alibaba.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl
    private RebalanceImpl rebalanceImpl = new RebalancePushImpl(this)
    private ArrayList<FilterMessageHook> filterMessageHookList = new ArrayList<FilterMessageHook>()

    public DefaultMQPushConsumerImpl(DefaultMQPushConsumer defaultMQPushConsumer, RPCHook rpcHook)
        设置defaultMQPushConsumer属性等于defaultMQPushConsumer参数
        设置rpcHook属性等于rpcHook参数
    public void subscribe(String topic, String subExpression) throws MQClientException          // 订阅一个主题
        try {
            执行FilterAPI.buildSubscriptionData(defaultMQPushConsumer.consumerGroup, topic, subExpression)返回SubscriptionData类型的实例，设置给subscriptionData变量
            执行rebalanceImpl.subscriptionInner.put(topic, subscriptionData)
            如果mQClientFactory属性不等于null
                执行mQClientFactory.sendHeartbeatToAllBrokerWithLock()
        } catch (Exception e) {
            抛异常MQClientException("subscription exception", e)
        }
    public void unsubscribe(String topic)                                                       // 解订阅一个主题
        执行rebalanceImpl.subscriptionInner.remove(topic)
    public void registerMessageListener(MessageListener messageListener)
        设置messageListenerInner属性等于messageListener参数
    public void start() throws MQClientException
        如果serviceState属性等于CREATE_JUST
            设置serviceState属性等于START_FAILED
            验证defaultMQPushConsumer.consumerGroup不能等于null；必须由字符、数字、-、_构成；长度不能超过255个字符；不能等于DEFAULT_CONSUMER，否则抛异常MQClientException("consumerGroup...")
            验证defaultMQPushConsumer.messageModel不能为空，否则抛异常MQClientException("messageModel is null")
            验证defaultMQPushConsumer.consumeFromWhere不能为空，否则抛异常MQClientException("consumeFromWhere is null")
            验证defaultMQPushConsumer.consumeTimestamp为yyyyMMddHHmmss格式并可以转换为Date类型，如果验证失败，抛异常MQClientException("consumeTimestamp is invalid, yyyyMMddHHmmss")
            验证defaultMQPushConsumer.allocateMessageQueueStrategy不能为空，否则抛异常MQClientException("allocateMessageQueueStrategy is null")
            验证defaultMQPushConsumer.subscription不能为空，否则抛异常MQClientException("subscription is null")
            验证defaultMQPushConsumer.messageListener不能为空，否则抛异常MQClientException("messageListener is null")
            验证defaultMQPushConsumer.messageListener必须是MessageListenerOrderly或MessageListenerConcurrently的实现，否则抛异常MQClientException("messageListener must be instanceof MessageListenerOrderly or MessageListenerConcurrently")
            验证defaultMQPushConsumer.consumeThreadMin大于等于1且小于等于1000，且不能超过defaultMQPushConsumer.consumeThreadMax，否则抛异常MQClientException("consumeThreadMin Out of range [1, 1000]")
            验证defaultMQPushConsumer.consumeThreadMax大于等于1且小于等于1000，否则抛异常MQClientException("consumeThreadMax Out of range [1, 1000]")
            验证defaultMQPushConsumer.consumeConcurrentlyMaxSpan大于等于1且小于等于65535，否则抛异常MQClientException("consumeConcurrentlyMaxSpan Out of range [1, 65535]")
            验证defaultMQPushConsumer.pullThresholdForQueue大于等于1且小于等于65535，否则抛异常MQClientException("pullThresholdForQueue Out of range [1, 65535]")
            验证defaultMQPushConsumer.pullInterval大于等于0且小于等于65535，否则抛异常MQClientException("pullInterval Out of range [0, 65535]")
            验证defaultMQPushConsumer.consumeMessageBatchMaxSize大于等于1且小于等于1024，否则抛异常MQClientException("consumeMessageBatchMaxSize Out of range [1, 1024]")
            验证defaultMQPushConsumer.pullBatchSize大于等于1且小于等于1024，否则抛异常MQClientException("pullBatchSize Out of range [1, 1024]")
            try {
                如果defaultMQPushConsumer.subscription不等于null，则进行遍历，设置entry变量等于当前元素    // 添加订阅主题列表
                    执行FilterAPI.buildSubscriptionData(defaultMQPushConsumer.consumerGroup, entry.key, entry.value)返回SubscriptionData类型的实例，设置给subscriptionData变量
                    执行rebalanceImpl.subscriptionInner.put(topic, subscriptionData)
                如果defaultMQPushConsumer.messageModel等于CLUSTERING
                    设置retryTopic变量等于%RETRY% + defaultMQPushConsumer.consumerGroup               // 订阅一个主题，当消费其他正常主题的消息失败后，会重发失败消息到该主题中，后续消费该主题来实现重新消费
                    执行FilterAPI.buildSubscriptionData(defaultMQPushConsumer.consumerGroup, retryTopic, "*")返回SubscriptionData类型的实例，设置给subscriptionData
                    执行rebalanceImpl.subscriptionInner.put(retryTopic, subscriptionData)
            } catch (Exception e) {
                抛异常MQClientException("subscription exception", e)
            }
            如果defaultMQPushConsumer.messageModel等于CLUSTERING                                     // 集群模式下，修改instanceName
                执行defaultMQPushConsumer.changeInstanceNameToPID()
            设置mQClientFactory属性等于MQClientManager.getInstance().getAndCreateMQClientInstance(defaultMQPushConsumer, rpcHook)     // 根据IP@instanceName生成Key确保MQClientManager实例唯一
            执行rebalanceImpl.setConsumerGroup(defaultMQPushConsumer.consumerGroup)
            执行rebalanceImpl.setMessageModel(defaultMQPushConsumer.messageModel)
            执行rebalanceImpl.setAllocateMessageQueueStrategy(defaultMQPushConsumer.allocateMessageQueueStrategy)
            执行rebalanceImpl.setmQClientFactory(mQClientFactory)
            设置pullAPIWrapper属性等于new PullAPIWrapper(mQClientFactory, defaultMQPushConsumer.consumerGroup, defaultMQPushConsumer.unitMode)
            执行pullAPIWrapper.registerFilterMessageHook(filterMessageHookList)
            如果defaultMQPushConsumer.offsetStore不等于null
                设置offsetStore属性等于defaultMQPushConsumer.offsetStore
            否则
                如果defaultMQPushConsumer.messageModel等于BROADCASTING                                                              // 广播模式，使用本地文件保存offset
                    设置offsetStore属性等于new LocalFileOffsetStore(mQClientFactory, defaultMQPushConsumer.consumerGroup)
                否则如果defaultMQPushConsumer.messageModel等于CLUSTERING
                    设置offsetStore属性等于new RemoteBrokerOffsetStore(mQClientFactory, defaultMQPushConsumer.consumerGroup)         // 集群模式，使用远程方式保存offset
            执行offsetStore.load()                                                                                                 // 加载之前持久化的offset信息，本地磁盘方式起作用
            如果messageListenerInner属性是MessageListenerOrderly类型
                设置consumeOrderly属性等于true
                设置consumeMessageService属性等于new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) messageListenerInner)
            否则如果messageListenerInner属性是MessageListenerConcurrently类型
                设置consumeOrderly属性等于false
                设置consumeMessageService属性等于new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) messageListenerInner)
            执行consumeMessageService.start()                                                                                     // 启动server，只针对顺序消费场景下，集群模式的起作用
            执行mQClientFactory.registerConsumer(defaultMQPushConsumer.consumerGroup, this)返回设置给registerOK变量
            如果registerOK变量等于false
                设置serviceState属性等于CREATE_JUST，执行consumeMessageService.shutdown()，抛异常MQClientException("The consumer group...")
            执行mQClientFactory.start()
            设置serviceState属性等于RUNNING
        否则如果serviceState属性等于RUNNING、START_FAILED、SHUTDOWN_ALREADY
            抛异常MQClientException("The PushConsumer service state not OK...")
        如果rebalanceImpl.subscriptionInner不等于null，则进行遍历，设置entry变量等于当前元素                                             // 更新所有订阅主题（正常消费主题和内置主题），用来维护topicRouteTable和brokerAddrTable属性，以及topicSubscribeInfoTable
            执行mQClientFactory.updateTopicRouteInfoFromNameServer(entry.key)
        执行mQClientFactory.sendHeartbeatToAllBrokerWithLock()                                                                    // 先同步执行一次心跳
        执行mQClientFactory.rebalanceImmediately()                                                                                // 触发一次doRebalance()平衡所有订阅主题的消息队列
    public void shutdown()
        如果serviceState属性等于RUNNING
            执行consumeMessageService.shutdown()
            执行persistConsumerOffset()
            执行mQClientFactory.unregisterConsumer(defaultMQPushConsumer.consumerGroup)
            执行mQClientFactory.shutdown()
            执行rebalanceImpl.destroy()
            设置serviceState属性等于SHUTDOWN_ALREADY

    public void persistConsumerOffset()                                                                                         // 持久化消息队列的offset信息
        try {
            如果serviceState属性不等于RUNNING，抛异常MQClientException("The consumer service state not OK...")
            执行offsetStore.persistAll(rebalanceImpl.processQueueTable.keySet())
        } catch (Exception e) {
        }
    public void doRebalance()                                                                                                   // 平衡所有订阅主题的消息队列
        如果rebalanceImpl属性不等于null
            执行rebalanceImpl.doRebalance()
    public Set<SubscriptionData> subscriptions()                                                                                // 获取当前进程（除了当前实例，还有其他实例）所有订阅主题的订阅信息，包括topic、group、subExpression
        设置subSet变量等于new HashSet<SubscriptionData>()实例
        添加rebalanceImpl.subscriptionInner.values()到subSet变量中
        返回subSet变量
    public boolean isSubscribeTopicNeedUpdate(String topic)                                                                     // 如果subscriptionInner中包含，但是topicSubscribeInfoTable中不包含，返回true，其余情况返回false
        如果rebalanceImpl.subscriptionInner不等于null并且包含topic参数
            如果rebalanceImpl.topicSubscribeInfoTable.containsKey(topic)等于true
                返回false
            否则返回true
        返回false
    public void updateTopicSubscribeInfo(String topic, Set<MessageQueue> info)                                                  // 如果subscriptionInner中包含，设置topic的可读消息队列
         如果rebalanceImpl.subscriptionInner不等于null并且包含topic参数
             执行rebalanceImpl.topicSubscribeInfoTable.put(topic, info)
    public void adjustThreadPool()                                                                                              // 拉取到的最后一条消息和brokerServer的最新offset的总差值如果大于阀值，增加消费线程，小于阀值，减少消费线程
        设置computeAccTotal变量等于computeAccumulationTotal()
        设置adjustThreadPoolNumsThreshold变量等于defaultMQPushConsumer.adjustThreadPoolNumsThreshold
        如果computeAccTotal变量大于adjustThreadPoolNumsThreshold变量
            执行consumeMessageService.incCorePoolSize()
        如果computeAccTotal变量小于adjustThreadPoolNumsThreshold * 0.8
            执行consumeMessageService.decCorePoolSize()
    private long computeAccumulationTotal()                                                                                     // 获取所有正在处理的队列中堆积消息的最后一条和消息拉取时brokerServer中最大Offset的差
        设置msgAccTotal变量等于0
        遍历rebalanceImpl.processQueueTable，设置entry等于当前元素
            执行msgAccTotal += entry.value.msgAccCnt
        返回msgAccTotal变量

    public void executePullRequestImmediately(PullRequest pullRequest)                                                          // 立即添加拉取请求到queue
        执行mQClientFactory.pullMessageService.executePullRequestImmediately(pullRequest)
    public ConsumerRunningInfo consumerRunningInfo()                                                                            // 返回consumer的运行时信息
        设置info变量等于new ConsumerRunningInfo()实例
        设置prop变量等于MixAll.object2Properties(defaultMQPushConsumer)                                                           // 添加consumer配置信息
        添加PROP_CONSUMEORDERLY和consumeOrderly属性到prop变量中                                                                    // 添加是否是顺序消费
        添加PROP_THREADPOOL_CORE_SIZE和consumeMessageService.getCorePoolSize()到prop变量中                                        // 添加线程池信息
        添加PROP_CONSUMER_START_TIMESTAMP和consumerStartTimestamp属性到prop变量中                                                  // 添加consumer启动时间
        执行info.setProperties(prop)
        设置subSet变量等于subscriptions()
        执行info.subscriptionSet.addAll(subSet)                                                                                 // 添加所有订阅信息包括topic/group/subExpression
        遍历rebalanceImpl.processQueueTable属性，设置entry变量等于当前元素                                                          // entry: Key: MessageQueue, Value: ProcessQueue
            设置pqinfo变量等于new ProcessQueueInfo()实例
            执行pqinfo.setCommitOffset(this.offsetStore.readOffset(entry.key, ReadOffsetType.MEMORY_FIRST_THEN_STORE))          // 添加offset信息
            // 填充运行时队列信息，包括
            //      队列中堆积第一条消息的offset和最后一条消息的offset
            //      队列中堆积消息个数大小
            //      队列中正在处理的一批消息的第一条消息的offset和最后一条消息的offset
            //      队列中正在处理的一批消息的消息个数大小
            //      队列是否处于lock状态
            //      队列执行的tryUnlock次数
            //      队列上次lock的时间戳
            //      队列是否处于废弃状态
            //      队列上次拉取的时间戳
            //      队列上次消费的时间戳
            执行entry.value.fillProcessQueueInfo(pqinfo)
            执行info.mqTable.put(entry.key, pqinfo)                                                                            // 添加消息队列信息和运行队列信息
        遍历subSet变量，设置sd变量等于当前元素
            设置consumeStatus变量等于mQClientFactory.consumerStatsManager.consumeStatus(defaultMQPushConsumer.consumerGroup, sd.topic)
            执行info.statusTable.put(sd.topic, consumeStatus)
        返回info变量
    public void updateConsumeOffset(MessageQueue mq, long offset)                                                             // 从内存中更新offset，如果出现并发插入或元素已存在，按这个offset为主
        执行offsetStore.updateOffset(mq, offset, false)

    public void pullMessage(PullRequest pullRequest)                                                                          // 执行拉取请求
        设置processQueue变量等于pullRequest.processQueue
        如果processQueue.dropped等于true                                                                                       // 如果队列被废弃，则退出方法
            退出方法
        执行processQueue.setLastPullTimestamp(System.currentTimeMillis())                                                     // 设置时间戳
        try {
            如果serviceState属性不等于RUNNING
                抛异常MQClientException("The consumer service state not OK...")
        } catch (MQClientException e) {
            执行mQClientFactory.pullMessageService.executePullRequestLater(pullRequest, 3000)
            退出方法
        }
        如果pause属性等于true                                                                                                  // 可暂时不考虑此种情况
            执行mQClientFactory.pullMessageService.executePullRequestLater(pullRequest, 1000)
            退出方法
            如果processQueue.msgCount.get()大于defaultMQPushConsumer.pullThresholdForQueue                                    // 当processQueue中消息堆积个数大于队列阀值后，会延迟拉取操作，直到消息个数小于此阀值
            执行mQClientFactory.pullMessageService.executePullRequestLater(pullRequest, 50)
            退出方法
        如果consumeOrderly属性等于false
            如果processQueue.getMaxSpan()大于defaultMQPushConsumer.consumeConcurrentlyMaxSpan                                // 并行消费场景，当processQueue中堆积消息的第一条消息和最后一条消息的offset差，超过阀值后，会延迟拉取操作，直到offset差小于此阀值
                执行mQClientFactory.pullMessageService.executePullRequestLater(pullRequest, 50)
                退出方法
        设置subscriptionData变量等于rebalanceImpl.subscriptionInner.get(pullRequest.messageQueue.topic)                       // 如果根据topic获取不到订阅信息，延迟，知道有订阅信息才进行下一步
        如果subscriptionData变量等于null
            执行mQClientFactory.pullMessageService.executePullRequestLater(pullRequest, 3000)
            退出方法
        设置beginTimestamp变量等于System.currentTimeMillis()
        创建new PullCallback()实例，设置给pullCallback变量
            public void onSuccess(PullResult pullResult)
                如果pullResult参数等于null
                    退出方法
                设置pullResult参数等于pullAPIWrapper.processPullResult(pullRequest.messageQueue, pullResult, subscriptionData)    // 解码和重新包装一下消息列表
                如果pullResult.pullStatus等于FOUND                                                                           // 如果状态是FOUND
                    执行pullRequest.setNextOffset(pullResult.nextBeginOffset)                                               // 据返回的nextBeginOffset，设置下次请求的offset开始位置，
                    如果pullResult.msgFoundList()等于null或者元素个数等于0                                                      // 如果msgFoundList元素个数为空则立即重新执行拉取请求
                        执行mQClientFactory.pullMessageService.executePullRequestImmediately(pullRequest)
                    否则
                        设置dispathToConsume变量等于processQueue.putMessage(pullResult.msgFoundList)                          // 添加消息到运行队列，返回值如果为true，表示当前消息队列不等于空并且没人消费
                        执行consumeMessageService.submitConsumeRequest(pullResult.msgFoundList, processQueue, pullRequest.messageQueue, dispathToConsume)
                        如果defaultMQPushConsumer.pullInterval大于0                                                          // 如果pullInterval大于0，则延迟拉取
                            执行mQClientFactory.pullMessageService.executePullRequestLater(pullRequest, defaultMQPushConsumer.pullInterval)
                        否则                                                                                                // 否则立即拉取
                            执行mQClientFactory.pullMessageService.executePullRequestImmediately(pullRequest)
                否则如果pullResult.pullStatus等于NO_NEW_MSG或NO_MATCHED_MSG                                                   // 如果状态是NO_NEW_MSG或NO_MATCHED_MSG，根据返回的nextBeginOffset，设置下次请求的offset开始位置，如果消息堆积则更新offset，最后立即重新执行拉取请求
                    执行pullRequest.setNextOffset(pullResult.nextBeginOffset)
                    如果pullRequest.processQueue.msgCount.get()等于0                                                         // msgCount包括堆积map和临时map的大小
                        执行offsetStore.updateOffset(pullRequest.messageQueue, pullRequest.nextOffset, true)
                    执行mQClientFactory.pullMessageService.executePullRequestImmediately(pullRequest)
                否则如果pullResult.pullStatus等于OFFSET_ILLEGAL                                                               // 如果状态是OFFSET_ILLEGAL，根据返回的nextBeginOffset，设置下次请求的offset开始位置和更新offset，废弃和删除运行队列
                    执行pullRequest.setNextOffset(pullResult.nextBeginOffset)
                    执行pullRequest.processQueue.setDropped(true)
                    创建new Runnable()实例，设置给runnable变量
                        public void run()
                            try {
                                执行offsetStore.updateOffset(pullRequest.messageQueue, pullRequest.nextOffset, false)
                                执行offsetStore.persist(pullRequest.messageQueue)
                                执行rebalanceImpl.removeProcessQueue(pullRequest.messageQueue)
                            } catch (Throwable e) {
                            }
                    执行mQClientFactory.pullMessageService.executeTaskLater(runnable, 10000)
            public void onException(Throwable e)
                执行mQClientFactory.pullMessageService.executePullRequestLater(pullRequest, 3000)                            // 执行出错，重新执行
        设置commitOffsetEnable变量等于false                                                                                   // 是否可以commitoffset，只对集群模式下生效，如果内存中已有offset，则设置为true
        设置commitOffsetValue变量等于0                                                                                        // commitoffset值，，只对集群模式下生效，设置内存中的offset值
        如果defaultMQPushConsumer.messageModel等于CLUSTERING
            设置commitOffsetValue变量等于offsetStore.readOffset(pullRequest.messageQueue, ReadOffsetType.READ_FROM_MEMORY)
            如果commitOffsetValue变量大于0
                设置commitOffsetEnable等于true
        设置subExpression变量等于null                                                                                         // 订阅表达式，非服务端类过滤场景生效，可能有值
        设置classFilter变量等于false                                                                                          // 是否是服务端类过滤
        如果subscriptionData变量不等于null
            如果defaultMQPushConsumer.postSubscriptionWhenPull等于true并且subscriptionData.classFilterMode等于false
                设置subExpression变量等于subscriptionData.subString
            设置classFilter变量等于subscriptionData.classFilterMode
        设置sysFlag变量等于PullSysFlag.buildSysFlag(commitOffsetEnable, true, subExpression != null, classFilter)             // 构建sysFlag，分别表示有集群场景有offset、是否可以延迟、是否有订阅表达式、是否是服务端过滤模式
        try {
            // 发布异步请求
            执行pullAPIWrapper.pullKernelImpl(pullRequest.messageQueue, subExpression, subscriptionData.subVersion, pullRequest.nextOffset, defaultMQPushConsumer.pullBatchSize, sysFlag, commitOffsetValue, 15000, 30000, CommunicationMode.ASYNC, pullCallback)
        } catch (Exception e) {
            执行mQClientFactory.pullMessageService.executePullRequestLater(pullRequest, 3000)                                // 如果拉取失败，延迟3000秒，在执行
        }
    public void executeHookBefore(ConsumeMessageContext context)
        如果consumeMessageHookList属性元素个数大于0
            遍历consumeMessageHookList属性，设置hook变量等于当前元素
                try {
                    执行hook.consumeMessageBefore(context)
                } catch (Throwable e) {
                }
    public void executeHookAfter(ConsumeMessageContext context)
        如果consumeMessageHookList属性元素个数大于0
            遍历consumeMessageHookList属性，设置hook变量等于当前元素
                try {
                    执行hook.consumeMessageAfter(context)
                } catch (Throwable e) {
                }
    // 告知brokermsg需要重写消息，如果重写失败，则使用Producer重新发送一条消息到"%RETRY%" + consumerGroup
    // 区别在于，消息失败场景，知道从哪个BrokerServer消费失败，所以会重新发送消息到指定的BrokerServer，而如果使用Producer发送，则按照可用消息队列进行发送
    public void sendMessageBack(MessageExt msg, int delayLevel, String brokerName) throws RemotingException, MQBrokerException, InterruptedException, MQClientException
        try {
            如果brokerName参数不等于null
                设置brokerAddr变量等于mQClientFactory.findBrokerAddressInPublish(brokerName)
            否则
                设置brokerAddr变量等于RemotingHelper.parseSocketAddressAddr(msg.storeHost)
            执行mQClientFactory.mQClientAPIImpl.consumerSendMessageBack(brokerAddr, msg, defaultMQPushConsumer.consumerGroup, delayLevel, 5000)
        } catch (Exception e) {
            设置newMsg变量等于new Message("%RETRY%" + defaultMQPushConsumer.consumerGroup), msg.body)
            设置originMsgId变量等于msg.properties.get("ORIGIN_MESSAGE_ID")
            如果originMsgId变量不等于null
                执行newMsg.putProperty("ORIGIN_MESSAGE_ID", originMsgId)
            否则
                执行newMsg.putProperty("ORIGIN_MESSAGE_ID", msg.msgId)
            执行newMsg.setFlag(msg.flag)
            执行newMsg.setProperties(msg.properties())
            添加RETRY_TOPIC和msg.topic到newMsg.properties中
            设置reTimes变量等于msg.reconsumeTimes + 1                 // 指定重发次数
            添加RECONSUME_TIME和reTimes变量到newMsg.properties中
            添加DELAY和3 + reTimes到newMsg.properties中               // 指定延迟级别
            执行mQClientFactory.defaultMQProducer.send(newMsg)
        }

## com.alibaba.rocketmq.common.filter.FilterAPI

    public static SubscriptionData buildSubscriptionData(String consumerGroup, String topic, String subString) throws Exception
        使用topic参数、subString参数创建new SubscriptionData()类型的实例，设置给subscriptionData变量
        如果subString参数等于null或者等于"*"或者等于""
            执行subscriptionData.setSubString("*")
        否则
            使用|分隔并进行遍历，设置tag变量等于当前元素
                执行subscriptionData.tagsSet.add(tag)
                执行subscriptionData.codeSet.add(tag.hashCode())
        返回subscriptionData

# com.alibaba.rocketmq.client.impl.consumer.RebalancePushImpl
    // Key: Topic, Value: SubscriptionData
    // 包含了所有正常订阅主题和内置主题，SubscriptionData是订阅信息，包含topic/subString或者topic/filterClassSource方式
    protected ConcurrentHashMap<String, SubscriptionData> subscriptionInner = new ConcurrentHashMap<String, SubscriptionData>();      // Key: topic
    // Key: MessageQueue, Value: ProcessQueue
    // 每个消息队列的处理情况，MessageQueue是一个抽象的概念，本身不具备行为，ProcessQueue才是拉完消息存放的地方和取消息消费的地方
    protected ConcurrentHashMap<MessageQueue, ProcessQueue> processQueueTable = new ConcurrentHashMap<MessageQueue, ProcessQueue>(64);
    // Key: Topic, Value: Set<MessageQueue>
    // 包含了所有正常订阅主题和内置主题的可读的消息队列列表
    protected ConcurrentHashMap<String, Set<MessageQueue>> topicSubscribeInfoTable = new ConcurrentHashMap<String, Set<MessageQueue>>()

    public void setConsumerGroup(String consumerGroup)
    public void setMessageModel(MessageModel messageModel)
    public void setAllocateMessageQueueStrategy(AllocateMessageQueueStrategy allocateMessageQueueStrategy)
    public void setmQClientFactory(MQClientInstance mQClientFactory)

    public RebalancePushImpl(DefaultMQPushConsumerImpl defaultMQPushConsumerImpl)
        执行this(null, null, null, null, defaultMQPushConsumerImpl)
    public RebalancePushImpl(String consumerGroup, MessageModel messageModel, AllocateMessageQueueStrategy allocateMessageQueueStrategy, MQClientInstance mQClientFactory, DefaultMQPushConsumerImpl defaultMQPushConsumerImpl)
        设置consumerGroup属性等于consumerGroup参数
        设置messageModel属性等于messageModel参数
        设置allocateMessageQueueStrategy属性等于allocateMessageQueueStrategy参数
        设置mQClientFactory属性等于mQClientFactory参数
        设置defaultMQPushConsumerImpl属性等于defaultMQPushConsumerImpl参数
    public void destroy()                                                                                         // 设置dropped等于true，然后清理processQueueTable
        遍历processQueueTable属性，设置entry变量等于当前元素
            执行entry.value.setDropped(true)
        执行processQueueTable.clear()
    public void doRebalance()                                                                                     // 平衡所有订阅主题的消息队列
        如果subscriptionInner属性不等于null，进行遍历，设置entry变量等于当前元素                                          // entry: Key: Topic, Value: SubscriptionData
            try {
                设置topic变量等于entry.key
                如果messageModel属性等于BROADCASTING                                                                // 对于广播模式
                    设置mqSet变量等于topicSubscribeInfoTable.get(topic)
                    如果mqSet变量不等于null                                                                         // 如果包含的可读消息队列队列不等于null
                        执行updateProcessQueueTableInRebalance(topic, mqSet)，返回boolean类型设置给changed变量
                        如果changed变量等于true
                            执行messageQueueChanged(topic, mqSet, mqSet)                                          // 空操作，update已经都干了
                否则如果messageModel属性等于CLUSTERING                                                              // 对于集群模式
                    设置mqSet变量等于topicSubscribeInfoTable.get(topic)                                            // 获取主题的所有可读消息队列
                    执行mQClientFactory.findConsumerIdList(topic, consumerGroup)，返回List<String>类型的实例，设置给cidAll变量      // 获取主题的一个brokerAddr，请求brokerAddr获取配置了相同group的clientId列表
                    如果mqSet变量不等于null并且cidAll变量不等于null                                                   // 如果都不等于null，使用strategy进行重新分配，获取当前clientId对应的消息队列列表
                        设置mqAll变量等于new ArrayList<MessageQueue>()实例
                        添加mqSet元素到aqAll变量中
                        执行Collections.sort(mqAll)
                        执行Collections.sort(cidAll)
                        设置allocateResult变量等于null
                        try {
                            设置allocateResult变量等于strategy.allocate(consumerGroup, mQClientFactory.clientId, mqAll, cidAll);
                        } catch (Throwable e) {
                            退出方法
                        }
                        设置allocateResultSet变量等于new HashSet<MessageQueue>()实例
                        如果allocateResult变量不等于null
                            添加allocateResult元素到allocateResultSet中
                       执行updateProcessQueueTableInRebalance(topic, allocateResultSet)，返回boolean类型设置给changed变量         // 更新当前clientId新的消息队列列表
                       如果changed变量等于true
                           执行messageQueueChanged(topic, mqSet, allocateResultSet)

            } catch (Exception e) {
            }
        // 重新整理一遍无用的
        遍历processQueueTable属性，设置entry变量等于当前元素
            如果subscriptionInner属性中不存在entry.key.topic
                执行processQueueTable.remove(mq)，返回实例设置给pq变量
                如果pq变量不等于null
                    执行pq.setDropped(true)
    private boolean updateProcessQueueTableInRebalance(String topic, Set<MessageQueue> mqSet)
        设置changed变量等于false
        遍历processQueueTable属性，设置entry变量等于当前元素                                                           // entry: Key: MessageQueue, Value: ProcessQueue
            设置mq变量等于entry.key
            设置pq变量等于entry.value
            如果mq.topic等于topic参数
                如果mqSet参数中不包含mq                                                                             // 如果不包含，说明不在消费这个消息队列了
                    执行pg.setDropped(true)
                    如果removeUnnecessaryMessageQueue(mq, pq)等于true                                             // 如果移除成功，删除当前消息队列，设置changed等于true，然后退出循环
                        删除entry，设置changed变量等于true
                        跳出循环
                // 过期的判定是，现在的时间和上一次拉取的时间差大于rocketmq.client.pull.pullMaxIdleTime:120000
                否则如果pg.isPullExpired()等于true                                                                     // 如果包含但是运行队列的pullExpired等于true，也尝试移除，如果移除成功，删除当前消息队列，设置changed等于true，然后退出循环
                    执行pq.setDropped(true)
                    如果removeUnnecessaryMessageQueue(mq, pq)等于true
                        删除entry，设置changed变量等于true
                        跳出循环
        设置pullRequestList变量等于new ArrayList<PullRequest>()实例
        遍历mqSet参数，设置mq变量等于当前元素
            如果processQueueTable属性中不包含mq变量                                                                 // 不存在的，创建PullRequest请求，请求offset，如果大于等于0，添加到pullRequestList中，设置changed等于true
                设置pullRequest变量等于new PullRequest()实例
                执行pullRequest.setConsumerGroup(consumerGroup)
                执行pullRequest.setMessageQueue(mq)
                执行pullRequest.setProcessQueue(new ProcessQueue())
                设置nextOffset变量等于computePullFromWhere(mq)
                如果nextOffset变量大于等于0
                    执行pullRequest.setNextOffset(nextOffset)
                    执行pullRequestList.add(pullRequest)
                    设置changed变量等于true
                    执行processQueueTable.put(mq, pullRequest.processQueue)                                      // 设置运行队列
        执行dispatchPullRequest(pullRequestList)
        返回changed变量
    public boolean removeUnnecessaryMessageQueue(MessageQueue mq, ProcessQueue pq)                               // 先持久化一下offset，然后移除对应的消息队列，对于顺序消费的集群场景，需要尝试unlock，如果成功，返回true，如果失败返回false，其他场景返回true
        执行defaultMQPushConsumerImpl.offsetStore.persist(mq)
        执行defaultMQPushConsumerImpl.offsetStore.removeOffset(mq)
        如果defaultMQPushConsumerImpl.consumeOrderly等于true并且defaultMQPushConsumerImpl.defaultMQPushConsumer.messageModel等于CLUSTERING
            try {
                if (pq.lockConsume.tryLock(1000, TimeUnit.MILLISECONDS)) {
                    try {
                        执行unlock(mq, true);
                        返回true;
                    } finally {
                        pq.lockConsume.unlock();
                    }
                } else {
                    pq.incTryUnlockTimes();
                }
            } catch (Exception e) {
            }
            返回false
        返回true
    public long computePullFromWhere(MessageQueue mq)                                                           // 计算开始消费的offset
        设置result变量等于-1
        设置consumeFromWhere变量等于defaultMQPushConsumerImpl.defaultMQPushConsumer.consumeFromWhere
        设置offsetStore变量等于defaultMQPushConsumerImpl.offsetStore
        如果consumeFromWhere变量等于CONSUME_FROM_LAST_OFFSET                                                      // 如果设置CONSUME_FROM_LAST_OFFSET，先从offsetStore读取，如果不存在，对于"%RETRY%"主题，设置为0，其他主题，请求brokerServer获取最大offset
            设置lastOffset变量等于offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE)
            如果lastOffset变量大于0
                设置result变量等于lastOffset变量
            否则如果lastOffset变量等于-1
                如果mq.topic.startsWith("%RETRY%)等于true
                    设置result变量等于0
                否则
                    try {
                        设置result变量等于mQClientFactory.mQAdminImpl.maxOffset(mq);
                    } catch (MQClientException e) {
                        设置result变量等于-1
                    }
            否则
                设置result变量等于-1
        否则如果变量等于CONSUME_FROM_FIRST_OFFSET                                                                 // 如果设置CONSUME_FROM_FIRST_OFFSET，只从offsetStore读取，如果不存在，返回0，其他情况返回-1
            设置lastOffset变量等于offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE)
            如果lastOffset变量大于等于0
                设置result变量等于lastOffset变量
            否则如果lastOffset变量等于-1
                设置result变量等于0
            否则
                设置result变量等于-1
        否则如果变量等于CONSUME_FROM_TIMESTAMP                                                                   // 如果设置CONSUME_FROM_TIMESTAMP，先从offsetStore读取，如果不存在，对于"%RETRY%"主题，请求brokerServer获取最大offset，其他主题请求brokerServer获取对应时间戳的offset
            设置lastOffset变量等于offsetStore.readOffset(mq, ReadOffsetType.READ_FROM_STORE)
            如果lastOffset变量大于等于0
                设置result变量等于lastOffset变量
            否则如果lastOffset变量等于-1
                如果mq.topic.startsWith("%RETRY%)等于true
                    try {
                        设置result变量等于mQClientFactory.mQAdminImpl.maxOffset(mq)
                    } catch (MQClientException e) {
                        设置result变量等于-1
                    }
                否则
                    try {
                        设置timestamp变量等于UtilAll.parseDate(defaultMQPushConsumerImpl.defaultMQPushConsumer.consumeTimestamp, UtilAll.yyyyMMddHHmmss).getTime()
                        设置result变量等于mQClientFactory.mQAdminImpl.searchOffset(mq, timestamp);
                    } catch (MQClientException e) {
                        设置result变量等于-1
                    }
            否则
                设置result变量等于-1
        返回result变量
    public void dispatchPullRequest(List<PullRequest> pullRequestList)                                        // 遍历列表，添加到pullRequest队列中
        遍历pullRequestList参数，设置pullRequest变量等于当前元素
            执行defaultMQPushConsumerImpl.executePullRequestImmediately(pullRequest)
    public void messageQueueChanged(String topic, Set<MessageQueue> mqAll, Set<MessageQueue> mqDivided)

    public void lockAll()
        // 根据processQueueTable整理brokerName到Set<MessageQueue>的映射
        执行buildProcessQueueTableByBrokerName()返回HashMap<String, Set<MessageQueue>>类型的实例，设置给brokerMqs变量                              // brokerMqs: Key: BrokerName, Value: Set<MessageQueue>
        遍历brokerMqs变量，设置entry变量等于当前元素
            如果entry.value的元素个数等于0
                继续下一次循环
            // 返回brokerName对应的0的brokerAddr，如果不存在返回null
            执行mQClientFactory.findBrokerAddressInSubscribe(entry.key, 0, true)返回FindBrokerResult类型的实例，设置给findBrokerResult变量         // MASTER_ID: 0
            如果findBrokerResult属性不等于null
                设置requestBody变量等于new LockBatchRequestBody()实例
                执行requestBody.setConsumerGroup(consumerGroup)
                执行requestBody.setClientId(mQClientFactory.clientId)
                执行requestBody.setMqSet(mqs)
                try {
                    // 告知brokerAddr，这些消息被clientId锁定，锁定成功的，标识locked等于true，同时设置最近锁定时间，锁定不成功的，标识locked等于false
                    执行mQClientFactory.mQClientAPIImpl.lockBatchMQ(findBrokerResult.brokerAddr, requestBody, 1000)返回Set<MessageQueue>类型的实例，设置给lockOKMQSet变量
                    遍历lockOKMQSet变量，设置mq变量等于当前元素
                        设置processQueue变量等于processQueueTable.get(mq)
                        如果processQueue变量不等于null
                            执行processQueue.setLocked(true)
                            执行processQueue.setLastLockTimestamp(System.currentTimeMillis())
                    遍历mqs变量，设置mq变量等于当前元素
                        如果lockOKMQSet变量中不包含mq变量
                            设置processQueue变量等于processQueueTable.get(mq)
                            如果processQueue变量不等于null
                                执行processQueue.setLocked(false)
                } catch (Exception e) {
                }
    private HashMap<String/* brokerName */, Set<MessageQueue>> buildProcessQueueTableByBrokerName()
        设置result变量等于new HashMap<String, Set<MessageQueue>>()实例
        遍历processQueueTable属性，设置entry等于当前元素
            设置mqs变量等于result.get(entry.key.brokerName)
            如果mqs变量等于null
                执行result.put(entry.key.brokerName, mqs = new HashSet<MessageQueue>())
            执行mqs.add(mq)
        返回result变量
    public boolean lock(MessageQueue mq)
        // 返回brokerName对应的0的brokerAddr，如果不存在返回null
        执行mQClientFactory.findBrokerAddressInSubscribe(mq.brokerName, 0, true)返回FindBrokerResult类型的实例，设置给findBrokerResult变量         // MASTER_ID: 0
        如果findBrokerResult属性不等于null
            设置requestBody变量等于new LockBatchRequestBody()实例
            执行requestBody.setConsumerGroup(consumerGroup)
            执行requestBody.setClientId(mQClientFactory.clientId)
            执行requestBody.setMqSet().add(mq)
            try {
                // 告知brokerAddr，这些消息被clientId锁定，锁定成功的，标识locked等于true，同时设置最近锁定时间，锁定不成功的，标识locked等于false
                执行mQClientFactory.mQClientAPIImpl.lockBatchMQ(findBrokerResult.brokerAddr, requestBody, 1000)返回Set<MessageQueue>类型的实例，设置给lockOKMQSet变量
                遍历lockOKMQSet变量，设置mq变量等于当前元素
                    设置processQueue变量等于processQueueTable.get(mq)
                    如果processQueue变量不等于null
                        执行processQueue.setLocked(true)
                        执行processQueue.setLastLockTimestamp(System.currentTimeMillis())
                如果lockOKMQSet变量中不包含mq变量，返回false，否则返回true
                遍历mqs变量，设置mq变量等于当前元素
            } catch (Exception e) {
            }
    public void unlockAll(boolean oneway)
        // 根据processQueueTable整理brokerName到Set<MessageQueue>的映射
        执行buildProcessQueueTableByBrokerName()返回HashMap<String, Set<MessageQueue>>类型的实例，设置给brokerMqs变量                              // brokerMqs: Key: BrokerName, Value: Set<MessageQueue>
        遍历brokerMqs变量，设置entry变量等于当前元素
            如果entry.value的元素个数等于0
                继续下一次循环
            // 返回brokerName对应的0的brokerAddr，如果不存在返回null
            执行mQClientFactory.findBrokerAddressInSubscribe(entry.key, 0, true)返回FindBrokerResult类型的实例，设置给findBrokerResult变量         // MASTER_ID: 0
            如果findBrokerResult属性不等于null
                设置requestBody变量等于new UnlockBatchRequestBody()实例
                执行requestBody.setConsumerGroup(consumerGroup)
                执行requestBody.setClientId(mQClientFactory.clientId)
                执行requestBody.setMqSet(mqs)
                try {
                    // 告知brokerAddr，这些消息被clientId解锁，然后遍历mqs，表示locked等于false
                    执行mQClientFactory.mQClientAPIImpl.unlockBatchMQ(findBrokerResult.brokerAddr, requestBody, 1000, oneway)
                    遍历mqs变量，设置mq变量等于当前元素
                        设置processQueue变量等于processQueueTable.get(mq)
                        如果processQueue变量不等于null
                            执行processQueue.setLocked(false)
                } catch (Exception e) {
                }
    public void removeProcessQueue(MessageQueue mq)                                                                                          // 废弃和删除消息队列
        设置prev变量等于processQueueTable.remove(mq)
        如果prev变量不等于null
            执行prev.setDropped(true)
            执行removeUnnecessaryMessageQueue(mq, prev)

# com.alibaba.rocketmq.client.impl.consumer.PullAPIWrapper
    private volatile boolean connectBrokerByUser = false
    private volatile long defaultBrokerId = MixAll.MASTER_ID
    private ConcurrentHashMap<MessageQueue, AtomicLong> pullFromWhichNodeTable = new ConcurrentHashMap<MessageQueue, AtomicLong>(32)       // Key: MessageQueue, Value: brokerId

    public PullAPIWrapper(MQClientInstance mQClientFactory, String consumerGroup, boolean unitMode)
        设置mQClientFactory属性等于mQClientFactory参数
        设置consumerGroup属性等于consumerGroup参数
        设置unitMode属性等于unitMode参数
    public void registerFilterMessageHook(ArrayList<FilterMessageHook> filterMessageHookList)
        设置filterMessageHookList属性等于filterMessageHookList参数
    // 执行拉取任务
    //      subVersion：可认为是订阅信息创建时间
    //      brokerSuspendMaxTimeMillis: 可认为是15000
    //      timeoutMillis: 可认为是30000
    //      communicationMode: 可认为是ASYNC
    public PullResult pullKernelImpl(MessageQueue mq, String subExpression, long subVersion, long offset, int maxNums, int sysFlag, long commitOffset, long brokerSuspendMaxTimeMillis, long timeoutMillis, CommunicationMode communicationMode, PullCallback pullCallback) throws MQClientException, RemotingException, MQBrokerException, InterruptedException
        // 获取brokerName对应的brokerId的brokerAddr，如果不存在并且brokerName的map不为空场景，获取第一个brokerId的brokerAddr，其他情况设置为null
        设置findBrokerResult变量等于mQClientFactory.findBrokerAddressInSubscribe(mq.brokerName, recalculatePullFromWhichNode(mq), false)
        如果findBrokerResult变量等于null
            执行mQClientFactory.updateTopicRouteInfoFromNameServer(mq.topic)
            设置findBrokerResult变量等于mQClientFactory.findBrokerAddressInSubscribe(mq.brokerName, recalculatePullFromWhichNode(mq), false)
        如果findBrokerResult变量等于null
            抛异常MQClientException("The broker name not exist", null)
        如果findBrokerResult.slave等于true
            设置sysFlag参数等于PullSysFlag.clearCommitOffsetFlag(sysFlag)                                                     // 如果得到的不是master，设置不可提交offset
        设置requestHeader变量等于new PullMessageRequestHeader()实例
        执行requestHeader.setConsumerGroup(consumerGroup);
        执行requestHeader.setTopic(mq.topic);
        执行requestHeader.setQueueId(mq.queueId);
        执行requestHeader.setQueueOffset(offset);
        执行requestHeader.setMaxMsgNums(maxNums);
        执行requestHeader.setSysFlag(sysFlag);
        执行requestHeader.setCommitOffset(commitOffset);
        执行requestHeader.setSuspendTimeoutMillis(brokerSuspendMaxTimeMillis);
        执行requestHeader.setSubscription(subExpression);
        执行requestHeader.setSubVersion(subVersion);
        设置brokerAddr变量等于findBrokerResult.brokerAddr
        如果PullSysFlag.hasClassFilterFlag(sysFlag)等于true
            设置brokerAddr变量等于computPullFromWhichFilterServer(mq.topic, brokerAddr)
        执行mQClientFactory.mQClientAPIImpl.pullMessage(brokerAddr, reuestHeader, timeoutMillis, communicationMode, pullCallback)并返回
    public long recalculatePullFromWhichNode(MessageQueue mq)               // connectBrokerByUser为true，返回默认的，如果pullFromWhichNodeTable存在消息队列，返回对应的，否则返回0
        如果connectBrokerByUser属性等于true
            返回defaultBrokerId属性
        设置suggest变量等于pullFromWhichNodeTable.get(mq)
        如果suggest变量不等于null
            返回suggest.get()
        返回0                                         // MASTER_ID: 0
    private String computPullFromWhichFilterServer(String topic, String brokerAddr) throws MQClientException                // 根据路由信息随机返回一个filterServer
        设置topicRouteTable变量等于mQClientFactory.topicRouteTable
        如果topicRouteTable变量不等于null
            设置list变量等于topicRouteTable.get(topic).filterServerTable.get(brokerAddr)
            如果list变量不等于null并且元素个数大于0
                执行list.get(randomNum() % list.size())并返回
        抛异常MQClientException("Find Filter Server Failed, Broker Addr:...")
    public PullResult processPullResult(MessageQueue mq, PullResult pullResult, SubscriptionData subscriptionData)          // 解码和重新包装一下消息列表
        设置projectGroupPrefix变量等于mQClientFactory.mQClientAPIImpl.projectGroupPrefix
        设置pullResultExt变量等于pullResult参数
        执行updatePullFromWhichNode(mq, pullResultExt.suggestWhichBrokerId)                                                  // 根据suggestWhichBrokerId更新pullFromWhichNodeTable对应消息队列的brokerId
        如果pullResult.pullStatus等于FOUND
            解序列化pullResultExt.messageBinary为List<MessageExt>类型，设置给msgList变量
            设置msgListFilterAgain变量等于msgList变量
            如果subscriptionData.tagsSet元素个数大于0并且subscriptionData.classFilterMode等于false                              // 如果设置了过滤表达式，进行过滤
                设置msgListFilterAgain变量等于new ArrayList<MessageExt>(msgList.size())实例
                遍历msgList变量，设置msg变量等于当前元素
                    如果msg.tags不等于null并且subscriptionData.tagsSet.contains(msg.tags)等于true
                        执行msgListFilterAgain.add(msg)
            如果filterMessageHookList属性元素个数大于0                                                                         // 执行拦截器
                设置filterMessageContext等于new FilterMessageContext()实例
                执行filterMessageContext.setUnitMode(unitMode)
                执行filterMessageContext.setMsgList(msgListFilterAgain)
                遍历filterMessageHookList属性，设置hook变量等于当前元素
                    try {
                        执行hook.filterMessage(context);
                    } catch (Throwable e) {
                    }
            如果projectGroupPrefix属性不等于null
                执行subscriptionData.setTopic(VirtualEnvUtil.clearProjectGroup(subscriptionData.topic, projectGroupPrefix))
                执行mq.setTopic(VirtualEnvUtil.clearProjectGroup(mq.topic, projectGroupPrefix))
                遍历msgListFilterAgain变量，设置msg变量等于当前元素
                    执行msg.setTopic(VirtualEnvUtil.clearProjectGroup(msg.topic(), projectGroupPrefix))                     // 如果发出去的topic包含projectGroupPrefix，去除前缀
                    执行msg.putProperty("MIN_OFFSET", Long.toString(pullResult.minOffset))                                  // 设置服务端返回的最小offset给消息
                    执行msg.putProperty("MAX_OFFSET", Long.toString(pullResult.maxOffset))                                  // 设置服务端返回的最大offset给消息
            否则
                遍历msgListFilterAgain变量，设置msg变量等于当前元素
                    执行msg.putProperty("MIN_OFFSET", Long.toString(pullResult.minOffset))                                  // 设置服务端返回的最小offset给消息
                    执行msg.putProperty("MAX_OFFSET", Long.toString(pullResult.maxOffset))                                  // 设置服务端返回的最大offset给消息
            执行pullResultExt.setMsgFoundList(msgListFilterAgain)
        执行pullResultExt.setMessageBinary(null)
        返回pullResult参数
    public void updatePullFromWhichNode(MessageQueue mq, long brokerId)                                                     // 更新pullFromWhichNodeTable对应消息队列的brokerId
        设置suggest变量等于pullFromWhichNodeTable.get(mq)
        如果suggest变量等于null
            执行pullFromWhichNodeTable.put(mq, new AtomicLong(brokerId))
        否则
            执行suggest.set(brokerId)

# com.alibaba.rocketmq.client.consumer.store.LocalFileOffsetStore

    private ConcurrentHashMap<MessageQueue, AtomicLong> offsetTable = new ConcurrentHashMap<MessageQueue, AtomicLong>()

    public LocalFileOffsetStore(MQClientInstance mQClientFactory, String groupName)
        设置mQClientFactory属性等于mQClientFactory参数
        设置groupName属性等于groupName参数
        设置storePath属性等于{rocketmq.client.localOffsetStoreDir}/{mQClientFactory.clientId}/{groupName}/offsets.json
    public void load() throws MQClientException                                                 // 加载本地文件之前的持久化信息到内存
        设置offsetSerializeWrapper变量等于readLocalOffset()
        如果offsetSerializeWrapper变量不等于null
            执行offsetTable.putAll(offsetSerializeWrapper.offsetTable)
    public void persistAll(Set<MessageQueue> mqs)                                               // 持久化mqs中topic和对应的offset信息到本地文件，内存中不会丢失不存在的
        如果mqs参数等于null或者元素个数等于0
            退出方法
        设置offsetSerializeWrapper属性等于new OffsetSerializeWrapper()
        遍历offsetTable属性，设置entry等于当前元素
            如果mqs.contain(entry.key)等于true
                添加entry.key和entry.value到offsetSerializeWrapper.offsetTable中
        序列化offsetSerializeWrapper属性，写入到storePath属性对应的文件
    public void persist(MessageQueue mq)
    public void removeOffset(MessageQueue mq)
    public long readOffset(MessageQueue mq, ReadOffsetType type)                                // READ_FROM_MEMORY表示只从内存读取，READ_FROM_STORE表示只从磁盘读取，MEMORY_FIRST_THEN_STORE先读内存，如果没有，读取磁盘。读取磁盘后会更新内存中指定mq
        如果mq参数不等于null
            如果type参数等于MEMORY_FIRST_THEN_STORE或READ_FROM_MEMORY
                设置offset变量等于offsetTable.get(mq)
                如果offset变量不等于null
                    返回offset.get()
                否则如果type参数等于READ_FROM_MEMORY
                    返回-1
            如果type参数等于READ_FROM_STORE
                OffsetSerializeWrapper offsetSerializeWrapper;
                try {
                    设置offsetSerializeWrapper变量等于readLocalOffset()
                } catch (MQClientException e) {
                    返回-1
                }
                如果offsetSerializeWrapper不等于null
                    设置offset变量等于offsetSerializeWrapper.offsetTable.get(mq)
                    如果offset变量不等于null
                        执行updateOffset(mq, offset.get(), false)
                        返回offset.get()
        返回-1
    private OffsetSerializeWrapper readLocalOffset() throws MQClientException               // 读取并解序列化返回本地文件信息
        如果storePath属性对应的文件内容不等于null，解序列化为OffsetSerializeWrapper类型并返回
    public void updateOffset(MessageQueue mq, long offset, boolean increaseOnly)            // 从内存中更新offset，如果出现并发插入，当increaseOnly等于false时，以后边的为主，否则按照谁大用谁的原则
        如果mq参数不等于null
            设置offsetOld变量等于offsetTable.get(mq)
            如果offsetOld变量等于null
                执行offsetTable.putIfAbsent(mq, new AtomicLong(offset))返回实例，设置给offsetOld变量
            如果offsetOld变量不等于null
                如果increaseOnly参数等于true
                    执行MixAll.compareAndIncreaseOnly(offsetOld, offset)                    // 如果offset大于OffsetOld，更新offsetOld到offset。
                否则
                    执行offsetOld.set(offset)
    public Map<MessageQueue, Long> cloneOffsetTable(String topic)
        设置cloneOffsetTable变量等于new HashMap<MessageQueue, Long>()实例
        遍历offsetTable属性，设置entry变量等于当前元素
            如果topic参数等于null或者entry.key.topic等于topic参数
                执行cloneOffsetTable.put(entry.key, entry.value.get())
        返回cloneOffsetTable变量

# com.alibaba.rocketmq.client.consumer.store.RemoteBrokerOffsetStore

    private ConcurrentHashMap<MessageQueue, AtomicLong> offsetTable = new ConcurrentHashMap<MessageQueue, AtomicLong>()

    public RemoteBrokerOffsetStore(MQClientInstance mQClientFactory, String groupName)
        设置mQClientFactory属性等于mQClientFactory参数
        设置groupName属性等于groupName参数
    public void load()
    public void persistAll(Set<MessageQueue> mqs)                                           // 持久化mqs中topic和对应的消息队列和offset信息到brokerServer，删除内存中不存在的消息队列和offset
        如果mqs参数等于null或者元素个数等于0
            退出方法
        设置unusedMQ变量等于new HashSet<MessageQueue>()实例
        遍历offsetTable属性，设置entry等于当前元素
            如果entry.value不等于null
                如果mqs.contain(entry.key)等于true
                    try {
                        执行updateConsumeOffsetToBroker(entry.key, entry.value)
                    } catch (Exception e) { }
                否则
                    添加entry.key到unusedMQ变量中
        如果unusedMQ变量元素个数大于0
            遍历unusedMQ变量，设置mq变量等于当前元素
                执行offsetTable.remove(mq)
    // 通过Oneway的方式，发送更新offset请求
    private void updateConsumeOffsetToBroker(MessageQueue mq, long offset) throws RemotingException, MQBrokerException, InterruptedException, MQClientException
        执行mQClientFactory.findBrokerAddressInAdmin(mq.brokerName)返回FindBrokerResult类型的实例，设置给findBrokerResult变量
        如果findBrokerResult变量等于null
            执行mQClientFactory.updateTopicRouteInfoFromNameServer(mq.topic)
            执行mQClientFactory.findBrokerAddressInAdmin(mq.brokerName)返回FindBrokerResult类型的实例，设置给findBrokerResult变量
        如果findBrokerResult变量等于null
            抛异常MQClientException("The broker...")
        根据mq.topic、groupName属性、mq.queueId、offset参数创建UpdateConsumerOffsetRequestHeader类型的实例，设置给requestHeader变量
        执行mQClientFactory.mQClientAPIImpl.updateConsumerOffsetOneway(findBrokerResult.brokerAddr, requestHeader, 1000 * 5)
    public void persist(MessageQueue mq)                                                    // 持久化mq和对应的offset信息到brokerServer
        设置offset变量等于offsetTable.get(mq)
        如果offset变量不等于null
            try {
                执行updateConsumeOffsetToBroker(mq, offset.get())
            } catch (Exception e) {
            }
    public void removeOffset(MessageQueue mq)                                               // 从内存中移除，后续持久化，该mq不参与
        如果mq不等于null
            执行offsetTable.remove(mq)
    public long readOffset(MessageQueue mq, ReadOffsetType type)                            // READ_FROM_MEMORY表示只从内存读取，READ_FROM_STORE表示只从brokerServer读取，MEMORY_FIRST_THEN_STORE先读内存，如果没有，读取brokerServer。读取brokerServer后会更新内存中指定mq
        如果mq参数不等于null
            如果type参数等于MEMORY_FIRST_THEN_STORE或READ_FROM_MEMORY
                设置offset变量等于offsetTable.get(mq)
                如果offset变量不等于null
                    返回offset.get()
                否则如果type参数等于READ_FROM_MEMORY
                    返回-1
            如果type参数等于READ_FROM_STORE
                try {
                    设置brokerOffset变量等于fetchConsumeOffsetFromBroker(mq)
                    设置offset变量等于new AtomicLong(brokerOffset)
                    执行updateOffset(mq, offset.get(), false)
                    返回brokerOffset变量
                } catch (MQBrokerException e) {
                    返回-1
                } catch (Exception e) {
                    返回-2
                }
        返回-1
    // 从brokerServer获取指定消息队列对于groupName的offset
    private long fetchConsumeOffsetFromBroker(MessageQueue mq) throws RemotingException, MQBrokerException, InterruptedException, MQClientException
        执行mQClientFactory.findBrokerAddressInAdmin(mq.brokerName)返回FindBrokerResult类型的实例，设置给findBrokerResult变量
        如果findBrokerResult变量等于null
            执行mQClientFactory.updateTopicRouteInfoFromNameServer(mq.topic)
            执行mQClientFactory.findBrokerAddressInAdmin(mq.brokerName)返回FindBrokerResult类型的实例，设置给findBrokerResult变量
        如果findBrokerResult变量等于null
            抛异常MQClientException("The broker...")
        根据mq.topic、groupName属性、mq.queueId创建QueryConsumerOffsetRequestHeader类型的实例，设置给requestHeader变量
        执行mQClientFactory.mQClientAPIImpl.queryConsumerOffset(findBrokerResult.brokerAddr, requestHeader, 1000 * 5)并返回
    public void updateOffset(MessageQueue mq, long offset, boolean increaseOnly)            // 从内存中更新offset，如果出现并发插入，当increaseOnly等于false时，以后边的为主，否则按照谁大用谁的原则
        如果mq参数不等于null
            设置offsetOld变量等于offsetTable.get(mq)
            如果offsetOld变量等于null
                执行offsetTable.putIfAbsent(mq, new AtomicLong(offset))返回实例，设置给offsetOld变量
            如果offsetOld变量不等于null
                如果increaseOnly参数等于true
                    执行MixAll.compareAndIncreaseOnly(offsetOld, offset)                    // 如果offset大于OffsetOld，更新offsetOld到offset。
                否则
                    执行offsetOld.set(offset)
    public Map<MessageQueue, Long> cloneOffsetTable(String topic)
        设置cloneOffsetTable变量等于new HashMap<MessageQueue, Long>()实例
        遍历offsetTable属性，设置entry变量等于当前元素
            如果topic参数等于null或者entry.key.topic等于topic参数
                执行cloneOffsetTable.put(entry.key, entry.value.get())
        返回cloneOffsetTable变量

# com.alibaba.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService

    public ConsumeMessageConcurrentlyService(DefaultMQPushConsumerImpl defaultMQPushConsumerImpl, MessageListenerConcurrently messageListener)
        设置defaultMQPushConsumerImpl属性等于defaultMQPushConsumerImpl参数
        设置messageListener属性等于messageListener参数
        设置defaultMQPushConsumer属性等于defaultMQPushConsumerImpl.defaultMQPushConsumer
        设置consumerGroup属性等于defaultMQPushConsumer.consumerGroup
        设置consumeRequestQueue属性等于new LinkedBlockingQueue<Runnable>()
        设置consumeExecutor属性等于new ThreadPoolExecutor(defaultMQPushConsumer.consumeThreadMin, defaultMQPushConsumer.consumeThreadMax, 1000 * 60, TimeUnit.MILLISECONDS, consumeRequestQueue, new ThreadFactoryImpl("ConsumeMessageThread_"))
        设置scheduledExecutorService属性等于Executors.newSingleThreadScheduledExecutor(new ThreadFactoryImpl("ConsumeMessageScheduledThread_"))
    public void start()
    public void shutdown()
        执行scheduledExecutorService.shutdown()
        执行consumeExecutor.shutdown()
    public void incCorePoolSize()
        设置corePoolSize变量等于consumeExecutor.corePoolSize
        如果corePoolSize变量小于defaultMQPushConsumer.consumeThreadMax
            执行consumeExecutor.setCorePoolSize(this.consumeExecutor.getCorePoolSize() + 1)
    public void decCorePoolSize()
        设置corePoolSize变量等于consumeExecutor.corePoolSize
        如果corePoolSize变量大于defaultMQPushConsumer.consumeThreadMin
            执行consumeExecutor.setCorePoolSize(this.consumeExecutor.getCorePoolSize() - 1)
    public ConsumeMessageDirectlyResult consumeMessageDirectly(MessageExt msg, String brokerName)                       // 直接消费一条消息
        设置result变量等于new ConsumeMessageDirectlyResult()实例
        执行result.setOrder(false)
        执行result.setAutoCommit(true)
        设置msgs变量等于new ArrayList<MessageExt>()实例
        执行msgs.add(msg)
        设置mq变量等于new MessageQueue()实例
        执行mq.setBrokerName(brokerName)
        执行mq.setTopic(msg.topic)
        执行mq.setQueueId(msg.queueId)
        设置context变量等于new ConsumeConcurrentlyContext(mq)实例
        执行resetRetryTopic(msgs)
        设置beginTime变量等于System.currentTimeMillis()
        try {
            设置status变量等于messageListener.consumeMessage(msgs, context)
            如果status变量等于null
                执行result.setConsumeResult(CMResult.CR_RETURN_NULL)
            否则如果status变量等于CONSUME_SUCCESS
                执行result.setConsumeResult(CMResult.CR_SUCCESS)
            否则如果status变量等于RECONSUME_LATER
                执行result.setConsumeResult(CMResult.CR_LATER)
        } catch (Throwable e) {
            执行result.setConsumeResult(CMResult.CR_THROW_EXCEPTION)
            执行result.setRemark(RemotingHelper.exceptionSimpleDesc(e))
        }
        执行result.setSpentTimeMills(System.currentTimeMillis() - beginTime)
        返回result变量
    public void resetRetryTopic(List<MessageExt> msgs)
        设置groupTopic变量等于"%RETRY%" + consumerGroup
        遍历msgs参数，设置msg变量等于当前元素
            设置retryTopic变量等于msg.properties.get("RETRY_TOPIC")
            如果retryTopic不等于null并且groupTopic变量等于msg.topic
                执行msg.setTopic(retryTopic)
    public int getCorePoolSize()
        返回consumeExecutor.getCorePoolSize()
    public void submitConsumeRequest(List<MessageExt> msgs, ProcessQueue processQueue, MessageQueue messageQueue, boolean dispathToConsume)     // 按consumeBatchSize指定的大小拆分msgs，每个子列表创建一个ConsumerRequest，启动一个新的线程执行
        设置consumeBatchSize变量等于defaultMQPushConsumer.consumeMessageBatchMaxSize
        如果msgs.size()小于等于consumeBatchSize
            执行consumeExecutor.submit(new ConsumeRequest(msgs, processQueue, messageQueue))
        否则
            for (int total = 0; total < msgs.size();) {
                List<MessageExt> msgThis = new ArrayList<MessageExt>(consumeBatchSize);
                for (int i = 0; i < consumeBatchSize; i++, total++) {
                    if (total < msgs.size()) {
                        msgThis.add(msgs.get(total));
                    }
                    else {
                        break;
                    }
                }
                执行consumeExecutor.submit(new ConsumeRequest(msgThis, processQueue, messageQueue));
            }
    public void processConsumeResult(ConsumeConcurrentlyStatus status, ConsumeConcurrentlyContext context, ConsumeRequest consumeRequest)       // 根据消费后的状态处理消息
        设置ackIndex变量等于context.ackIndex
        如果consumeRequest.msgs元素个数等于0
            退出方法
        如果status参数等于CONSUME_SUCCESS
            如果ackIndex变量大于等于consumeRequest.msgs.size()
                设置ackIndex变量等于consumeRequest.msgs.size() - 1
            设置ok变量等于ackIndex + 1
            设置failed变量等于consumeRequest.msgs.size() - ok
            执行defaultMQPushConsumerImpl.mQClientFactory.consumerStatsManager.incConsumeOKTPS(consumerGroup, consumeRequest.messageQueue.topic, ok)
            执行defaultMQPushConsumerImpl.mQClientFactory.consumerStatsManager.incConsumeFailedTPS(consumerGroup, consumeRequest.messageQueue.topic, failed)
        否则如果status参数等于RECONSUME_LATER
            设置ackIndex变量等于-1
            执行defaultMQPushConsumerImpl.mQClientFactory.consumerStatsManager.incConsumeFailedTPS(consumerGroup, consumeRequest.messageQueue.topic, consumeRequest.msgs.size())
        如果defaultMQPushConsumer.messageModel等于CLUSTERING
            设置msgBackFailed变量等于new ArrayList<MessageExt>(consumeRequest.msgs.size())                                // 集群模式下，如果status等于RECONSUME_LATER，处理方式为
            for (int i = ackIndex + 1; i < consumeRequest.msgs.size(); i++) {
                设置msg变量等于consumeRequest.msgs.get(i)
                设置result变量等于sendMessageBack(msg, context)
                如果result变量等于false                                                                                   // 告知broker需要重新消费和重新发送消息均出现失败，添加到失败列表中
                    执行msg.setReconsumeTimes(msg.reconsumeTimes + 1)
                    执行msgBackFailed.add(msg)
            }
            如果msgBackFailed变量元素个数大于0                                                                             // 如果大于0个，从msgs重移除，然后重新消费移除的
                执行consumeRequest.msgs.removeAll(msgBackFailed)
                执行submitConsumeRequestLater(msgBackFailed, consumeRequest.processQueue, consumeRequest.messageQueue)
        设置offset变量等于consumeRequest.processQueue.removeMessage(consumeRequest.msgs)                                  // 移除msgs，如果运行队列中还存在堆积，返回第一条消息的offset
        如果offset变量大于0
            执行defaultMQPushConsumerImpl.offsetStore.updateOffset(consumeRequest.messageQueue, offset, true)            // 更新offset
    public boolean sendMessageBack(MessageExt msg, ConsumeConcurrentlyContext context)                                  // 如果返回false，表示告知broker重新存储消息和使用Producer重新发送消息均出现失败
        设置delayLevel变量等于context.delayLevelWhenNextConsume
        try {
            执行defaultMQPushConsumerImpl.sendMessageBack(msg, delayLevel, context.messageQueue.brokerName)
            返回true
        } catch (Exception e) {
        }
        返回false
    private void submitConsumeRequestLater(List<MessageExt> msgs, ProcessQueue processQueue, MessageQueue messageQueue) // 延迟5000毫秒，消费msgs
        延迟5000毫秒执行任务
            执行submitConsumeRequest(msgs, processQueue, messageQueue, true)

## com.alibaba.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService.ConsumeRequest

    public ConsumeRequest(List<MessageExt> msgs, ProcessQueue processQueue, MessageQueue messageQueue)
        设置msgs属性等于msgs参数
        设置processQueue属性等于processQueue参数
        设置messageQueue属性等于messageQueue参数
    public void run()
        如果processQueue.dropped等于true                                                                                 // 如果运行队列已被废弃，退出方法
            退出方法
        设置context变量等于new ConsumeConcurrentlyContext(messageQueue)实例                                               // 默认设置ackIndex属性等于Integer.MAX_VALUE
        ConsumeConcurrentlyStatus status = null
        设置consumeMessageContext变量等于null
        如果defaultMQPushConsumerImpl.consumeMessageHookList元素个数大于0
            设置consumeMessageContext变量等于new ConsumeMessageContext()实例
            执行consumeMessageContext.setConsumerGroup(defaultMQPushConsumer.consumerGroup)
            执行consumeMessageContext.setMq(messageQueue)
            执行consumeMessageContext.setMsgList(msgs)
            执行consumeMessageContext.setSuccess(false)
            执行defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext)
        设置beginTimestamp变量等于System.currentTimeMillis()                                                              // 设置开始时间戳
        try {
            执行resetRetryTopic(msgs)
            设置status变量等于messageListener.consumeMessage(Collections.unmodifiableList(msgs), context)                 // 进行消费
        } catch (Throwable e) {
        }
        设置consumeRT变量等于System.currentTimeMillis() - beginTimestamp                                                  // 记录消费时间
        如果status变量等于null
            设置status变量等于RECONSUME_LATER                                                                             // 表示需要重新消费
        如果defaultMQPushConsumerImpl.consumeMessageHookList元素个数大于0
            执行consumeMessageContext.setStatus(status.toString())
            执行consumeMessageContext.setSuccess(ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status)
            执行defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext)
        执行defaultMQPushConsumerImpl.mQClientFactory.consumerStatsManager.incConsumeRT(consumerGroup, messageQueue.topic, consumeRT)
        如果processQueue.dropped等于false
            执行processConsumeResult(status, context, this)                                                              // 根据消息状态处理消息

# com.alibaba.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService
    private static long MaxTimeConsumeContinuously = Long.parseLong(System.getProperty("rocketmq.client.maxTimeConsumeContinuously", "60000"));

    public ConsumeMessageOrderlyService(DefaultMQPushConsumerImpl defaultMQPushConsumerImpl, MessageListenerOrderly messageListener)
        设置defaultMQPushConsumerImpl属性等于defaultMQPushConsumerImpl参数
        设置messageListener属性等于messageListener参数
        设置defaultMQPushConsumer属性等于defaultMQPushConsumerImpl.defaultMQPushConsumer
        设置consumerGroup属性等于defaultMQPushConsumer.consumerGroup
        设置consumeRequestQueue属性等于new LinkedBlockingQueue<Runnable>()
        设置consumeExecutor属性等于new ThreadPoolExecutor(defaultMQPushConsumer.consumeThreadMin, defaultMQPushConsumer.consumeThreadMax, 1000 * 60, TimeUnit.MILLISECONDS, consumeRequestQueue, new ThreadFactoryImpl("ConsumeMessageThread_"))
        设置scheduledExecutorService属性等于Executors.newSingleThreadScheduledExecutor(new ThreadFactoryImpl("ConsumeMessageScheduledThread_"))
    public void start()                                                                                             // 集群模式下，开启定时任务，每隔一段时间执行锁定mq操作
        如果defaultMQPushConsumerImpl.defaultMQPushConsumer.messageModel等于CLUSTERING
            执行定时任务，延迟1秒钟执行，任务间隔为{rocketmq.client.rebalance.lockInterval:20000}毫秒
                执行lockMQPeriodically()
    public synchronized void lockMQPeriodically()
        如果stopped等于false
            执行defaultMQPushConsumerImpl.rebalanceImpl.lockAll()
    public void shutdown()
        设置stopped属性等于true
        执行scheduledExecutorService.shutdown()
        执行consumeExecutor.shutdown()
        如果defaultMQPushConsumerImpl.defaultMQPushConsumer.messageModel等于CLUSTERING
            执行unlockAllMQ()
    public synchronized void unlockAllMQ()
        执行defaultMQPushConsumerImpl.rebalanceImpl.unlockAll(false)
    public void incCorePoolSize()
    public void decCorePoolSize()
    public ConsumeMessageDirectlyResult consumeMessageDirectly(MessageExt msg, String brokerName)                       // 直接消费一条消息
        设置result变量等于new ConsumeMessageDirectlyResult()实例
        执行result.setOrder(false)
        设置msgs变量等于new ArrayList<MessageExt>()实例
        执行msgs.add(msg)
        设置mq变量等于new MessageQueue()实例
        执行mq.setBrokerName(brokerName)
        执行mq.setTopic(msg.topic)
        执行mq.setQueueId(msg.queueId)
        设置context变量等于new ConsumeOrderlyContext(mq)实例
        设置beginTime变量等于System.currentTimeMillis()
        try {
            设置status变量等于messageListener.consumeMessage(msgs, context)
            如果status变量等于null
                执行result.setConsumeResult(CMResult.CR_RETURN_NULL)
            否则如果status变量等于SUCCESS
                执行result.setConsumeResult(CMResult.CR_SUCCESS)
            否则如果status变量等于SUSPEND_CURRENT_QUEUE_A_MOMENT
                执行result.setConsumeResult(CMResult.CR_LATER)
        } catch (Throwable e) {
            执行result.setConsumeResult(CMResult.CR_THROW_EXCEPTION)
            执行result.setRemark(RemotingHelper.exceptionSimpleDesc(e))
        }
        执行result.setAutoCommit(context.autoCommit)
        执行result.setSpentTimeMills(System.currentTimeMillis() - beginTime)
        返回result变量
    public int getCorePoolSize()
        返回consumeExecutor.getCorePoolSize()
    public void submitConsumeRequest(List<MessageExt> msgs, ProcessQueue processQueue, MessageQueue messageQueue, boolean dispathToConsume)
        // dispathToConsume等于true表示添加消息到运行队列时，队列内部的消息队列不等于空并且没人消费，即如果正在有人消费，不会开启新的线程
        如果dispathToConsume参数等于true
            设置consumeRequest变量等于new ConsumeRequest(processQueue, messageQueue)
            执行consumeExecutor.submit(consumeRequest)
    private void submitConsumeRequestLater(ProcessQueue processQueue, MessageQueue messageQueue, long suspendTimeMillis)
        如果suspendTimeMillis参数小于10，设置suspendTimeMillis参数等于10
        如果suspendTimeMillis参数大于3000，设置suspendTimeMillis参数等于3000
        延迟suspendTimeMillis毫秒执行任务
            执行submitConsumeRequest(null, processQueue, messageQueue, true)
    public void tryLockLaterAndReconsume(MessageQueue mq, ProcessQueue processQueue, long delayMills)                   // 延迟delayMills毫秒执行任务，给mq加锁，如果加锁成功，10毫秒后执行submitConsumeRequest，否则3秒后执行submitConsumeRequest
        延迟delayMills毫秒执行任务
            设置lockOK变量等于lockOneMQ(mq)
            如果lockOK变量等于true
                执行submitConsumeRequestLater(processQueue, mq, 10)
            否则
                执行submitConsumeRequestLater(processQueue, mq, 3000)
    public synchronized boolean lockOneMQ(MessageQueue mq)
        如果stopped属性等于false
            执行defaultMQPushConsumerImpl.rebalanceImpl.lock(mq)并返回
        返回false
    public boolean processConsumeResult(List<MessageExt> msgs, ConsumeOrderlyStatus status, ConsumeOrderlyContext context, ConsumeRequest consumeRequest)
        设置continueConsume变量等于true
        设置commitOffset变量等于-1
        如果context.autoCommit等于true                                                                                   // 默认为true
            如果status参数等于SUCCESS
                设置commitOffset变量等于consumeRequest.processQueue.commit()                                              // 清空临时map，返回临时map中的最后一个元素的key + 1，设置msgCount - 临时map个数
                执行defaultMQPushConsumerImpl.mQClientFactory.consumerStatsManager.incConsumeOKTPS(consumerGroup, consumeRequest.messageQueue.topic, msgs.size())
            否则如果status参数等于SUSPEND_CURRENT_QUEUE_A_MOMENT
                执行consumeRequest.processQueue.makeMessageToCosumeAgain(msgs)                                           // 从临时map中移除并加入到堆积map中
                // 延迟suspendCurrentQueueTimeMillis毫秒执行任务，给messageQueue加锁，如果加锁成功，10毫秒后执行submitConsumeRequest，否则3秒后执行submitConsumeRequest
                执行submitConsumeRequestLater(consumeRequest.processQueue, consumeRequest.messageQueue, context.suspendCurrentQueueTimeMillis)
                // 设置当前线程不在继续消费了
                设置continueConsume变量等于false
                执行defaultMQPushConsumerImpl.mQClientFactory.consumerStatsManager.incConsumeFailedTPS(consumerGroup, consumeRequest.messageQueue.topic, msgs.size())
        否则
            如果status参数等于SUCCESS
                执行defaultMQPushConsumerImpl.mQClientFactory.consumerStatsManager.incConsumeOKTPS(consumerGroup, consumeRequest.messageQueue.topic, msgs.size())
            否则如果status参数等于SUSPEND_CURRENT_QUEUE_A_MOMENT
                执行consumeRequest.processQueue.makeMessageToCosumeAgain(msgs)                                           // 从临时map中移除并加入到堆积map中
                // 延迟suspendCurrentQueueTimeMillis毫秒执行任务，给messageQueue加锁，如果加锁成功，10毫秒后执行submitConsumeRequest，否则3秒后执行submitConsumeRequest
                执行submitConsumeRequestLater(consumeRequest.processQueue, consumeRequest.messageQueue, context.suspendCurrentQueueTimeMillis)
                // 设置当前线程不在继续消费了
                设置continueConsume变量等于false
                执行defaultMQPushConsumerImpl.mQClientFactory.consumerStatsManager.incConsumeFailedTPS(consumerGroup, consumeRequest.messageQueue.topic, msgs.size())
        如果commitOffset变量大于等于0
            执行defaultMQPushConsumerImpl.offsetStore.updateOffset(consumeRequest.messageQueue, commitOffset, false)    // autoCommit等于true的时候，更新offset
        返回continueConsume变量                                                                                          // 返回是否继续消费

## com.alibaba.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService.ConsumeRequest
    public ConsumeRequest(ProcessQueue processQueue, MessageQueue messageQueue)
        设置processQueue属性等于processQueue参数
        设置messageQueue属性等于messageQueue参数
    public void run()
        如果processQueue.dropped等于true                                                                                 // 如果运行队列已被废弃，退出方法
            退出方法
        执行messageQueueLock.fetchLockObject(this.messageQueue)返回Object类型的实例，设置给objLock参数
        synchronized (objLock) {                                                                                        // 使用mq持有的value同步（mq本身会不断创建新的实例，无法保证同步）
            // isLockExpired()是否过期根据当前时间减去上次锁定时间戳的差值是否大于rocketmq.client.rebalance.lockMaxLiveTime:30000
            如果defaultMQPushConsumerImpl.defaultMQPushConsumer.messageModel等于BROADCASTING或者processQueue.locked等于true并且isLockExpired()等于false
                设置beginTime变量等于System.currentTimeMillis()
                for (boolean continueConsume = true; continueConsume;) {
                    如果processQueue.dropped等于true                                                                            // 如果运行队列已被废弃，退出方法
                        退出循环
                    如果defaultMQPushConsumerImpl.defaultMQPushConsumer.messageModel等于CLUSTERING并且processQueue.locked等于false  // 如果集群模式下运行队列锁失效，延迟10毫秒执行任务，给messageQueue加锁，如果加锁成功，10毫秒后执行submitConsumeRequest，否则3秒后执行submitConsumeRequest，退出方法
                        执行tryLockLaterAndReconsume(messageQueue, processQueue, 10)
                        退出循环
                    如果defaultMQPushConsumerImpl.defaultMQPushConsumer.messageModel等于CLUSTERING并且processQueue.isLockExpired()等于true  // 如果集群模式下运行队列锁过期，延迟10毫秒执行任务，给messageQueue加锁，如果加锁成功，10毫秒后执行submitConsumeRequest，否则3秒后执行submitConsumeRequest，退出方法
                        执行tryLockLaterAndReconsume(messageQueue, processQueue, 10)
                        退出循环
                    如果System.currentTimeMillis() - beginTime大于MaxTimeConsumeContinuously                                    // 如果循环逻辑执行超过MaxTimeConsumeContinuously，延迟10毫秒执行任务，给messageQueue加锁，如果加锁成功，10毫秒后执行submitConsumeRequest，否则3秒后执行submitConsumeRequest，退出方法
                        执行submitConsumeRequestLater(processQueue, messageQueue, 10)
                        退出循环
                    设置consumeBatchSize变量等于defaultMQPushConsumer.consumeMessageBatchMaxSize
                    // 从运行队列中取出consumeBatchSize个元素
                    //      设置lastConsumeTimestamp等于当前时间
                    //      将取出的元素导入临时map中
                    //      如果返回0条信息，设置consuming等于false
                    执行processQueue.takeMessags(consumeBatchSize)返回List<MessageExt>类型的实例，设置给msgs变量                   // 缺个字母e
                    如果msgs变量的元素个数大于0
                        设置context变量等于new ConsumeOrderlyContext(messageQueue)
                        ConsumeOrderlyStatus status = null
                        设置consumeMessageContext变量等于null
                        如果defaultMQPushConsumerImpl.consumeMessageHookList元素个数大于0
                            设置consumeMessageContext变量等于new ConsumeMessageContext()实例
                            执行consumeMessageContext.setConsumerGroup(defaultMQPushConsumer.consumerGroup)
                            执行consumeMessageContext.setMq(messageQueue)
                            执行consumeMessageContext.setMsgList(msgs)
                            执行consumeMessageContext.setSuccess(false)
                            执行defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext)
                        设置beginTimestamp变量等于System.currentTimeMillis()
                        try {
                            执行processQueue.lockConsume.lock()
                            如果processQueue.dropped等于true
                                退出循环
                            设置status变量等于messageListener.consumeMessage(Collections.unmodifiableList(msgs), context)
                        } catch (Throwable e) {
                        } finally {
                            执行processQueue.lockConsume.unlock()
                        }
                        设置consumeRT变量等于System.currentTimeMillis() - beginTimestamp                                       // 记录消费时间
                        如果status变量等于null
                            设置status变量等于SUSPEND_CURRENT_QUEUE_A_MOMENT                                                   // 如果等于null，设置状态等于SUSPEND_CURRENT_QUEUE_A_MOMENT
                        如果defaultMQPushConsumerImpl.consumeMessageHookList元素个数大于0
                            执行consumeMessageContext.setStatus(status.toString())
                            执行consumeMessageContext.setSuccess(ConsumeOrderlyStatus.SUCCESS == status)
                            执行defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext)
                        执行defaultMQPushConsumerImpl.mQClientFactory.consumerStatsManager.incConsumeRT(consumerGroup, messageQueue.topic, consumeRT)
                        设置continueConsume变量等于processConsumeResult(msgs, status, context, this)                           // 根据消息状态处理消息，根据返回值决定是否继续消费
                    否则
                        设置continueConsume变量等于false                                                                       // 队列中没有元素了，退出循环
                }
            否则                                                                                                        // 集群模式下，lock等于false或者lock已过期
                如果processQueue.dropped等于true                                                                         // 如果运行队列已被废弃，退出方法
                    退出方法
                执行tryLockLaterAndReconsume(messageQueue, processQueue, 100)                                            // 延迟100毫秒执行任务，给messageQueue加锁，如果加锁成功，10毫秒后执行submitConsumeRequest，否则3秒后执行submitConsumeRequest
        }

### com.alibaba.rocketmq.client.impl.consumer.MessageQueueLock

    public Object fetchLockObject(MessageQueue mq)
        设置objLock变量等于mqLockTable.get(mq)
        如果objLock变量等于null
            设置objLock变量等于new Object()实例
            设置prevLock变量等于mqLockTable.putIfAbsent(mq, objLock)
            如果prevLock变量不等于null
                设置objLock变量等于prevLock变量
        返回objLock变量

# com.alibaba.rocketmq.client.impl.factory.MQClientInstance
    private ConcurrentHashMap<String, MQConsumerInner> consumerTable = new ConcurrentHashMap<String, MQConsumerInner>()                         // Key: group, Value: DefaultMQPushConsumerImpl
    // Key: Topic       Value: QueueData列表、BrokerData列表、filterServerTable信息，或者可能只包含orderTopicConf信息
    // 记录了所有订阅消息（正常订阅主题和内置主题）信息
    private ConcurrentHashMap<String, TopicRouteData> topicRouteTable = new ConcurrentHashMap<String, TopicRouteData>()
    private Lock lockHeartbeat = new ReentrantLock()
    private Lock lockNamesrv = new ReentrantLock()
    // Key: Broker Name    Value: Key: BrokerId  Value: Address
    // 同一个name下边可能有一组id和addr，其中0表示master，其他的是slave
    // addr一定要在topicRouteTable中存在
    private ConcurrentHashMap<String, HashMap<Long, String>> brokerAddrTable = new ConcurrentHashMap<String, HashMap<Long, String>>()
    private ServiceState serviceState = ServiceState.CREATE_JUST

    public MQClientInstance(ClientConfig clientConfig, int instanceIndex, String clientId, RPCHook rpcHook)
        设置clientConfig属性等于clientConfig参数
        设置instanceIndex属性等于instanceIndex参数
        设置nettyClientConfig属性等于new NettyClientConfig()实例
        执行nettyClientConfig.setClientCallbackExecutorThreads(clientConfig.clientCallbackExecutorThreads)
        设置clientRemotingProcessor属性等于new ClientRemotingProcessor(this)实例                                                          // 用于接收brokerServer发出的请求
        设置mQClientAPIImpl属性等于new MQClientAPIImpl(this.nettyClientConfig, this.clientRemotingProcessor, rpcHook)实例
        如果clientConfig.namesrvAddr不等于null                                                                                           // 如果指定了namesevers，用;分隔成列表设置nameserver
            执行mQClientAPIImpl.updateNameServerAddressList(clientConfig.namesrvAddr)
        设置clientId属性等于clientId参数
        设置mQAdminImpl属性等于new MQAdminImpl(this)实例
        设置pullMessageService属性等于new PullMessageService(this)实例
        设置rebalanceService属性等于new RebalanceService(this)实例
        设置defaultMQProducer属性等于new DefaultMQProducer("CLIENT_INNER_PRODUCER")实例                                                   // 创建producer，用于发送消费失败的消息
        执行defaultMQProducer.resetClientConfig(clientConfig)
        设置consumerStatsManager属性等于new ConsumerStatsManager(this.scheduledExecutorService)实例
    public boolean registerConsumer(String group, MQConsumerInner consumer)                                                            // 注册group，用于检测心跳和获取所有正常订阅主题及内置主题来维护topicRouteTable和brokerAddrTable属性
        如果group等于null或者consumer等于null
            返回false
        执行consumerTable.putIfAbsent(group, consumer)，设置返回值给prev变量
        如果prev变量不等于null
            返回false
        返回true
    public void start() throws MQClientException
        如果serviceState属性等于CREATE_JUST
            设置serviceState属性等于START_FAILED
            如果clientConfig.namesrvAddr等于null                                                                                        // 如果没有手动指定nameservers，先通过http拉取nameservers并设置
                执行clientConfig.setNamesrvAddr(mQClientAPIImpl.fetchNameServerAddr())
            执行mQClientAPIImpl.start()
            如果clientConfig.namesrvAddr等于null                                                                                        // 如果没有拉取到，执行定时任务，每隔120秒拉取重新拉取一次
                启动定时任务，延迟10秒钟执行，任务间隔为120秒
                    try {
                        执行mQClientAPIImpl.fetchNameServerAddr()
                    } catch (Exception e) { }
            执行定时任务，延迟10毫秒执行，任务间隔为clientConfig.pollNameServerInteval毫秒                                                   // 每隔一段时间，需要获取所有主题（正常订阅的和内置的），用来维护topicRouteTable和brokerAddrTable属性，以及topicSubscribeInfoTable
                执行updateTopicRouteInfoFromNameServer()
            执行定时任务，延迟1000毫秒执行，任务间隔为clientConfig.heartbeatBrokerInterval毫秒                                               // 每隔一段时间，重新整理一遍brokerAddrTable，然后和其中所有的BrokerAddr做心跳
                try {
                    执行cleanOfflineBroker()
                    执行sendHeartbeatToAllBrokerWithLock()
                } catch (Exception e) { }
            指定定时任务，延迟10秒钟执行，任务执行间隔为clientConfig.persistConsumerOffsetInterval毫秒                                       // 每隔一段时间，定时持久化所有mq的offset信息
                try {
                    执行persistAllConsumerOffset()
                } catch (Exception e) { }
            执行定时任务，延迟1分钟执行，任务执行间隔为1分钟                                                                                 // 每隔一段时间，定时调整消息的消费线程池，只对并行消费场景有效
                try {
                    执行adjustThreadPool()
                } catch (Exception e) { }
            执行pullMessageService.start()
            执行rebalanceService.start()                                                                                              // 定时做rebalance，允许外部立即触发rebalance
            执行defaultMQProducer.defaultMQProducerImpl.start(false)
            设置serviceState属性等于RUNNING
        否则如果serviceState属性等于START_FAILED
            抛异常MQClientException("The Factory object...")
    private void cleanOfflineBroker()                                                                                                // 重新整理一遍brokerAddrTable，保证每个brokerAddr确实被某个topic使用
        try {
            if (lockNamesrv.tryLock(3000, TimeUnit.MILLISECONDS))
                try {
                    遍历brokerAddrTable属性，设置entry变量等于当前元素                                                                    // entry: Key(brokerName), Value(Map<BrokerId, BrokerAddress>)
                        遍历entry.value，设置subEntry变量等于当前元素
                            如果isBrokerAddrExistInTopicRouteTable(subEntry.value)等于false
                                移除当前元素
                        如果entry.value元素个数等于空
                            移除当前元素
                } finally {
                    lockNamesrv.unlock();
                }
        } catch (InterruptedException e) { }
    private boolean isBrokerAddrExistInTopicRouteTable(String addr)                                                                  // 检测指定brokerAddress是否被topicRouteTable中某个topic使用
        遍历topicRouteTable属性，设置entry变量等于当前元素                                                                                // entry: Key: Topic, Value: QueueData列表、BrokerData列表、filterServerTable信息，或者可能只包含orderTopicConf信息
            遍历entry.value.brokerDatas，设置subEntry变量等于当前元素
                如果subEntry.brokerAddrs不等于null并且包含addr参数，返回true
        返回false
    public void sendHeartbeatToAllBrokerWithLock()                                                                                   // 和brokerAddrTable中所有BrokerAddr做心跳
        如果lockHeartbeat.tryLock()等于true
            try {
                执行sendHeartbeatToAllBroker()
                执行uploadFilterClassSource()
            } catch (Exception e) {
            } finally {
                lockHeartbeat.unlock()
            }
    private void sendHeartbeatToAllBroker()
        创建new HeartbeatData()实例，设置给heartbeatData变量
        执行heartbeatData.setClientID(clientId)
        遍历consumerTable属性，设置entry变量等于当前元素                                                                                  // entry: Key: Group, Value: DefaultMQPushConsumerImpl
            如果entry.value不等于null
                ConsumerData consumerData = new ConsumerData();
                consumerData.setGroupName(entry.value.defaultMQPushConsumer.consumerGroup)
                consumerData.setConsumeType(CONSUME_PASSIVELY)
                consumerData.setMessageModel(entry.value.defaultMQPushConsumer.messageModel)
                consumerData.setConsumeFromWhere(entry.value.defaultMQPushConsumer.consumeFromWhere)
                consumerData.subscriptionDataSet.addAll(entry.value.subscriptions())
                consumerData.setUnitMode(entry.value.defaultMQPushConsumer.unitMode)
                heartbeatData.consumerDataSet.add(consumerData);
        如果heartbeatData.consumerDataSet.isEmpty()等于false
            遍历brokerAddrTable属性，设置entry变量等于当前元素                                                                            // entry: Key(brokerName), Value(Map<BrokerId, BrokerAddress>)
                如果entry.value不等于null
                    遍历entry.value，设置subEntry变量等于当前元素
                        如果subEntry.value不等于null
                            如果heartbeatData.consumerDataSet.isEmpty()等于true并且subEntry.key不等于0                                  // MASTER_ID: 0
                                continue
                            try {
                                mQClientAPIImpl.sendHearbeat(addr, heartbeatData, 3000);
                            } catch (Exception e) {
                            }
    private void uploadFilterClassSource()                                                                                           // 如果配置的SubscriptionData为类模式，上传过滤类名及源码到filterServer实现类似于协处理器这样的服务端过滤
        遍历consumerTable属性，设置entry变量等于当前元素                                                                                  // entry: Key: Group, Value: DefaultMQPushConsumerImpl
            遍历entry.value.subscriptions()，设置sub变量等于当前元素
                如果sub.classFilterMode等于true并且sub.filterClassSource不等于null
                    设置consumerGroup变量等于entry.value.defaultMQPushConsumer.consumerGroup
                    设置className变量等于sub.subString
                    设置topic变量等于sub.topic
                    设置filterClassSource变量等于sub.filterClassSource
                    try {
                        执行uploadFilterClassToAllFilterServer(consumerGroup, className, topic, filterClassSource)
                    } catch (Exception e) {
                    }
    private void uploadFilterClassToAllFilterServer(String consumerGroup, String fullClassName, String topic, String filterClassSource) throws UnsupportedEncodingException
        byte[] classBody = null;
        int classCRC = 0;
        try {
            设置classBody变量等于filterClassSource.getBytes("UTF-8")
            设置classCRC变量等于UtilAll.crc32(classBody)
        } catch (Exception e1) {
        }
        设置topicRouteData变量等于topicRouteTable.get(topic)
        如果topicRouteData变量不等于null并且topicRouteData.filterServerTable不等于null并且topicRouteData.filterServerTable元素个数大于0    // filterServerTable: Key: brokerServer Value: List<filterServer>
            遍历topicRouteData.filterServerTable属性，设置entry等于当前元素
                遍历entry.value，设置fsAddr变量等于当前元素
                    try {
                        this.mQClientAPIImpl.registerMessageFilterClass(fsAddr, consumerGroup, topic, fullClassName, classCRC, classBody, 5000);
                    } catch (Exception e) {
                    }
    private void persistAllConsumerOffset()
        遍历consumerTable属性，设置entry变量等于当前元素                                                                                 // entry: Key: Group, Value: DefaultMQPushConsumerImpl
            执行entry.value.persistConsumerOffset()
    public void adjustThreadPool()
        遍历consumerTable属性，设置entry变量等于当前元素                                                                                 // entry: Key: Group, Value: DefaultMQPushConsumerImpl
            如果entry.value不等于null
                执行entry.value.adjustThreadPool()方法
    public void unregisterConsumer(String group)                                                                                    // 对brokerAddrTable中所有的BrokerAddr取消注册
        执行consumerTable.remove(group)
        try {
            如果this.lockHeartbeat.tryLock(3000, TimeUnit.MILLISECONDS)等于true
                try {
                    执行unregisterClient(null, group);
                } catch (Exception e) {
                } finally {
                    this.lockHeartbeat.unlock();
                }
        } catch (InterruptedException e) { }
    private void unregisterClient(String producerGroup, String consumerGroup)
        遍历brokerAddrTable属性，设置entry等于当前元素                                                                                 // entry: Key(brokerName), Value(Map<BrokerId, BrokerAddress>)
            如果entry.value不等于null，遍历entry.value，设置subEntry等于当前元素
                如果subEntry.value不等于null
                    try {
                        this.mQClientAPIImpl.unregisterClient(subEntry.value, clientId, producerGroup, consumerGroup, 3000);
                    } catch (RemotingException e) {
                    } catch (MQBrokerException e) {
                    } catch (InterruptedException e) {
                    }
    public void shutdown()
        如果consumerTable属性元素个数大于0，退出
        如果serviceState属性等于RUNNING
            执行defaultMQProducer.defaultMQProducerImpl.shutdown(false)
            设置serviceState属性等于SHUTDOWN_ALREADY
            执行pullMessageService.shutdown(true)
            关闭定时任务
            执行mQClientAPIImpl.shutdown()
            执行rebalanceService.shutdown()
            执行MQClientManager.getInstance().removeClientFactory(this.clientId)
    public void updateTopicRouteInfoFromNameServer()                                                                              // 更新所有正常订阅的主题和内置主题
        设置topicList变量等于new HashSet<String>()实例
        遍历consumerTable属性，设置entry变量等于当前元素                                                                               // entry: Key: Group, Value: DefaultMQPushConsumerImpl
            如果entry.value不等于null并且entry.value.subscriptions()不等于null
                遍历entry.value.subscriptions()，设置subEntry变量等于当前元素                                                         // 添加DefaultMQPushConsumerImpl所有正常订阅的主题和内置主题
                    执行topicList.add(subEntry.topic)
        遍历topicList变量，设置entry变量等于当前元素
            执行updateTopicRouteInfoFromNameServer(entry)
    public boolean updateTopicRouteInfoFromNameServer(String topic)
        执行updateTopicRouteInfoFromNameServer(topic, false, null)并返回
    public boolean updateTopicRouteInfoFromNameServer(String topic, boolean isDefault, DefaultMQProducer defaultMQProducer)
        try {
            if (lockNamesrv.tryLock(3000, TimeUnit.MILLISECONDS)) {
                try {
                    如果isDefault参数等于true并且defaultMQProducer参数不等于null
                        设置topicRouteData变量等于mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer("TBW102", 1000 * 3)       // 随机选中一台nameserver，获取TBW102主题TopicRouteData信息，相当于获取默认配置
                        如果topicRouteData变量不等于null
                            遍历topicRouteData.queueDatas，设置data等于当前元素
                            设置queueNums等于Math.min(defaultMQProducer.defaultTopicQueueNums, data.readQueueNums)
                            执行data.setReadQueueNums(queueNums)
                            执行data.setWriteQueueNums(queueNums)
                    否则
                        设置topicRouteData变量等于mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3)                 // 随机选中一台nameserver，获取TopicRouteData信息
                    如果topicRouteData变量不等于null
                        设置changed变量等于topicRouteDataIsChange(topicRouteTable.get(topic), topicRouteData)                      // 检测queue列表和broker列表是否相等
                        如果changed变量等于false
                            设置changed变量等于isNeedUpdateTopicRouteInfo(topic)                                                   // 检测主题是否在subscriptionInner中存在，在topicSubscribeInfoTable中不存在
                        如果changed变量等于true
                            遍历topicRouteData.brokerDatas，设置bd变量等于当前元素
                                执行brokerAddrTable.put(bd.brokerName, bd.brokerAddrs)                                            // 维护brokerAddrTable
                            设置subscribeInfo变量等于topicRouteData2TopicSubscribeInfo(topic, topicRouteData)
                            遍历consumerTable属性，设置entry变量等于当前元素
                                如果entry.value不等于null
                                    执行entry.value.updateTopicSubscribeInfo(topic, subscribeInfo)                                // 更新主题和主题可读的队列关系
                            执行topicRouteTable.put(topic, cloneTopicRouteData)                                                   // 维护topicRouteTable
                            返回true
                } catch (Exception e) {
                } finally {
                    lockNamesrv.unlock();
                }
            }
        } catch (InterruptedException e) { }
        返回false
    private boolean topicRouteDataIsChange(TopicRouteData olddata, TopicRouteData nowdata)                                      // 比较对应的queueDatas和brokerDatas属性
        如果olddata等于null或者nowdata等于null
            返回true
        设置old变量等于olddata.cloneTopicRouteData()
        设置now变量等于nowdata.cloneTopicRouteData()
        对old.queueDatas进行排序
        对old.brokerDatas进行排序
        对now.queueDatas进行排序
        对now.brokerDatas进行排序
        如果old等于now，返回true，否则返回false
    private boolean isNeedUpdateTopicRouteInfo(String topic)                                                                    // 如果这个主题在subscriptionInner中存在，在topicSubscribeInfoTable中不存在，则需要更新
        遍历consumerTable属性，设置entry变量等于当前元素                                                                             // entry: Key: Group, Value: DefaultMQPushConsumerImpl
            如果entry.value不等于null并且entry.value.isSubscribeTopicNeedUpdate(topic)等于true
                返回true
        返回false
    public static Set<MessageQueue> topicRouteData2TopicSubscribeInfo(String topic, TopicRouteData route)                       // 将route转换为MessageQueue列表，主要是填充可读的MessageQueue
        设置mqList变量new HashSet<MessageQueue>()实例
        遍历route.queueDatas，设置qd变量等于当前元素
            如果qd.perm有读权限
                循环qd.readQueueNums次，设置i变量等于循环索引                                                                       // 对topic主题添加readQueueNums个消息队列
                    创建new MessageQueue(topic, qd.brokerName, i)实例添加到mqList变量中
        返回mqList变量

    public void rebalanceImmediately()                                                                                         // 平衡所有订阅主题的消息队列
        执行rebalanceService.wakeup()
    public void doRebalance()                                                                                                  // 平衡所有订阅主题的消息队列
        遍历consumerTable属性，设置entry等于当前元素                                                                               // entry: Key: Group, Value: DefaultMQPushConsumerImpl
            如果entry.value不等于null
                执行entry.value.doRebalance()
    public FindBrokerResult findBrokerAddressInAdmin(String brokerName)                                                        // 尽可能返回brokerName对应的master的brokerAddr，如果不存在，返回最后一个非master的brokerAddr，如果没有，返回null
        设置brokerAddr变量等于null
        设置slave变量等于false
        设置found变量等于true
        设置map变量等于brokerAddrTable.get(brokerName)                                                                          // entry: Key(brokerName), Value(Map<BrokerId, BrokerAddress>)
        如果map变量不等于null并且元素个数大于0，进行遍历，设置entry变量等于当前元素
            设置brokerAddr变量等于entry.value
            如果brokerAddr变量不等于null
                设置found变量等于true
                如果entry.key等于0                      // MASTER_ID: 0
                    设置slave变量等于false
                    退出循环
                否则
                    设置slave变量等于true
        如果found变量等于true
            返回new FindBrokerResult(brokerAddr, slave)实例
        返回null
    public List<String> findConsumerIdList(String topic, String group)                                                        // 获取当前topic的一个brokerAddr，请求brokerAddr获取配置了相同group的clientId列表
        设置brokerAddr变量等于findBrokerAddrByTopic(topic)
        如果brokerAddr变量等于null
            执行updateTopicRouteInfoFromNameServer(topic)
            设置brokerAddr变量等于findBrokerAddrByTopic(topic)
        如果brokerAddr变量不等于null
            try {
                执行mQClientAPIImpl.getConsumerIdListByGroup(brokerAddr, group, 3000)并返回
            } catch (Exception e) {
            }
    public String findBrokerAddrByTopic(String topic)                                                                         // 返回当前topic的brokerServer列表中第一个brokerServer的brokerAddr
        设置topicRouteData变量等于topicRouteTable.get(topic)                                                                    // entry: Key: Topic, Value: QueueData列表、BrokerData列表、filterServerTable信息，或者可能只包含orderTopicConf信息
        如果topicRouteData变量不等于null
            设置brokers变量等于topicRouteData.brokerDatas
                如果brokers变量元素个数大于0
                    返回brokers.get(0).selectBrokerAddr()
        返回null
    public FindBrokerResult findBrokerAddressInSubscribe(String brokerName, long brokerId, boolean onlyThisBroker)           // 返回brokerName对应的brokerId的brokerAddr，如果不存在并且onlyThisBroker等于false，对于brokerName的map不为空场景，返回第一个brokerId的brokerAddr，其他情况返回null
        设置brokerAddr变量等于null
        设置slave变量等于false
        设置found变量等于false
        设置map变量等于brokerAddrTable.get(brokerName)                                                                         // entry: Key(brokerName), Value(Map<BrokerId, BrokerAddress>)
        如果map变量不等于null并且元素个数大于0
            设置brokerAddr变量等于map.get(brokerId)
            如果brokerId等于0                   // MASTER_ID: 0
                设置slave变量等于false
            否则
                设置slave变量等于true
            如果brokerAddr变量不等于null
                设置found变量等于true
            否则
                设置found变量等于false
            如果find变量等于false并且onlyThisBroker变量等于false
                设置entry变量等于map.entrySet().iterator().next()
                设置brokerAddr变量等于entry.value
                如果entry.key不等于0
                    设置slave变量等于true
                否则
                    设置found变量等于false
                设置found变量等于true
        如果found变量等于true
            返回new FindBrokerResult(brokerAddr, slave)实例
        返回null
    public MQConsumerInner selectConsumer(String group)
        执行consumerTable.get(group)并返回                                                                                   // entry: Key: Group, Value: DefaultMQPushConsumerImpl
    public String findBrokerAddressInPublish(String brokerName)                                                            // 返回brokerName对应的masterId的brokerAdddr
        设置map变量等于brokerAddrTable.get(brokerName)                                                                       // map: Map<BrokerId, BrokerAddress>
        如果map变量不等于null并且元素个数大于0
            返回map.get(0)                    // MASTER_ID: 0
        返回null
    public void resetOffset(String topic, String group, Map<MessageQueue, Long> offsetTable)                                // 强制根据offsetTable重置offset
        设置consumer变量等于null
        try {
            设置impl变量等于consumerTable.get(group)                                                                          // entry: Key: Group, Value: DefaultMQPushConsumerImpl
            如果impl变量不等于null
                设置consumer变量等于impl变量
            否则
                退出方法
            设置processQueueTable变量等于consumer.rebalanceImpl.processQueueTable                                             // Key: MessageQueue, Value: ProcessQueue
            遍历processQueueTable变量，设置entry变量等于当前元素
                如果entry.key.topic等于topic                                                                                 // 如果消息队列处理的主题等于topic，废弃对应的ProcessQueue
                    执行entry.value.setDropped(true)
                    执行entry.value.clear()
            遍历offsetTable变量，设置entry变量等于当前元素
                执行consumer.updateConsumeOffset(entry.key, entry.value)                                                    // 强制更新offset
            执行consumer.offsetStore.persistAll(offsetTable.keySet())                                                       // 以offsetTable为主，持久化一遍offset
            try {
                TimeUnit.SECONDS.sleep(10);
            } catch (InterruptedException e) {
            }
            遍历offsetTable变量，设置entry变量等于当前元素
                执行consumer.updateConsumeOffset(entry.key, entry.value)                                                    // 强制更新offset
            执行consumer.offsetStore.persistAll(offsetTable.keySet())                                                       // 以offsetTable为主，持久化一遍offset
            遍历offsetTable变量，设置entry变量等于当前元素
                执行processQueueTable.remove(entry.key)                                                                     // 移除processQueueTable中的消息队列，然后重新rebalance
        } finally {
            执行consumer.rebalanceImpl.doRebalance()
        }
    public Map<MessageQueue, Long> getConsumerStatus(String topic, String group)                                           // 复制一份group对应的DefaultMQPushConsumerImpl持有的topic的offset数据并返回
        设置consumer变量等于consumerTable.get(group)
        如果consumer变量不等于null
            返回consumer.offsetStore.cloneOffsetTable(topic)
        返回Collections.EMPTY_MAP
    public ConsumerRunningInfo consumerRunningInfo(String consumerGroup)                                                  // 获取consumerGroup的运行时信息，同时添加nameserver、consumer_type、client版本并返回
        设置consumer变量等于consumerTable.get(consumerGroup)
        设置consumerRunningInfo变量等于consumer.consumerRunningInfo()
        设置nsList变量等于mQClientAPIImpl.remotingClient.getNameServerAddressList()                                         // 返回所有nameserver地址拼接成字符串
        设置nsAddr变量等于""
        如果nsList变量不等于null，进行遍历，设置addr变量等于当前元素
            执行nsAddr = nsAddr + addr + ";"
        添加PROP_NAMESERVER_ADDR和nsAddr到consumerRunningInfo.properties中                                                  // 添加所有nameserver地址给consumerRunningInfo
        添加PROP_CONSUME_TYPE和CONSUME_PASSIVELY到consumerRunningInfo.properties中                                          // 添加consume_type给consumerRunningInfo
        添加PROP_CLIENT_VERSION和MQVersion.getVersionDesc(MQVersion.CurrentVersion)到consumerRunningInfo.properties中       // 添加当前版本给consumerRunningInfo
        返回consumerRunningInfo变量
    public ConsumeMessageDirectlyResult consumeMessageDirectly(MessageExt msg, String consumerGroup, String brokerName)   // 消费服务端发送的消息
        设置consumer变量等于consumerTable.get(consumerGroup)
        如果consumer变量不等于null
            返回consumer.consumeMessageService.consumeMessageDirectly(msg, brokerName)
        返回null

## com.alibaba.rocketmq.client.impl.consumer.RebalanceService

    public RebalanceService(MQClientInstance mqClientFactory)
        设置mqClientFactory属性等于mqClientFactory参数
    public void start()
    public void shutdown()
    public void run()                                                                                                           // 定时做rebalance，允许外部立即触发rebalance
        循环执行，直到stop属性等于true
            如果hasNotified属性等于true
                设置hasNotified属性等于false
            否则
                wait(10 * 1000)
                设置hasNotified属性等于false
            执行mqClientFactory.doRebalance()
    public void wakeup()
        如果hasNotified属性等于false
            设置hasNotified属性等于true
            执行notify()

## com.alibaba.rocketmq.client.impl.consumer.PullMessageService

    public PullMessageService(MQClientInstance mQClientFactory)
        设置mqClientFactory属性等于mqClientFactory参数
    public void start()
    public void shutdown(boolean interrupt)
    public void run()                                                                                                           // pullequest处理器，从queue中获取拉取请求并执行，此处是单线程处理所有请求
        循环执行，直到stop属性等于true
            try {
                设置pullRequest变量等于pullRequestQueue.take()
                如果pullRequest变量不等于null
                    执行pullMessage(pullRequest)
            } catch (Exception e) {}
    private void pullMessage(PullRequest pullRequest)                                                                           // 获取对应的DefaultMQPushConsumerImpl实例，执行拉取请求
        设置consumer变量等于mQClientFactory.selectConsumer(pullRequest.consumerGroup)
        如果consumer变量不等于null
            执行consumer.pullMessage(pullRequest)
    public void executePullRequestImmediately(PullRequest pullRequest)                                                          // 添加拉取请求到queue中
        try {
          执行pullRequestQueue.put(pullRequest);
        } catch (InterruptedException e) {
        }
    public void executePullRequestLater(PullRequest pullRequest, long timeDelay)                                                // 延迟timeDelay毫秒，添加拉取请求到queue中
        延迟timeDelay毫秒执行executePullRequestImmediately(pullRequest)
    public void executeTaskLater(Runnable r, long timeDelay)
        延迟timeDelay毫秒执行r

## com.alibaba.rocketmq.client.stat.ConsumerStatsManager

    public ConsumerStatsManager(ScheduledExecutorService scheduledExecutorService)
        // 全是统计用的，不看了

### com.alibaba.rocketmq.client.impl.ClientRemotingProcessor

    public ClientRemotingProcessor(MQClientInstance mqClientFactory)
        设置mqClientFactory属性等于mqClientFactory参数
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request)
        否则如果request.code等于40                  // NOTIFY_CONSUMER_IDS_CHANGED
            执行notifyConsumerIdsChanged(ctx, request)并返回
        否则如果request.code等于220                 // RESET_CONSUMER_CLIENT_OFFSET
            执行resetOffset(ctx, request)并返回
        否则如果request.code等于221                 // GET_CONSUMER_STATUS_FROM_CLIENT
            执行getConsumeStatus(ctx, request)并返回
        否则如果request.code等于307                 // GET_CONSUMER_RUNNING_INFO
            执行getConsumerRunningInfo(ctx, request)并返回
        否则如果request.code等于309                 // CONSUME_MESSAGE_DIRECTLY
            执行consumeMessageDirectly(ctx, request)并返回
    public RemotingCommand notifyConsumerIdsChanged(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException         // 接收服务端传递的clientId列表发生变更通知，需要立即rebalance
        执行mqClientFactory.rebalanceImmediately()
    public RemotingCommand resetOffset(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException                      // 接收服务端重置offset请求
        解序列化request参数为ResetOffsetRequestHeader类型的实例，设置给requestHeader
        设置offsetTable变量等于new HashMap<MessageQueue, Long>()实例
        如果request.body不等于null
            解序列化request.body为ResetOffsetBody类型的实例，设置给body变量
            设置offsetTable变量等于body.offsetTable
        执行mqClientFactory.resetOffset(requestHeader.topic, requestHeader.group, offsetTable)
        返回null
    public RemotingCommand getConsumeStatus(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException                 // 接收服务端获取topic下group的所有消息队列的offset请求
        设置response变量等于RemotingCommand.createResponseCommand(null)
        解序列化request参数为GetConsumerStatusRequestHeader类型的实例，设置给requestHeader
        设置offsetTable变量等于mqClientFactory.getConsumerStatus(requestHeader.topioc, requestHeader.group)
        根据offsetTable变量创建GetConsumerStatusBody类型的实例，设置给body变量
        执行body.setMessageQueueTable(offsetTable)
        执行response.setBody(body.encode())
        执行response.setCode(0)                                                                   // SUCCESS: 0
        返回response变量
    // 接收服务端获取group的consumer运行时信息请求并返回，包括
    //      consumer实例配置信息、是否是顺序消费、线程池信息、consumer启动时间、所有订阅信息（topic/group/subExpression）
    //      nameserver列表、consumer_type（推还是拉）、client版本
    //      消息队列信息和运行队列信息，运行时队列信息包括
    //              offset信息、队列中堆积第一条消息的offset和最后一条消息的offset、队列中堆积消息个数大小、队列中正在处理的一批消息的第一条消息的offset和最后一条消息的offset、队列中正在处理的一批消息的消息个数大小
    //              队列是否处于lock状态、队列执行的tryUnlock次数、队列上次lock的时间戳、队列是否处于废弃状态、队列上次拉取的时间戳、队列上次消费的时间戳
    private RemotingCommand getConsumerRunningInfo(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        设置response变量等于RemotingCommand.createResponseCommand(null)
        解序列化request参数为GetConsumerRunningInfoRequestHeader类型的实例，设置给requestHeader
        设置consumerRunningInfo变量等于mqClientFactory.consumerRunningInfo(requestHeader.consumerGroup)
        如果consumerRunningInfo变量不等于null
            如果requestHeader.jstackEnable等于true                                                                                              // 根据参数，返回当前所有栈信息
                执行consumerRunningInfo.setJstack(UtilAll.jstack())
            执行response.setCode(0)                                                               // SUCCESS: 0
            执行response.setBody(consumerRunningInfo.encode())
        否则
            执行response.setCode(1)                                                               // SYSTEM_ERROR: 1
            执行response.setRemark("The Consumer Group not exist in this consumer")
        返回response变量
    private RemotingCommand consumeMessageDirectly(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException        // 接收和消费服务端发送过来的消息
        设置response变量等于RemotingCommand.createResponseCommand(null)
        解序列化request参数为ConsumeMessageDirectlyResultRequestHeader类型的实例，设置给requestHeader
        解序列化request.body为MessageExt类型的实例，设置给msg变量
        执行mqClientFactory.consumeMessageDirectly(msg, requestHeader.consumerGroup, requestHeader.brokerName)返回ConsumeMessageDirectlyResult类型的实例，设置给result变量
        如果result变量不等于null
            执行response.setCode(0)                                                               // SUCCESS: 0
            执行response.setBody(result.encode())
        否则
            执行response.setCode(1)                                                               // SYSTEM_ERROR: 1
            执行response.setRemark("The Consumer Group not exist in this consumer")
        返回response变量

# com.alibaba.rocketmq.client.impl.MQAdminImpl

    public MQAdminImpl(MQClientInstance mQClientFactory)
        设置mQClientFactory属性等于mQClientFactory参数
    // 返回brokerName对应的masterId的brokerAdddr，然后向对应的brokerAddr发出请求，获取消息队列的最大offset
    public long maxOffset(MessageQueue mq) throws MQClientException
        设置brokerAddr变量等于mQClientFactory.findBrokerAddressInPublish(mq.brokerName)
        如果brokerAddr变量等于null
            执行mQClientFactory.updateTopicRouteInfoFromNameServer(mq.topic)
            设置brokerAddr变量等于mQClientFactory.findBrokerAddressInPublish(mq.brokerName)
        如果brokerAddr变量不等于null
            try {
                返回mQClientFactory.mQClientAPIImpl.getMaxOffset(brokerAddr, mq.topic, mq.queueId, 1000 * 3)
            } catch (Exception e) {
                抛异常new MQClientException("Invoke Broker addr exception", e)
            }
        抛异常MQClientException("The broker name not exist", null)
    // 返回brokerName对应的masterId的brokerAdddr，然后想对应的brokerAddr发出请求，获取消息队列对应时间戳的offset
    public long searchOffset(MessageQueue mq, long timestamp) throws MQClientException
        设置brokerAddr变量等于mQClientFactory.findBrokerAddressInPublish(mq.brokerName)
        如果brokerAddr变量等于null
            执行mQClientFactory.updateTopicRouteInfoFromNameServer(mq.topic)
            设置brokerAddr变量等于mQClientFactory.findBrokerAddressInPublish(mq.brokerName)
        如果brokerAddr变量不等于null
            try {
                返回mQClientFactory.mQClientAPIImpl.searchOffset(brokerAddr, mq.topic, mq.queueId, timestamp, 1000 * 3)
            } catch (Exception e) {
                抛异常new MQClientException("Invoke Broker addr exception", e)
            }
        抛异常MQClientException("The broker name not exist", null)

# com.alibaba.rocketmq.client.impl.MQClientAPIImpl
    private TopAddressing topAddressing = new TopAddressing("http://{rocketmq.namesrv.domain:jmenv.tbsite.net}:8080/rocketmq/{rocketmq.namesrv.domain.subgroup:nsaddr}")
    private String nameSrvAddr = null
    private String projectGroupPrefix                                                       // 区分不同应用下相同的主题

    public String fetchNameServerAddr()                                                     // 获取和设置namserver列表
        设置addrs变量等于topAddressing.fetchNSAddr()                                          // 请求url获取对应的响应文本，每次调用，都会重新请求
        如果addrs变量不等于null并且不等于nameSrvAddr属性
            设置nameSrvAddr属性等于addrs变量
            执行updateNameServerAddressList(addrs)
        返回nameSrvAddr属性
    public void updateNameServerAddressList(String addrs)                                   // 用;分隔成列表设置nameserver
        使用;分隔addrs参数成List，设置给lst变量
        执行remotingClient.updateNameServerAddressList(lst)

    public MQClientAPIImpl(NettyClientConfig nettyClientConfig, ClientRemotingProcessor clientRemotingProcessor, RPCHook rpcHook)
        设置remotingClient属性等于new NettyRemotingClient(nettyClientConfig, null)实例
        设置clientRemotingProcessor属性等于clientRemotingProcessor参数
        执行remotingClient.registerRPCHook(rpcHook)
        // 注册请求处理器，用于接收brokerServer主动发过来的请求
        执行remotingClient.registerProcessor(40, clientRemotingProcessor, null)                   // NOTIFY_CONSUMER_IDS_CHANGED: 40
        执行remotingClient.registerProcessor(220, clientRemotingProcessor, null)                  // RESET_CONSUMER_CLIENT_OFFSET: 220
        执行remotingClient.registerProcessor(221, clientRemotingProcessor, null)                  // GET_CONSUMER_STATUS_FROM_CLIENT: 221
        执行remotingClient.registerProcessor(307, clientRemotingProcessor, null)                  // GET_CONSUMER_RUNNING_INFO: 307
        执行remotingClient.registerProcessor(309, clientRemotingProcessor, null)                  // CONSUME_MESSAGE_DIRECTLY: 309
    public void start()
        执行remotingClient.start()
        设置projectGroupPrefix属性等于getProjectGroupByIp(localAddress, 3000)                 // 以PROJECT_CONFIG作为namesapce，当前ip作为key，随机选中一台nameserver，发起同步GET_KV_CONFIG请求，超时时间为3秒钟
    public String getProjectGroupByIp(String ip, long timeoutMillis) throws RemotingException, MQClientException, InterruptedException
        执行getKVConfigValue("PROJECT_CONFIG", ip, timeoutMillis)并返回
    public String getKVConfigValue(String namespace, String key, long timeoutMillis) throws RemotingException, MQClientException, InterruptedException
        根据namespace和key，创建new GetKVConfigRequestHeader()实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(101, requestHeader)                   // GET_KV_CONFIG: 101
        执行remotingClient.invokeSync(null, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            解序列化response变量为GetKVConfigResponseHeader类型的实例，设置给responseHeader
            返回responseHeader.value
        抛异常MQClientException(response.code, response.remark)
    public void shutdown()
        执行remotingClient.shutdown()
    // 随机选中一台nameserver，发起同步GET_ROUTEINTO_BY_TOPIC请求，超时时间为timeoutMillis毫秒，TopicRouteData信息包括QueueData列表、BrokerData列表、filterServerTable信息，或者可能只包含orderTopicConf信息
    public TopicRouteData getTopicRouteInfoFromNameServer(String topic, long timeoutMillis) throws RemotingException, MQClientException, InterruptedException
        如果projectGroupPrefix属性不等于null
            设置topicWithProjectGroup变量等于VirtualEnvUtil.buildWithProjectGroup(topic, projectGroupPrefix)
        否则
            设置topicWithProjectGroup变量等于topic参数
        根据topicWithProjectGroup变量创建new GetRouteInfoRequestHeader()实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(105, requestHeader)                   // GET_ROUTEINTO_BY_TOPIC: 105
        执行remotingClient.invokeSync(null, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            如果response.body不等于null，解序列化为TopicRouteData类型并返回
        抛异常MQClientException(response.code, response.remark)
    // 随机选中一台nameserver，发起同步GET_ROUTEINTO_BY_TOPIC请求，超时时间为timeoutMillis毫秒，TopicRouteData信息包括QueueData列表、BrokerData列表、filterServerTable信息，或者可能只包含orderTopicConf信息
    public TopicRouteData getDefaultTopicRouteInfoFromNameServer(String topic, long timeoutMillis) throws RemotingException, MQClientException, InterruptedException
        根据topic变量创建new GetRouteInfoRequestHeader()实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(105, requestHeader)                   // GET_ROUTEINTO_BY_TOPIC: 105
        执行remotingClient.invokeSync(null, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            如果response.body不等于null，解序列化为TopicRouteData类型并返回
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送同步QUERY_CONSUMER_OFFSET请求，获取某个主题下某个消息队列对于某个groupName的offset，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public long queryConsumerOffset(String addr, QueryConsumerOffsetRequestHeader requestHeader, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            执行requestHeader.setConsumerGroup(VirtualEnvUtil.buildWithProjectGroup(requestHeader.consumerGroup, projectGroupPrefix))
            执行requestHeader.setTopic(VirtualEnvUtil.buildWithProjectGroup(requestHeader.topic, projectGroupPrefix))
        设置request变量等于RemotingCommand.createRequestCommand(14, requestHeader)                   // QUERY_CONSUMER_OFFSET: 14
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            解序列化response为QueryConsumerOffsetResponseHeader类型的实例，设置给responseHeader变量
            返回responseHeader.offset
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送同步GET_MAX_OFFSET请求，获取某个主题下某个消息队列当前最大的offset，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public long getMaxOffset(String addr, String topic, int queueId, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            设置topicWithProjectGroup变量等于VirtualEnvUtil.buildWithProjectGroup(topic, projectGroupPrefix)
        否则
            设置topicWithProjectGroup变量等于topic参数
        根据topicWithProjectGroup变量、queueId参数，创建new GetMaxOffsetRequestHeader()实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(30, requestHeader)                   // GET_MAX_OFFSET: 30
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            解序列化response为GetMaxOffsetResponseHeader类型的实例，设置给responseHeader变量
            返回responseHeader.offset
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送同步SEARCH_OFFSET_BY_TIMESTAMP请求，获取某个主题下某个消息队列在某个时间戳下的offset，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public long searchOffset(String addr, String topic, int queueId, long timestamp, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            设置topicWithProjectGroup变量等于VirtualEnvUtil.buildWithProjectGroup(topic, projectGroupPrefix)
        否则
            设置topicWithProjectGroup变量等于topic参数
        根据topicWithProjectGroup变量、queueId参数、timestamp参数、创建new SearchOffsetRequestHeader()实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(29, requestHeader)                   // SEARCH_OFFSET_BY_TIMESTAMP: 29
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            解序列化response为SearchOffsetResponseHeader类型的实例，设置给responseHeader变量
            返回responseHeader.offset
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr，以Oneway的方式发送UPDATE_CONSUMER_OFFSET请求，更新某个主题下某个消息队列对于某个groupName的offset，超时时间为timeoutMillis毫秒
    public void updateConsumerOffsetOneway(String addr, UpdateConsumerOffsetRequestHeader requestHeader, long timeoutMillis) throws RemotingConnectException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException, InterruptedException
        如果projectGroupPrefix属性不等于null
            执行requestHeader.setConsumerGroup(VirtualEnvUtil.buildWithProjectGroup(requestHeader.consumerGroup, projectGroupPrefix))
            执行requestHeader.setTopic(VirtualEnvUtil.buildWithProjectGroup(requestHeader.topic, projectGroupPrefix))
        设置request变量等于RemotingCommand.createRequestCommand(15, requestHeader)                   // UPDATE_CONSUMER_OFFSET: 15
        执行remotingClient.invokeOneway(addr, request, timeoutMillis)
    // 对指定filterServer发送同步REGISTER_MESSAGE_FILTER_CLASS请求，用于注册服务端协处理器，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public void registerMessageFilterClass(String addr, String consumerGroup, String topic, String className, int classCRC, byte[] classBody, long timeoutMillis) throws RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException, InterruptedException, MQBrokerException
        根据consumerGroup参数、className参数、topic参数、classCRC参数创建RegisterMessageFilterClassRequestHeader类型的实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(302, requestHeader)                   // REGISTER_MESSAGE_FILTER_CLASS: 302
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            退出方法
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送同步GET_CONSUMER_LIST_BY_GROUP请求，获取配置了相同consumerGroup的clientId列表，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public List<String> getConsumerIdListByGroup(String addr, String consumerGroup, long timeoutMillis) throws RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException, MQBrokerException, InterruptedException
        设置consumerGroupWithProjectGroup变量等于consumerGroup参数
        如果projectGroupPrefix属性不等于null
            设置consumerGroupWithProjectGroup变量等于VirtualEnvUtil.buildWithProjectGroup(consumerGroup, projectGroupPrefix)
        根据consumerGroupWithProjectGroup变量创建GetConsumerListByGroupRequestHeader类型的实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(38, requestHeader)                   // GET_CONSUMER_LIST_BY_GROUP: 38
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            如果response.body不等于null
                解序列化response.body为GetConsumerListByGroupResponseBody类型的实例，设置给body变量
                返回body.consumerIdList()
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送LOCK_BATCH_MQ请求，标识这些mq已被当前clientId锁定，超时时间为timeoutMillis毫秒
    public Set<MessageQueue> lockBatchMQ(String addr, LockBatchRequestBody requestBody, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
           执行requestBody.setConsumerGroup(VirtualEnvUtil.buildWithProjectGroup(requestBody.consumerGroup, projectGroupPrefix))
           遍历requestBody.mqSet，设置messageQueue等于当前元素
               执行messageQueue.setTopic(VirtualEnvUtil.buildWithProjectGroup(messageQueue.topic, projectGroupPrefix))
        设置request变量等于RemotingCommand.createRequestCommand(41, null)                   // LOCK_BATCH_MQ: 41
        执行request.setBody(requestBody.encode())
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            解序列化response.body为LockBatchResponseBody类型的实例，设置给responseBody变量
            设置messageQueues变量等于responseBody.lockOKMQSet
            如果projectGroupPrefix属性不等于null
               遍历messageQueues变量，设置messageQueue变量等于当前元素
                   执行messageQueue.setTopic(VirtualEnvUtil.buildWithProjectGroup(messageQueue.topic, projectGroupPrefix))
            返回messageQueues变量
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送UNLOCK_BATCH_MQ请求，标识这些mq已被当前clientId解锁定，超时时间为timeoutMillis毫秒
    public void unlockBatchMQ(String addr, UnlockBatchRequestBody requestBody, long timeoutMillis, boolean oneway) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            执行requestBody.setConsumerGroup(VirtualEnvUtil.buildWithProjectGroup(requestBody.consumerGroup, projectGroupPrefix))
            遍历requestBody.mqSet，设置messageQueue等于当前元素
                执行messageQueue.setTopic(VirtualEnvUtil.buildWithProjectGroup(messageQueue.topic, projectGroupPrefix))
        设置request变量等于RemotingCommand.createRequestCommand(42, requestHeader)                   // UNLOCK_BATCH_MQ: 42
        执行request.setBody(requestBody.encode())
        如果oneway参数等于true
            执行remotingClient.invokeOneway(addr, request, timeoutMillis)
        否则
            执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
            如果response.code等于0                                                                       // SUCCESS: 0
                退出方法
            抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr/filterAddr发送PULL_MESSAGE请求，超时时间为timeoutMillis毫秒
    public PullResult pullMessage(String addr, PullMessageRequestHeader requestHeader, long timeoutMillis, CommunicationMode communicationMode, PullCallback pullCallback) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            执行requestHeader.setConsumerGroup(VirtualEnvUtil.buildWithProjectGroup(requestHeader.consumerGroup, projectGroupPrefix))
            执行requestHeader.setTopic(VirtualEnvUtil.buildWithProjectGroup(requestHeader.topic, projectGroupPrefix))
        设置request变量等于RemotingCommand.createRequestCommand(11, requestHeader)                   // PULL_MESSAGE: 11
        如果communicationMode等于ASYNC
            执行pullMessageAsync(addr, request, timeoutMillis, pullCallback)
            返回null
        否则如果communicationMode等于SYNC
            执行pullMessageSync(addr, request, timeoutMillis)并返回
        否则
            抛异常
    private void pullMessageAsync(String addr, RemotingCommand request, long timeoutMillis, PullCallback pullCallback) throws RemotingException, InterruptedException
        创建new InvokeCallback()实例，设置给callback变量
            public void operationComplete(ResponseFuture responseFuture)
                设置response变量等于responseFuture.responseCommand
                如果response变量不等于null
                    try {
                        执行pullCallback.onSuccess(processPullResponse(response));
                    } catch (Exception e) {
                        执行pullCallback.onException(e);
                    }
                否则
                    如果responseFuture.sendRequestOK等于false
                        执行pullCallback.onException(new MQClientException("send request failed", responseFuture.cause))
                    否则如果responseFuture.isTimeout()等于true
                        执行pullCallback.onException(new MQClientException("wait response timeout "+ responseFuture.timeoutMillis + "ms", responseFuture.cause))
                    否则
                        执行pullCallback.onException(new MQClientException("unknow reseaon", responseFuture.cause))
        执行remotingClient.invokeAsync(addr, request, timeoutMillis, callback)
    private PullResult pullMessageSync(String addr, RemotingCommand request, long timeoutMillis) throws RemotingException, InterruptedException, MQBrokerException
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        执行processPullResponse(response)并返回
    private PullResult processPullResponse(RemotingCommand response) throws MQBrokerException, RemotingCommandException
        如果response.code等于0                  // SUCCESS: 0
            设置pullStatus变量等于FOUND
        否则如果response.code等于19              // PULL_NOT_FOUND: 19
            设置pullStatus变量等于NO_NEW_MSG
        否则如果response.code等于20              // PULL_RETRY_IMMEDIATELY: 20
            设置pullStatus变量等于NO_MATCHED_MSG
        否则如果response.code等于21              // PULL_OFFSET_MOVED: 21
            设置pullStatus变量等于OFFSET_ILLEGAL
        否则
            抛异常MQBrokerException(response.code, response.remark)
        解序列化response为PullMessageResponseHeader类型的实例，设置给responseHeader变量
        返回new PullResultExt(pullStatus, responseHeader.nextBeginOffset, responseHeader.minOffset, responseHeader.maxOffset, null, responseHeader.suggestWhichBrokerId, response.body)实例
    // 对指定brokerAddr发送同步CONSUMER_SEND_MSG_BACK请求，告知需要重新消费，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public void consumerSendMessageBack(String addr, MessageExt msg, String consumerGroup, int delayLevel, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        设置consumerGroupWithProjectGroup变量等于consumerGroup
        如果projectGroupPrefix属性不等于null
            设置consumerGroupWithProjectGroup变量等于VirtualEnvUtil.buildWithProjectGroup(consumerGroup, projectGroupPrefix)
            执行msg.setTopic(VirtualEnvUtil.buildWithProjectGroup(msg.topic, projectGroupPrefix))
        根据consumerGroupWithProjectGroup变量、msg.topic、msg.commitLogOffset、delayLevel参数、msg.msgId创建new ConsumerSendMsgBackRequestHeader()实例，设置给requestHeader参数
        设置request变量等于RemotingCommand.createRequestCommand(36, requestHeader)                    // CONSUMER_SEND_MSG_BACK: 36
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            退出方法
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送同步UNREGISTER_CLIENT请求，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public void unregisterClient(String addr, String clientID, String producerGroup, String consumerGroup, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            设置consumerGroupWithProjectGroup变量等于VirtualEnvUtil.buildWithProjectGroup(consumerGroup, projectGroupPrefix)
        否则
            设置consumerGroupWithProjectGroup变量等于consumerGroup参数
        根据clientID参数、consumerGroupWithProjectGroup变量创建new UnregisterClientRequestHeader()实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(35, requestHeader)                    // UNREGISTER_CLIENT: 35
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            退出方法
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送同步HEART_BEAT请求，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public void sendHearbeat(String addr, HeartbeatData heartbeatData, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            遍历heartbeatData.consumerDataSet，设置consumerData变量等于当前元素
                执行consumerData.setGroupName(VirtualEnvUtil.buildWithProjectGroup(consumerData.groupName, projectGroupPrefix))
        设置request变量等于RemotingCommand.createRequestCommand(34, null)                             // HEART_BEAT: 34
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            退出方法
        抛异常MQClientException(response.code, response.remark)

# com.alibaba.rocketmq.remoting.netty.NettyRemotingClient

    protected NettyEventExecuter nettyEventExecuter = new NettyEventExecuter();

    public NettyRemotingClient(NettyClientConfig nettyClientConfig, ChannelEventListener channelEventListener)
        使用nettyClientConfig.clientOnewaySemaphoreValue初始化semaphoreOneway属性                   // 控制onway并发
        使用nettyClientConfig.clientAsyncSemaphoreValue初始化semaphoreAsync属性                     // 控制async并发
        设置nettyClientConfig属性等于nettyClientConfig参数
        设置channelEventListener属性等于channelEventListener参数
        使用nettyClientConfig.clientCallbackExecutorThreads创建Fix线程池，设置给publicExecutor属性    // 用于callback回复场景
        初始化eventLoopGroupWorker属性，线程数为1                                                    // netty的worker
    public void registerRPCHook(RPCHook rpcHook)
        设置rpcHook属性等于rpcHook参数
    public void registerProcessor(int requestCode, NettyRequestProcessor processor, ExecutorService executor)
        如果executor参数等于null
            设置executor参数等于publicExecutor属性
        执行processorTable.put(requestCode, new Pair<NettyRequestProcessor, ExecutorService>(processor, executor))              // 注册处理器和执行该请求的业务线程池
    public void start()
        设置defaultEventExecutorGroup属性等于new DefaultEventExecutorGroup(nettyClientConfig.clientWorkerThreads)实例
        创建Client
            使用eventLoopGroupWorker作为worker
            设置SO_KEEPALIVE等于false                                                             // 非长连接
            设置SO_SNDBUF等于nettyClientConfig.clientSocketSndBufSize
            设置SO_RCVBUF等于nettyClientConfig.clientSocketRcvBufSize
            添加Handler
                defaultEventExecutorGroup   new NettyEncoder()                                                                  // 使用defaultEventExecutorGroup执行编码器
                defaultEventExecutorGroup   new NettyDecoder()                                                                  // 使用defaultEventExecutorGroup执行解码器
                defaultEventExecutorGroup   new new IdleStateHandler(0, 0, nettyClientConfig.clientChannelMaxIdleTimeSeconds)   // 使用defaultEventExecutorGroup执行，指定时间内未发生读操作或写操作，触发ALL_IDLE的IdleStateEvent
                defaultEventExecutorGroup   new NettyConnetManageHandler()                                                      // 使用defaultEventExecutorGroup执行
                defaultEventExecutorGroup   new NettyClientHandler()                                                            // 使用defaultEventExecutorGroup执行
        执行定时任务，延迟3秒钟执行，执行间隔为1秒钟
            执行scanResponseTable()
        如果channelEventListener不等于null
            执行nettyEventExecuter.start()
    public void scanResponseTable()
        遍历responseTable属性，设置当前元素给entry变量
            如果当前系统时间戳和entry.value.beginTimestamp的差值 - 1000大于等于entry.value.timeoutMillis                             // 触发即将超时的Future
                移除当前元素
                try {
                    执行entry.value.executeInvokeCallback()
                } catch (Exception e) {
                } finally {
                    执行entry.value.release()
                }
    public void shutdown()
        取消正在执行的一个定时任务
        遍历channelTables属性，设置cw变量等于当前元素
            执行closeChannel(null, cw.channel)
        执行channelTables.clear()
        关闭eventLoopGroupWorker属性
        执行nettyEventExecuter.shutdown()
        关闭defaultEventExecutorGroup属性
        关闭publicExecutor属性
    public void closeChannel(Channel channel)
        如果channel参数等于null，退出方法
        try {
            if (lockChannelTables.tryLock(3000, TimeUnit.MILLISECONDS))
                try {
                    遍历channelTables属性，设置entry变量等于当前元素
                        如果entry.value不等于null并且entry.value.channel等于channel参数
                            移除当前元素
                            执行channel.close()
                            退出循环
                } catch (Exception e) {
                } finally {
                    lockChannelTables.unlock();
                }
        } catch (InterruptedException e) { }
    public void closeChannel(String addr, Channel channel)
        如果channel参数等于null，退出方法
        try {
            if (lockChannelTables.tryLock(3000, TimeUnit.MILLISECONDS))
                try {
                    设置prevCW变量等于channelTables.get(addrRemote)
                    如果prevCW不等于null并且等于channel
                        执行channelTables.remove(addrRemote)
                    执行channel.close()
                } catch (Exception e) {
                } finally {
                    lockChannelTables.unlock();
                }
        } catch (InterruptedException e) { }
    public void updateNameServerAddressList(List<String> addrs)                                                         // 设置nameserver，比较类似equals效果，因为最终需要乱排，所以用size + contain比较
        如果addrs参数元素个数大于0
            设置old变量等于namesrvAddrList.get()
            如果old变量等于null或者old.size()不等于addrs.size()或者old.containAll(addrs)等于false
                执行Collections.shuffle(addrs)
                执行namesrvAddrList.set(addrs)
    public List<String> getNameServerAddressList()
        返回namesrvAddrList.get()
    private Channel getAndCreateChannel(String addr) throws InterruptedException                                        // addr为null返回当前正在用的，否则返回指定addr的连接
        如果addr参数等于null
            返回getAndCreateNameserverChannel()
        设置cw变量等于channelTables.get(addr)
        如果cw变量不等于null并且cw.isOK()等于true
            返回cw.channel
        返回createChannel(addr)
    private Channel getAndCreateNameserverChannel() throws InterruptedException                                         // 获取当前正在用的，如果不存在或已过期，按递增方式，创建一个新的
        设置addr变量等于namesrvAddrChoosed.get()
        如果addr变量不等于null
            设置cw变量等于channelTables.get(addr)
            如果cw不等于null并且cw.isOK()等于true
                返回cw.channel
        设置addrList变量等于namesrvAddrList.get()
        if (lockNamesrvChannel.tryLock(3000, TimeUnit.MILLISECONDS)) {
            try {
                设置addr变量等于namesrvAddrChoosed.get()
                如果addr变量不等于null
                    设置cw变量等于channelTables.get(addr)
                    如果cw变量不等于null并且cw.isOK()等于true
                        返回cw.channel
                如果addrList变量不等于null并且元素个数大于0
                    循环addrList.size()次
                        设置index变量等于Math.abs(namesrvIndex.incrementAndGet()) % addrList.size()
                        设置newAddr变量等于addrList.get(index)
                        执行namesrvAddrChoosed.set(newAddr)
                        设置channelNew变量等于createChannel(newAddr)
                        如果channelNew变量不等于null
                            返回channelNew变量
            } catch (Exception e) {
            } finally {
                lockNamesrvChannel.unlock();
            }
        }
        返回null
    private Channel createChannel(String addr) throws InterruptedException                                              // 创建指定addr的连接，如果之前创建完毕，返回创建好的
        设置cw变量等于channelTables.get(addr)
        如果cw变量不等于null并且cw.isOK()等于true
            返回cw.channel
        if (lockChannelTables.tryLock(3000, TimeUnit.MILLISECONDS)) {
            try {
                设置createNewConnection变量等于false
                cw变量等于channelTables.get(addr)
                如果cw变量不等于null
                    如果cw.isOK()等于true
                        返回cw.channel
                    否则如果cw.channelFuture.isDone()等于false
                        设置createNewConnection变量等于false
                    否则
                        执行channelTables.remove(addr)
                        设置createNewConnection变量等于true
                否则
                    设置createNewConnection变量等于true
                如果createNewConnection等于true
                    执行bootstrap.connect(RemotingHelper.string2SocketAddress(addr))返回ChannelFuture类型的实例，设置给channelFuture变量
                    执行channelTables.put(addr, cw = new ChannelWrapper(channelFuture))
            } catch (Exception e) {
            } finally {
                lockChannelTables.unlock()
            }
        }
        如果cw变量不等于null
            设置channelFuture变量等于cw.channelFuture
            如果channelFuture.awaitUninterruptibly(nettyClientConfig.connectTimeoutMillis)等于true
                如果cw.isOK()等于true
                    返回cw.channel
        返回null
    // 同步执行
    public RemotingCommand invokeSync(String addr, RemotingCommand request, long timeoutMillis) throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException
        设置channel变量等于getAndCreateChannel(addr)
        如果channel变量不等于null并且channel.isActive()等于true
            try {
                如果rpcHook属性不等于null，执行rpcHook.doBeforeRequest(addr, request)
                执行invokeSyncImpl(channel, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response变量
                如果rpcHook属性不等于null，执行rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(channel), request, response)
                返回response变量
            } catch (RemotingSendRequestException e) {
                执行closeChannel(addr, channel)
                抛异常e
            } catch (RemotingTimeoutException e) {
                抛异常e
            }
        否则
            执行closeChannel(addr, channel)
            抛异常RemotingConnectException(addr)
    // Oneway执行
    public void invokeOneway(String addr, RemotingCommand request, long timeoutMillis) throws InterruptedException, RemotingConnectException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException
        设置channel变量等于getAndCreateChannel(addr)
        如果channel变量不等于null并且channel.isActive()等于true
            try {
                如果rpcHook属性不等于null，执行rpcHook.doBeforeRequest(addr, request)
                执行invokeOnewayImpl(channel, request, timeoutMillis)
            } catch (RemotingSendRequestException e) {
                执行closeChannel(addr, channel)
                抛异常e
            }
        否则
            执行closeChannel(addr, channel)
            抛异常RemotingConnectException(addr)
    // 异步执行
    public void invokeAsync(String addr, RemotingCommand request, long timeoutMillis, InvokeCallback invokeCallback) throws InterruptedException, RemotingConnectException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException
        设置channel变量等于getAndCreateChannel(addr)
        如果channel变量不等于null并且channel.isActive()等于true
            try {
                如果rpcHook属性不等于null，执行rpcHook.doBeforeRequest(addr, request)
                执行invokeAsyncImpl(channel, request, timeoutMillis, invokeCallback)
            } catch (RemotingSendRequestException e) {
                执行closeChannel(addr, channel)
                抛异常e
            }
        否则
            执行closeChannel(addr, channel)
            抛异常RemotingConnectException(addr)
    // 同步发送请求，超时时间为timeoutMills毫秒
    // 如果发送请求失败（responseCommand等于null，responseFuture.sendRequestOK等于false），抛RemotingSendRequestException
    // 如果发送请求成功，响应超时（responseCommand等于null，responseFuture.sendRequestOK等于true），抛RemotingTimeoutException
    public RemotingCommand invokeSyncImpl(Channel channel, RemotingCommand request, long timeoutMillis) throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException
        try {
            设置responseFuture变量等于new ResponseFuture(request.opaque, timeoutMillis, null, null)
            执行responseTable.put(request.opaque, responseFuture)
            创建new ChannelFutureListener()实例，设置给listener变量
                public void operationComplete(ChannelFuture f) throws Exception
                    如果f.isSuccess()等于true
                        执行responseFuture.setSendRequestOK(true)
                        退出方法
                    执行responseFuture.setSendRequestOK(false)
                    执行responseTable.remove(request.opaque)
                    执行responseFuture.setCause(f.cause())
                    执行responseFuture.putResponse(null)
            执行channel.writeAndFlush(request).addListener(listener)
            设置responseCommand等于responseFuture.waitResponse(timeoutMillis)
            如果responseCommand等于null
                如果responseFuture.isSendRequestOK()等于true
                    抛异常RemotingTimeoutException(RemotingHelper.parseChannelRemoteAddr(channel), timeoutMillis, responseFuture.cause)
                否则
                    抛异常RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), responseFuture.cause)
            返回responseCommand
        } finally {
            执行responseTable.remove(request.opaque)
        }
    // 异步发送请求，超时时间为timeoutMills毫秒
    // 如果发送请求失败（申请令牌失败，timeoutMills小于等于0），抛RemotingTooMuchRequestException
    // 如果发送请求失败（申请令牌失败，timeoutMills大于0），抛RemotingTimeoutException
    // 如果发送请求失败（写入channel出现异常），释放令牌，抛RemotingSendRequestException
    // 如果发送请求失败（responseFuture.sendRequestOK等于false），执行callback，释放令牌
    // 如果发送请求成功，利用定时任务检测响应超时（responseFuture.sendRequestOK等于true），超时后执行callback，释放令牌
    public void invokeAsyncImpl(Channel channel, RemotingCommand request, long timeoutMillis, InvokeCallback invokeCallback) throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException
        如果semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS)等于true
            设置responseFuture变量等于new ResponseFuture(request.opaque, timeoutMillis, invokeCallback, new SemaphoreReleaseOnlyOnce(semaphoreAsync))
            执行responseTable.put(request.opaque, responseFuture)
            try {
                创建new ChannelFutureListener()实例，设置给listener变量
                    public void operationComplete(ChannelFuture f) throws Exception
                        如果f.isSuccess()等于true
                            执行responseFuture.setSendRequestOK(true)
                            退出方法
                        执行responseFuture.setSendRequestOK(false)
                        执行responseFuture.putResponse(null)
                        执行responseTable.remove(request.opaque)
                        try {
                            执行responseFuture.executeInvokeCallback();
                        } catch (Throwable e) {
                        } finally {
                            执行responseFuture.release();
                        }
                执行channel.writeAndFlush(request).addListener(listener)
            } catch (Exception e) {
                执行responseFuture.release()
                抛异常RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e)
            }
        否则
            如果timeoutMillis参数小于等于0
                抛异常RemotingTooMuchRequestException("invokeAsyncImpl invoke too fast")
            否则
                抛异常RemotingTimeoutException("invokeAsyncImpl tryAcquire semaphore timeout...")
    // oneway方式发送请求，超时时间为timeoutMills毫秒
    // 如果发送请求失败（申请令牌失败，timeoutMills小于等于0），抛RemotingTooMuchRequestException
    // 如果发送请求失败（申请令牌失败，timeoutMills大于0），抛RemotingTimeoutException
    // 如果发送请求失败（写入channel出现异常），释放令牌，抛RemotingSendRequestException
    // 如果发送请求成功，释放令牌
    public void invokeOnewayImpl(Channel channel, RemotingCommand request, long timeoutMillis) throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException
       执行request.markOnewayRPC()
       如果semaphoreOneway.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS)等于true
           设置once变量等于new SemaphoreReleaseOnlyOnce(semaphoreAsync)
           try {
               创建new ChannelFutureListener()实例，设置给listener变量
                   public void operationComplete(ChannelFuture f) throws Exception
                       执行once.release()
               执行channel.writeAndFlush(request).addListener(listener)
           } catch (Exception e) {
               执行once.release()
               抛异常RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e)
           }
       否则
           如果timeoutMillis参数小于等于0
               抛异常RemotingTooMuchRequestException("invokeAsyncImpl invoke too fast")
           否则
               抛异常RemotingTimeoutException("invokeAsyncImpl tryAcquire semaphore timeout...")
    public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception
        如果msg参数不等于null
            如果msg.type等于REQUEST_COMMAND
                执行processRequestCommand(ctx, cmd)                                                                             // 处理请求
            如果msg.type等于RESPONSE_COMMAND
                执行processResponseCommand(ctx, cmd)                                                                            // 处理响应
    // 接收传输过来的请求，用对应的处理器来执行
    public void processRequestCommand(ChannelHandlerContext ctx, RemotingCommand cmd)
        设置matched变量等于processorTable.get(cmd.code)
        如果matched变量等于null
            设置matched变量等于defaultRequestProcessor属性                                                                       // 匹配不到用默认处理器
        如果matched变量等于null
            创建RemotingCommand.createResponseCommand(REQUEST_CODE_NOT_SUPPORTED, "request type code not supported")实例，设置给response变量
            执行response.setOpaque(cmd.opaque)
            执行ctx.writeAndFlush(response)
        否则
            创建new Runnable()实例，设置给run变量
                public void run()
                    如果rpcHook属性不等于null
                        执行rpcHook.doBeforeRequest(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd)
                    执行pair.object1.processRequest(ctx, cmd)返回RemotingCommand类型的实例，设置给response变量
                    如果rpcHook属性不等于null
                        执行rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response)
                    如果cmd.isOnewayRPC()等于false                                                                             // twoway形式，要响应
                        如果response不等于null
                            执行response.setOpaque(cmd.opaque)
                            执行response.markResponseType()
                            try {
                                执行ctx.writeAndFlush(response)
                            } catch (Throwable e) {}
            try {
                执行matched.object2.submit(run)                                                                               // 利用对应处理器的业务线程池处理请求
            } catch (RejectedExecutionException e) {
                如果cmd.isOnewayRPC()等于false
                    创建RemotingCommand.createResponseCommand(SYSTEM_BUSY, "too many requests and system thread pool busy, please try another server")实例，设置给response变量
                    执行response.setOpaque(cmd.opaque)
                    执行ctx.writeAndFlush(response)
            }
    // 处理接收到的响应，填充ResponseFuture
    public void processResponseCommand(ChannelHandlerContext ctx, RemotingCommand cmd)
        设置responseFuture变量等于responseTable.get(cmd.opaque)
        如果responseFuture变量不等于null
            执行responseFuture.setResponseCommand(cmd)
            执行responseFuture.release()
            执行responseTable.remove(cmd.opaque)
            如果responseFuture.invokeCallback不等于null
                设置runInThisThread变量等于false
                如果publicExecutor属性不等于null
                    try {
                        执行executor.submit(new Runnable() {
                            @Override
                            public void run() {
                                try {
                                    执行responseFuture.executeInvokeCallback();
                                } catch (Throwable e) { }
                            }
                        })
                    } catch (Exception e) {
                        设置runInThisThread变量等于true
                    }
                否则
                    设置runInThisThread变量等于true
            否则
                执行responseFuture.putResponse(cmd)                                                                           // 释放countDownLatch

## com.alibaba.rocketmq.remoting.netty.NettyRemotingClient.NettyConnetManageHandler

    public void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) throws Exception
        执行super.connect(ctx, remoteAddress, localAddress, promise)
        如果channelEventListener属性不等于null，执行nettyEventExecuter.putNettyEvent(new NettyEvent(NettyEventType.CONNECT, remoteAddress .toString(), ctx.channel()))
    public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception
        执行closeChannel(ctx.channel())
        执行super.disconnect(ctx, promise)
        如果channelEventListener属性不等于null，执行nettyEventExecuter.putNettyEvent(new NettyEvent(NettyEventType.CLOSE, remoteAddress .toString(), ctx.channel()))
    public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception
        执行closeChannel(ctx.channel())
        执行super.close(ctx, promise)
        如果channelEventListener属性不等于null，执行nettyEventExecuter.putNettyEvent(new NettyEvent(NettyEventType.CLOSE, remoteAddress .toString(), ctx.channel()))
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception                                  // 错误关闭连接
        执行closeChannel(ctx.channel())
        如果channelEventListener属性不等于null，执行nettyEventExecuter.putNettyEvent(new NettyEvent(NettyEventType.EXCEPTION, remoteAddress .toString(), ctx.channel()))
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception
        如果evt参数是IdleStateEvent类型
            如果evt.state()等于ALL_IDLE                                                                                         // 超时关闭连接
                执行closeChannel(ctx.channel())
                如果channelEventListener属性不等于null，执行nettyEventExecuter.putNettyEvent(new NettyEvent(NettyEventType.IDLE, remoteAddress .toString(), ctx.channel()))
        执行ctx.fireUserEventTriggered(evt)

## com.alibaba.rocketmq.remoting.netty.NettyRemotingAbstract.NettyEventExecuter

    public void putNettyEvent(NettyEvent event)
        如果eventQueue属性大小超过10000，丢弃事件，否则添加事件到eventQueue属性
    public void run()
        while (!this.isStoped()) {                                                                                            // 以下几类事件都是在单独的线程中执行的
            try {
                获取eventQueue中的事件
                    如果事件不等于null并且channelEventListener属性不等于null
                        如果事件类型等于IDLE
                            执行channelEventListener.onChannelIdle(event.remoteAddr, event.channel)
                        如果事件类型等于CLOSE
                            执行channelEventListener.onChannelClose(event.remoteAddr, event.channel)
                        如果事件类型等于CONNECT
                            执行channelEventListener.onChannelConnect(event.remoteAddr, event.channel)
                        如果事件类型等于EXCEPTION
                            执行channelEventListener.onChannelException(event.remoteAddr, event.channel)
            } catch (Exception e) {
            }
        }

## com.alibaba.rocketmq.remoting.netty.NettyRemotingClient.NettyClientHandler

    protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception
        执行processMessageReceived(ctx, msg)
        