# com.alibaba.rocketmq.broker.BrokerStartup

    public static void main(String[] args)
        创建new BrokerConfig()实例，设置给brokerConfig变量
        创建new NettyServerConfig()实例，设置给nettyServerConfig变量
        设置nettyServerConfig.listenPort属性等于10911
        创建new NettyClientConfig()实例，设置给nettyClientConfig变量
        创建new MessageStoreConfig()实例，设置给messageStoreConfig变量
        如果messageStoreConfig.getBrokerRole()等于SLAVE                 // 如果是Slave，修改配置
            执行messageStoreConfig.setAccessMessageInMemoryMaxRatio(messageStoreConfig.accessMessageInMemoryMaxRatio - 10)
        如果指定了配置文件
            使用配置文件填充brokerConfig变量的相关属性
            使用配置文件填充nettyServerConfig变量的相关属性
            使用配置文件填充nettyClientConfig变量的相关属性
            使用配置文件填充messageStoreConfig变量的相关属性
        使用命令行参数填充brokerConfig变量的相关属性
        验证namesrvConfig.rocketmqHome设置过，否则程序退出
        如果brokerConfig.namesrvAddr不等于null                           // 检测NameServer设置是否正确
            使用;分隔并进行遍历，将分隔后的字符串使用:分隔，分别作为host和port创建Socket实例，如果出现异常，则程序退出
        如果messageStoreConfig.brokerRole等于ASYNC_MASTER或SYNC_MASTER
            执行brokerConfig.setBrokerId(0)                                               // MASTER_ID: 0
        否则如果messageStoreConfig.brokerRole等于SLAVE
            验证brokerConfig.brokerId必须大于0，否则抛异常
        执行messageStoreConfig.setHaListenPort(nettyServerConfig.listenPort + 1)          // Master监听Slave请求的端口，默认为服务端口+1，listenPort默认为8888
        创建new BrokerController(brokerConfig, nettyServerConfig, nettyClientConfig, messageStoreConfig)实例，设置给controller变量
        执行controller.initialize()，返回boolean类型，设置给initResult变量
        如果initResult变量等于false
            执行controller.shutdown()并退出
        添加钩子
           执行controller.shutdown()
        执行controller.start()

# com.alibaba.rocketmq.broker.BrokerController
    private List<SendMessageHook> sendMessageHookList = new ArrayList<SendMessageHook>()
    private List<ConsumeMessageHook> consumeMessageHookList = new ArrayList<ConsumeMessageHook>()
    RebalanceLockManager rebalanceLockManager = new RebalanceLockManager()
    
    public BrokerController(BrokerConfig brokerConfig, NettyServerConfig nettyServerConfig, NettyClientConfig nettyClientConfig, MessageStoreConfig messageStoreConfig)
        设置brokerConfig属性等于brokerConfig参数
        设置nettyServerConfig属性等于nettyServerConfig参数
        设置nettyClientConfig属性等于nettyClientConfig参数
        设置messageStoreConfig属性等于messageStoreConfig参数
        设置consumerOffsetManager属性等于new ConsumerOffsetManager(this)实例
        设置topicConfigManager属性等于new TopicConfigManager(this)实例
        设置pullMessageProcessor属性等于new PullMessageProcessor(this)实例
        设置pullRequestHoldService属性等于new PullRequestHoldService(this)实例
        设置consumerIdsChangeListener属性等于new DefaultConsumerIdsChangeListener(this)实例
        设置consumerManager属性等于new ConsumerManager(consumerIdsChangeListener)实例
        设置producerManager属性等于new ProducerManager()实例
        设置clientHousekeepingService属性等于new ClientHousekeepingService(this)实例
        设置broker2Client属性等于new Broker2Client(this)实例
        设置subscriptionGroupManager属性等于new SubscriptionGroupManager(this)实例
        设置brokerOuterAPI属性等于new BrokerOuterAPI(nettyClientConfig)实例
        设置filterServerManager属性等于new FilterServerManager(this)实例
        如果brokerConfig.namesrvAddr不等于null                                           // 默认值: System.getProperty("rocketmq.namesrv.addr", System.getenv("NAMESRV_ADDR"))
            执行brokerOuterAPI.updateNameServerAddressList(brokerConfig.namesrvAddr)
        设置slaveSynchronize属性等于new SlaveSynchronize(this)
        设置sendThreadPoolQueue属性等于new LinkedBlockingQueue<Runnable>(brokerConfig.sendThreadPoolQueueCapacity)        // 默认值: 100000
        设置pullThreadPoolQueue属性等于new LinkedBlockingQueue<Runnable>(brokerConfig.pullThreadPoolQueueCapacity)        // 默认值: 100000
        设置brokerStatsManager属性等于new BrokerStatsManager(brokerConfig.brokerClusterName)                              // 默认值: DefaultCluster
        设置storeHost属性等于new InetSocketAddress(brokerConfig.brokerIP1, nettyServerConfig.listenPort)                  // brokerIP1默认值: 本机ip
    public boolean initialize()
        设置result变量等于true
        设置result变量等于result && topicConfigManager.load()             // 加载主题配置信息
        设置result变量等于result && consumerOffsetManager.load()          // 加载消费者offset信息
        设置result变量等于result && subscriptionGroupManager.load()       // 加载订阅信息
        如果result等于true
            try {
                设置messageStore变量等于new DefaultMessageStore(messageStoreConfig, brokerStatsManager)实例
            } catch (IOException e) {
                设置result变量等于false
            }
        设置result变量等于result && messageStore.load()                   // 加载主题
        如果result变量等于true
            设置remotingServer属性等于new NettyRemotingServer(nettyServerConfig, clientHousekeepingService)
            // sendMessageThreadPoolNums默认值: 16 + Runtime.getRuntime().availableProcessors() * 4
            设置sendMessageExecutor属性等于new ThreadPoolExecutor(brokerConfig.sendMessageThreadPoolNums, brokerConfig.sendMessageThreadPoolNums, 1000 * 60, TimeUnit.MILLISECONDS, sendThreadPoolQueue, new ThreadFactoryImpl("SendMessageThread_"))
            // pullMessageThreadPoolNums默认值: 16 + Runtime.getRuntime().availableProcessors() * 2
            设置pullMessageExecutor属性等于new ThreadPoolExecutor(brokerConfig.pullMessageThreadPoolNums, brokerConfig.pullMessageThreadPoolNums, 1000 * 60, TimeUnit.MILLISECONDS, pullThreadPoolQueue, new ThreadFactoryImpl("PullMessageThread_"))
            // adminBrokerThreadPoolNums默认值: 16
            设置adminBrokerExecutor属性等于Executors.newFixedThreadPool(brokerConfig.adminBrokerThreadPoolNums, new ThreadFactoryImpl("AdminBrokerThread_"))
            // clientManageThreadPoolNums默认值: 16
            设置clientManageExecutor属性等于Executors.newFixedThreadPool(brokerConfig.clientManageThreadPoolNums, new ThreadFactoryImpl("ClientManageThread_"))

            设置sendProcessor变量等于new SendMessageProcessor(this)实例
            执行sendProcessor.registerSendMessageHook(sendMessageHookList)
            执行remotingServer.registerProcessor(310, sendProcessor, sendMessageExecutor)                                 // SEND_MESSAGE_V2: 310
            执行remotingServer.registerProcessor(36, sendProcessor, sendMessageExecutor)                                  // CONSUMER_SEND_MSG_BACK: 36

            执行remotingServer.registerProcessor(11, pullMessageProcessor, pullMessageExecutor)                           // PULL_MESSAGE: 11
            执行pullMessageProcessor.registerConsumeMessageHook(consumeMessageHookList)

            设置queryProcessor变量等于new QueryMessageProcessor(this)实例
            执行remotingServer.registerProcessor(12, queryProcessor, pullMessageExecutor)                                 // QUERY_MESSAGE: 12
            执行remotingServer.registerProcessor(33, queryProcessor, pullMessageExecutor)                                 // VIEW_MESSAGE_BY_ID: 33

            设置clientProcessor变量等于new ClientManageProcessor(this)实例
            执行clientProcessor.registerConsumeMessageHook(consumeMessageHookList)
            执行remotingServer.registerProcessor(34, clientProcessor, clientManageExecutor)                               // HEART_BEAT: 34
            执行remotingServer.registerProcessor(35, clientProcessor, clientManageExecutor)                               // UNREGISTER_CLIENT: 35
            执行remotingServer.registerProcessor(38, clientProcessor, clientManageExecutor)                               // GET_CONSUMER_LIST_BY_GROUP: 38
            执行remotingServer.registerProcessor(15, clientProcessor, clientManageExecutor)                               // UPDATE_CONSUMER_OFFSET: 15
            执行remotingServer.registerProcessor(14, clientProcessor, clientManageExecutor)                               // QUERY_CONSUMER_OFFSET: 14
            执行remotingServer.registerProcessor(37, new EndTransactionProcessor(this), sendMessageExecutor)              // END_TRANSACTION: 37
            执行remotingServer.registerDefaultProcessor(new AdminBrokerProcessor(this), adminBrokerExecutor)              // 添加默认事件处理器

            设置brokerStats变量等于new BrokerStats((DefaultMessageStore) messageStore)实例
            执行定时任务，延迟到隔天的00:00:00.000执行，执行间隔为24小时
                try {
                    执行brokerStats.record()
                } catch (Exception e) { }
            执行定时任务，延迟10秒执行，执行间隔为brokerConfig.flushConsumerOffsetInterval毫秒          // 默认值: 1000 * 5
                try {
                    执行consumerOffsetManager.persist()
                } catch (Exception e) { }
            执行定时任务，延迟10分钟执行，执行间隔为60分钟
                try {
                    执行consumerOffsetManager.scanUnsubscribedTopic()
                } catch (Exception e) { }
            如果brokerConfig.namesrvAddr不等于null
                执行brokerOuterAPI.updateNameServerAddressList(brokerConfig.namesrvAddr)
            否则如果brokerConfig.fetchNamesrvAddrByAddressServer等于true
                执行定时任务，延迟10秒钟执行，执行间隔为120秒钟
                    try {
                        执行brokerOuterAPI.fetchNameServerAddr()
                    } catch (Exception e) { }
            如果messageStoreConfig.brokerRole等于SLAVE
                如果messageStoreConfig.haMasterAddress不等于null并且长度大于等于6
                    执行messageStore.updateHaMasterAddress(messageStoreConfig.haMasterAddress)
                    设置updateMasterHAServerAddrPeriodically属性等于false
                否则
                    设置updateMasterHAServerAddrPeriodically属性等于true                  // 如果没指定，每次根据nameserver返回的为准
                执行定时任务，延迟10秒钟执行，执行间隔为60秒钟
                    try {
                        执行slaveSynchronize.syncAll()
                    } catch (Exception e) { }
        返回result变量
    public void start()
        如果messageStore属性不等于null
            执行messageStore.start()
        如果remotingServer属性不等于null
            执行remotingServer.start()
        如果brokerOuterAPI属性不等于null
            执行brokerOuterAPI.start()
        如果pullRequestHoldService属性不等于null
            执行pullRequestHoldService.start()
        如果clientHousekeepingService属性不等于null
            执行clientHousekeepingService.start()
        如果filterServerManager属性不等于null
            执行filterServerManager.start()
        执行registerBrokerAll(true, false)
        执行定时任务，延迟10秒钟执行，执行间隔为30秒钟
            try {
                执行registerBrokerAll(true, false)
            } catch (Exception e) { }
        如果brokerStatsManager属性不等于null
            执行brokerStatsManager.start()
        执行addDeleteTopicTask()
    public synchronized void registerBrokerAll(boolean checkOrderConfig, boolean oneway)
        设置topicConfigWrapper变量等于topicConfigManager.buildTopicConfigSerializeWrapper()               // 当前所有主题的配置信息
        如果brokerConfig.brokerPermission没有写权限或者读权限                                               // 默认值: 支持读和写
            设置topicConfigTable变量等于new ConcurrentHashMap<String, TopicConfig>(topicConfigWrapper.topicConfigTable)
            遍历topicConfigTable.values()，设置topicConfig变量等于当前元素
                topicConfig.setPerm(brokerConfig.brokerPermission)
            执行topicConfigWrapper.setTopicConfigTable(topicConfigTable)
        // 请求nameserver，注册brokerServer信息
        设置registerBrokerResult变量等于brokerOuterAPI.registerBrokerAll(brokerConfig.brokerClusterName, getBrokerAddr(), brokerConfig.brokerName, brokerConfig.brokerId, getHAServerAddr(), topicConfigWrapper, filterServerManager.buildNewFilterServerList(), oneway)      // buildNewFilterServerList返回当前brokerServer启动的所有fliterServer列表
        如果registerBrokerResult变量不等于null
            如果updateMasterHAServerAddrPeriodically等于true并且registerBrokerResult.haServerAddr不等于null                // 对slave模式有用
                执行messageStore.updateHaMasterAddress(registerBrokerResult.haServerAddr)
            执行slaveSynchronize.setMasterAddr(registerBrokerResult.masterAddr)                                          // 对slave模式有用
            如果checkOrderConfig变量等于true
                执行topicConfigManager.updateOrderTopicConfig(registerBrokerResult.kvTable)                              // 根据kvTable，重新设置顺序主题和非顺序主题信息
    public void addDeleteTopicTask()
        执行任务，延迟5分钟执行
            try {
                执行messageStore.cleanUnusedTopic(topicConfigManager.topicConfigTable.keySet())                          // 根据当前brokerServer的持有的主题，删除不在使用的主题
            } catch (Exception e) { }
    public String getBrokerAddr()
        返回brokerConfig.brokerIP1 + ":" + nettyServerConfig.listenPort                                                  // brokerIP1默认值: 本机IP
    public String getHAServerAddr()
        返回brokerConfig.brokerIP2 + ":" + messageStoreConfig.haListenPort                                               // brokerIP2默认值: 本机IP
    public void shutdown()
        如果brokerStatsManager属性不等于null
            执行brokerStatsManager.shutdown()
        如果clientHousekeepingService属性不等于null
            执行clientHousekeepingService.shutdown()
        如果pullRequestHoldService属性不等于null
            执行pullRequestHoldService.shutdown()
        如果remotingServer属性不等于null
            执行remotingServer.shutdown()
        如果messageStore属性不等于null
            执行messageStore.shutdown()
        关闭开启的定时任务
        // 请求nameserver，解注册brokerServer信息
        执行brokerOuterAPI.unregisterBrokerAll(brokerConfig.brokerClusterName, getBrokerAddr(), brokerConfig.brokerName, brokerConfig.brokerId)
        如果sendMessageExecutor属性不等于null
            执行sendMessageExecutor.shutdown()
        如果pullMessageExecutor属性不等于null
            执行pullMessageExecutor.shutdown()
        如果adminBrokerExecutor属性不等于null
            执行adminBrokerExecutor.shutdown()
        如果brokerOuterAPI属性不等于null
            执行brokerOuterAPI.shutdown()
        执行consumerOffsetManager.persist()
        如果filterServerManager属性不等于null
            执行filterServerManager.shutdown()
