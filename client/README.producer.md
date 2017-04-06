# com.alibaba.rocketmq.client.producer.DefaultMQProducer
    public void setInstanceName(String instanceName)                                        // 默认值: System.getProperty("rocketmq.client.name", "DEFAULT")
    public void setNamesrvAddr(String namesrvAddr)                                          // 默认值: System.getProperty("rocketmq.namesrv.addr", System.getenv("NAMESRV_ADDR"))
    public void setSendMsgTimeout(int sendMsgTimeout)                                       // 默认值: 3000
    public void setMaxMessageSize(int maxMessageSize)                                       // 默认值: 1024 * 128
    public void setCompressMsgBodyOverHowmuch(int compressMsgBodyOverHowmuch)               // 默认值: 1024 * 4
    public void setRetryTimesWhenSendFailed(int retryTimesWhenSendFailed)                   // 默认值: 2，发送出现异常的重试次数
    public void setRetryAnotherBrokerWhenNotStoreOK(boolean retryAnotherBrokerWhenNotStoreOK)       // 默认值: false，同步发送返回非SEND_OK情况下，是否继续发送给其他broker
    public void setClientCallbackExecutorThreads(int clientCallbackExecutorThreads)         // 默认值: Runtime.getRuntime().availableProcessors()

    public void changeInstanceNameToPID()                                                   // 如果是DEFAULT，设置instanceName为进程PID，后边需要根据ip + instanceName确定MQClientManager
        如果instanceName属性等于DEFAULT
            设置instanceName属性等于当前进程PID
    public void resetClientConfig(ClientConfig cc)
        重置namesrvAddr、clientIP、instanceName、clientCallbackExecutorThreads、pollNameServerInteval、heartbeatBrokerInterval属性

    public DefaultMQProducer(String producerGroup)
        执行this(producerGroup, null)
    public DefaultMQProducer(String producerGroup, RPCHook rpcHook)
        设置producerGroup属性等于producerGroup参数
        设置defaultMQProducerImpl属性等于new DefaultMQProducerImpl(this, rpcHook)实例
    public void start() throws MQClientException
        执行defaultMQProducerImpl.start()
    public void shutdown()
        执行defaultMQProducerImpl.shutdown()
    public SendResult send(Message msg)
        执行defaultMQProducerImpl.send(msg)并返回
    public void send(Message msg, SendCallback sendCallback)
        执行defaultMQProducerImpl.send(msg, sendCallback)并返回
    public void sendOneway(Message msg) throws MQClientException, RemotingException, InterruptedException
        执行defaultMQProducerImpl.sendOneway(msg)
    public SendResult send(Message msg, MessageQueueSelector selector, Object arg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException
        执行defaultMQProducerImpl.send(msg, selector, arg)并返回
    public void createTopic(String key, String newTopic, int queueNum, int topicSysFlag)
        执行defaultMQProducerImpl.createTopic(key, newTopic, queueNum, topicSysFlag)
    public MessageExt viewMessage(String msgId) throws RemotingException, MQBrokerException, InterruptedException, MQClientException
        执行defaultMQProducerImpl.viewMessage(msgId)并返回

# com.alibaba.rocketmq.client.producer.TransactionMQProducer

    public TransactionMQProducer(String producerGroup)
        执行super(producerGroup)
    public TransactionMQProducer(String producerGroup, RPCHook rpcHook)
        执行super(producerGroup,rpcHook)
    public TransactionSendResult sendMessageInTransaction(Message msg, LocalTransactionExecuter tranExecuter, Object arg) throws MQClientException
        执行defaultMQProducerImpl.sendMessageInTransaction(msg, tranExecuter, arg)并返回

# com.alibaba.rocketmq.client.impl.producer.DefaultMQProducerImpl
    private ServiceState serviceState = ServiceState.CREATE_JUST;
    // Key: Topic, Value: TopicPublishInfo
    // TopicPublishInfo主要包含可写的队列（发送主题和内置主题），以及非select场景下选择队列的标识
    private ConcurrentHashMap<String, TopicPublishInfo> topicPublishInfoTable = new ConcurrentHashMap<String, TopicPublishInfo>()
    private ArrayList<SendMessageHook> sendMessageHookList = new ArrayList<SendMessageHook>()
    private ArrayList<CheckForbiddenHook> checkForbiddenHookList = new ArrayList<CheckForbiddenHook>()

    public DefaultMQProducerImpl(DefaultMQProducer defaultMQProducer, RPCHook rpcHook)
        设置defaultMQProducer属性等于defaultMQProducer参数
        设置rpcHook属性等于rpcHook参数
    public void start() throws MQClientException
        执行start(true)
    public void start(boolean startFactory) throws MQClientException
        如果serviceState属性等于CREATE_JUST
            设置serviceState属性等于START_FAILED
            验证defaultMQProducer.producerGroup不能等于null；必须由字符、数字、-、_构成；长度不能超过255个字符；不能等于DEFAULT_PRODUCER，否则抛异常MQClientException("producerGroup...")
            如果defaultMQProducer.producerGroup不等于CLIENT_INNER_PRODUCER                                                        // CLIENT_INNER_PRODUCER为内部Consumer使用
                执行defaultMQProducer.changeInstanceNameToPID()
            设置mQClientFactory属性等于MQClientManager.getInstance().getAndCreateMQClientInstance(defaultMQProducer, rpcHook)     // 根据IP@instanceName生成Key确保MQClientManager实例唯一
            执行mQClientFactory.registerProducer(defaultMQProducer.producerGroup, this)返回设置给registerOK变量
            如果registerOK变量等于false
                设置serviceState属性等于CREATE_JUST，抛异常MQClientException("The producer group...")
            执行topicPublishInfoTable.put("TBW102", new TopicPublishInfo())                                                      // 添加内置主题，后续会重新更新其TopicPublishInfo
            如果startFactory参数等于true
                执行mQClientFactory.start()
            设置serviceState属性等于RUNNING
        否则如果serviceState属性等于RUNNING、START_FAILED、SHUTDOWN_ALREADY
            抛异常MQClientException("The producer service state not OK...")
        执行mQClientFactory.sendHeartbeatToAllBrokerWithLock()                                                                   // 先同步执行一次心跳，不过updateTopicRouteInfoFromNameServer在下边是异步的，此处可能什么都更新不到
    public void shutdown()
        执行shutdown(true)
    public void shutdown(boolean shutdownFactory)
        如果serviceState属性等于RUNNING
            执行mQClientFactory.unregisterProducer(defaultMQProducer.producerGroup)
            如果shutdownFactory参数等于true
                执行mQClientFactory.shutdown()
            设置serviceState属性等于SHUTDOWN_ALREADY
    // 事物消息强调了可靠性（2次请求保证），在确保消息落盘成功且主从完全可用情况下（写入的是prepare消息，consumer只消费非事务型和提交型事物消息），根据传递的事务执行器决定是否是否回滚，其他情况一律回滚
    public TransactionSendResult sendMessageInTransaction(Message msg, LocalTransactionExecuter tranExecuter, Object arg) throws MQClientException
        如果tranExecuter参数等于null
            抛异常MQClientException("tranExecutor is null", null)
        校验msg参数不能等于null；msg.body不能等于null；msg.body.length不能等于0；msg.body.length必须小于defaultMQProducer.maxMessageSize，否则抛异常MQClientException("the message...")
        添加TRAN_MSG和true到msg参数中
        添加PGROUP和defaultMQProducer.producerGroup到msg参数中
        try {
            设置sendResult变量等于send(msg);
        } catch (Exception e) {
            抛异常MQClientException("send message Exception", e);
        }
        如果sendResult.sendStatus变量等于FLUSH_DISK_TIMEOUT或FLUSH_SLAVE_TIMEOUT或SLAVE_NOT_AVAILABLE
            设置localTransactionState变量等于ROLLBACK_MESSAGE
        否则如果sendResult.sendStatus变量等于SEND_OK
            try {
                如果sendResult.transactionId不等于null
                    添加__transactionId__和sendResult.transactionId到msg参数中                                                 // brokerServer目前没有支持执行sendResult.setTransactionId
                设置localTransactionState变量等于tranExecuter.executeLocalTransactionBranch(msg, arg)
                如果localTransactionState变量等于null，设置localTransactionState变量等于UNKNOW
            } catch (Throwable e) {
                设置localException变量等于e
            }
        否则
            设置localTransactionState变量等于UNKNOW
        try {
            设置transactionId变量等于sendResult.transactionId
            设置brokerAddr变量等于mQClientFactory.findBrokerAddressInPublish(sendResult.messageQueue.brokerName)
            解序列化sendResult.msgId为MessageId类型的实例，设置给id变量
            根据transactionId、id.offset、localTransactionState、defaultMQProducer.producerGroup、sendResult.queueOffset、sendResult.msgId创建EndTransactionRequestHeader类型的实例，设置给requestHeader变量
            执行mQClientFactory.mQClientAPIImpl.endTransactionOneway(brokerAddr, requestHeader, localException != null ? localException.message : null, defaultMQProducer.sendMsgTimeout)
        } catch (Exception e) { }
        根据sendResult变量和localTransactionState变量创建TransactionSendResult类型的实例并返回
    public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException
        执行sendDefaultImpl(msg, CommunicationMode.SYNC, null, defaultMQProducer.sendMsgTimeout)并返回
    public void send(Message msg, SendCallback sendCallback) throws MQClientException, RemotingException, InterruptedException
        try {
            执行sendDefaultImpl(msg, CommunicationMode.ASYNC, sendCallback, defaultMQProducer.sendMsgTimeout);
        } catch (MQBrokerException e) {
            throw new MQClientException("unknown exception", e);
        }
    public void sendOneway(Message msg) throws MQClientException, RemotingException, InterruptedException
        try {
            执行sendDefaultImpl(msg, CommunicationMode.ONEWAY, null, defaultMQProducer.sendMsgTimeout);
        } catch (MQBrokerException e) {
            throw new MQClientException("unknow exception", e);
        }
    public SendResult send(Message msg, MessageQueueSelector selector, Object arg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException
        执行sendSelectImpl(msg, selector, arg, CommunicationMode.SYNC, null, defaultMQProducer.sendMsgTimeout)
    public void createTopic(String key, String newTopic, int queueNum, int topicSysFlag) throws MQClientException
        如果serviceState不等于RUNNING
            抛异常MQClientException("The producer service state not OK...")
        校验newTopic不能等于null；必须由字符、数字、-、_构成、长度不能超过255个字符；不能等于TBW102，否则抛异常MQClientException("the specified topic...")
        执行mQClientFactory.mQAdminImpl.createTopic(key, newTopic, queueNum, topicSysFlag)
    public MessageExt viewMessage(String msgId) throws RemotingException, MQBrokerException, InterruptedException, MQClientException
        如果serviceState不等于RUNNING
            抛异常MQClientException("The producer service state not OK...")
        执行mQClientFactory.mQAdminImpl.viewMessage(msgId)并返回

    private TopicPublishInfo tryToFindTopicPublishInfo(String topic)
        设置topicPublishInfo变量等于topicPublishInfoTable.get(topic)
        如果topicPublishInfo等于null或者topicPublishInfo.ok()等于false
            执行topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo())                                              // 先加个位置，然后在执行更新
            执行mQClientFactory.updateTopicRouteInfoFromNameServer(topic)
            设置topicPublishInfo变量等于topicPublishInfoTable.get(topic)
        如果topicPublishInfo.isHaveTopicRouterInfo()等于true或者topicPublishInfo不等于null并且topicPublishInfo.ok()等于true      // 返回正常的
            返回topicPublishInfo变量
        否则
            执行mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, defaultMQProducer)                           // 获取内置主题的信息来填充topic的信息
            设置topicPublishInfo变量等于topicPublishInfoTable.get(topic)
            返回topicPublishInfo变量
    public Set<String> getPublishTopicList()                                                            // 获取当前进程（除了当前实例，还有其他实例）发送过消息的主题和内置主题
        设置topicList变量等于new HashSet<String>()实例
        遍历topicPublishInfoTable属性，设置entry变量等于当前元素
            执行topicList.add(entry.key)
        返回topicList
    public boolean isPublishTopicNeedUpdate(String topic)                                               // 主题是否不存在或主题队列为空都需要更新
        设置prev变量等于topicPublishInfoTable.get(topic)
        如果prev变量等于null或者prev.ok()等于false，返回true，否则返回false
    public void updateTopicPublishInfo(String topic, TopicPublishInfo info)                             // 所有发生变化的都更新
        如果info参数不等于null并且topic参数不等于null
            设置prev变量等于topicPublishInfoTable.put(topic, info)
            如果prev变量不等于null
                执行info.sendWhichQueue.set(prev.sendWhichQueue.get())

    private SendResult sendSelectImpl(Message msg, MessageQueueSelector selector, Object arg, CommunicationMode communicationMode, SendCallback sendCallback, long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException
        如果serviceState不等于RUNNING
            抛异常MQClientException("The producer service state not OK...")
        校验msg参数不能等于null；msg.body不能等于null；msg.body.length不能等于0；msg.body.length必须小于defaultMQProducer.maxMessageSize，否则抛异常MQClientException("the message...")
        执行tryToFindTopicPublishInfo(msg.topic)，返回TopicPublishInfo类型的实例，设置给topicPublishInfo变量
        如果topicPublishInfo变量不等于null并且topicPublishInfo.ok()等于true
            MessageQueue mq = null;
            try {
                // 按select选择一个消息队列
                设置mq变量等于selector.select(topicPublishInfo.messageQueueList, msg, arg)
            } catch (Throwable e) {
                throw new MQClientException("select message queue throwed exception.", e);
            }
            如果mq不等于null
                执行sendKernelImpl(msg, mq, communicationMode, sendCallback, timeout)并返回
            否则
                抛异常MQClientException("select message queue return null.")
        抛异常MQClientException("select message queue return null.")
    private SendResult sendDefaultImpl(Message msg, CommunicationMode communicationMode, SendCallback sendCallback, long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException
        如果serviceState不等于RUNNING
            抛异常MQClientException("The producer service state not OK...")
        校验msg参数不能等于null；msg.body不能等于null；msg.body.length不能等于0；msg.body.length必须小于defaultMQProducer.maxMessageSize，否则抛异常MQClientException("the message...")
        设置endTimestamp变量和beginTimestamp变量等于System.currentTimeMillis()
        执行tryToFindTopicPublishInfo(msg.topic)，返回TopicPublishInfo类型的实例，设置给topicPublishInfo变量
        如果topicPublishInfo变量不等于null并且topicPublishInfo.ok()等于true
            MessageQueue mq = null;
            Exception exception = null;
            SendResult sendResult = null;
            // 一共尝试tryTimes + 1次，总超时时间也延长1秒钟
            String[] brokersSent = new String[1 + defaultMQProducer.retryTimesWhenSendFailed]
            for (int times = 0; times < 1 + defaultMQProducer.retryTimesWhenSendFailed && (endTimestamp - beginTimestamp) < defaultMQProducer.sendMsgTimeout + 1000; times++)
                如果mq变量等于null，设置lastBrokerName变量等于null，否则设置lastBrokerName变量等于mq.brokerName
                执行topicPublishInfo.selectOneMessageQueue(lastBrokerName)返回MessageQueue类型的实例，设置给tmpmq变量                  // 这里是一个递增（轮询）方式的选择
                如果tmpmq变量不等于null
                    设置mq变量等于tmpmq变量
                    设置brokersSent[times]等于mq.brokerName
                    try {
                        设置sendResult变量等于sendKernelImpl(msg, mq, communicationMode, sendCallback, timeout)
                        设置endTimestamp变量等于System.currentTimeMillis()
                        如果communicationMode参数等于ASYNC或ONEWAY                                                                  // 异步或Oneway也可能出现异常
                            返回null
                        否则如果communicationMode参数等于等于SYNC
                            如果sendResult.sendStatus不等于SendStatus.SEND_OK
                                如果defaultMQProducer.retryAnotherBrokerWhenNotStoreOK不等于true                                   // 也可能重新尝试，有可能产生2条相同消息
                                    返回sendResult变量
                    } catch (RemotingException e) {
                        设置exception变量等于e参数
                        设置endTimestamp变量等于System.currentTimeMillis()
                    } catch (MQClientException e) {
                        设置exception变量等于e参数
                        设置endTimestamp变量等于System.currentTimeMillis()
                    } catch (MQBrokerException e) {
                        设置exception变量等于e参数
                        设置endTimestamp变量等于System.currentTimeMillis()
                        如果e.responseCode不等于17和14和1和16和204和205  // TOPIC_NOT_EXIST: 17    SERVICE_NOT_AVAILABLE: 14    SYSTEM_ERROR: 1    NO_PERMISSION: 16    NO_BUYER_ID: 204    NOT_IN_CURRENT_UNIT: 205
                            如果sendResult变量不等于null                                                                           // 如果不等于null，则是上一次执行的结果
                                返回sendResult变量
                            否则抛异常e
                    } catch (InterruptedException e) {
                        抛异常e
                    }
                否则
                    退出循环
            如果sendResult变量不等于null
                返回sendResult变量
            抛异常MQClientException(exception)
        设置nsList变量等于mQClientFactory.mQClientAPIImpl.getNameServerAddressList()
        如果nsList变量等于null或者元素个数等于0
            抛异常MQClientException("No name server address, please set it...")
        抛异常MQClientException("No route info of this topic...")
    private SendResult sendKernelImpl(Message msg, MessageQueue mq, CommunicationMode communicationMode, SendCallback sendCallback, long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException
        设置brokerAddr变量等于mQClientFactory.findBrokerAddressInPublish(mq.brokerName)
        如果brokerAddr变量等于null
            执行tryToFindTopicPublishInfo(mq.topic)
            设置brokerAddr变量等于mQClientFactory.findBrokerAddressInPublish(mq.brokerName)
        SendMessageContext context = null
        如果brokerAddr变量不等于null
            设置prevBody变量等于msg.body
            try {
                int sysFlag = 0
                如果msg.body.length大于等于defaultMQProducer.compressMsgBodyOverHowmuch
                    try {
                        设置msg.body等于UtilAll.compress(msg.body, Integer.parseInt(System.getProperty("rocketmq.message.compressLevel", "5")))     // 内部不是用snappy，批量或量大的时候可以考虑
                        执行sysFlag |= 1
                    } catch (Throwable e) {
                    }
                如果msg.properties.get("TRAN_MSG")不等于null并且等于true
                    执行sysFlag |= 4                                                          // TransactionPreparedType: 4
                如果checkForbiddenHookList属性元素个数大于0
                    创建new CheckForbiddenContext()实例，设置给checkForbiddenContext变量
                    checkForbiddenContext.setNameSrvAddr(defaultMQProducer.namesrvAddr);
                    checkForbiddenContext.setGroup(defaultMQProducer.producerGroup);
                    checkForbiddenContext.setCommunicationMode(communicationMode);
                    checkForbiddenContext.setBrokerAddr(brokerAddr);
                    checkForbiddenContext.setMessage(msg);
                    checkForbiddenContext.setMq(mq);
                    checkForbiddenContext.setUnitMode(defaultMQProducer.unitMode);
                    executeCheckForbiddenHook(checkForbiddenContext);                                                                               // 可以抛异常
                如果sendMessageHookList属性元素个数不等于0
                    创建new SendMessageContext()实例，设置给context变量
                    context.setProducerGroup(defaultMQProducer.producerGroup);
                    context.setCommunicationMode(communicationMode);
                    context.setBornHost(defaultMQProducer.clientIP);
                    context.setBrokerAddr(brokerAddr);
                    context.setMessage(msg);
                    context.setMq(mq);
                    executeSendMessageHookBefore(context);
                创建new SendMessageRequestHeader()实例，设置给requestHeader变量
                requestHeader.setProducerGroup(defaultMQProducer.producerGroup);
                requestHeader.setTopic(msg.topic);
                requestHeader.setDefaultTopic(defaultMQProducer.createTopicKey);
                requestHeader.setDefaultTopicQueueNums(defaultMQProducer.defaultTopicQueueNums);
                requestHeader.setQueueId(mq.queueId);
                requestHeader.setSysFlag(sysFlag);
                requestHeader.setBornTimestamp(System.currentTimeMillis());
                requestHeader.setFlag(msg.flag);
                requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.properties));
                requestHeader.setReconsumeTimes(0);
                requestHeader.setUnitMode(defaultMQProducer.unitMode);
                如果requestHeader.topic以%RETRY%开头                                                                                                 // 消费失败的消息，是通过发送到%RETRY%类主题的方式来重新消费
                    设置reconsumeTimes变量等于msg.properties.get("RECONSUME_TIME")
                    如果reconsumeTimes变量不等于null
                        requestHeader.setReconsumeTimes(new Integer(reconsumeTimes))
                        执行msg.clearProperty("RECONSUME_TIME")
                执行mQClientFactory.mQClientAPIImpl.sendMessage(brokerAddr, mq.brokerName, msg, requestHeader,timeout, communicationMode, sendCallback)返回SendResult类型的实例，设置给sendResult变量
                如果sendMessageHookList属性元素个数不等于0
                    执行context.setSendResult(sendResult)
                    执行executeSendMessageHookAfter(context)
                返回sendResult变量
            } catch (RemotingException e) {
                如果sendMessageHookList属性元素个数不等于0
                    执行context.setException(e)
                    执行executeSendMessageHookAfter(context)
                抛异常e
            } catch (MQBrokerException e) {
                如果sendMessageHookList属性元素个数不等于0
                    执行context.setException(e)
                    执行executeSendMessageHookAfter(context)
                抛异常e
            } catch (InterruptedException e) {
                如果sendMessageHookList属性元素个数不等于0
                    执行context.setException(e)
                    执行executeSendMessageHookAfter(context)
                抛异常e
            } fianlly {
                执行msg.setBody(prevBody)
            }
        抛异常MQClientException("The broker...")
    public void executeCheckForbiddenHook(CheckForbiddenContext context) throws MQClientException
        如果checkForbiddenHookList属性元素个数不等于0，遍历checkForbiddenHookList属性，设置hook等于当前元素
            执行hook.checkForbidden(context)
    public void executeSendMessageHookBefore(SendMessageContext context)
        如果sendMessageHookList属性元素个数不等于0，遍历sendMessageHookList属性，设置hook等于当前元素
            try {
                执行hook.sendMessageBefore(context);
            } catch (Throwable e) { }
    public void executeSendMessageHookAfter(SendMessageContext context)
        如果sendMessageHookList属性元素个数不等于0，遍历sendMessageHookList属性，设置hook等于当前元素
            try {
                执行hook.sendMessageAfter(context);
            } catch (Throwable e) { }

# com.alibaba.rocketmq.client.impl.factory.MQClientInstance
    private ConcurrentHashMap<String, MQProducerInner> producerTable = new ConcurrentHashMap<String, MQProducerInner>()                         // Key: group, Value: DefaultMQProducerImpl
    // Key: Topic       Value: QueueData列表、BrokerData列表、filterServerTable信息，或者可能只包含orderTopicConf信息
    // 记录了所有发送过消息的主题和内置主题信息
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
        设置mQClientAPIImpl属性等于new MQClientAPIImpl(this.nettyClientConfig, null, rpcHook)实例
        如果clientConfig.namesrvAddr不等于null                                                       // 如果指定了namesevers，用;分隔成列表设置nameserver
            执行mQClientAPIImpl.updateNameServerAddressList(clientConfig.namesrvAddr)
        设置clientId属性等于clientId参数
        设置mQAdminImpl属性等于new MQAdminImpl(this)实例
    public boolean registerProducer(String group, DefaultMQProducerImpl producer)                  // 注册group，用于检测心跳和获取所有发布过的主题及内置主题来维护topicRouteTable和brokerAddrTable属性
        如果group等于null或者producer等于null
            返回false
        执行producerTable.putIfAbsent(group, producer)，设置返回值给prev变量
        如果prev变量不等于null
            返回false
        返回true
    public void start() throws MQClientException
        如果serviceState属性等于CREATE_JUST
            设置serviceState属性等于START_FAILED
            如果clientConfig.namesrvAddr等于null                                                    // 如果没有手动指定nameservers，先通过http拉取nameservers并设置
                执行clientConfig.setNamesrvAddr(mQClientAPIImpl.fetchNameServerAddr())
            执行mQClientAPIImpl.start()
            如果clientConfig.namesrvAddr等于null                                                    // 如果没有拉取到，执行定时任务，每隔120秒拉取重新拉取一次
                启动定时任务，延迟10秒钟执行，任务间隔为120秒
                    try {
                        执行mQClientAPIImpl.fetchNameServerAddr()
                    } catch (Exception e) { }
            执行定时任务，延迟10毫秒执行，任务间隔为clientConfig.pollNameServerInteval毫秒               // 每隔一段时间，需要获取所有主题（发布过消息和内置的），用来维护topicRouteTable和brokerAddrTable属性，以及topicPublishInfoTable
                执行updateTopicRouteInfoFromNameServer()
            执行定时任务，延迟1000毫秒执行，任务间隔为clientConfig.heartbeatBrokerInterval毫秒           // 每隔一段时间，重新整理一遍brokerAddrTable，然后和其中所有的BrokerAddr做心跳
                try {
                    执行cleanOfflineBroker()
                    执行sendHeartbeatToAllBrokerWithLock()
                } catch (Exception e) { }
            设置serviceState属性等于RUNNING
        否则如果serviceState属性等于START_FAILED
            抛异常MQClientException("The Factory object...")
    private void cleanOfflineBroker()                                                             // 重新整理一遍brokerAddrTable，保证每个brokerAddr确实被某个topic使用
        try {
            if (lockNamesrv.tryLock(3000, TimeUnit.MILLISECONDS))
                try {
                    遍历brokerAddrTable属性，设置entry变量等于当前元素                                 // entry: Key(brokerName), Value(Map<BrokerId, BrokerAddress>)
                        遍历entry.value，设置subEntry变量等于当前元素
                            如果isBrokerAddrExistInTopicRouteTable(subEntry.value)等于false
                                移除当前元素
                        如果entry.value元素个数等于空
                            移除当前元素
                } finally {
                    lockNamesrv.unlock();
                }
        } catch (InterruptedException e) { }
    private boolean isBrokerAddrExistInTopicRouteTable(String addr)                               // 检测指定brokerAddress是否被topicRouteTable中某个topic使用
        遍历topicRouteTable属性，设置entry变量等于当前元素                                             // entry: Key: Topic, Value: QueueData列表、BrokerData列表、filterServerTable信息，或者可能只包含orderTopicConf信息
            遍历entry.value.brokerDatas，设置subEntry变量等于当前元素
                如果subEntry.brokerAddrs不等于null并且包含addr参数，返回true
        返回false
    public void sendHeartbeatToAllBrokerWithLock()                                                // 和brokerAddrTable中所有BrokerAddr做心跳
        如果lockHeartbeat.tryLock()等于true
            try {
                执行sendHeartbeatToAllBroker()
            } catch (Exception e) {
            } finally {
                lockHeartbeat.unlock()
            }
    private void sendHeartbeatToAllBroker()
        创建new HeartbeatData()实例，设置给heartbeatData变量
        执行heartbeatData.setClientID(clientId)
        遍历producerTable属性，设置entry变量等于当前元素                                               // entry: Key: Group, Value: DefaultMQProducerImpl
            如果entry.value不等于null
                ProducerData producerData = new ProducerData();
                producerData.setGroupName(entry.key);
                heartbeatData.producerDataSet.add(producerData);
        如果heartbeatData.producerDataSet.isEmpty()等于false
            遍历brokerAddrTable属性，设置entry变量等于当前元素                                         // entry: Key(brokerName), Value(Map<BrokerId, BrokerAddress>)
                如果entry.value不等于null
                    遍历entry.value，设置subEntry变量等于当前元素
                        如果subEntry.value不等于null
                            如果subEntry.key不等于0                                                // MASTER_ID: 0
                                continue
                            try {
                                mQClientAPIImpl.sendHearbeat(subEntry.value, heartbeatData, 3000);
                            } catch (Exception e) {
                            }
    public void unregisterProducer(String group)                                                 // 对brokerAddrTable中所有的BrokerAddr取消注册
        执行producerTable.remove(group)
        try {
            如果lockHeartbeat.tryLock(3000, TimeUnit.MILLISECONDS)等于true
                try {
                    执行unregisterClient(group, null);
                } catch (Exception e) {
                } finally {
                    lockHeartbeat.unlock();
                }
        } catch (InterruptedException e) { }
    private void unregisterClient(String producerGroup, String consumerGroup)
        遍历brokerAddrTable属性，设置entry等于当前元素                                                // entry: Key(brokerName), Value(Map<BrokerId, BrokerAddress>)
            如果entry.value不等于null，遍历entry.value，设置subEntry等于当前元素
                如果subEntry.value不等于null
                    try {
                        mQClientAPIImpl.unregisterClient(subEntry.value, clientId, producerGroup, consumerGroup, 3000);
                    } catch (RemotingException e) {
                    } catch (MQBrokerException e) {
                    } catch (InterruptedException e) {
                    }
    public void shutdown()
        如果producerTable属性元素个数大于1，退出
        如果serviceState属性等于RUNNING
            设置serviceState属性等于SHUTDOWN_ALREADY
            关闭定时任务
            执行mQClientAPIImpl.shutdown()
            执行MQClientManager.getInstance().removeClientFactory(clientId)
    public void updateTopicRouteInfoFromNameServer()                                             // 更新所有发送过消息的主题和内置主题
        设置topicList变量等于new HashSet<String>()实例
        遍历producerTable属性，设置entry变量等于当前元素                                              // entry: Key: Group, Value: DefaultMQProducerImpl
            如果entry.value不等于null
                执行topicList.addAll(entry.value.getPublishTopicList())                           // 添加DefaultMQProducerImpl所有发送过消息的主题和内置主题列表
        遍历topicList变量，设置entry变量等于当前元素
            执行updateTopicRouteInfoFromNameServer(entry)
    public boolean updateTopicRouteInfoFromNameServer(String topic)
        执行updateTopicRouteInfoFromNameServer(topic, false, null)并返回
    public boolean updateTopicRouteInfoFromNameServer(String topic, boolean isDefault, DefaultMQProducer defaultMQProducer)
        try {
            if (lockNamesrv.tryLock(3000, TimeUnit.MILLISECONDS)) {
                try {
                    如果isDefault参数等于true并且defaultMQProducer参数不等于null
                        设置topicRouteData变量等于mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer("TBW102", 1000 * 3)      // 随机选中一台nameserver，获取TBW102主题TopicRouteData信息，相当于获取默认配置
                        如果topicRouteData变量不等于null
                            遍历topicRouteData.queueDatas，设置data等于当前元素
                            设置queueNums等于Math.min(defaultMQProducer.defaultTopicQueueNums, data.readQueueNums)
                            执行data.setReadQueueNums(queueNums)
                            执行data.setWriteQueueNums(queueNums)
                    否则
                        设置topicRouteData变量等于mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3)                // 随机选中一台nameserver，获取TopicRouteData信息
                    如果topicRouteData变量不等于null
                        设置changed变量等于topicRouteDataIsChange(topicRouteTable.get(topic), topicRouteData)                     // 检测queue列表和broker列表是否相等
                        如果changed变量等于false
                            设置changed变量等于isNeedUpdateTopicRouteInfo(topic)                                                  // 检测主题是否不存在或主题队列为空
                        如果changed变量等于true
                            遍历topicRouteData.brokerDatas，设置bd变量等于当前元素
                                执行brokerAddrTable.put(bd.brokerName, bd.brokerAddrs)                                          // 维护brokerAddrTable
                            设置publishInfo变量等于topicRouteData2TopicPublishInfo(topic, topicRouteData)
                            遍历producerTable属性，设置entry变量等于当前元素
                                如果entry.value不等于null
                                    执行entry.value.updateTopicPublishInfo(topic, publishInfo)                                  // 更新主题和主题可写的队列关系
                            执行topicRouteTable.put(topic, topicRouteData)                                                      // 维护topicRouteTable
                            返回true
                } catch (Exception e) {
                } finally {
                    lockNamesrv.unlock();
                }
            }
        } catch (InterruptedException e) { }
        返回false
    private boolean topicRouteDataIsChange(TopicRouteData olddata, TopicRouteData nowdata)      // 比较对应的queueDatas和brokerDatas属性
        如果olddata等于null或者nowdata等于null
            返回true
        设置old变量等于olddata.cloneTopicRouteData()
        设置now变量等于nowdata.cloneTopicRouteData()
        对old.queueDatas进行排序
        对old.brokerDatas进行排序
        对now.queueDatas进行排序
        对now.brokerDatas进行排序
        如果old等于now，返回true，否则返回false
    private boolean isNeedUpdateTopicRouteInfo(String topic)                                    // 如果这个主题不存在或主题的队列为空，则需要更新
        遍历producerTable属性，设置entry变量等于当前元素
            如果entry.value不等于null并且entry.value.isPublishTopicNeedUpdate(topic)等于true
                返回true
        返回false
    public static TopicPublishInfo topicRouteData2TopicPublishInfo(String topic, TopicRouteData route)      // 将route转换为TopicPublishInfo，主要是填充可写的MessageQueue
        设置info变量等于new TopicPublishInfo()实例
        如果route.orderTopicConf不等于null并且route.orderTopicConf长度大于0                        // orderTopicConf: (brokerName1:maxQueueId);(brokerName2:maxQueueId);(brokerName3:maxQueueId)
            遍历route.orderTopicConf.split(";")，设置broker变量等于当前元素
                设置item变量等于broker.split(":")
                循环item[1]次，设置i变量等于当前循环标识
                    执行info.messageQueueList.add(new MessageQueue(topic, item[0], i))
            执行info.setOrderTopic(true)
        否则
            对route.queueDatas进行排序和遍历，设置qd变量等于当前元素
                如果qd.perm有写权限
                    设置brokerData变量等于null
                    遍历route.brokerDatas，设置bd变量等于当前元素
                        如果bd.brokerName等于qd.brokerName
                            设置brokerData变量等于bd
                            退出循环
                如果brokerData不等于null并且brokerData.brokerAddrs中包含0                         // MASTER_ID: 0
                    循环qd.writeQueueNums次，设置i变量等于当前循环标识
                        info.messageQueueList.add(new MessageQueue(topic, qd.brokerName, i))
            执行info.setOrderTopic(false)
        返回info变量
    public String findBrokerAddressInPublish(String brokerName)                                // 返回brokerName对应的masterId的brokerAdddr
        设置map变量等于brokerAddrTable.get(brokerName)                                           // map: Map<BrokerId, BrokerAddress>
        如果map变量不等于null并且元素个数大于0
            返回map.get(0)                    // MASTER_ID: 0
        返回null

# com.alibaba.rocketmq.client.impl.MQAdminImpl

    public MQAdminImpl(MQClientInstance mQClientFactory)
        设置mQClientFactory属性等于mQClientFactory参数
    // 根据nameserver获取指定key的TopicRouteData信息，包括QueueData列表、BrokerData列表、filterServerTable信息，或者可能只包含orderTopicConf信息，然后遍历BrokerData列表，对所有的Broker发起创建主题的请求
    // 顺序的意思是读和写一次对应，非顺序表示的是一个写队列，可以对应多个读队列
    // 可能出现对某个Broker创建失败，其他创建成功，这种情况，最终还是抛出异常
    // 这里的key起到的是一个默认主题或内部公共主题的作用，只是为了能够获取对应的BrokerData列表
    public void createTopic(String key, String newTopic, int queueNum, int topicSysFlag) throws MQClientException
        try {
            执行mQClientFactory.mQClientAPIImpl.getTopicRouteInfoFromNameServer(key, 1000 * 3)返回TopicRouteData类型的实例，设置给topicRouteData变量
            如果topicRouteData.brokerDatas不等于null并且元素个数大于0
                执行Collections.sort(topicRouteData.brokerDatas)
                MQClientException exception = null;
                遍历topicRouteData.brokerDatas，设置brokerData变量等于当前元素
                    设置addr变量等于brokerData.brokerAddrs.get(0)             // MASTER_ID: 0
                    如果addr变量不等于null
                        创建new TopicConfig(newTopic)实例，设置给topicConfig变量
                        topicConfig.setReadQueueNums(queueNum)
                        topicConfig.setWriteQueueNums(queueNum)
                        topicConfig.setTopicSysFlag(topicSysFlag)
                        try {
                            mQClientFactory.mQClientAPIImpl.createTopic(addr, key, topicConfig, 1000 * 3);
                        } catch (Exception e) {
                            设置exception变量等于new MQClientException("create topic to broker exception", e)实例
                        }
                如果exception变量不等于null
                    跑异常exception
            否则
                抛异常MQClientException("Not found broker, maybe key is wrong", null)
        } catch (Exception e) {
            抛异常new MQClientException("create new topic failed", e)
        }
    // 解码msgId，根据其offset获取发送过的消息
    public MessageExt viewMessage(String msgId) throws RemotingException, MQBrokerException, InterruptedException, MQClientException
        try {
            执行MessageDecoder.decodeMessageId(msgId)返回MessageId类型的实例，设置给messageId变量
            执行mQClientFactory.mQClientAPIImpl.viewMessage(RemotingUtil.socketAddress2String(messageId.address), messageId.offset, 1000 * 3)并返回
        } catch (UnknownHostException e) {
            throw new MQClientException("message id illegal", e);
        }

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
        执行remotingClient.registerRPCHook(rpcHook)
    public void start()
        执行remotingClient.start()
        获取本地地址，设置给localAddress变量
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

    // 对指定brokerAddr发送SEND_MESSAGE_V2请求，超时时间为timeoutMillis毫秒
    // 发送消息包括同步、异步、Oneway三种方式
    public SendResult sendMessage(String addr, String brokerName, Message msg, SendMessageRequestHeader requestHeader, long timeoutMillis, CommunicationMode communicationMode, SendCallback sendCallback) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            执行msg.setTopic(VirtualEnvUtil.buildWithProjectGroup(msg.topic, projectGroupPrefix))
            执行requestHeader.setProducerGroup(VirtualEnvUtil.buildWithProjectGroup(requestHeader.producerGroup, projectGroupPrefix))
            执行requestHeader.setTopic(VirtualEnvUtil.buildWithProjectGroup(requestHeader.topic, projectGroupPrefix))
        设置requestHeaderV2变量等于SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader)
        设置request变量等于RemotingCommand.createRequestCommand(310, requestHeaderV2)                   // SEND_MESSAGE_V2: 310
        执行request.setBody(msg.body)
        如果communicationMode参数等于ONEWAY
            执行remotingClient.invokeOneway(addr, request, timeoutMillis)
        否则如果communicationMode参数等于ASYNC
            执行sendMessageAsync(addr, brokerName, msg, timeoutMillis, request, sendCallback)
            返回null
        否则如果communicationMode参数等于SYNC
            执行sendMessageSync(addr, brokerName, msg, timeoutMillis, request)并返回
        返回null
    private SendResult sendMessageSync(String addr, String brokerName, Message msg, long timeoutMillis, RemotingCommand request) throws RemotingException, MQBrokerException, InterruptedException
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        执行processSendResponse(brokerName, msg, response)并返回
    private void sendMessageAsync(String addr, String brokerName, Message msg, long timeoutMillis, RemotingCommand request, SendCallback sendCallback) throws RemotingException, InterruptedException
        创建new InvokeCallback()类型的实例，设置给callback变量
            public void operationComplete(ResponseFuture responseFuture)
                如果sendCallback变量等于null，退出方法
                设置response变量等于responseFuture.responseCommand
                如果response变量不等于null
                    try {
                        sendCallback.onSuccess(processSendResponse(brokerName, msg, response));
                    } catch (Exception e) {
                        sendCallback.onException(e);
                    }
                否则
                    如果responseFuture.isSendRequestOK()等于false
                        执行sendCallback.onException(new MQClientException("send request failed", responseFuture.cause))
                    否则如果responseFuture.isTimeout()等于true
                        执行sendCallback.onException(new MQClientException("wait response timeout "+ responseFuture.timeoutMillis + "ms", responseFuture.cause))
                    否则
                        执行sendCallback.onException(new MQClientException("unknow reseaon", responseFuture.cause))
        执行remotingClient.invokeAsync(addr, request, timeoutMillis, callback)
    private SendResult processSendResponse(String brokerName, Message msg, RemotingCommand response) throws MQBrokerException, RemotingCommandException
        如果response.code等于10或者12或者11或者0                       // FLUSH_DISK_TIMEOUT: 10   FLUSH_SLAVE_TIMEOUT: 12     SLAVE_NOT_AVAILABLE: 11     SUCCESS: 0
            如果response.code等于10
                设置sendStatus变量等于FLUSH_DISK_TIMEOUT
            否则如果response.code等于12
                设置sendStatus变量等于FLUSH_SLAVE_TIMEOUT
            否则如果response.code等于11
                设置sendStatus变量等于SLAVE_NOT_AVAILABLE
            否则如果response.code等于0
                设置sendStatus变量等于SEND_OK
            解序列化response变量为SendMessageResponseHeader类型的实例，设置给responseHeader
            创建new MessageQueue(msg.topic, brokerName, responseHeader.queueId)实例，设置给messageQueue变量
            设置sendResult变量等于new SendResult(sendStatus, responseHeader.msgId, messageQueue, responseHeader.queueOffset, projectGroupPrefix)      // 记录消息的msgId、brokerName、queueId、queueOffset
            执行sendResult.setTransactionId(responseHeader.transactionId)
            返回sendResult
    // 以Oneway的方式，确认事物消息最终的状态（提交还是回滚）
    public void endTransactionOneway(String addr, EndTransactionRequestHeader requestHeader, String remark, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            执行requestHeader.setProducerGroup(VirtualEnvUtil.buildWithProjectGroup(requestHeader.producerGroup, projectGroupPrefix))
        设置request变量等于RemotingCommand.createRequestCommand(37, null)                             // END_TRANSACTION: 37
        执行request.setRemark(remark)
        执行remotingClient.invokeOneway(addr, request, timeoutMillis)
    // 对指定brokerAddr发送同步UPDATE_AND_CREATE_TOPIC请求，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    // 创建主题为topicConfig.topicName，defaultTopic在服务端没起作用，主题配置主要包括读和写的队列个数，主题权限（可读、可写等）、是否是顺序主题等，是否是顺序主题，在服务端创建的时候，也不会起作用，而是通过同步nameserver顺序主题后，才进行设置
    public void createTopic(String addr, String defaultTopic, TopicConfig topicConfig, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException, MQClientException
        如果projectGroupPrefix属性不等于null
            设置topicWithProjectGroup变量等于VirtualEnvUtil.buildWithProjectGroup(topicConfig.topicName, projectGroupPrefix)
        否则
            设置topicWithProjectGroup变量等于topicConfig.topicName
        根据topicWithProjectGroup变量、defaultTopic参数、topicConfig.readQueueNums、topicConfig.writeQueueNums、topicConfig.perm、topicConfig.topicFilterType、topicConfig.topicSysFlag、topicConfig.order创建new CreateTopicRequestHeader()实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(17, requestHeader)                    // UPDATE_AND_CREATE_TOPIC: 17
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            退出方法
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送同步VIEW_MESSAGE_BY_ID请求，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    // 根据发送消息返回的queueOffset进行查看
    public MessageExt viewMessage(String addr, long phyoffset, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        根据phyoffset参数创建new ViewMessageRequestHeader()实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(33, requestHeader)                    // VIEW_MESSAGE_BY_ID: 33
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            解序列化response.body为MessageExt类型的实例，设置给messageExt变量
            如果projectGroupPrefix属性不等于null
                执行messageExt.setTopic(VirtualEnvUtil.clearProjectGroup(messageExt.topic, projectGroupPrefix))
            返回messageExt变量
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送同步UNREGISTER_CLIENT请求，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public void unregisterClient(String addr, String clientID, String producerGroup, String consumerGroup, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            设置producerGroupWithProjectGroup变量等于VirtualEnvUtil.buildWithProjectGroup(producerGroup, projectGroupPrefix)
        否则
            设置producerGroupWithProjectGroup变量等于producerGroup参数
        根据clientID参数、producerGroupWithProjectGroup变量创建new UnregisterClientRequestHeader()实例，设置给requestHeader变量
        设置request变量等于RemotingCommand.createRequestCommand(35, requestHeader)                    // UNREGISTER_CLIENT: 35
        执行remotingClient.invokeSync(addr, request, timeoutMillis)返回RemotingCommand类型的实例，设置给response
        如果response.code等于0                                                                       // SUCCESS: 0
            退出方法
        抛异常MQClientException(response.code, response.remark)
    // 对指定brokerAddr发送同步HEART_BEAT请求，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public void sendHearbeat(String addr, HeartbeatData heartbeatData, long timeoutMillis) throws RemotingException, MQBrokerException, InterruptedException
        如果projectGroupPrefix属性不等于null
            遍历heartbeatData.producerDataSet，设置producerData变量等于当前元素
                执行producerData.setGroupName(VirtualEnvUtil.buildWithProjectGroup(producerData.groupName, projectGroupPrefix))
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
    // 接收传输过来的请求，用对应的处理器来执行（server直接调用我）
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
    // 处理请求后得到的响应，填充ResponseFuture（我先调用server，然后server执行writeAndFlush）
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
        while (!isStoped()) {                                                                                            // 以下几类事件都是在单独的线程中执行的
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
        