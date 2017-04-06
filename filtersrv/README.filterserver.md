# com.alibaba.rocketmq.filtersrv.FiltersrvStartup

    public static void main(String[] args)
        创建new FiltersrvConfig()实例，设置给filtersrvConfig变量
        创建new NettyServerConfig()实例，设置给nettyServerConfig变量
        如果指定了配置文件
            使用配置文件填充filtersrvConfig变量的相关属性
            如果配置文件中包含listenPort属性
                执行filtersrvConfig.setConnectWhichBroker("127.0.0.1:" + listenPort)
        执行nettyServerConfig.setListenPort(0)                    // 设置为0表示自动分配端口号
        执行nettyServerConfig.setServerAsyncSemaphoreValue(filtersrvConfig.fsServerAsyncSemaphoreValue)
        执行nettyServerConfig.setServerCallbackExecutorThreads(filtersrvConfig.fsServerCallbackExecutorThreads)
        执行nettyServerConfig.setServerWorkerThreads(filtersrvConfig.fsServerWorkerThreads)
        使用命令行参数填充filtersrvConfig变量的相关属性
        验证filtersrvConfig.rocketmqHome设置过，否则程序退出
        创建new FiltersrvController(filtersrvConfig, nettyServerConfig)实例，设置给controller变量
        执行controller.initialize()，返回boolean类型，设置给initResult变量
        如果initResult变量等于false
            执行controller.shutdown()并退出
        添加钩子
           执行controller.shutdown()
        执行controller.start()

# com.alibaba.rocketmq.filtersrv.FiltersrvController

    private FilterServerOuterAPI filterServerOuterAPI = new FilterServerOuterAPI()
    private DefaultMQPullConsumer defaultMQPullConsumer = new DefaultMQPullConsumer("FILTERSRV_CONSUMER")

    public FiltersrvController(FiltersrvConfig filtersrvConfig, NettyServerConfig nettyServerConfig)
        设置filtersrvConfig属性等于filtersrvConfig参数
        设置nettyServerConfig属性等于nettyServerConfig参数
        设置filterClassManager属性等于new FilterClassManager(this)实例
    public boolean initialize()
        设置remotingServer属性等于new NettyRemotingServer(nettyServerConfig)实例
        设置remotingExecutor属性等于Executors.newFixedThreadPool(nettyServerConfig.serverWorkerThreads, new ThreadFactoryImpl("RemotingExecutorThread_"))
        执行remotingServer.registerDefaultProcessor(new DefaultRequestProcessor(this), remotingExecutor)
        启动定时任务，延迟3秒钟执行，任务执行间隔为10毫秒
            执行registerFilterServerToBroker()
        执行defaultMQPullConsumer.setBrokerSuspendMaxTimeMillis(defaultMQPullConsumer.brokerSuspendMaxTimeMillis - 1000)
        执行defaultMQPullConsumer.setConsumerTimeoutMillisWhenSuspend(defaultMQPullConsumer.consumerTimeoutMillisWhenSuspend - 1000)
        执行defaultMQPullConsumer.setNamesrvAddr(filtersrvConfig.namesrvAddr)                                            // nameserver为brokerServer命令行启动filterServer时传递
        执行defaultMQPullConsumer.setInstanceName(String.valueOf(UtilAll.getPid()))
        返回true
    // 注册serverIp:port到brokerServer
    public void registerFilterServerToBroker()
        try {
            设置responseHeader变量等于filterServerOuterAPI.registerFilterServerToBroker(filtersrvConfig.connectWhichBroker, localAddr())
            执行defaultMQPullConsumer.defaultMQPullConsumerImpl.pullAPIWrapper.setDefaultBrokerId(responseHeader.brokerId)
            如果brokerName属性等于null
                设置brokerName属性等于responseHeader.brokerName
        } catch (Exception e) {
            System.exit(-1)
        }
    public String localAddr()
        返回filtersrvConfig.filterServerIP + ":" + remotingServer.port
    public void start() throws Exception
        执行defaultMQPullConsumer.start()
        执行remotingServer.start()
        执行filterServerOuterAPI.start()
        执行defaultMQPullConsumer.defaultMQPullConsumerImpl.pullAPIWrapper.setConnectBrokerByUser(true)
        执行filterClassManager.start()
        执行filterServerStatsManager.start()
    public void shutdown()
        执行remotingServer.shutdown()
        执行remotingExecutor.shutdown()
        停止定时任务
        执行defaultMQPullConsumer.shutdown()
        执行filterServerOuterAPI.shutdown()
        执行filterClassManager.shutdown()
        执行filterServerStatsManager.shutdown()

# com.alibaba.rocketmq.filtersrv.filter.FilterClassManager

    // Key: Topic@ConsumerGroup, Value: FilterClassInfo
    private ConcurrentHashMap<String, FilterClassInfo> filterClassTable = new ConcurrentHashMap<String, FilterClassInfo>(128)

    public FilterClassManager(FiltersrvController filtersrvController)
        设置filtersrvController属性等于filtersrvController参数
        设置filterClassFetchMethod属性等于new HttpFilterClassFetchMethod(filtersrvController.filtersrvConfig.filterClassRepertoryUrl)实例              // filterClassRepertoryUrl: http://fsrep.tbsite.net/filterclass
    public void start()
        如果filtersrvController.filtersrvConfig.clientUploadFilterClassEnable等于false
            执行定时任务，延迟1分钟执行，任务执行间隔为1分钟
                执行fetchClassFromRemoteHost()
    // 同步所有最新的FilterClass
    private void fetchClassFromRemoteHost()
        遍历filterClassTable属性，设置entry变量等于当前元素
            try {
                设置filterClassInfo变量等于entry.value
                设置topicAndGroup变量等于entry.key.split("@")
                设置responseStr变量等于filterClassFetchMethod.fetch(topicAndGroup[0], topicAndGroup[1], filterClassInfo.className)
                设置filterSourceBinary变量等于
                设置classCRC变量等于UtilAll.crc32(responseStr.getBytes("UTF-8"))
                如果classCRC变量不等于filterClassInfo.classCRC
                    执行filterClassInfo.setMessageFilter((MessageFilter) (DynaCode.compileAndLoadClass(filterClassInfo.className, responseStr).newInstance()))
                    执行filterClassInfo.setClassCRC(classCRC)
            } catch (Exception e) {
            }
    public void shutdown()
        关闭定时任务
    // 根据配置决定是否支持注册FilterClass，如果配置支持，当不存在或不一致时，重新更新
    public boolean registerFilterClass(String consumerGroup, String topic, String className, int classCRC, byte[] filterSourceBinary)
        设置key变量等于topic + "@" + consumerGroup
        设置registerNew变量等于false
        设置filterClassInfoPrev变量等于filterClassTable.get(key)
        如果filterClassInfoPrev变量等于null
            设置registerNew变量等于true
        否则
            如果filtersrvController.filtersrvConfig.clientUploadFilterClassEnable等于true
                如果filterClassInfoPrev.classCRC != classCRC && classCRC != 0等于true
                    设置registerNew变量等于true
        如果registerNew变量等于true
            synchronized (compileLock)
                设置filterClassInfoPrev变量等于filterClassTable.get(key)
                如果filterClassInfoPrev变量不等于null并且filterClassInfoPrev.classCRC == classCRC等于true
                    返回true
                try {
                    设置filterClassInfoNew变量等于new FilterClassInfo()
                    执行filterClassInfoNew.setClassName(className);
                    执行filterClassInfoNew.setClassCRC(0);
                    执行filterClassInfoNew.setMessageFilter(null);
                    如果filtersrvController.filtersrvConfig.clientUploadFilterClassEnable等于true
                        设置javaSource变量等于new String(filterSourceBinary, "UTF-8")
                        执行filterClassInfoNew.setMessageFilter((MessageFilter) (DynaCode.compileAndLoadClass(className, javaSource).newInstance()))
                        执行filterClassInfoNew.setClassCRC(classCRC)
                    执行filterClassTable.put(key, filterClassInfoNew)
                } catch (Throwable e) {
                    返回false
                }
            }
        返回true
    // 返回FilterClass
    public FilterClassInfo findFilterClass(String consumerGroup, String topic)
        执行filterClassTable.get(consumerGroup + "@" + topic))并返回

# com.alibaba.rocketmq.filtersrv.processor.DefaultRequestProcessor

    public DefaultRequestProcessor(FiltersrvController filtersrvController)
        设置filtersrvController属性等于filtersrvController参数
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws Exception
        如果request.code等于302                                                                                            // REGISTER_MESSAGE_FILTER_CLASS: 302
            执行registerMessageFilterClass(ctx, request)并返回
        否则如果request.code等于11                                                                                          // PULL_MESSAGE: 11
            执行pullMessageForward(ctx, request)并返回
        返回null
    // 接收Consumer发送的FilterClass，注册到filterClassManager中
    private RemotingCommand registerMessageFilterClass(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        解序列化request参数为RegisterMessageFilterClassRequestHeader类型的实例，设置给requestHeader变量
        try {
            设置ok变量等于filtersrvController.filterClassManager.registerFilterClass(requestHeader.consumerGroup, requestHeader.topic, requestHeader.className, requestHeader.classCRC, request.body)
            如果ok变量等于false
                抛异常Exception("registerFilterClass error")
        } catch (Exception e) {
            执行response.setCode(1)                                                                                       // SYSTEM_ERROR: 1
            执行response.setRemark(e.getMessage())
            返回response变量
        }
        执行response.setCode(0)                                                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response变量
    // 用正常PULL API消费消息，当拉取到新消息后，使用filterClass进行服务端过滤，过滤后如果存在元素，发送给客户端，否则，返回重新拉取标识（特点是和BrokerServer部署在同一台机器上，走本地网卡，同时使用BrokerServer所在机器的cpu资源）
    private RemotingCommand pullMessageForward(ChannelHandlerContext ctx, RemotingCommand request) throws Exception
        执行RemotingCommand.createResponseCommand(PullMessageResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        设置responseHeader变量等于response.customHeader
        解序列化request为PullMessageRequestHeader类型的实例，设置给requestHeader变量
        执行response.setOpaque(request.opaque)
        设置pullConsumer变量等于filtersrvController.defaultMQPullConsumer
        设置findFilterClass变量等于filtersrvController.filterClassManager.findFilterClass(requestHeader.consumerGroup, requestHeader.topic)
        如果findFilterClass等于null
            执行response.setCode(1)                                                                       // SYSTEM_ERROR: 1
            执行response.setRemark("Find Filter class failed, not registered")
            返回response变量
        如果findFilterClass.messageFilter等于null
             执行response.setCode(1)                                                                      // SYSTEM_ERROR: 1
             执行response.setRemark("Find Filter class failed, registered but no class")
             返回response变量
        执行responseHeader.setSuggestWhichBrokerId(0)                                                     // MASTER_ID: 0
        设置mq变量等于new MessageQueue()实例
        执行mq.setTopic(requestHeader.topic)
        执行mq.setQueueId(requestHeader.queueId)
        执行mq.setBrokerName(filtersrvController.brokerName)
        设置offset变量等于requestHeader.queueOffset
        设置maxNums变量等于requestHeader.maxMsgNums
        创建new PullCallback()实例，设置给pullCallback变量
            public void onSuccess(PullResult pullResult)
                执行responseHeader.setMaxOffset(pullResult.maxOffset)
                执行responseHeader.setMinOffset(pullResult.minOffset)
                执行responseHeader.setNextBeginOffset(pullResult.nextBeginOffset)
                执行response.setRemark(null)

                如果pullResult.pullStatus等于FOUND
                    执行response.setCode(0)                                       // SUCCESS: 0
                    设置msgListOK变量等于new ArrayList<MessageExt>()实例
                    try {
                        遍历pullResult.msgFoundList，设置msg变量等于当前元素
                        如果findFilterClass.messageFilter.match(msg)等于true
                            执行msgListOK.add(msg)
                        如果msgListOK变量元素个数大于0
                            执行eturnResponse(requestHeader.consumerGroup, requestHeader.topic, ctx, response, msgListOK)
                            退出方法
                        否则
                            执行response.setCode(20)                              // PULL_RETRY_IMMEDIATELY: 20
                    } catch (Throwable e) {
                        执行response.setCode(1)                                                           // SYSTEM_ERROR: 1
                        执行response.setRemark(e.getMessage())
                        执行returnResponse(requestHeader.consumerGroup, requestHeader.topic, ctx, response, null)
                        退出方法
                    }
                否则如果pullResult.pullStatus等于NO_MATCHED_MSG
                    执行response.setCode(20)                                      // PULL_RETRY_IMMEDIATELY: 20
                否则如果pullResult.pullStatus等于NO_NEW_MSG
                    执行response.setCode(19)                                      // PULL_NOT_FOUND: 19
                否则如果pullResult.pullStatus等于OFFSET_ILLEGAL
                    执行response.setCode(21)                                      // PULL_OFFSET_MOVED: 21
                执行returnResponse(requestHeader.consumerGroup, requestHeader.topic, ctx, response, null)
        执行pullConsumer.pullBlockIfNotFound(mq, null, offset, maxNums, pullCallback)
        返回null
    private void returnResponse(String group, String topic, ChannelHandlerContext ctx, RemotingCommand response, List<MessageExt> msgList)
        如果msgList参数不等于null
            序列化msgList为byte[]，设置给body变量
            执行response.setBody(body)
            执行filtersrvController.filterServerStatsManager.incGroupGetNums(group, topic, msgList.size())
            执行filtersrvController.filterServerStatsManager.incGroupGetSize(group, topic, body.length)
        try {
            执行ctx.writeAndFlush(response)
        } catch (Throwable e) {
        }ƒ

# com.alibaba.rocketmq.filtersrv.FilterServerOuterAPIƒ

    public FilterServerOuterAPI()
        设置remotingClient属性等于new NettyRemotingClient(new NettyClientConfig())实例
    public void start()
        执行remotingClient.start()
    public void shutdown()
        执行remotingClient.shutdown()
    // 注册filterServerAddr到brokerServer
    public RegisterFilterServerResponseHeader registerFilterServerToBroker(String brokerAddr, String filterServerAddr) throws RemotingCommandException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException, InterruptedException, MQBrokerException
        设置requestHeader变量等于new RegisterFilterServerRequestHeader()实例
        执行requestHeader.setFilterServerAddr(filterServerAddr)
        执行RemotingCommand.createRequestCommand(301, requestHeader)返回RemotingCommand类型的实例，设置给request变量           // REGISTER_FILTER_SERVER: 301
        执行remotingClient.invokeSync(brokerAddr, request, 3000)返回RemotingCommand类型的实例，设置给response变量
        如果response.code等于0                                                                                             // SUCCESS: 0
            解序列化response变量为RegisterFilterServerResponseHeader类型的实例并返回
        抛异常MQBrokerException(response.code, response.remark)

// client源码和server源码参考其他工程

