# com.alibaba.rocketmq.namesrv.NamesrvStartup

    public static void main(String[] args)
        创建new NamesrvConfig()实例，设置给namesrvConfig变量
        创建new NettyServerConfig()实例，设置给nettyServerConfig变量
        设置nettyServerConfig.listenPort属性等于9876
        如果指定了配置文件
            使用配置文件填充namesrvConfig变量的相关属性
            使用配置文件填充nettyServerConfig变量的相关属性
        使用命令行参数填充namesrvConfig变量的相关属性
        验证namesrvConfig.rocketmqHome设置过，否则程序退出
        创建new NamesrvController(namesrvConfig, nettyServerConfig)实例，设置给controller变量
        执行controller.initialize()
        添加钩子
           执行controller.shutdown()
        执行controller.start()

# com.alibaba.rocketmq.namesrv.NamesrvController

    public NamesrvController(NamesrvConfig namesrvConfig, NettyServerConfig nettyServerConfig)
        设置namesrvConfig属性等于namesrvConfig参数
        设置nettyServerConfig属性等于nettyServerConfig参数
        设置kvConfigManager属性等于new KVConfigManager(this)实例
        设置routeInfoManager属性等于new RouteInfoManager()实例
        设置brokerHousekeepingService属性等于new BrokerHousekeepingService(this)实例
    public boolean initialize()
        执行kvConfigManager.load()
        设置remotingServer属性等于new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService)实例
        使用nettyServerConfig.serverWorkerThreads创建Fix线程池，设置给remotingExecutor属性
        执行remotingServer.registerDefaultProcessor(new DefaultRequestProcessor(this), this.remotingExecutor)         // 注册默认请求处理器和执行该请求器的线程池
        添加定时任务，延迟5秒执行，执行间隔为10秒钟
            执行routeInfoManager.scanNotActiveBroker()
        添加定时任务，延迟5秒执行，执行间隔为10秒钟
            执行kvConfigManager.printAllPeriodically()
    public void start()
        执行remotingServer.start()
    public void shutdown()
        执行remotingServer.shutdown();
        关闭remotingExecutor线程池
        关闭开启的两个定时任务

## com.alibaba.rocketmq.namesrv.routeinfo.BrokerHousekeepingService

    public BrokerHousekeepingService(NamesrvController namesrvController)
        设置namesrvController属性等于namesrvController参数
    public void onChannelClose(String remoteAddr, Channel channel)                                // 连接关闭，调用destory
        执行namesrvController.routeInfoManager.onChannelDestroy(remoteAddr, channel)
    public void onChannelException(String remoteAddr, Channel channel)                            // 出现异常，调用destory
        执行namesrvController.routeInfoManager.onChannelDestroy(remoteAddr, channel)
    public void onChannelIdle(String remoteAddr, Channel channel)                                 // 触发连接空闲，调用destory
        执行namesrvController.routeInfoManager.onChannelDestroy(remoteAddr, channel)

# com.alibaba.rocketmq.remoting.netty.NettyRemotingServer

    protected NettyEventExecuter nettyEventExecuter = new NettyEventExecuter();

    public NettyRemotingServer(NettyServerConfig nettyServerConfig, ChannelEventListener channelEventListener)
        使用nettyServerConfig.serverOnewaySemaphoreValue初始化semaphoreOneway属性                   // 控制onway并发
        使用nettyServerConfig.serverAsyncSemaphoreValue初始化semaphoreAsync属性                     // 控制async并发
        设置nettyServerConfig属性等于nettyServerConfig参数
        设置channelEventListener属性等于channelEventListener参数
        使用nettyServerConfig.serverCallbackExecutorThread创建Fix线程池，设置给publicExecutor属性     // 用于callback回复场景
        初始化eventLoopGroupBoss属性，线程数为1                                                      // netty的boss
        使用nettyServerConfig.serverSelectorThreads初始化eventLoopGroupWorker属性                   // netty的worker
    public void registerDefaultProcessor(NettyRequestProcessor processor, ExecutorService executor)
        创建new Pair(processor, executor)实例，设置给defaultRequestProcessor属性                     // 默认处理器和执行该请求的业务线程池
    public void start()
        设置defaultEventExecutorGroup属性等于new DefaultEventExecutorGroup(nettyServerConfig.serverWorkerThreads)实例
        创建Server
            使用eventLoopGroupBoss作为boss，eventLoopGroupWorker作为worker
            设置SO_KEEPALIVE等于false                                                             // 非长连接
            设置SO_SNDBUF等于nettyServerConfig.serverSocketSndBufSize
            设置SO_RCVBUF等于nettyServerConfig.serverSocketRcvBufSize
            绑定nettyServerConfig.listenPort端口
            添加Handler
                defaultEventExecutorGroup   new NettyEncoder()                                                                  // 使用defaultEventExecutorGroup执行编码器
                defaultEventExecutorGroup   new NettyDecoder()                                                                  // 使用defaultEventExecutorGroup执行解码器
                defaultEventExecutorGroup   new new IdleStateHandler(0, 0, nettyServerConfig.serverChannelMaxIdleTimeSeconds)   // 使用defaultEventExecutorGroup执行，指定时间内未发生读操作或写操作，触发ALL_IDLE的IdleStateEvent
                defaultEventExecutorGroup   new NettyConnetManageHandler()                                                      // 使用defaultEventExecutorGroup执行
                defaultEventExecutorGroup   new NettyServerHandler()                                                            // 使用defaultEventExecutorGroup执行
            如果nettyServerConfig.serverPooledByteBufAllocatorEnable等于true
                设置ALLOCATOR等于PooledByteBufAllocator.DEFAULT                                    // 池化内部默认申请DirectyMemory，可以通过系统参数控制申请HeapMemory
        如果channelEventListener不等于null
            执行nettyEventExecuter.start()
    public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception
        如果msg参数不等于null
            如果msg.type等于REQUEST_COMMAND                     // nameserver不主动发出请求，因此没有RESPONSE_COMMAND需要处理
                执行processRequestCommand(ctx, cmd)                                                                             // 处理请求// 处理响应
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
                    执行pair.getObject1().processRequest(ctx, cmd)返回RemotingCommand类型的实例，设置给response变量
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
    public void shutdown()
        关闭eventLoopGroupBoss属性和eventLoopGroupWorker属性
        执行nettyEventExecuter.shutdown()
        关闭defaultEventExecutorGroup属性
        关闭publicExecutor属性

## com.alibaba.rocketmq.remoting.netty.NettyRemotingServer.NettyConnetManageHandler

    public void channelActive(ChannelHandlerContext ctx) throws Exception
        如果channelEventListener属性不等于null，执行nettyEventExecuter.putNettyEvent(new NettyEvent(NettyEventType.CONNECT, remoteAddress .toString(), ctx.channel()))
    public void channelInactive(ChannelHandlerContext ctx) throws Exception
        如果channelEventListener属性不等于null，执行nettyEventExecuter.putNettyEvent(new NettyEvent(NettyEventType.CLOSE, remoteAddress .toString(), ctx.channel()))
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception
        如果evt参数是IdleStateEvent类型
            如果evt.state()等于ALL_IDLE                                                                                         // 超时关闭连接
                执行channel.close()
                如果channelEventListener属性不等于null，执行nettyEventExecuter.putNettyEvent(new NettyEvent(NettyEventType.IDLE, remoteAddress .toString(), ctx.channel()))
        执行ctx.fireUserEventTriggered(evt)
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception                                  // 错误关闭连接
        如果channelEventListener属性不等于null，执行nettyEventExecuter.putNettyEvent(new NettyEvent(NettyEventType.EXCEPTION, remoteAddress .toString(), ctx.channel()))
        执行channel.close()

## com.alibaba.rocketmq.remoting.netty.NettyRemotingAbstract.NettyEventExecuter

    public void putNettyEvent(NettyEvent event)
        如果eventQueue属性大小超过10000，丢弃事件，否则添加事件到eventQueue属性
    public void run()
        while (!this.isStoped()) {                                                                                            // 以下几类事件都是在单独的线程中执行的
            try {
                获取eventQueue中的事件
                    如果事件不等于null并且channelEventListener属性不等于null
                        如果事件类型等于IDLE
                            执行channelEventListener.onChannelIdle(event.getRemoteAddr(), event.getChannel())
                        如果事件类型等于CLOSE
                            执行channelEventListener.onChannelClose(event.getRemoteAddr(), event.getChannel())
                        如果事件类型等于CONNECT
                            执行channelEventListener.onChannelConnect(event.getRemoteAddr(), event.getChannel())
                        如果事件类型等于EXCEPTION
                            执行channelEventListener.onChannelException(event.getRemoteAddr(), event.getChannel())
            } catch (Exception e) {
            }
        }

## com.alibaba.rocketmq.remoting.netty.NettyRemotingServer.NettyServerHandler

    protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception
        执行processMessageReceived(ctx, msg)                                                                                  // 执行command

# com.alibaba.rocketmq.namesrv.processor.DefaultRequestProcessor

    public DefaultRequestProcessor(NamesrvController namesrvController)
        设置namesrvController属性等于namesrvController参数
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        如果request.code为100                                      // PUT_KV_CONFIG
            解码request为PutKVConfigRequestHeader                  // String namespace;     String key;     String value
            执行namesrvController.kvConfigManager.putKVConfig(header.namespace, header.key, header.value)
            设置response.code为0并返回                              // SUCCESS
        如果request.code为101                                      // GET_KV_CONFIG
            解码request为GetKVConfigRequestHeader                  // String namespace;     String key
            执行namesrvController.kvConfigManager.getKVConfig(header.namespace, header.key)返回value
            如果value不等于null
                填充到response.header中，设置response.code为0并返回   // SUCCESS
            否则设置response.code为22并返回                          // QUERY_NOT_FOUND
        如果request.code为102                                      // DELETE_KV_CONFIG
            解码request为DeleteKVConfigRequestHeader               // String namespace;     String key
            执行namesrvController.kvConfigManager.deleteKVConfig(header.namespace, header.key)返回value
            设置response.code为0并返回                              // SUCCESS
        如果request.code为103                                      // REGISTER_BROKER
            解码request为RegisterBrokerRequestHeader               // String brokerName;    String brokerAddr;      String clusterName;     String haServerAddr;        Long brokerId
            如果request.body不等于null
                解码为RegisterBrokerBody并设置给registerBrokerBody
            否则
                创建new RegisterBrokerBody()实例设置给registerBrokerBody
                执行registerBrokerBody.topicConfigSerializeWrapper.dataVersion.setCounter(new AtomicLong(0))
                执行registerBrokerBody.topicConfigSerializeWrapper.dataVersion.setTimestatmp(0)
            // 注册并返回masterId、haServerAddr、顺序主题列表给请求方
            执行namesrvController.routeInfoManager.registerBroker(header.clusterName, header.brokerAddr, header.brokerName, header.brokerId, header.haServerAddr, registerBrokerBody.topicConfigSerializeWrapper, registerBrokerBody.filterServerList, ctx.channel())返回result
            执行responseHeader.setHaServerAddr(result.haServerAddr)
            执行responseHeader.setMasterAddr(result.masterAddr)
            设置response.body等于namesrvController.kvConfigManager().getKVListByNamespace("ORDER_TOPIC_CONFIG")
            设置response.code为0并返回                              // SUCCESS
        如果request.code为104                                      // UNREGISTER_BROKER
            解码request为UnRegisterBrokerRequestHeader             // String brokerName;    String brokerAddr;      String clusterName;     Long brokerId
            // 解注册
            执行namesrvController.routeInfoManager.unregisterBroker(header.clusterName, header.brokerAddr, header.brokerName, header.brokerId)
            设置response.code为0并返回                              // SUCCESS
        如果request.code为105                                      // GET_ROUTEINTO_BY_TOPIC
            解码request为RegisterBrokerRequestHeader               // String topic
            执行namesrvController.routeInfoManager.pickupTopicRouteData(header.topic)返回topicRouteData
            如果topicRouteData等于null
                设置response.code为17并返回                         // TOPIC_NOT_EXIST
            否则
                // 返回包含QueueData列表、BrokerData列表、filterServerTable信息的主题配置信息，或者可能只包含当前topic的orderTopicConf信息的主题配置信息
                设置topicRouteData.orderTopicConf等于namesrvController.kvConfigManager.getKVConfig("ORDER_TOPIC_CONFIG", header.topic)
                解码topicRouteData作为response.body
                设置response.code为0并返回                          // SUCCESS
        如果request.code为106                                      // GET_BROKER_CLUSTER_INFO
            设置response.body等于namesrvController.routeInfoManager.getAllClusterInfo()
            设置response.code为0并返回                              // SUCCESS
        如果request.code为205                                      // WIPE_WRITE_PERM_OF_BROKER
            解码request为WipeWritePermOfBrokerRequestHeader        // String brokerName
            设置response.wipeTopicCount等于namesrvController.routeInfoManager.wipeWritePermOfBrokerByLock(header.brokerName)
            设置response.code为0并返回                              // SUCCESS
        如果request.code为206                                      // GET_ALL_TOPIC_LIST_FROM_NAMESERVER
            设置response.body等于namesrvController.routeInfoManager.getAllTopicList()
            设置response.code为0并返回                              // SUCCESS
        如果request.code为216                                      // DELETE_TOPIC_IN_NAMESRV
            解码request为DeleteTopicInNamesrvRequestHeader         // String topic
            执行namesrvController.routeInfoManager().deleteTopic(header.topic)
            设置response.code为0并返回                              // SUCCESS
        如果request.code为217                                      // GET_KV_CONFIG_BY_VALUE
            解码request为GetKVConfigRequestHeader                  // String namespace;     String key
            执行namesrvController.kvConfigManager.getKVConfigByValue(header.namespace, header.key)返回value
            如果value等于null
                设置response.code为22并返回                         // QUERY_NOT_FOUND
            否则
                设置response.header等于value
                设置response.code为0并返回                          // SUCCESS
        如果request.code为218                                      // DELETE_KV_CONFIG_BY_VALUE
            解码request为DeleteKVConfigRequestHeader               // String namespace;     String key
            执行namesrvController.kvConfigManager.deleteKVConfigByValue(header.namespace, header.key)
            设置response.code为0并返回                              // SUCCESS
        如果request.code为219                                      // GET_KVLIST_BY_NAMESPACE
            解码request为GetKVListByNamespaceRequestHeader         // String namespace
            执行namesrvController.kvConfigManager.getKVListByNamespace(header.namespace)返回jsonValue
            如果jsonValue等于null
                设置response.code为22并返回                         // QUERY_NOT_FOUND
            否则
                设置response.header等于jsonValue
                设置response.code为0并返回                          // SUCCESS
        如果request.code为224                                      // GET_TOPICS_BY_CLUSTER
            解码request为GetTopicsByClusterRequestHeader           // String cluster
            设置response.body等于namesrvController.routeInfoManager().topicsByCluster(header.cluster)
            设置response.code为0并返回                              // SUCCESS
        如果request.code为304                                      // GET_SYSTEM_TOPIC_LIST_FROM_NS
            设置response.body等于namesrvController.routeInfoManager().getSystemTopicList()
            设置response.code为0并返回                              // SUCCESS
        如果request.code为311                                      // GET_UNIT_TOPIC_LIST
            设置response.body等于namesrvController.routeInfoManager().getUnitTopics()
            设置response.code为0并返回                              // SUCCESS
        如果request.code为312                                      // GET_HAS_UNIT_SUB_TOPIC_LIST
            设置response.body等于namesrvController.routeInfoManager().getHasUnitSubTopicList()
            设置response.code为0并返回                              // SUCCESS
        如果request.code为313                                      // GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST
            设置response.body等于namesrvController.routeInfoManager().getHasUnitSubUnUnitTopicList()
            设置response.code为0并返回                              // SUCCESS

# com.alibaba.rocketmq.namesrv.kvconfig.KVConfigManager

    private HashMap<String/* Namespace */, HashMap<String/* Key */, String/* Value */>> configTable = new HashMap<String, HashMap<String, String>>();

    public KVConfigManager(NamesrvController namesrvController)
        设置namesrvController属性等于namesrvController参数
    public void load()
        如果namesrvController.namesrvConfig.kvConfigPath属性不等于null
            读取其对应的文件内容，反序列化为KVConfigSerializeWrapper类型的实例，设置给kvConfigSerializeWrapper变量
            如果kvConfigSerializeWrapper变量不等于null
                执行configTable.putAll(kvConfigSerializeWrapper.configTable);
    public void printAllPeriodically()
        打印configTable中所有K/V信息到日志
    public void putKVConfig(String namespace, String key, String value)
        写入namespace参数、key参数、value参数到configTable属性中，如果存在，则覆盖
        执行persist()方法
    public void persist()
        创建new KVConfigSerializeWrapper()实例，设置给kvConfigSerializeWrapper变量
        执行kvConfigSerializeWrapper.setConfigTable(this.configTable)
        序列化kvConfigSerializeWrapper，如果内容不等于null，写入到namesrvController.namesrvConfig.kvConfigPath对应的文件
    public String getKVConfig(String namespace, String key)                               // 获取namespace下key的配置，比如namespace等于ORDER_TOPIC_CONFIG，key等于主题名称
        返回configTable属性中指定namespace下key对应的value
    public void deleteKVConfig(String namespace, String key)
        删除configTable属性中指定namespace下的key
        执行persist()方法
    public void deleteKVConfigByValue(String namespace, String value)
        删除configTable属性中指定namespace下的HashMap中的value等于此value的key
        执行persist()方法
    public byte[] getKVListByNamespace(String namespace)
        返回configTable属性中指定namespace下的HashMap序列化后的byte[]
    public String getKVConfigByValue(String namespace, String value)
        返回configTable属性中指定namespace下的HashMap中的value等于此value的key，key和key之间使用;分隔

# com.alibaba.rocketmq.namesrv.routeinfo.RouteInfoManager
    // BrokerAddr信息，BrokerLiveInfo主要包括客户端channel、对应的haServerAddr、创建时间、主题配置快照版本
    private HashMap<String, BrokerLiveInfo> brokerLiveTable;                              // Key: brokerAddr
    // filterServer信息，每个brokerAddr下可能包含多个FilterServer
    private HashMap<String, List<String>> filterServerTable                               // Key: brokerAddr      Value: Filter Server
    // brokerName信息，相同brokerName下存在master/slave
    private HashMap<String, BrokerData> brokerAddrTable;                                  // Key: brokerName      Value: HashMap<BrokerId, BrokerAddress>
    // 集群信息
    private HashMap<String, Set<String>> clusterAddrTable;                                // Key: clusterName     Value: brokerNames
    // 主题配置信息，QueueData包含brokerName，brokername的读写消息队列个数、主题权限、系统标识
    private HashMap<String, List<QueueData>> topicQueueTable;                             // Key: Topic

    public void scanNotActiveBroker()                                                           // 扫描已过期的broker，删除元素并关闭其channel，维护RouteInfo内存结构
        遍历brokerLiveTable属性，设置entry等于当前元素
            如果当前时间和entry.value.lastUpdateTimestamp的时间差大于2分钟
                执行entry.value.channel.close()
                移除当前元素
                执行onChannelDestroy(entry.key, entry.value.channel)
    public void onChannelDestroy(String remoteAddr, Channel channel)                            // 维护完整的删除结构
        如果brokerLiveTable属性中，存在entry.value.channel等于channel参数，设置brokerAddrFound变量等于entry.key
        如果brokerAddrFound变量等于null
            设置brokerAddrFound变量等于removeAddr
        如果brokerAddrFound变量不等于null
            执行brokerLiveTable.remove(brokerAddrFound)
            执行filterServerTable.remove(brokerAddrFound)
            遍历brokerAddrTable属性，设置entry等于当前元素
                如果entry.value.brokerAddrs中存在brokerAddrFound，则从entry.value.brokerAddrs中删除，设置brokerNameFound变量等于entry.value.brokerName
                如果entry.value.brokerAddrs元素个数为0，删除entry，设置removeBrokerName变量等于true
            如果brokerNameFound变量不等于null并且removeBrokerName变量等于true
                遍历clusterAddrTable属性，设置entry等于当前元素
                    如果entry.value中存在brokerNameFound，从entry.value中移除brokerNameFound
                    如果entry.value元素个数为0，删除entry
            如果removeBrokerName变量等于true
                遍历topicQueueTable属性，设置entry等于当前元素
                    如果entry.value中存在brokerNameFound，从entry.value中移除brokerNameFound
                    如果entry.value元素个数为0，删除entry
    public RegisterBrokerResult registerBroker(String clusterName, String brokerAddr, String brokerName, long brokerId, String haServerAddr, TopicConfigSerializeWrapper topicConfigWrapper, List<String> filterServerList, Channel channel)
        创建new RegisterBrokerResult()实例，设置给result变量
        添加clusterName变量和brokerName变量到clusterAddrTable属性中
        添加brokerName变量、brokerId变量、brokerAddr变量到brokerAddrTable属性中，如果brokerId是第一次添加，设置registerFirst变量等于true
        // 如果当前broker是master，
        如果topicConfigWrapper不等于null并且brokerId等于0            // MASTER_ID: 0
            如果isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.dataVersion)等于true或者registerFirst变量等于true
            // 如果快照版本发生变更或者是第一次添加
            如果topicConfigWrapper.topicConfigTable元素个数大于0，进行遍历，设置entry等于当前元素
                执行createAndUpdateQueueData(brokerName, entry.value)
        添加brokerAddr变量、new BrokerLiveInfo(System.currentTimeMillis(), topicConfigWrapper.dataVersion(), channel, haServerAddr)到brokerLiveTable属性中
        如果filterServerList变量不等于null
            // 如果为0，则删除，否则添加
            如果filterServerList变量元素个数等于0，执行filterServerTable.remove(brokerAddr)
            否则，执行filterServerTable.put(brokerAddr, filterServerList)
        如果brokerId不等于0                                        // MASTER_ID: 0
            // 对于非master的broker请求，获取master地址和master对应的ha地址给slave
            设置masterAddr变量等于brokerAddrTable.get(brokerName).brokerAddrs.get(0)
            如果masterAddr变量不等于null
                设置brokerLiveInfo变量等于brokerLiveTable.get(masterAddr)
                如果brokerLiveInfo不等于null
                    执行result.setHaServerAddr(brokerLiveInfo.haServerAddr)
                    执行result.setMasterAddr(masterAddr)
        返回result变量
    private boolean isBrokerTopicConfigChanged(String brokerAddr, DataVersion dataVersion)                              // 判断快照版本是否发生变化
        设置prev等于brokerLiveTable.get(brokerAddr)
        如果prev等于null或者prev.dataVersion不等于dataVersion参数
            返回true
        返回false
    // 如果topic下不存在对应的brokerName信息或者存在，但是配置不等，则添加或更新
    private void createAndUpdateQueueData(String brokerName, TopicConfig topicConfig)
        通过brokerName、topicConfig.writeQueueNums、topicConfig.readQueueNums、topicConfig.perm、topicConfig.topicSysFlag创建QueueData类型的实例，设置给queueData变量
        设置queueDataList变量等于topicQueueTable.get(topicConfig.topicName)
        如果queueDataList变量等于null
            创建new LinkedList<QueueData>()实例，设置给queueDataList变量，添加queueData变量到queueDataList变量中
            添加topicConfig.topicName和实例到topicQueueTable属性中
        否则
            遍历queueDataList参数，设置entry等于当前元素
                如果entry.brokerName等于brokerName
                    如果entry和queueData变量不等
                        移除entry
                    否则
                        设置addNewOne变量等于true
            如果addNewOne变量等于true
                添加queueData变量到queueDataList变量中
    public void unregisterBroker(String clusterName, String brokerAddr, String brokerName, long brokerId)
        执行brokerLiveTable.remove(brokerAddr)                                            // 从BrokerAddr信息中移除
        执行filterServerTable.remove(brokerAddr)                                          // 从filterServer信息中移除
        执行brokerAddrTable.get(brokerName).brokerAddrs.remove(brokerId)                  // 从brokerName对应的列表中移除brokerId
        如果brokerAddrTable.get(brokerName).brokerAddrs元素个数等于0                        // 如果brokerName对应的列表为空，删除brokerName信息
            执行brokerAddrTable.remove(brokerName)，设置removeBrokerName变量等于true
        如果removeBrokerName变量等于true
            执行clusterAddrTable.get(clusterName).remove(brokerName)                      // 如果删除了brokerName，继续维护集群信息
            如果clusterAddrTable.get(clusterName)元素个数等于0
                执行clusterAddrTable.remove(clusterName)
            执行removeTopicByBrokerName(brokerName)
    private void removeTopicByBrokerName(String brokerName)                              // 如果删除了brokerName，遍历所有topic配置，删除对应brokerName信息
        遍历topicQueueTable属性，设置entry等于当前元素
            如果entry.value中存在brokerName，从entry.value中移除brokerName
            如果entry.value元素个数为0，删除entry
    public TopicRouteData pickupTopicRouteData(String topic)
        设置topicRouteData变量等于new TopicRouteData()实例
        设置queueDataList变量等于topicQueueTable.get(topic)
        如果queueDataList变量不等于null
            添加queueDataList变量到topicRouteData变量中                                     // 整合topic配置，包括brokerId和brokerAddr列表，以及brokerAddr和和filterServer列表
            整理queueDataList变量中，所有的brokerName，并进行遍历
                设置brokerData变量等于brokerAddrTable.get(brokerName)
                如果brokerData变量不等于null
                    添加brokerData变量到topicRouteData变量中
                    设置foundBrokerData变量等于true
                    遍历brokerData.brokerAddrs，设置entry等于当前元素，添加filterServerTable.get(entry.value)到topicRouteData变量中
        如果foundBrokerData变量等于true
            返回topicRouteData变量
        返回null
    public byte[] getAllClusterInfo()
        创建new ClusterInfo()实例，设置给clusterInfoSerializeWrapper变量
        添加brokerAddrTable、clusterAddrTable到clusterInfoSerializeWrapper变量中
        序列化clusterInfoSerializeWrapper变量并返回
    public int wipeWritePermOfBrokerByLock(String brokerName)
        遍历topicQueueTable属性，设置entry等于当前元素
            如果entry.value中存在brokerName，移除对应的write权限，设置wipeTopicCnt++
        返回wipeTopicCnt
    public byte[] getAllTopicList()
        创建new TopicList()实例，设置给topicList变量
        添加topicQueueTable.keySet()到topicList变量中
        序列化topicList变量并返回
    public void deleteTopic(String topic)
        执行topicQueueTable.remove(topic)
    public byte[] getTopicsByCluster(String cluster)
        创建new TopicList()实例，设置给topicList变量
        设置brokerNameSet变量等于clusterAddrTable.get(cluster)
        遍历brokerNameSet变量，设置brokerName等于当前元素
            遍历topicQueueTable属性，设置entry等于当前元素
                如果entry.value中存在brokerName，添加entry.key到topicList变量中
        序列化topicList变量并返回
    public byte[] getSystemTopicList()
        创建new TopicList()实例，设置给topicList变量
        遍历clusterAddrTable属性，设置entry等于当前元素
            添加entry.key到topicList变量中                ？这是clusterName
            添加entry.value到topicList变量中
        获取brokerAddrTable属性中，第一个元素的value.brokerAddrs中第一个value，添加到topicList变量中
        序列化topicList变量并返回
    public byte[] getUnitTopics()
        创建new TopicList()实例，设置给topicList变量
        遍历clusterAddrTable属性，设置entry等于当前元素
            如果entry.value元素个数大于0，并且第0个元素的topicSynFlag包含1，添加entry.key到topicList变量中
        序列化topicList变量并返回
    public byte[] getHasUnitSubTopicList()
        创建new TopicList()实例，设置给topicList变量
        遍历clusterAddrTable属性，设置entry等于当前元素
            如果entry.value元素个数大于0，并且第0个元素的topicSynFlag包含2，添加entry.key到topicList变量中
        序列化topicList变量并返回
    public byte[] getHasUnitSubUnUnitTopicList()
        创建new TopicList()实例，设置给topicList变量
        遍历clusterAddrTable属性，设置entry等于当前元素
            如果entry.value元素个数大于0，并且第0个元素的topicSynFlag不包含1但是包含2，添加entry.key到topicList变量中
        序列化topicList变量并返回
        