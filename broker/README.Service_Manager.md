# com.alibaba.rocketmq.broker.topic.TopicConfigManager

    private transient Lock lockTopicConfigTable = new ReentrantLock()
    private Set<String> systemTopicList = new HashSet<String>()
    // 主题配置信息
    private ConcurrentHashMap<String, TopicConfig> topicConfigTable = new ConcurrentHashMap<String, TopicConfig>(1024)
    private DataVersion dataVersion = new DataVersion()

    public TopicConfigManager(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
        {
            // 添加SELF_TEST_TOPIC主题
            设置topicConfig变量等于new TopicConfig("SELF_TEST_TOPIC")
            执行systemTopicList.add(topic)
            执行topicConfig.setReadQueueNums(1)
            执行topicConfig.setWriteQueueNums(1)
            执行topicConfigTable.put(topicConfig.topicName, topicConfig)
        }
        {
            // 添加TBW102公共配置主题，同时也是系统主题
            如果brokerController.brokerConfig.autoCreateTopicEnable等于true
            设置topicConfig变量等于new TopicConfig("TBW102")
            执行systemTopicList.add(topic)
            执行topicConfig.setReadQueueNums(brokerController.brokerConfig.defaultTopicQueueNums)
            执行topicConfig.setWriteQueueNums(brokerController.brokerConfig.defaultTopicQueueNums)
            执行topicConfig.setPerm(PermName.PERM_INHERIT | PermName.PERM_READ | PermName.PERM_WRITE)
            执行topicConfigTable.put(topicConfig.topicName, topicConfig)
        }
        {
            // 添加测试主题
            设置topicConfig变量等于new TopicConfig("BenchmarkTest")
            执行systemTopicList.add(topic)
            执行topicConfig.setReadQueueNums(1024)
            执行topicConfig.setWriteQueueNums(1024)
            执行topicConfigTable.put(topicConfig.topicName, topicConfig)
        }
        {
            // 添加brokerClusterName系统主题              // 集群名字
            设置topicConfig变量等于new TopicConfig(brokerController.brokerConfig.brokerClusterName)
            执行systemTopicList.add(topic)
            设置perm变量等于PermName.PERM_INHERIT
            如果brokerController.brokerConfig.clusterTopicEnable等于true
                perm等于perm | PermName.PERM_READ | PermName.PERM_WRITE
            执行topicConfig.setPerm(perm)
            执行topicConfigTable.put(topicConfig.topicName, topicConfig)
        }
        {
            // 添加brokerName系统主题                     // 服务器标识
            设置topicConfig变量等于new TopicConfig(brokerController.brokerConfig.brokerName)
            执行systemTopicList.add(topic)
            设置perm变量等于PermName.PERM_INHERIT
            如果brokerController.brokerConfig.brokerTopicEnable等于true
                perm等于perm | PermName.PERM_READ | PermName.PERM_WRITE
            执行topicConfig.setPerm(perm)
            执行topicConfig.setReadQueueNums(1)
            执行topicConfig.setWriteQueueNums(1)
            执行topicConfigTable.put(topicConfig.topicName, topicConfig)
        }
        {
            // 添加OFFSET_MOVED_EVENT系统主题，拉消息请求可能触发加消息事件
            设置topicConfig变量等于new TopicConfig("OFFSET_MOVED_EVENT")
            执行systemTopicList.add(topic)
            执行topicConfig.setReadQueueNums(1)
            执行topicConfig.setWriteQueueNums(1)
            执行topicConfigTable.put(topicConfig.topicName, topicConfig)
        }
    // 加载已有配置
    public boolean load()
        try {
            读取brokerController.messageStoreConfig.storePathRootDir/config/topics.json文件内容作为String类型，设置给jsonString变量
            如果jsonString变量等于null
                执行loadBak()并返回
            否则
                解序列化jsonString变量作为TopicConfigSerializeWrapper类型，设置给topicConfigSerializeWrapper变量
                如果topicConfigSerializeWrapper变量不等于null
                    执行topicConfigTable.putAll(topicConfigSerializeWrapper.topicConfigTable)
                    执行dataVersion.assignNewOne(topicConfigSerializeWrapper.dataVersion)
                返回true
        } catch (Exception e) {
            执行loadBak()并返回
        }
    private boolean loadBak()
       try {
           读取brokerController.messageStoreConfig.storePathRootDir/config/topics.json.bak文件内容作为String类型，设置给jsonString变量
           如果jsonString变量不等于null
                解序列化jsonString变量作为TopicConfigSerializeWrapper类型，设置给topicConfigSerializeWrapper变量
                如果topicConfigSerializeWrapper变量不等于null
                    执行topicConfigTable.putAll(topicConfigSerializeWrapper.topicConfigTable)
                    执行dataVersion.assignNewOne(topicConfigSerializeWrapper.dataVersion)
                返回true
       } catch (Exception e) {
           返回false
       }
       返回true
    // 创建序列化Bean
    public TopicConfigSerializeWrapper buildTopicConfigSerializeWrapper()
        设置topicConfigSerializeWrapper变量等于new TopicConfigSerializeWrapper()实例
        执行topicConfigSerializeWrapper.setTopicConfigTable(topicConfigTable)
        执行topicConfigSerializeWrapper.setDataVersion(dataVersion)
        返回topicConfigSerializeWrapper变量
    // 根据nameserver返回的顺序主题信息，重新整理主题配置
    // 是否是顺序主题，貌似没啥意义，只是在put消息的时候，如果当前BrokerServer没有写权限同时是顺序主题，则无法写入
    public void updateOrderTopicConfig(KVTable orderKVTableFromNs)
        如果orderKVTableFromNs参数不等于null并且orderKVTableFromNs.table不等于null
            设置isChange变量等于false
            遍历orderKVTableFromNs.table.keySet()，设置topic变量等于当前元素
                设置topicConfig变量等于topicConfigTable.get(topic)
                如果topicConfig变量不等于null并且topicConfig.order等于false            // 如果原来是非顺序主题，变更为顺序主题
                    执行topicConfig.setOrder(true)
                    设置isChange变量等于true
            遍历topicConfigTable.keySet()，设置topic变量等于当前元素
                如果orderKVTableFromNs.table.keySet().contains(topic)等于false
                    设置topicConfig变量等于topicConfigTable.get(topic)
                    如果topicConfig.order等于true                                   // 如果原来是顺序主题，变更为非顺序主题
                        执行topicConfig.setOrder(false)
                        设置isChange变量等于true
            如果isChange变量等于true
                执行dataVersion.nextVersion()                                      // 更新数据版本
                执行persist()                                                      // 持久化所有主题配置
    // 持久化配置信息
    public synchronized void persist()
        设置topicConfigSerializeWrapper变量等于buildTopicConfigSerializeWrapper()
        序列化topicConfigSerializeWrapper变量为JSON格式的字符串，设置给jsonString变量
        如果jsonString变量不等于null
            try {
                写入jsonString变量值到brokerController.messageStoreConfig.storePathRootDir/config/topics.json文件中
            } catch (Exception e) {
            }
    // 获取指定主题配置
    public TopicConfig selectTopicConfig(String topic)
        执行topicConfigTable.get(topic)并返回
    // 返回指定主题是否是顺序主题
    public boolean isOrderTopic(String topic)
        设置topicConfig变量等于topicConfigTable.get(topic)
        如果topicConfig变量等于null
            返回false
        返回topicConfig.order
    // 创建新的主题，如果主题不存在创建成功后然后更新主题配置到nameserver
    public TopicConfig createTopicInSendMessageBackMethod(String topic, int clientDefaultTopicQueueNums, int perm, int topicSysFlag)
        设置topicConfig变量等于topicConfigTable.get(topic)
        如果topicConfig变量不等于null
            返回topicConfig变量
        设置createNew变量等于false
        try {
            if (this.lockTopicConfigTable.tryLock(3000, TimeUnit.MILLISECONDS)) {
                try {
                    设置topicConfig变量等于topicConfigTable.get(topic)
                    如果topicConfig变量不等于null
                        返回topicConfig变量
                    设置topicConfig变量等于new TopicConfig(topic)实例
                    执行topicConfig.setReadQueueNums(clientDefaultTopicQueueNums)
                    执行topicConfig.setWriteQueueNums(clientDefaultTopicQueueNums)
                    执行topicConfig.setPerm(perm)
                    执行topicConfig.setTopicSysFlag(topicSysFlag)
                    执行topicConfigTable.put(topic, topicConfig);
                    设置createNew变量等于true
                    执行dataVersion.nextVersion()
                    执行persist();
                } finally {
                    lockTopicConfigTable.unlock();
                }
            }
        } catch (InterruptedException e) {
        }
        如果createNew变量等于true
            执行brokerController.registerBrokerAll(false, true)
        返回topicConfig变量
    // 更新主题配置
    public void updateTopicConfig(TopicConfig topicConfig)
        执行topicConfigTable.put(topicConfig.topicName, topicConfig)
        执行dataVersion.nextVersion()
        执行persist()
    // 检查主题是否可以发送消息
    public boolean isTopicCanSendMessage(String topic)
        如果主题名称等于TBW102或者等于brokerController.brokerConfig.brokerClusterName，返回false
        否则
            返回true

# com.alibaba.rocketmq.broker.offset.ConsumerOffsetManager
    // Key: Topic@Group, Value: Key: QueueId, Value: Offset
    private ConcurrentHashMap<String, ConcurrentHashMap<Integer, Long>> offsetTable = new ConcurrentHashMap<String, ConcurrentHashMap<Integer, Long>>(512)

    public ConsumerOffsetManager(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    // 加载之前的offset配置
    public boolean load()
        try {
            读取brokerController.messageStoreConfig.storePathRootDir/config/consumerOffset.json文件内容作为String类型，设置给jsonString变量
            如果jsonString变量等于null
                执行loadBak()并返回
            否则
                解序列化jsonString变量作为ConsumerOffsetManager类型，设置给obj变量
                如果obj变量不等于null
                    设置offsetTable属性等于obj.offsetTable
                返回true
        } catch (Exception e) {
            执行loadBak()并返回
        }
    private boolean loadBak()
       try {
           读取brokerController.messageStoreConfig.storePathRootDir/config/consumerOffset.json.bak文件内容作为String类型，设置给jsonString变量
           如果jsonString变量不等于null
                解序列化jsonString变量作为ConsumerOffsetManager类型，设置给obj变量
                如果obj变量不等于null
                    设置offsetTable属性等于obj.offsetTable
                返回true
       } catch (Exception e) {
           返回false
       }
       返回true
    // 持久化offset配置
    public synchronized void persist()
        序列化this为JSON格式的字符串，设置给jsonString变量
        如果jsonString变量不等于null
            try {
                写入jsonString变量值到brokerController.messageStoreConfig.storePathRootDir/config/consumerOffset.json文件中
            } catch (Exception e) {
            }
    // 扫描过期的offset配置并删除（没有订阅信息，同时逻辑队列消费映射元素列表为空或者的逻辑队列offset全部小于当前逻辑队列的最小offset）
    public void scanUnsubscribedTopic()
        遍历offsetTable属性，设置entry变量等于当前元素
            设置arrays变量等于entry.key.split("@")
            如果arrays变量不等于null并且元素个数等于2
                设置topic变量等于arrays[0]
                设置group变量等于arrays[1]
                如果brokerController.consumerManager.findSubscriptionData(group, topic)等于null并且offsetBehindMuchThanData(topic, entry.value)等于true
                    删除entry变量
    private boolean offsetBehindMuchThanData(String topic, ConcurrentHashMap<Integer, Long> table)
        设置result变量等于!table.isEmpty()
        遍历table参数，设置entry变量等于当前元素
            设置minOffsetInStore变量等于brokerController.messageStore.getMinOffsetInQuque(topic, entry.key)               // ??应该是queue
            如果entry.value大于minOffsetInStore
                设置result变量等于false
        返回result变量
    // 获取指定Consumer消费的逻辑队列的offset
    public long queryOffset(String group, String topic, int queueId)
        设置map变量等于offsetTable.get(topic + "@" + group)
        如果map变量不等于null
            设置offset变量等于map.get(queueId)
            如果offset变量不等于null
                返回offset变量
        返回-1
    // commit操作，更新指定Consumer消费的逻辑队列的offset
    public void commitOffset(String group, String topic, int queueId, long offset)
        设置map变量等于offsetTable.get(topic + "@" + group)
        如果map变量等于null
            设置map变量等于new ConcurrentHashMap<Integer, Long>(32)
            执行map.put(queueId, offset)
            执行offsetTable.put(key, map)
        否则
            执行map.put(queueId, offset)

# com.alibaba.rocketmq.broker.subscription.SubscriptionGroupManager
    // 订阅组信息
    // Key: ConsumerGroup, Value: SubscriptionGroupConfig
    private ConcurrentHashMap<String, SubscriptionGroupConfig> subscriptionGroupTable = new ConcurrentHashMap<String, SubscriptionGroupConfig>(1024)
    private DataVersion dataVersion = new DataVersion()

    public SubscriptionGroupManager(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
        {
            // 工具专用的订阅组
            设置subscriptionGroupConfig变量等于new SubscriptionGroupConfig()实例
            执行subscriptionGroupConfig.setGroupName("TOOLS_CONSUMER");
            执行subscriptionGroupTable.put("TOOLS_CONSUMER", subscriptionGroupConfig);
        }
        {
            // FilterServer使用的订阅组
            设置subscriptionGroupConfig变量等于new SubscriptionGroupConfig()实例
            执行subscriptionGroupConfig.setGroupName("FILTERSRV_CONSUMER");
            执行subscriptionGroupTable.put("FILTERSRV_CONSUMER", subscriptionGroupConfig);
        }
        {
            // 测试的
            设置subscriptionGroupConfig变量等于new SubscriptionGroupConfig()实例
            执行subscriptionGroupConfig.setGroupName("SELF_TEST_C_GROUP");
            执行subscriptionGroupTable.put("SELF_TEST_C_GROUP", subscriptionGroupConfig);
        }
    // 加载配置
    public boolean load()
        try {
            读取brokerController.messageStoreConfig.storePathRootDir/config/subscriptionGroup.json文件内容作为String类型，设置给jsonString变量
            如果jsonString变量等于null
                执行loadBak()并返回
            否则
                解序列化jsonString变量作为SubscriptionGroupManager类型，设置给obj变量
                如果obj变量不等于null
                    执行subscriptionGroupTable.putAll(obj.subscriptionGroupTable)
                    执行dataVersion.assignNewOne(obj.dataVersion)
                返回true
        } catch (Exception e) {
            执行loadBak()并返回
        }
    private boolean loadBak()
       try {
           读取brokerController.messageStoreConfig.storePathRootDir/config/subscriptionGroup.json.bak文件内容作为String类型，设置给jsonString变量
           如果jsonString变量不等于null
                解序列化jsonString变量作为SubscriptionGroupManager类型，设置给obj变量
                如果obj变量不等于null
                    执行subscriptionGroupTable.putAll(obj.subscriptionGroupTable)
                    执行dataVersion.assignNewOne(obj.dataVersion)
                返回true
       } catch (Exception e) {
           返回false
       }
       返回true
    // 返回指定订阅组配置，如果为空，根据配置决定是否新建，如果新建，新建成功后需要持久化配置
    public SubscriptionGroupConfig findSubscriptionGroupConfig(String group)
        设置subscriptionGroupConfig变量等于subscriptionGroupTable.get(group)
        如果subscriptionGroupConfig变量等于null
            如果brokerController.brokerConfig.autoCreateSubscriptionGroup等于true                   // autoCreateSubscriptionGroup默认值: true
                设置subscriptionGroupConfig变量等于new SubscriptionGroupConfig()实例
                执行subscriptionGroupConfig.setGroupName(group)
                执行subscriptionGroupTable.putIfAbsent(group, subscriptionGroupConfig)
                执行dataVersion.nextVersion()
                执行persist()
        返回subscriptionGroupConfig变量

# com.alibaba.rocketmq.broker.longpolling.PullRequestHoldService

    // Key: Topic@QueueId, Value: ManyPullRequest
    private ConcurrentHashMap<String, ManyPullRequest> pullRequestTable = new ConcurrentHashMap<String, ManyPullRequest>(1024)

    public PullRequestHoldService(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    public void start()
    public void shutdown()
    public void shutdown(boolean interrupt)
    public void run()                                                                                                           // 定时做rebalance，允许外部立即触发rebalance
        循环执行，直到stop属性等于true
            try {
                如果hasNotified属性等于true
                    设置hasNotified属性等于false
                否则
                    wait(1000)
                    设置hasNotified属性等于false
                执行checkHoldRequest()
            } catch (Exception e) {
            }
    //
    public void wakeup()
        如果hasNotified属性等于false
            设置hasNotified属性等于true
            执行notify()
    private void checkHoldRequest()
        遍历pullRequestTable属性，设置entry变量等于当前元素
            设置kArray变量等于entry.key.split("@")
            如果kArray变量不等于null并且元素个数等于2
                设置topic变量等于kArray[0]
                设置queueId变量等于Integer.parseInt(kArray[0])
                设置offset变量等于brokerController.messageStore.getMaxOffsetInQuque(topic, queueId)                       // ??应该是queue
                执行notifyMessageArriving(topic, queueId, offset)
    // 存消息成功后调用，通知消费指定逻辑队列的消费者当前最大的offset（消息是同步写入物理队列，异步构建逻辑队列，因此实际还是依赖逻辑队列是否构建）
    public void notifyMessageArriving(String topic, int queueId, long maxOffset)
        设置mpr变量等于pullRequestTable.get(topic + "@" + queueId)
        如果mpr变量不等于null                                              // 执行过拉消息请求，且出现过拉不到最新消息情况
            设置requestList变量等于mpr.cloneListAndClear()
            如果requestList变量不等于null                                  // 不等于null表示有请求
                设置replayList变量等于new ArrayList<PullRequest>()
                遍历requestList变量，设置request变量等于当前元素              // 遍历所有请求
                    如果maxOffset变量大于request.pullFromThisOffset        // 新消息的offset大于拉请求的offset，可以拉取，异步执行拉请求
                        try {
                            执行brokerController.pullMessageProcessor.excuteRequestWhenWakeup(request.clientChannel, request.requestCommand)
                        } catch (RemotingCommandException e) {
                        }
                        继续下一次循环
                    否则
                        // 获取当前逻辑逻辑队列的最大offset
                        设置newestOffset变量等于brokerController.messageStore.getMaxOffsetInQuque(topic, queueId)         // ??应该是queue
                        // 逻辑队列的最大offset大于拉请求的offset，可以拉取，异步执行拉请求
                        如果newestOffset变量大于request.pullFromThisOffset
                            try {
                                执行brokerController.pullMessageProcessor.excuteRequestWhenWakeup(request.clientChannel, request.requestCommand)
                            } catch (RemotingCommandException e) {
                            }
                            继续下一次循环
                    // 如果超时了，强制执行一次拉取
                    如果当前系统时间戳大于等于request.suspendTimestamp + request.timeoutMillis
                        try {
                            执行brokerController.pullMessageProcessor.excuteRequestWhenWakeup(request.clientChannel, request.requestCommand)
                        } catch (RemotingCommandException e) {
                        }
                        继续下一次循环
                    // 其他情况继续下一次执行
                    执行replayList.add(request)
                如果replayList变量元素个数大于0
                    执行mpr.addPullRequest(replayList)
    // Consumer拉取消息时，如果没有新的消息，则把pullRequest加入到请求列表中
    public void suspendPullRequest(String topic, int queueId, PullRequest pullRequest)
        设置mpr变量等于pullRequestTable.get(topic + "@" + queueId)
        如果mpr变量不等于null
            设置mpr变量new ManyPullRequest()实例
            设置prev变量等于pullRequestTable.putIfAbsent(key, mpr)
            如果prev变量不等于null
                设置mpr变量等于prev变量
        执行mpr.addPullRequest(pullRequest)

# com.alibaba.rocketmq.broker.client.ConsumerManager

    // Key: Group, Value: ConsumerGroupInfo
    private ConcurrentHashMap<String, ConsumerGroupInfo> consumerTable = new ConcurrentHashMap<String, ConsumerGroupInfo>(1024)

    public ConsumerManager(ConsumerIdsChangeListener consumerIdsChangeListener)
        设置consumerIdsChangeListener属性等于consumerIdsChangeListener参数
    // Channel关闭事件，如果存在且删除成功，通知group下的clientId列表发生变化，需要重新均衡消息队列
    public void doChannelCloseEvent(String remoteAddr, Channel channel)
        遍历groupChannelTable属性，设置entry变量等于当前元素
            如果entry.value.doChannelCloseEvent(remoteAddr, channel)等于true
                如果entry.value.channelInfoTable元素个数等于0
                    执行consumerTable.remove(entry.key)
                执行consumerIdsChangeListener.consumerIdsChanged(entry.key, entry.value.getAllChannel())
    // 扫描过期的（lastUpdateTimestamp会不断更新）并删除
    public void scanNotActiveChannel()
        遍历consumerTable属性，设置entry变量等于当前元素
            遍历entry.value.channelInfoTable，设置subEntry变量等于当前元素
                如果系统当前时间戳减去subEntry.value.lastUpdateTimestamp的差值大于120 * 1000
                    执行RemotingUtil.closeChannel(subEntry.value.channel)
                    移除subEntry变量
            如果entry.value.channelInfoTable元素个数等于0
                移除entry变量
    // 获取指定consumerGroup的消费组信息
    public ConsumerGroupInfo getConsumerGroupInfo(String group)
        执行consumerTable.get(group)并返回
    // 获取指定consumerGroup下的clientId对应的Consumer的客户端信息
    public ClientChannelInfo findChannel(String group, String clientId)
        设置consumerGroupInfo变量等于consumerTable.get(group)
        如果consumerGroupInfo变量不等于null
            返回consumerGroupInfo.findChannel(clientId)
        返回null
    // 注册consumer客户端信息，如果存在，更新lastUpdateTimestamp，如果添加成功，通知group下的clientId列表发生变化，需要重新均衡消息队列
    public boolean registerConsumer(String group, ClientChannelInfo clientChannelInfo, ConsumeType consumeType, MessageModel messageModel, ConsumeFromWhere consumeFromWhere, Set<SubscriptionData> subList)
        设置consumerGroupInfo变量等于consumerTable.get(group)
        如果consumerGroupInfo变量不等于null
            设置tmp变量等于new ConsumerGroupInfo(group, consumeType, messageModel, consumeFromWhere)实例
            设置prev变量等于consumerTable.putIfAbsent(group, tmp)
            如果prev变量不等于null
                设置consumerGroupInfo变量等于prev变量
            否则
                设置consumerGroupInfo变量等于tmp变量
        设置r1变量等于consumerGroupInfo.updateChannel(clientChannelInfo, consumeType, messageModel, consumeFromWhere)
        设置r2变量等于consumerGroupInfo.updateSubscription(subList)
        如果r1变量等于true或者r2变量等于true
            执行consumerIdsChangeListener.consumerIdsChanged(group, consumerGroupInfo.getAllChannel())
    // 解注册consumer客户端信息，如果解注册成功，通知group下的clientId列表发生变化，需要重新均衡消息队列
    public void unregisterConsumer(String group, ClientChannelInfo clientChannelInfo)
        设置consumerGroupInfo变量等于consumerTable.get(group)
        如果consumerGroupInfo变量不等于null
            执行consumerGroupInfo.unregisterChannel(clientChannelInfo)
            如果consumerGroupInfo.channelInfoTable元素个数为0
                执行consumerTable.remove(group)
            执行consumerIdsChangeListener.consumerIdsChanged(group, consumerGroupInfo.getAllChannel())

# com.alibaba.rocketmq.broker.client.DefaultConsumerIdsChangeListener

    public DefaultConsumerIdsChangeListener(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    // 通知指定Consumer列表，group下的clientId列表发生变化，需要重新均衡消息队列
    public void consumerIdsChanged(String group, List<Channel> channels)
        如果channels参数不等于null并且brokerController.brokerConfig.notifyConsumerIdsChangedEnable等于true
            遍历channels参数，设置chl变量等于当前元素
                执行brokerController.broker2Client.notifyConsumerIdsChanged(chl, group)

# com.alibaba.rocketmq.broker.client.ProducerManager

    private Lock groupChannelLock = new ReentrantLock()
    // Key: group name, Value: Key: Channel, Value: ClientChannelInfo
    private HashMap<String, HashMap<Channel, ClientChannelInfo>> groupChannelTable = new HashMap<String, HashMap<Channel, ClientChannelInfo>>()

    // Channel关闭事件，移除指定Channel
    public void doChannelCloseEvent(String remoteAddr, Channel channel)
        如果channel参数不等于null
            try {
                if (groupChannelLock.tryLock(3000, TimeUnit.MILLISECONDS)) {
                    try {
                        遍历groupChannelTable属性，设置entry变量等于当前元素
                            执行entry.value.remove(channel)
                    }finally {
                        groupChannelLock.unlock();
                    }
                }
            } catch (InterruptedException e) {
            }
    // 扫描过期的（lastUpdateTimestamp会不断更新）并删除
    public void scanNotActiveChannel()
         try {
             if (groupChannelLock.tryLock(3000, TimeUnit.MILLISECONDS)) {
                 try {
                     遍历groupChannelTable属性，设置entry变量等于当前元素
                        遍历entry.value，设置subEntry变量等于当前元素
                            如果系统当前时间戳减去subEntry.value.lastUpdateTimestamp的差值大于120 * 1000
                                移除subEntry变量
                                执行RemotingUtil.closeChannel(subEntry.value.channel)
                 }finally {
                     groupChannelLock.unlock();
                 }
             }
         } catch (InterruptedException e) {
         }
    // 注册producer客户端信息，最后更新lastUpdateTimestamp
    public void registerProducer(String group, ClientChannelInfo clientChannelInfo)
        try {
            设置clientChannelInfoFound变量等于null
            if (groupChannelLock.tryLock(3000, TimeUnit.MILLISECONDS)) {
                try {
                    设置channelTable变量等于groupChannelTable.get(group)
                    如果channelTable变量等于null
                        设置channelTable变量等于new HashMap<Channel, ClientChannelInfo>()
                        执行groupChannelTable.put(group, channelTable)
                    设置clientChannelInfoFound变量等于channelTable.get(clientChannelInfo.channel)
                    如果clientChannelInfoFound变量等于null
                        执行channelTable.put(clientChannelInfo.channel, clientChannelInfo)
                } finally {
                    groupChannelLock.unlock();
                }
                如果clientChannelInfoFound变量不等于null
                    执行clientChannelInfoFound.setLastUpdateTimestamp(System.currentTimeMillis())
            }
        } catch (InterruptedException e) {
        }
    // 解注册producer客户端信息，删除对应group下的channel，最后更新lastUpdateTimestamp
    public void unregisterProducer(String group, ClientChannelInfo clientChannelInfo)
        try {
            设置clientChannelInfoFound变量等于null
            if (groupChannelLock.tryLock(3000, TimeUnit.MILLISECONDS)) {
                try {
                    设置channelTable变量等于groupChannelTable.get(group)
                    如果channelTable变量不等于null并且元素个数大于0
                        执行channelTable.remove(clientChannelInfo.channel)
                        如果channelTable变量元素个数等于0
                            执行groupChannelTable.remove(group)
                } finally {
                    groupChannelLock.unlock();
                }
                如果clientChannelInfoFound变量不等于null
                    执行clientChannelInfoFound.setLastUpdateTimestamp(System.currentTimeMillis())
            }
        } catch (InterruptedException e) {
        }

# com.alibaba.rocketmq.broker.filtersrv.FilterServerManager

    // Key: Channel, Value: FilterServerInfo
    // Channel表示注册链接
    // FilterServerInfo表示filterServer监听端口
    private ConcurrentHashMap<Channel, FilterServerInfo> filterServerTable = new ConcurrentHashMap<Channel, FilterServerInfo>(16)

    public FilterServerManager(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    public void start()
        执行定时任务，延迟5秒钟执行，任务执行间隔为30秒
            try {
                执行createFilterServer()              // 命令行启动FilterServer
            } catch (Exception e) {
            }
    // 启动一个FilterServer
    public void createFilterServer()
        设置more变量等于brokerController.brokerConfig.filterServerNums - filterServerTable.size()
        设置cmd变量等于"sh rocketmqHome/bin/startfsrv.sh -c 启动broker时指定的配置文件 -n nameserver地址"
        循环more次，设置i等于当前循环索引
            执行FilterServerUtil.callShell(cmd, log)
    public void shutdown()
        关闭定时任务
    // Channel关闭事件，移除指定Channel
    public void doChannelCloseEvent(String remoteAddr, Channel channel)
        执行filterServerTable.remove(channel)
    // 扫描过期的（lastUpdateTimestamp会不断更新）并删除
    public void scanNotActiveChannel()
        遍历filterServerTable属性，设置entry变量等于当前元素
            如果当前时间减去entry.value.lastUpdateTimestamp的差值大于30000
                移除entry变量
                执行RemotingUtil.closeChannel(entry.key)
    // 注册FilterServer客户端信息，更新lastUpdateTimestamp
    public void registerFilterServer(Channel channel, String filterServerAddr)
        设置filterServerInfo属性等于filterServerTable.get(channel)
        如果filterServerInfo属性不等于null
            执行filterServerInfo.setLastUpdateTimestamp(System.currentTimeMillis())
        否则
            设置filterServerInfo变量等于new FilterServerInfo()实例
            执行filterServerInfo.setFilterServerAddr(filterServerAddr);
            执行filterServerInfo.setLastUpdateTimestamp(System.currentTimeMillis());
            执行filterServerTable.put(channel, filterServerInfo);
    // 返回当前所有FilterServer的serverAddr
    public List<String> buildNewFilterServerList()
        设置addr变量等于new ArrayList<String>()实例
        遍历filterServerTable属性，设置entry变量等于当前元素
            执行addr.add(entry.value.filterServerAddr)
        返回addr变量

## com.alibaba.rocketmq.broker.filtersrv.FilterServerUtil

    public static void callShell(String shellString, Logger log)
        Process process = null;
        try {
            String[] cmdArray = splitShellString(shellString);
            process = Runtime.getRuntime().exec(cmdArray);
            process.waitFor();
            log.info("callShell: <{}> OK", shellString);
        }
        catch (Throwable e) {
            log.error("callShell: readLine IOException, " + shellString, e);
        } finally {
            if (null != process)
                process.destroy();
        }

# com.alibaba.rocketmq.broker.slave.SlaveSynchronize

    // Slave专用，用于同步master信息
    public SlaveSynchronize(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    // 设置master地址
    public void setMasterAddr(String masterAddr)
        设置masterAddr属性等于masterAddr参数
    // 同步主题配置信息、消费者offset信息、延迟队列offset信息、订阅组配置
    public void syncAll()
        执行syncTopicConfig()
        执行syncConsumerOffset()
        执行syncDelayOffset()
        执行syncSubscriptionGroupConfig()
    // 请求master获取主题配置信息，如果版本不一致，更新和同步
    private void syncTopicConfig()
        设置masterAddrBak变量等于masterAddr属性
        如果masterAddrBak变量不等于null
            try {
                设置topicWrapper变量等于brokerController.brokerOuterAPI.getAllTopicConfig(masterAddrBak)
                如果brokerController.topicConfigManager.dataVersion不等于topicWrapper.dataVersion
                    执行brokerController.topicConfigManager.dataVersion.assignNewOne(topicWrapper.dataVersion)
                    执行brokerController.topicConfigManager.topicConfigTable.clear()
                    执行brokerController.topicConfigManager.topicConfigTable.putAll(topicWrapper.topicConfigTable)
                    执行brokerController.topicConfigManager.persist()
            } catch (Exception e) {
            }
    // 请求master获取消费者offset信息，更新和同步
    private void syncConsumerOffset()
        设置masterAddrBak变量等于masterAddr属性
        如果masterAddrBak变量不等于null
            try {
                设置offsetWrapper变量等于brokerController.brokerOuterAPI.getAllConsumerOffset(masterAddrBak)
                执行brokerController.consumerOffsetManager.offsetTable.putAll(offsetWrapper.offsetTable)
                执行brokerController.consumerOffsetManager.persist()
            } catch (Exception e) {
            }
    // 请求master获取延迟队列offset信息，同步
    private void syncDelayOffset()
        设置masterAddrBak变量等于masterAddr属性
        如果masterAddrBak变量不等于null
            try {
                设置delayOffset变量等于brokerController.brokerOuterAPI.getAllDelayOffset(masterAddrBak)
                如果delayOffset变量不等于null
                    设置fileName变量等于brokerController.messageStoreConfig.storePathRootDir/config/delayOffset.json
                    try {
                        写入delayOffset变量内容到fileName变量对应的文件
                    } catch (IOException e) {
                    }
            } catch (Exception e) {
            }
    // 请求master获取订阅组配置信息，如果版本不一致，更新和同步
    private void syncSubscriptionGroupConfig()
        设置masterAddrBak变量等于masterAddr属性
        如果masterAddrBak变量不等于null
            try {
                设置subscriptionWrapper变量等于brokerController.brokerOuterAPI.getAllSubscriptionGroupConfig(masterAddrBak)
                如果brokerController.subscriptionGroupManager.dataVersion不等于subscriptionWrapper.dataVersion
                    执行brokerController.subscriptionGroupManager.dataVersion.assignNewOne(topicWrapper.dataVersion)
                    执行brokerController.subscriptionGroupManager.subscriptionGroupTable.clear()
                    执行brokerController.subscriptionGroupManager.subscriptionGroupTable.putAll(topicWrapper.topicConfigTable)
                    执行brokerController.subscriptionGroupManager.persist()
            } catch (Exception e) {
            }

# com.alibaba.rocketmq.broker.client.rebalance.RebalanceLockManager

    // 顺序消费场景专用

    private Lock lock = new ReentrantLock();
    // Key: Group, Value: Key: MessageQueue, Value: LockEntry
    private ConcurrentHashMap<String, ConcurrentHashMap<MessageQueue, LockEntry>> mqLockTable = new ConcurrentHashMap<String, ConcurrentHashMap<MessageQueue, LockEntry>>(1024)

    // group消费组的clientId申请独占mqs消息队列，返回独占成功的
    public Set<MessageQueue> tryLockBatch(String group, Set<MessageQueue> mqs, String clientId)
        设置lockedMqs变量等于new HashSet<MessageQueue>(mqs.size())实例
        设置notLockedMqs变量等于new HashSet<MessageQueue>(mqs.size())实例
        遍历mqs参数，设置mq变量等于当前元素
            如果isLocked(group, mq, clientId)等于true               // 如果之前已被当前clientId占用，则继续占用
                执行lockedMqs.add(mq)
            否则
                执行notLockedMqs.add(mq)
        如果notLockedMqs变量元素个数大于0                             //
            try {
                执行lock.lockInterruptibly()
                try {
                    设置groupValue变量等于mqLockTable.get(group)
                    如果groupValue变量等于null
                        设置groupValue变量等于new ConcurrentHashMap<MessageQueue, LockEntry>(32)
                        执行mqLockTable.put(group, groupValue)
                    遍历notLockedMqs变量，设置mq变量等于当前元素             // 遍历没有锁住的队列
                        设置lockEntry变量等于groupValue.get(mq)
                        如果lockEntry变量等于null                         // 没被独占过，设置独占
                            设置lockEntry变量等于new LockEntry()实例
                            执行lockEntry.setClientId(clientId)
                            执行groupValue.put(mq, lockEntry)
                        如果lockEntry.isLocked(clientId)等于true         // 已经锁定
                            执行lockEntry.setLastUpdateTimestamp(System.currentTimeMillis())
                            执行lockedMqs.add(mq)
                            继续下一次循环
                        设置oldClientId变量等于lockEntry.clientId
                        如果lockEntry.isExpired()等于true                // 被其他client独占，但是过期了，设置独占
                            执行lockEntry.setClientId(clientId)
                            执行lockEntry.setLastUpdateTimestamp(System.currentTimeMillis())
                            执行lockedMqs.add(mq)
                            继续下一次循环
                    }
                } finally {
                    执行lock.unlock();
                }
            } catch (InterruptedException e) {
            }
        返回lockedMqs变量
    // group消费组的clientId是否对mq独占过，如果是，更新lastUpdateTimestamp
    private boolean isLocked(String group, MessageQueue mq, String clientId)
        设置groupValue变量等于mqLockTable.get(group)
        如果groupValue变量不等于null
            设置lockEntry变量等于groupValue.get(mq)
            如果lockEntry变量不等于null
                设置locked变量等于lockEntry.isLocked(clientId)
                如果locked变量等于true
                    执行lockEntry.setLastUpdateTimestamp(System.currentTimeMillis())
                返回locked变量
        返回false
    // group消费组的clientId申请解除独占的mqs消息队列
    public void unlockBatch(String group, Set<MessageQueue> mqs, String clientId)
        try {
             执行lock.lockInterruptibly()
             try {
                设置groupValue变量等于mqLockTable.get(group)
                如果groupValue变量不等于null
                    遍历mqs变量，设置mq变量等于当前元素
                        设置lockEntry变量等于groupValue.get(mq)
                        如果lockEntry变量不等于null
                            如果lockEntry.clientId等于clientId
                                执行groupValue.remove(mq)
             } finally {
                 执行lock.unlock();
             }
         } catch (InterruptedException e) {
         }

# com.alibaba.rocketmq.store.stats.BrokerStatsManager

    public BrokerStatsManager(String clusterName)

# com.alibaba.rocketmq.store.stats.BrokerStats

    public BrokerStats(DefaultMessageStore defaultMessageStore)



        