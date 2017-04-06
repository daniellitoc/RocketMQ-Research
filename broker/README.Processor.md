# com.alibaba.rocketmq.broker.processor.SendMessageProcessor
    private List<ConsumeMessageHook> consumeMessageHookList                 // 消费每条消息会回调
    private List<SendMessageHook> sendMessageHookList                       // 发送每条消息会回调

    public SendMessageProcessor(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
        设置storeHost属性等于new InetSocketAddress(brokerController.brokerConfig.brokerIP1, brokerController.nettyServerConfig.listenPort)
    public void registerSendMessageHook(List<SendMessageHook> sendMessageHookList)
        设置sendMessageHookList属性等于sendMessageHookList参数
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        如果request.code等于36                                                                 // CONSUMER_SEND_MSG_BACK: 36
            执行consumerSendMsgBack(ctx, request)并返回
        否则                                                                                  // SEND_MESSAGE_V2: 310
            解序列化request为SendMessageRequestHeaderV2类型的实例，设置给requestHeader变量
            如果requestHeader变量等于null
                返回null
            如果sendMessageHookList属性不等于null并且元素个数大于0                                // 消息轨迹：可以记录到达broker的消息
                设置mqtraceContext变量等于new SendMessageContext()实例
                执行mqtraceContext.setProducerGroup(requestHeader.producerGroup)
                执行mqtraceContext.setTopic(requestHeader.topic)
                执行mqtraceContext.setMsgProps(requestHeader.properties)
                执行mqtraceContext.setBornHost(RemotingHelper.parseChannelRemoteAddr(ctx.channel()))
                执行mqtraceContext.setBrokerAddr(brokerController.getBrokerAddr())
            否则
                设置mqtraceContext变量等于null
            执行executeSendMessageHookBefore(ctx, request, mqtraceContext)
            执行sendMessage(ctx, request, mqtraceContext, requestHeader)返回RemotingCommand类型的实例，设置给response
            执行executeSendMessageHookAfter(response, mqtraceContext)                         // 消息轨迹：可以记录发送成功的消息
            返回response
    public void executeSendMessageHookBefore(ChannelHandlerContext ctx, RemotingCommand request, SendMessageContext context)
        如果sendMessageHookList属性不等于null并且元素个数大于0
            遍历sendMessageHookList属性，设置hook变量等于当前元素
                try {
                    解序列化request为SendMessageRequestHeader类型的实例，设置给requestHeader变量
                    执行context.setProducerGroup(requestHeader.producerGroup)
                    执行context.setTopic(requestHeader.topic)
                    执行context.setBodyLength(request.body.length)
                    执行context.setMsgProps(requestHeader.properties)
                    执行context.setBornHost(RemotingHelper.parseChannelRemoteAddr(ctx.channel()))
                    执行context.setBrokerAddr(brokerController.getBrokerAddr())
                    执行context.setQueueId(requestHeader.queueId)
                    执行hook.sendMessageBefore(context)
                    执行requestHeader.setProperties(context.msgProps)
                } catch (Throwable e) {
                }
    public void executeSendMessageHookAfter(RemotingCommand response, SendMessageContext context)
        如果sendMessageHookList属性不等于null并且元素个数大于0
            遍历sendMessageHookList属性，设置hook变量等于当前元素
                try {
                    如果response不等于null
                        设置responseHeader等于response.readCustomHeader()
                        context.setMsgId(responseHeader.msgId)
                        context.setQueueId(responseHeader.queueId)
                        context.setQueueOffset(responseHeader.queueOffset)
                        context.setCode(response.code)
                        context.setErrorMsg(response.remark)
                    执行hook.sendMessageAfter(context)
                } catch (Throwable e) {
                }
    // 发送消息到存储
    private RemotingCommand sendMessage(ChannelHandlerContext ctx, RemotingCommand request, SendMessageContext mqtraceContext, SendMessageRequestHeader requestHeader) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(SendMessageResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        设置responseHeader变量等于response.readCustomHeader()
        执行response.setOpaque(request.opaque)
        执行response.setCode(-1)
        执行msgCheck(ctx, requestHeader, response)
        如果response.code不等于-1
            返回response变量
        设置body变量等于request.body
        设置queueIdInt变量等于requestHeader.queueId
        设置topicConfig变量等于brokerController.topicConfigManager.selectTopicConfig(requestHeader.topic)
        如果queueIdInt小于0                                                                     // 如果小于0，随机分配一个写队列ID
            设置queueIdInt等于Math.abs(random.nextInt() % 99999999) % topicConfig.writeQueueNums
        设置sysFlag变量等于requestHeader.sysFlag
        如果topicConfig.topicFilterType等于MULTI_TAG                                            // 多标签需要设置标识
            设置sysFlag变量等于sysFlag | 2                                                      // MultiTagsFlag: 2
        设置msgInner变量等于new MessageExtBrokerInner()实例
        执行msgInner.setTopic(requestHeader.topic)
        执行msgInner.setBody(body)
        执行msgInner.setFlag(requestHeader.flag)
        执行msgInner.setProperties(MessageDecoder.string2messageProperties(requestHeader.properties))
        执行msgInner.setPropertiesString(requestHeader.properties)
        执行msgInner.setTagsCode(MessageExtBrokerInner.tagsString2tagsCode(topicConfig.topicFilterType, msgInner.properties.get("TAGS")))
        执行msgInner.setQueueId(queueIdInt)
        执行msgInner.setSysFlag(sysFlag)
        执行msgInner.setBornTimestamp(requestHeader.bornTimestamp)
        执行msgInner.setBornHost(ctx.channel().remoteAddress())
        执行msgInner.setStoreHost(storeHost)
        执行msgInner.setReconsumeTimes(requestHeader.reconsumeTimes == null ? 0 : requestHeader.reconsumeTimes)
        如果brokerController.brokerConfig.rejectTransactionMessage等于true                     // rejectTransactionMessage默认值: false，表示不拒绝
            设置traFlag变量等于msgInner.properties.get("TRAN_MSG")
            如果traFlag变量不等于null
                执行response.setCode(16)                                                      // NO_PERMISSION: 16
                执行response.seRemark("the broker[" + brokerController.brokerConfig.brokerIP1 + "] sending transaction message is forbidden")
                返回response变量
        设置putMessageResult变量等于brokerController.messageStore.putMessage(msgInner)         // 存储消息
        如果putMessageResult变量不等于null
            设置sendOK变量等于false
            // PUT_OK/FLUSH_DISK_TIMEOUT/FLUSH_SLAVE_TIMEOUT/SLAVE_NOT_AVAILABLE算成功，其他算失败
            如果putMessageResult.putMessageStatus等于PUT_OK
                设置sendOK变量等于true
                执行response.setCode(0)                                                       // SUCCESS: 0
            否则如果putMessageResult.putMessageStatus等于FLUSH_DISK_TIMEOUT
                执行response.setCode(10)                                                      // FLUSH_DISK_TIMEOUT: 10
                设置sendOK变量等于true
            否则如果putMessageResult.putMessageStatus等于FLUSH_SLAVE_TIMEOUT
                执行response.setCode(12)                                                      // FLUSH_SLAVE_TIMEOUT: 12
                设置sendOK变量等于true
            否则如果putMessageResult.putMessageStatus等于SLAVE_NOT_AVAILABLE
                执行response.setCode(11)                                                      // SLAVE_NOT_AVAILABLE: 11
                设置sendOK变量等于true
            否则如果putMessageResult.putMessageStatus等于CREATE_MAPEDFILE_FAILED
                执行response.setCode(1)                                                       // SYSTEM_ERROR: 1
                执行response.setRemark("create maped file failed, please make sure OS and JDK both 64bit.")
            否则如果putMessageResult.putMessageStatus等于MESSAGE_ILLEGAL
                执行response.setCode(13)                                                      // MESSAGE_ILLEGAL: 13
                执行response.setRemark("the message is illegal, maybe length not matched.")
            否则如果putMessageResult.putMessageStatus等于SERVICE_NOT_AVAILABLE
                执行response.setCode(14)                                                      // SERVICE_NOT_AVAILABLE: 14
                执行response.setRemark("service not available now, maybe disk full, maybe your broker machine memory too small.")
            否则如果putMessageResult.putMessageStatus等于UNKNOWN_ERROR
                执行response.setCode(1)                                                       // SYSTEM_ERROR: 1
                执行response.setRemark("UNKNOWN_ERROR")
            否则
                执行response.setCode(1)                                                       // SYSTEM_ERROR: 1
                执行response.setRemark("UNKNOWN_ERROR DEFAULT")
            如果sendOK等于true
                // 统计
                执行brokerController.brokerStatsManager.incTopicPutNums(msgInner.topic)
                执行brokerController.brokerStatsManager.incTopicPutSize(msgInner.topic, putMessageResult.appendMessageResult.wroteBytes)
                执行brokerController.brokerStatsManager.incBrokerPutNums()
                执行response.setRemark(null)
                执行responseHeader.setMsgId(putMessageResult.appendMessageResult.msgId)
                执行responseHeader.setQueueId(queueIdInt)
                执行responseHeader.setQueueOffset(putMessageResult.appendMessageResult.logicsOffset)                      // 没有设置事物id逻辑，所以客户端获取不到
                如果request.isOnewayRPC()等于false                                             // oneway方式不需要写回
                    try {
                        ctx.writeAndFlush(response)                                           // 写回
                    } catch (Throwable e) {
                    }
                如果brokerController.brokerConfig.longPollingEnable等于true                    // longPollingEnable默认值: true
                    // 通知可消费的位置（实际还是依赖逻辑队列是否构建）
                    执行brokerController.pullRequestHoldService.notifyMessageArriving(requestHeader.topic, queueIdInt, putMessageResult.appendMessageResult.logicsOffset + 1)
                如果sendMessageHookList属性不等于null并且元素个数大于0                             // 消息轨迹：可以记录发送成功的消息
                    执行mqtraceContext.setMsgId(responseHeader.msgId)
                    执行mqtraceContext.setQueueId(responseHeader.queueId)
                    执行mqtraceContext.setQueueOffset(responseHeader.queueOffset)
                返回null
        否则
            执行response.setCode(1)                                                           // SYSTEM_ERROR: 1
            执行response.setRemark("store putMessage return null")
        返回response
    protected RemotingCommand msgCheck(ChannelHandlerContext ctx, SendMessageRequestHeader requestHeader, RemotingCommand response)
        // 不支持BrokerServer没有写权限并且主题是顺序主题的场景
        如果brokerController.brokerConfig.brokerPermission没有写权限并且brokerController.topicConfigManager.isOrderTopic(requestHeader.topic)等于true
            执行response.setCode(1)                                                           // SYSTEM_ERROR: 1
            执行response.setRemark("the topic is conflict with system reserved words.")
            返回response参数
        // 不支持主题名称等于TBW102或者等于brokerController.brokerConfig.brokerClusterName场景
        如果brokerController.topicConfigManager.isTopicCanSendMessage(requestHeader.topic)等于false
            执行response.setCode(1)                                                           // SYSTEM_ERROR: 1
            执行response.setRemark("the topic is conflict with system reserved words.")
            返回response参数
        设置topicConfig变量等于brokerController.topicConfigManager.selectTopicConfig(requestHeader.topic)
        如果topicConfig变量等于null               // 获取不到Topic配置，就创建一份新的主题配置
            设置topicSysFlag变量等于0
            如果requestHeader.unitMode等于true
                如果requestHeader.topic.startsWith("%RETRY%")等于true
                    设置topicSysFlag变量等于TopicSysFlag.buildSysFlag(false, true)
                否则
                    设置topicSysFlag变量等于TopicSysFlag.buildSysFlag(true, false)
            设置topicConfig变量等于brokerController.topicConfigManager.createTopicInSendMessageMethod(requestHeader.topic, requestHeader.defaultTopic, RemotingHelper.parseChannelRemoteAddr(ctx.channel()), requestHeader.defaultTopicQueueNums, topicSysFlag)
            如果topicConfig变量等于null
                如果requestHeader.topic.startsWith("%RETRY%")等于true
                    设置topicConfig变量等于brokerController.topicConfigManager.createTopicInSendMessageBackMethod(requestHeader.topic, 1, PermName.PERM_WRITE | PermName.PERM_READ, topicSysFlag)
            如果topicConfig变量等于null
                执行response.setCode(17)                                                      // TOPIC_NOT_EXIST: 17
                执行response.setRemark("topic not exist, apply first please!")
                返回response
        设置queueIdInt变量等于requestHeader.queueId
        设置idValid变量等于Math.max(topicConfig.writeQueueNums, topicConfig.readQueueNums)
        如果queueIdInt大于等于idValid                                                          // ??验证队列是否超过限制，这里是不是应该只验证写队列个数，读队列在consumer client端生效，不过broker上完全依赖写队列个数实现逻辑队列
            执行response.setCode(1)                                                           // SYSTEM_ERROR: 1
            执行response.setRemark("request queueId is illagal")
            返回response
        返回response
    // 将消息重新写一份到%RETRY%主题或者%DLQ%主题
    private RemotingCommand consumerSendMsgBack(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        解序列化request为ConsumerSendMsgBackRequestHeader类型的实例，设置给requestHeader变量
        如果consumeMessageHookList属性不等于null并且个数大于0并且requestHeader.originMsgId不等于null          // 消息轨迹：可以记录消费失败的消息
            设置context变量等于new ConsumeMessageContext()实例
            执行context.setConsumerGroup(requestHeader.group)
            执行context.setTopic(requestHeader.originTopic)
            执行context.setClientHost(RemotingHelper.parseChannelRemoteAddr(ctx.channel()))
            执行context.setSuccess(false)
            执行context.setStatus(ConsumeConcurrentlyStatus.RECONSUME_LATER.toString())
            设置messageIds变量等于new HashMap<String, Long>()实例
            执行messageIds.put(requestHeader.originMsgId, requestHeader.offset)
            执行context.setMessageIds(messageIds)
            执行executeConsumeMessageHookAfter(context)
        设置subscriptionGroupConfig变量等于brokerController.subscriptionGroupManager.findSubscriptionGroupConfig(requestHeader.group)
        如果subscriptionGroupConfig变量等于null                                                // 不存在订阅组配置
            执行response.setCode(26)                                                          // SUBSCRIPTION_GROUP_NOT_EXIST: 26
            执行response.setRemark("subscription group not exist")
            返回response变量
        如果brokerController.brokerConfig.brokerPermission没有写权限                            // 当前BrokerServer没有写权限
            执行response.setCode(16)                                                          // NO_PERMISSION: 16
            执行response.setRemark("the broker sending message is forbidden")
            返回response变量
        如果subscriptionGroupConfig.retryQueueNums小于等于0                                    // ??不支持重新存储消息，则返回成功
            执行response.setCode(0)                                                           // SUCCESS: 0
            执行response.setRemark(null)
            返回response
        设置newTopic变量等于"%RETRY%" + requestHeader.group                                    // 将消息写入到新的主题（重试主题）
        设置queueIdInt变量等于Math.abs(random.nextInt() % 99999999) % subscriptionGroupConfig.retryQueueNums
        设置topicSysFlag变量等于0
        如果requestHeader.unitMode等于true
            设置topicSysFlag变量等于TopicSysFlag.buildSysFlag(false, true)                     // 如果是单元化模式，则对topic进行设置
        // 获取或创建新的主题配置
        设置topicConfig变量等于brokerController.topicConfigManager.createTopicInSendMessageBackMethod(newTopic, subscriptionGroupConfig.retryQueueNums, PermName.PERM_WRITE | PermName.PERM_READ, topicSysFlag)
        如果topicConfig变量等于null                                                            // 创建主题配置失败
            执行response.setCode(1)                                                           // SYSTEM_ERROR: 1
            执行response.setRemark("topic not exist")
            返回response变量
        如果topicConfig.perm没有写权限                                                          // 主题没有写入权限
            执行response.setCode(16)                                                          // NO_PERMISSION: 16
            执行response.setRemark("the topic sending message is forbidden")
            返回response变量
        设置msgExt变量等于brokerController.messageStore.lookMessageByOffset(requestHeader.offset)
        如果msgExt变量等于null                                                                 // 原消息不在当前BrokerServer上或者对应的文件已被清理
            执行response.setCode(1)                                                           // SYSTEM_ERROR: 1
            执行response.setRemark("look message by offset failed")
            返回response变量
        设置retryTopic变量等于msgExt.properties.get("RETRY_TOPIC")
        如果retryTopic变量等于null
            执行msgExt.properties.put("RETRY_TOPIC", msgExt.topic)                            // 设置RETRY_TOPIC为原主题
        执行msgExt.setWaitStoreMsgOK(false)
        // 如果超过了最大重试次数，或者延迟级别小于0，则使用%DLQ% + group主题存储，如果延迟级别等于0，设置延迟级别等于3 + 重试次数
        设置delayLevel变量等于requestHeader.delayLevel
        如果msgExt.reconsumeTimes大于等于subscriptionGroupConfig.retryMaxTimes或者delayLevel小于0
            设置newTopic变量等于%DLQ% + requestHeader.group                                    // 存储重试多次都失败的或者延迟级别小于0的
            设置queueIdInt变量等于Math.abs(random.nextInt() % 99999999) % 1
            设置topicConfig变量等于brokerController.topicConfigManager.createTopicInSendMessageBackMethod(newTopic, 1, PermName.PERM_WRITE, 0)        // 只有一个逻辑队列、只有写权限
            如果topicConfig变量等于null
                执行response.setCode(1)                                                       // SYSTEM_ERROR: 1
                执行response.setRemark("topic not exist")
                返回response变量
        否则
            如果delayLevel变量等于0
                设置delayLevel变量等于3 + msgExt.reconsumeTimes
            执行msgExt.setDelayTimeLevel(delayLevel)
        // 创建一个新的消息并存储
        设置msgInner变量等于new MessageExtBrokerInner()实例
        执行msgInner.setTopic(newTopic)
        执行msgInner.setBody(msgExt.body)
        执行msgInner.setFlag(msgExt.flag)
        执行msgInner.properties.put(msgExt.properties)
        执行msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgExt.properties))
        执行msgInner.setTagsCode(MessageExtBrokerInner.tagsString2tagsCode(null, msgExt.properties.get("TAGS")))
        执行msgInner.setQueueId(queueIdInt)
        执行msgInner.setSysFlag(msgExt.sysFlag)
        执行msgInner.setBornTimestamp(msgExt.bornTimestamp)
        执行msgInner.setBornHost(msgExt.bornHost)
        执行msgInner.setStoreHost(storeHost)
        执行msgInner.setReconsumeTimes(msgExt.reconsumeTimes + 1)
        // 原消息的消息ID
        设置originMsgId变量等于msgExt.properties.get("ORIGIN_MESSAGE_ID")
        执行msgInner.properties.put("ORIGIN_MESSAGE_ID", originMsgId == null ? msgExt.msgId : originMsgId)
        设置putMessageResult变量等于brokerController.messageStore.putMessage(msgInner)
        如果putMessageResult变量不等于null
            如果putMessageResult.putMessageStatus变量等于PUT_OK
                设置backTopic变量等于msgExt.topic
                设置correctTopic变量等于msgExt.properties.get("RETRY_TOPIC")
                如果correctTopic变量不等于null
                    设置backTopic变量等于correctTopic变量
                // 统计失败
                执行brokerController.brokerStatsManager.incSendBackNums(requestHeader.group, backTopic)
                执行response.setCode(0)                                                       // SUCCESS: 0
                执行response.setRemark(null)
                返回response变量
            执行response.setCode(1)                                                           // SYSTEM_ERROR: 0
            执行response.setRemark(putMessageResult.putMessageStatus.name())
            返回response变量
        // 存储消息失败
        执行response.setCode(1)                                                               // SYSTEM_ERROR: 0
        执行response.setRemark("putMessageResult is null")
        返回response变量
    public void executeConsumeMessageHookAfter(ConsumeMessageContext context)
        如果consumeMessageHookList属性不等于null并且个数大于0
            遍历consumeMessageHookList属性，设置hook变量等于当前元素
                try {
                    hook.consumeMessageAfter(context)
                } catch (Throwable e) {
                }

# com.alibaba.rocketmq.broker.processor.EndTransactionProcessor

    public EndTransactionProcessor(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    // 消费者真正消费的是非事务型和事物提交型，这里的实现机制不会改变消息状态，而是类似consumerSendMsgBack一样，读取原有的消息并实现一份事物提交型消息或回滚型消息并写入
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        解序列化request为EndTransactionRequestHeader类型的实例，设置给requestHeader变量
        // 非事物类型的直接返回null
        如果requestHeader.fromTransactionCheck等于true
            如果requestHeader.commitOrRollback等于0                                                 // TransactionNotType: 0
                返回null
            否则如果requestHeader.commitOrRollback等于8                                              // TransactionCommitType: 8
            否则如果requestHeader.commitOrRollback等于12                                             // TransactionRollbackType: 12
            否则
                返回null
        否则
            如果requestHeader.commitOrRollback等于0                                                 // TransactionNotType: 0
                返回null
            否则如果requestHeader.commitOrRollback等于8                                              // TransactionCommitType: 8
            否则如果requestHeader.commitOrRollback等于12                                             // TransactionRollbackType: 12
            否则
                返回null
        设置msgExt变量等于brokerController.messageStore.lookMessageByOffset(requestHeader.commitLogOffset)            // 查找的是物理队列
        如果msgExt变量不等于null
            设置pgroupRead变量等于msgExt.properties.get("PGROUP")
            如果requestHeader.producerGroup不等于pgroupRead变量                                      // 校验producerGroup是否发生变化
                执行response.setCode(1)                                                            // SYSTEM_ERROR: 1
                执行response.setRemark("the producer group wrong")
                返回response
            如果msgExt.queueOffset不等于requestHeader.tranStateTableOffset                          // 校验逻辑队列offset是否发生变化
                执行response.setCode(1)                                                            // SYSTEM_ERROR: 1
                执行response.setRemark("the transaction state table offset wrong")
                返回response
            如果msgExt.commitLogOffset不等于requestHeader.commitLogOffset                           // 校验物理队列offset是否发生变化
                执行response.setCode(1)                                                            // SYSTEM_ERROR: 1
                执行response.setRemark("the commit log offset wrong")
                返回response
            设置msgInner变量等于new MessageExtBrokerInner()实例
            执行msgInner.setBody(msgExt.body)
            执行msgInner.setFlag(msgExt.flag)
            执行msgInner.properties.put(msgExt.properties)
            设置topicFilterType变量等于(msgInner.sysFlag & 2) == 2 ? MULTI_TAG : SINGLE_TAG                   // MultiTagsFlag: 2
            设置tagsCodeValue变量等于MessageExtBrokerInner.tagsString2tagsCode(topicFilterType, msgInner.properties.get("TAGS"))
            执行msgInner.setTagsCode(tagsCodeValue)
            执行msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgExt.properties))
            执行msgInner.setSysFlag(msgExt.sysFlag)
            执行msgInner.setBornTimestamp(msgExt.bornTimestamp)
            执行msgInner.setBornHost(msgExt.bornHost)
            执行msgInner.setStoreHost(msgExt.storeHost)
            执行msgInner.setReconsumeTimes(msgExt.reconsumeTimes)
            执行msgInner.setWaitStoreMsgOK(false)
            执行msgInner.properties.remove("DELAY")
            执行msgInner.setTopic(msgExt.topic)
            执行msgInner.setQueueId(msgExt.queueId)
            执行msgInner.setSysFlag(MessageSysFlag.resetTransactionValue(msgInner.sysFlag, requestHeader.commitOrRollback))
            执行msgInner.setQueueOffset(requestHeader.tranStateTableOffset)
            执行msgInner.setPreparedTransactionOffset(requestHeader.commitLogOffset)
            执行msgInner.setStoreTimestamp(msgExt.storeTimestamp)
            如果requestHeader.commitOrRollback等于12                                                   // TransactionRollbackType: 12
                执行msgInner.setBody(null)                                                            // 回滚型事物消息设置body等于null
            设置putMessageResult变量等于brokerController.messageStore.putMessage(msgInner)
            如果putMessageResult变量不等于null
                如果putMessageResult.putMessageStatus等于PUT_OK、FLUSH_DISK_TIMEOUT、FLUSH_SLAVE_TIMEOUT、SLAVE_NOT_AVAILABLE
                    执行response.setCode(0)                                                           // SUCCESS: 0
                    执行response.setRemark(null)
                否则如果putMessageResult.putMessageStatus等于CREATE_MAPEDFILE_FAILED
                    执行response.setCode(1)                                                           // SYSTEM_ERROR: 1
                    执行response.setRemark("create maped file failed.")
                否则如果putMessageResult.putMessageStatus等于MESSAGE_ILLEGAL
                    执行response.setCode(13)                                                          // MESSAGE_ILLEGAL: 13
                    执行response.setRemark("the message is illegal, maybe length not matched.")
                否则如果putMessageResult.putMessageStatus等于SERVICE_NOT_AVAILABLE
                    执行response.setCode(14)                                                          // SERVICE_NOT_AVAILABLE: 14
                    执行response.setRemark("service not available now.")
                否则如果putMessageResult.putMessageStatus等于UNKNOWN_ERROR
                    执行response.setCode(1)                                                           // SYSTEM_ERROR: 1
                    执行response.setRemark("UNKNOWN_ERROR")
                否则
                    执行response.setCode(1)                                                           // SYSTEM_ERROR: 1
                    执行response.setRemark("UNKNOWN_ERROR DEFAULT")
                返回response变量
            否则
                执行response.setCode(1)                                                               // SYSTEM_ERROR: 1
                执行response.setRemark("store putMessage return null")
        否则
            执行response.setCode(1)                                                                   // SYSTEM_ERROR: 1
            执行response.setRemark("find prepared transaction message failed")
            返回response
        返回response

# com.alibaba.rocketmq.broker.processor.ClientManageProcessor
    private List<ConsumeMessageHook> consumeMessageHookList                                          // 消费每条消息会回调

    public ClientManageProcessor(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    public void registerConsumeMessageHook(List<ConsumeMessageHook> consumeMessageHookList)
        设置consumeMessageHookList属性等于consumeMessageHookList参数
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        如果request.code等于34                                                                             // HEART_BEAT: 34
            执行heartBeat(ctx, request)并返回
        否则如果request.code等于35                                                                          // UNREGISTER_CLIENT: 35
            执行unregisterClient(ctx, request)并返回
        否则如果request.code等于38                                                                          // GET_CONSUMER_LIST_BY_GROUP: 38
            执行getConsumerListByGroup(ctx, request)并返回
        否则如果request.code等于15                                                                          // UPDATE_CONSUMER_OFFSET: 15
            执行updateConsumerOffset(ctx, request)并返回
        否则如果request.code等于14                                                                          // QUERY_CONSUMER_OFFSET: 14
            执行queryConsumerOffset(ctx, request)并返回
        返回null
    // 心跳内容包含了Producer和Consumer的相关信息，为定时任务，用于维护BrokerServer的consumer信息和producer信息以及topicConfig信息
    public RemotingCommand heartBeat(ChannelHandlerContext ctx, RemotingCommand request)
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        解序列化request.body为HeartbeatData类型的实例，设置给heartbeatData变量
        设置clientChannelInfo变量等于new ClientChannelInfo(ctx.channel(), heartbeatData.clientID, request.language, request.version)
        // 不存在则注册主题配置和消费者ClientId
        遍历heartbeatData.consumerDataSet，设置data变量等于当前元素
            设置subscriptionGroupConfig变量等于brokerController.subscriptionGroupManager.findSubscriptionGroupConfig(data.groupName)
            如果subscriptionGroupConfig变量不等于null
                设置topicSysFlag变量等于0
                如果data.unitMode等于true
                    设置topicSysFlag变量等于TopicSysFlag.buildSysFlag(false, true)                         // 如果是单元化模式，则对topic进行设置
                设置newTopic变量等于"%RETRY%" + data.groupName
                执行brokerController.topicConfigManager.createTopicInSendMessageBackMethod(newTopic, subscriptionGroupConfig.retryQueueNums, PermName.PERM_WRITE | PermName.PERM_READ, topicSysFlag)
                执行brokerController.consumerManager.registerConsumer(data.groupName, clientChannelInfo, data.consumeType, data.messageModel, data.consumeFromWhere, data.subscriptionDataSet)
        // 不存在注册Producer ClientId
        遍历heartbeatData.producerDataSet，设置data变量等于当前元素
            执行brokerController.producerManager.registerProducer(data.groupName, clientChannelInfo)
        执行response.setCode(0)                                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response变量
    // Producer或Consumer关闭时调用，用于维护BrokerServer的consumer信息和producer信息
    public RemotingCommand unregisterClient(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(UnregisterClientResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        解序列化request为UnregisterClientRequestHeader类型的实例，设置给requestHeader变量
        设置clientChannelInfo变量等于new ClientChannelInfo(ctx.channel(), heartbeatData.clientID, request.language, request.version)
        // 解注册Producer中当前clientId
        如果requestHeader.producerGroup不等于null
            执行brokerController.producerManager.unregisterProducer(requestHeader.producerGroup, clientChannelInfo)
        // 解注册Consumer中当前clientId
        如果requestHeader.consumerGroup不等于null
            执行brokerController.consumerManager.unregisterConsumer(requestHeader.consumerGroup, clientChannelInfo)
        执行response.setCode(0)                                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response变量
    // 返回consumerGroup下的所有Consumer的clientId
    public RemotingCommand getConsumerListByGroup(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(GetConsumerListByGroupResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        解序列化request为GetConsumerListByGroupRequestHeader类型的实例，设置给requestHeader变量
        设置consumerGroupInfo变量等于brokerController.consumerManager.getConsumerGroupInfo(requestHeader.consumerGroup)
        如果consumerGroupInfo不等于null
            设置clientIds变量等于consumerGroupInfo.getAllClientId()
            如果clientIds变量元素个数大于0
                设置body变量等于new GetConsumerListByGroupResponseBody()
                执行body.setConsumerIdList(clientIds)
                执行response.setBody(body.encode())
                执行response.setCode(0)                                                                           // SUCCESS: 0
                执行response.setRemark(null)
                返回response
        执行response.setCode(1)                                                                                   // SYSTEM_ERROR: 1
        执行response.setRemark("no consumer for this group")
        返回response变量
    // 更新某个消费组下的Consumer消费的topic的某个逻辑队列的offset等于指定的offset
    private RemotingCommand updateConsumerOffset(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(UpdateConsumerOffsetResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        解序列化request为UpdateConsumerOffsetRequestHeader类型的实例，设置给requestHeader变量
        如果consumeMessageHookList属性不等于null并且元素个数大于0                                                     // 消息轨迹：可以记录已经消费成功并提交 offset 的消息记录
            设置context变量等于new ConsumeMessageContext()实例
            执行context.setConsumerGroup(requestHeader.consumerGroup)
            执行context.setTopic(requestHeader.topic)
            执行context.setClientHost(RemotingHelper.parseChannelRemoteAddr(ctx.channel()))
            执行context.setSuccess(true)
            执行context.setStatus(ConsumeConcurrentlyStatus.CONSUME_SUCCESS.toString())
            设置storeHost变量等于new InetSocketAddress(brokerController.brokerConfig.brokerIP1, brokerController.nettyServerConfig.listenPort)
            设置preOffset变量等于brokerController.consumerOffsetManager.queryOffset(requestHeader.consumerGroup, requestHeader.topic, requestHeader.queueId)
            设置messageIds变量等于brokerController.messageStore.getMessageIds(requestHeader.topic, requestHeader.queueId, preOffset, requestHeader.commitOffset, storeHost)
            执行context.setMessageIds(messageIds)
            执行executeConsumeMessageHookAfter(context)
        执行brokerController.consumerOffsetManager.commitOffset(requestHeader.consumerGroup, requestHeader.topic, requestHeader.queueId, requestHeader.commitOffset)
        执行response.setCode(0)                                                                               // SUCCESS: 0
        执行response.setRemark(null)
        返回response变量
    // 返回offset
    private RemotingCommand queryConsumerOffset(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(QueryConsumerOffsetResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        设置responseHeader变量等于response.readCustomHeader()
        解序列化request为QueryConsumerOffsetRequestHeader类型的实例，设置给requestHeader变量
        // 返回某个消费组下的Consumer消费的topic的某个逻辑队列的offset，如果存在直接返回
        设置offset变量等于brokerController.consumerOffsetManager.queryOffset(requestHeader.consumerGroup, requestHeader.topic, requestHeader.queueId)
        如果offset大于0
            执行responseHeader.setOffset(offset)
            执行response.setCode(0)                                                                           // SUCCESS: 0
            执行response.setRemark(null)
        否则                  // 不存在情况
            // 获取主题的逻辑队列的最小offset（-1表示不存在指定的逻辑队列），如果小于等于0并且queueId逻辑队列的offset 0不在磁盘上（topic中的queueId逻辑队列中，0对应的物理offset位置和物理队列的最大offset的差值大于物理队列可使用的内存容量，表示在磁盘上，其他情况都不在），返回0
            // 逻辑队列对应的offset 0不在磁盘上，主要是区分消息是否上线不久
            设置minOffset变量等于brokerController.messageStore.getMinOffsetInQuque(requestHeader.topic, requestHeader.queueId)                // ??应该是queue
            如果minOffset小于等于0并且brokerController.messageStore.checkInDiskByConsumeOffset(requestHeader.topic, requestHeader.queueId, 0)等于false
                执行responseHeader.setOffset(0L)
                执行response.setCode(0)                                                                       // SUCCESS: 0
                执行response.setRemark(null)
            否则
                执行response.setCode(22)                                                                      // QUERY_NOT_FOUND: 22
                执行response.setRemark("Not found, V3_0_6_SNAPSHOT maybe this group consumer boot first")
        返回response变量

# com.alibaba.rocketmq.broker.processor.QueryMessageProcessor

    public QueryMessageProcessor(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request)
        如果request.code等于12                                                                              // QUERY_MESSAGE
            此部分不做代码分析
        如果request.code等于33                                                                              // VIEW_MESSAGE_BY_ID: 33
            执行viewMessageById(ctx, request)并返回
        返回null
    // 返回物理offset对应的消息内容
    public RemotingCommand viewMessageById(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        解序列化request为ViewMessageRequestHeader类型的实例，设置给requestHeader变量
        执行response.setOpaque(request.opaque)
        设置selectMapedBufferResult变量等于brokerController.messageStore.selectOneMessageByOffset(requestHeader.offset)
        如果selectMapedBufferResult不等于null
            执行response.setCode(0)                                                                       // SUCCESS: 0
            执行response.setRemark(null)
            try {
                设置fileRegion变量等于new OneMessageTransfer(response.encodeHeader(selectMapedBufferResult.size), selectMapedBufferResult)
                创建new ChannelFutureListener()类型的实例，设置给listener变量
                    public void operationComplete(ChannelFuture future) throws Exception
                        执行selectMapedBufferResult.release()
                执行ctx.channel().writeAndFlush(fileRegion).addListener(listener)
            } catch (Throwable e) {
                执行selectMapedBufferResult.release()
            }
            返回null
        否则
            执行response.setCode(1)                                                                       // SYSTEM_ERROR: 1
            执行response.setRemark("can not find message by the offset")
        返回response变量

# com.alibaba.rocketmq.broker.processor.PullMessageProcessor

    public PullMessageProcessor(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    public void registerConsumeMessageHook(List<ConsumeMessageHook> sendMessageHookList)
        设置consumeMessageHookList属性等于consumeMessageHookList参数
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行processRequest(ctx.channel(), request, true)并返回
    // 拉消息的逻辑
    private RemotingCommand processRequest(Channel channel, RemotingCommand request, boolean brokerAllowSuspend) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(PullMessageResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        设置responseHeader变量等于response.readCustomHeader()
        解序列化request为PullMessageRequestHeader类型的实例，设置给requestHeader变量
        执行response.setOpaque(request.opaque)
        如果brokerController.brokerConfig.brokerPermission没有读权限                            // 验证是否有读权限
            执行response.setCode(16)                                                          // NO_PERMISSION: 16
            执行response.setRemark("the broker pulling message is forbidden")
            返回response变量
        设置subscriptionGroupConfig变量等于brokerController.subscriptionGroupManager.findSubscriptionGroupConfig(requestHeader.consumerGroup)
        如果subscriptionGroupConfig变量等于null                                                // 验证是否存在订阅组
            response.setCode(26)                                                             // SUBSCRIPTION_GROUP_NOT_EXIST: 26
            response.setRemark("subscription group not exist")
            返回response变量
        如果subscriptionGroupConfig.consumeEnable等于false                                    // 验证订阅组可提供消费
            response.setCode(16)                                                             // NO_PERMISSION: 16
            response.setRemark("subscription group no permission")
            返回response变量
        设置hasSuspendFlag变量等于PullSysFlag.hasSuspendFlag(requestHeader.sysFlag)                       // 表示拉取不到消息时是否立即通知给客户端，还是等待直到超时
        设置hasCommitOffsetFlag变量等于PullSysFlag.hasCommitOffsetFlag(requestHeader.sysFlag)
        设置hasSubscriptionFlag变量等于PullSysFlag.hasSubscriptionFlag(requestHeader.sysFlag)
        设置suspendTimeoutMillisLong变量等于hasSuspendFlag ? requestHeader.suspendTimeoutMillis : 0       // 计算等待超时时间
        设置topicConfig变量等于brokerController.topicConfigManager.selectTopicConfig(requestHeader.topic)
        如果topicConfig变量等于null                                                           // 验证主题配置必须存在
            执行response.setCode(17)                                                         // TOPIC_NOT_EXIST: 17
            执行response.setRemark("topic not exist, apply first please!")
            返回response变量
        如果topicConfig.perm没有读权限                                                         // 主题配置提供读权限
            执行response.setCode(16)                                                         // NO_PERMISSION: 16
            执行response.setRemark("the topic pulling message is forbidden")
            返回response变量
        如果requestHeader.queueId小于0或者大于等于topicConfig.readQueueNums                     // ??拉取的消息队列必须小于readQueueNums，是不是完全应该用写队列
            执行response.setCode(1)                                                          // SYSTEM_ERROR: 1
            执行response.setRemark("queueId is illagal")
            返回response变量
        设置subscriptionData变量等于null
        如果hasSubscriptionFlag变量等于true
            // 如果拉消息的请求中传递了订阅标识，用传递过来的构建订阅拦截器配置
            try {
                设置subscriptionData变量等于FilterAPI.buildSubscriptionData(requestHeader.consumerGroup, requestHeader.topic, requestHeader.subscription)
            } catch (Exception e) {
                response.setCode(23)                                                        // SUBSCRIPTION_PARSE_FAILED: 23
                response.setRemark("parse the consumer's subscription failed")
                返回response变量
            }
        否则
            设置consumerGroupInfo变量等于brokerController.consumerManager.getConsumerGroupInfo(requestHeader.consumerGroup)
            如果consumerGroupInfo变量等于null                                                 // 验证必须存在消费组
                response.setCode(24)                                                        // SUBSCRIPTION_NOT_EXIST: 24
                response.setRemark("the consumer's group info not exist")
                返回response变量
            如果subscriptionGroupConfig.consumeBroadcastEnable等于false并且consumerGroupInfo.messageModel等于BROADCASTING       // 验证是否支持广播模式
                response.setCode(16)                                                        // NO_PERMISSION: 16
                response.setRemark("the consumer group can not consume by broadcast way")
                返回response变量
            设置subscriptionData变量等于consumerGroupInfo.findSubscriptionData(requestHeader.topic)                            // 获取对应的订阅拦截器配置
            如果subscriptionData变量等于null                                                  // 验证不能为null
                response.setCode(24)                                                        // SUBSCRIPTION_NOT_EXIST: 24
                response.setRemark("the consumer's subscription not exist")
                返回response变量
            如果subscriptionData.subVersion小于等于requestHeader.subVersion                   // 验证BrokerServer上存储的必须是最新
                response.setCode(25)                                                        // SUBSCRIPTION_NOT_LATEST: 25
                response.setRemark("the consumer's subscription not latest")
                返回response变量
        // 根据queueOffset拉取指定逻辑队列
        设置getMessageResult变量等于brokerController.messageStore.getMessage(requestHeader.consumerGroup, requestHeader.topic, requestHeader.queueId, requestHeader.queueOffset, requestHeader.maxMsgNums, subscriptionData)
        如果getMessageResult变量不等于null
            执行response.setRemark(getMessageResult.status.name())
            执行responseHeader.setNextBeginOffset(getMessageResult.nextBeginOffset)                   // 返回下一次的开始位置
            执行responseHeader.setMinOffset(getMessageResult.minOffset)                               // 逻辑队列最小offset
            执行responseHeader.setMaxOffset(getMessageResult.maxOffset)                               // 逻辑队列最大offset
            如果getMessageResult.suggestPullingFromSlave等于true                                       // 如果最后一条消息对应的物理offset和物理队列中的最大offset的差值超过了物理队列可使用的内存量，推荐用从
                执行responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.whichBrokerWhenConsumeSlowly)
            否则
                执行responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.brokerId)          // 消费正常，按照订阅组配置指定
            如果getMessageResult.status等于FOUND
                执行response.setCode(0)                                                               // SUCCESS: 0
                如果consumeMessageHookList属性不等于null并且元素个数大于0                                 // 消息轨迹：可以记录客户端拉取的消息记录（不表示消费成功）
                    设置context变量等于new ConsumeMessageContext()实例
                    执行context.setConsumerGroup(requestHeader.consumerGroup)
                    执行context.setTopic(requestHeader.topic)
                    执行context.setClientHost(RemotingHelper.parseChannelRemoteAddr(channel))
                    执行context.setStoreHost(brokerController.getBrokerAddr())
                    执行context.setQueueId(requestHeader.queueId)
                    设置storeHost变量等于new InetSocketAddress(brokerController.brokerConfig.brokerIP1, brokerController.nettyServerConfig.listenPort)
                    设置messageIds变量等于brokerController.messageStore.getMessageIds(requestHeader.topic, requestHeader.queueId, requestHeader.queueOffset, requestHeader.queueOffset + getMessageResult.getMessageCount(), storeHost)
                    执行context.setMessageIds(messageIds)
                    执行context.setBodyLength(getMessageResult.bufferTotalSize / getMessageResult.getMessageCount())
                    执行executeConsumeMessageHookAfter(context)
            否则如果getMessageResult.status等于MESSAGE_WAS_REMOVING
                执行response.setCode(20)                                                          // PULL_RETRY_IMMEDIATELY: 20
            否则如果getMessageResult.status等于NO_MATCHED_LOGIC_QUEUE或NO_MESSAGE_IN_QUEUE         // 表示服务端暂时没有这个队列
                如果requestHeader.queueOffset不等于0
                    执行response.setCode(21)                                                      // PULL_OFFSET_MOVED: 21
                否则
                    执行response.setCode(19)                                                      // PULL_NOT_FOUND: 19
            否则如果getMessageResult.status等于NO_MATCHED_MESSAGE
                执行response.setCode(20)                                                          // PULL_RETRY_IMMEDIATELY: 20
            否则如果getMessageResult.status等于OFFSET_FOUND_NULL
                执行response.setCode(19)                                                          // PULL_NOT_FOUND: 19
            否则如果getMessageResult.status等于OFFSET_OVERFLOW_BADLY
                执行response.setCode(21)                                                          // PULL_OFFSET_MOVED: 21
            否则如果getMessageResult.status等于OFFSET_OVERFLOW_ONE
                执行response.setCode(19)                                                          // PULL_NOT_FOUND: 19
            否则如果getMessageResult.status等于OFFSET_TOO_SMALL
                执行response.setCode(21)                                                          // PULL_OFFSET_MOVED: 21
            否则
                抛异常
            如果response.code等于0                                                                // SUCCESS: 0
                // 统计
                执行brokerController.brokerStatsManager.incGroupGetNums(requestHeader.consumerGroup, requestHeader.topic, getMessageResult.getMessageCount())
                执行brokerController.brokerStatsManager.incGroupGetSize(requestHeader.consumerGroup, requestHeader.topic, getMessageResult.bufferTotalSize)
                执行brokerController.brokerStatsManager.incBrokerGetNums(getMessageResult.getMessageCount())
                // 回写拉取到的消息内容
                try {
                    设置fileRegion变量等于new ManyMessageTransfer(response.encodeHeader(getMessageResult.bufferTotalSize), getMessageResult)
                    创建new ChannelFutureListener()类型的实例，设置给listener变量
                        public void operationComplete(ChannelFuture future) throws Exception
                            执行getMessageResult.release()
                    执行channel.writeAndFlush(fileRegion).addListener(listener)
                } catch (Throwable e) {
                    执行getMessageResult.release()
                }
                设置response等于null
            否则如果response.code等于19                                                                // PULL_NOT_FOUND: 19
                // 如果允许推迟，则进行推迟拉消息请求
                如果brokerAllowSuspend等于true并且hasSuspendFlag等于true
                    设置pollingTimeMills变量等于suspendTimeoutMillisLong
                    如果brokerController.brokerConfig.longPollingEnable等于false
                        设置pollingTimeMills变量等于brokerController.brokerConfig.shortPollingTimeMills
                    设置pullRequest变量等于new PullRequest(request, channel, pollingTimeMills, brokerController.messageStore.now(), requestHeader.queueOffset)
                    执行brokerController.pullRequestHoldService.suspendPullRequest(requestHeader.topic, requestHeader.queueId, pullRequest)
                    设置response等于null
            否则如果response.code等于20                                                                // PULL_RETRY_IMMEDIATELY: 20
                // 让客户端马上重新执行拉取请求
            否则如果response.code等于21                                                                // PULL_OFFSET_MOVED: 21
                如果brokerController.messageStoreConfig.brokerRole不等于SLAVE或者brokerController.brokerConfig.offsetCheckInSlave等于true            // 记录事件
                    设置mq变量等于new MessageQueue()实例
                    执行mq.setTopic(requestHeader.topic)
                    执行mq.setQueueId(requestHeader.queueId)
                    执行mq.setBrokerName(brokerController.brokerConfig.brokerName)
                    设置event变量等于new OffsetMovedEvent()实例
                    执行event.setConsumerGroup(requestHeader.consumerGroup)
                    执行event.setMessageQueue(mq)
                    执行event.setOffsetRequest(requestHeader.queueOffset)
                    执行event.setOffsetNew(getMessageResult.nextBeginOffset)
                    try {
                        设置msgInner变量等于new MessageExtBrokerInner()
                        执行msgInner.setTopic("OFFSET_MOVED_EVENT")
                        执行msgInner.setTags(event.consumerGroup)
                        执行msgInner.setDelayTimeLevel(0)
                        执行msgInner.setKeys(event.consumerGroup)
                        执行msgInner.setBody(event.encode())
                        执行msgInner.setFlag(0)
                        执行msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.properties))
                        执行msgInner.properties.set("TAGS", MessageExtBrokerInner.tagsString2tagsCode(TopicFilterType.SINGLE_TAG, msgInner.properties.get("TAGS")))
                        执行msgInner.setQueueId(0)
                        执行msgInner.setSysFlag(0)
                        执行msgInner.setBornTimestamp(System.currentTimeMillis())
                        执行msgInner.setBornHost(RemotingUtil.string2SocketAddress(brokerController.brokerAddr))
                        执行msgInner.setStoreHost(msgInner.bornHost)
                        执行msgInner.setReconsumeTimes(0)
                        执行brokerController.messageStore.putMessage(msgInner)
                    } catch (Exception e) {
                    }
                否则
                    执行responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.brokerId)
                    执行response.setCode(20)                                                          // PULL_RETRY_IMMEDIATELY: 20
            否则
                抛异常
        否则              // 拉不到消息
            执行response.setCode(1)                                                                   // SYSTEM_ERROR: 1
            执行response.setRemark("store getMessage return null")

        设置storeOffsetEnable变量等于brokerAllowSuspend
        设置storeOffsetEnable变量等于storeOffsetEnable && hasCommitOffsetFlag
        设置storeOffsetEnable变量等于storeOffsetEnable && brokerController.messageStoreConfig.brokerRole != BrokerRole.SLAVE
        如果storeOffsetEnable变量等于true
            // 如果拉取不到消息允许等待且有commitOffset标识，且当前Broker不是slave，则更新offset为commitOffset
            执行brokerController.consumerOffsetManager.commitOffset(requestHeader.consumerGroup, requestHeader.topic, requestHeader.queueId, requestHeader.commitOffset)
        返回response
    // 异步执行拉请求，response不等于null表示没有正常拉取到新消息
    public void excuteRequestWhenWakeup(Channel channel, RemotingCommand request) throws RemotingCommandException
        创建new Runnable()实例，设置给run变量
            public void run()
                try {
                    执行processRequest(channel, request, false)返回RemotingCommand类型的实例，设置给response变量
                    如果response变量不等于null
                        执行response.setOpaque(request.opaque)
                        执行response.markResponseType()
                        try {
                            执行channel.writeAndFlush(response)
                        } catch (Throwable e) {
                        }
                } catch (RemotingCommandException e1) {
                }
        执行brokerController.pullMessageExecutor.submit(run)

# com.alibaba.rocketmq.broker.processor.AdminBrokerProcessor

    public AdminBrokerProcessor(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request)
        如果reuqest.code等于17                                           // UPDATE_AND_CREATE_TOPIC: 17
            执行updateAndCreateTopic(ctx, request)并返回
        否则如果request.code等于21                                       // GET_ALL_TOPIC_CONFIG: 21
            执行getAllTopicConfig(ctx, request)并返回
        否则如果request.code等于29                                       // SEARCH_OFFSET_BY_TIMESTAMP: 29
            执行searchOffsetByTimestamp(ctx, request)并返回
        否则如果request.code等于30                                       // GET_MAX_OFFSET: 30
            执行getMaxOffset(ctx, request)并返回
        否则如果request.code等于41                                       // LOCK_BATCH_MQ: 41
            执行lockBatchMQ(ctx, request)并返回
        否则如果request.code等于42                                       // UNLOCK_BATCH_MQ: 42
            执行unlockBatchMQ(ctx, request)并返回
        否则如果request.code等于43                                       // GET_ALL_CONSUMER_OFFSET: 43
            执行getAllConsumerOffset(ctx, request)并返回
        否则如果request.code等于45                                       // GET_ALL_DELAY_OFFSET: 45
            执行getAllDelayOffset(ctx, request)并返回
        否则如果request.code等于201                                      // GET_ALL_SUBSCRIPTIONGROUP_CONFIG: 201
            执行getAllSubscriptionGroup(ctx, request)并返回
        否则如果reuqest.code等于222                                      // INVOKE_BROKER_TO_RESET_OFFSET: 222
            执行resetOffset(ctx, request)并返回
        否则如果request.code等于223                                      // INVOKE_BROKER_TO_GET_CONSUMER_STATUS: 223
            执行getConsumerStatus(ctx, request)并返回
        否则如果request.code等于301                                      // REGISTER_FILTER_SERVER: 301
            执行registerFilterServer(ctx, request)并返回
        否则如果reuqest.code等于307                                      // GET_CONSUMER_RUNNING_INFO: 307
            执行getConsumerRunningInfo(ctx, request)并返回
        否则如果reuqest.code等于309                                      // CONSUME_MESSAGE_DIRECTLY: 309
            执行consumeMessageDirectly(ctx, request)并返回
        返回null
    // 更新或创建主题配置
    private RemotingCommand updateAndCreateTopic(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        解序列化request为CreateTopicRequestHeader类型的实例，设置给requestHeader变量
        如果requestHeader.topic等于brokerController.brokerConfig.brokerClusterName                    // 校验是否冲突
            执行response.setCode(1)                                                                   // SYSTEM_ERROR: 1
            执行response.setRemark("the topic is conflict with system reserved words.")
            返回response
        设置topicConfig变量等于new TopicConfig(requestHeader.topic)
        topicConfig.setReadQueueNums(requestHeader.readQueueNums)
        topicConfig.setWriteQueueNums(requestHeader.writeQueueNums)
        topicConfig.setTopicFilterType(TopicFilterType.valueOf(requestHeader.topicFilterType))
        topicConfig.setPerm(requestHeader.perm)
        topicConfig.setTopicSysFlag(requestHeader.topicSysFlag == null ? 0 : requestHeader .topicSysFlag)
        执行brokerController.topicConfigManager.updateTopicConfig(topicConfig)
        执行brokerController.registerBrokerAll(false, true)
        执行response.setCode(0)                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response
    // 获取某个主题下queueId逻辑队列中指定timestamp对应的offset
    private RemotingCommand searchOffsetByTimestamp(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(SearchOffsetResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        设置responseHeader变量等于response.readCustomHeader()
        解序列化request为SearchOffsetRequestHeader类型的实例，设置给requestHeader变量
        设置offset变量等于brokerController.messageStore.getOffsetInQueueByTime(requestHeader.topic, requestHeader.queueId, requestHeader.timestamp)
        执行responseHeader.setOffset(offset)
        执行response.setCode(0)                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response
    // 获取某个主题的queueId逻辑队列当前最大的offset
    private RemotingCommand getMaxOffset(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(GetMaxOffsetResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        设置responseHeader变量等于response.readCustomHeader()
        解序列化request为GetMaxOffsetRequestHeader类型的实例，设置给requestHeader变量
        设置offset变量等于brokerController.messageStore.getMaxOffsetInQuque(requestHeader.topic, requestHeader.queueId)       // ??应该是queue
        执行responseHeader.setOffset(offset)
        执行response.setCode(0)                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response
    // consumerGroup消费组的clientId申请独占对应的消息队列
    private RemotingCommand lockBatchMQ(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        解序列化request.body为LockBatchRequestBody类型的实例，设置给requestBody变量
        设置lockOKMQSet变量等于brokerController.rebalanceLockManager.tryLockBatch(requestBody.consumerGroup, requestBody.mqSet, requestBody.clientId)
        设置responseBody变量等于new LockBatchResponseBody()
        执行responseBody.setLockOKMQSet(lockOKMQSet)
        执行response.setBody(responseBody.encode())
        执行response.setCode(0)                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response
    // consumerGroup消费组的clientId申请解除独占对应的消息队列
    private RemotingCommand unlockBatchMQ(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        解序列化request.body为UnlockBatchRequestBody类型的实例，设置给requestBody变量
        执行brokerController.rebalanceLockManager.unlockBatch(requestBody.consumerGroup, requestBody.mqSet, requestBody.clientId)
        执行response.setCode(0)                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response
    // 获取所有主题配置
    private RemotingCommand getAllTopicConfig(ChannelHandlerContext ctx, RemotingCommand request)
        执行RemotingCommand.createResponseCommand(GetAllTopicConfigResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        设置topicConfigSerializeWrapper变量等于brokerController.topicConfigManager.buildTopicConfigSerializeWrapper()
        序列化topicConfigSerializeWrapper变量为JSON格式的字符串，设置给content变量
        如果content变量不等于null并且长度大于0
            try {
                执行response.setBody(content.getBytes("UTF-8"))
            } catch (UnsupportedEncodingException e) {
                执行response.setCode(1)                                                   // SYSTEM_ERROR: 1
                执行response.setRemark("UnsupportedEncodingException " + e)
                返回response
            }
        否则
            执行response.setCode(1)                                                       // SYSTEM_ERROR: 1
            执行response.setRemark("No topic in this broker")
            返回response
        执行response.setCode(0)                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response
    // 获取所有offset消费情况信息
    private RemotingCommand getAllConsumerOffset(ChannelHandlerContext ctx, RemotingCommand request)
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        序列化brokerController.consumerOffsetManager为JSON格式的字符串，设置给content变量
        如果content变量不等于null并且长度大于0
            try {
                执行response.setBody(content.getBytes("UTF-8"))
            } catch (UnsupportedEncodingException e) {
                执行response.setCode(1)                                                   // SYSTEM_ERROR: 1
                执行response.setRemark("UnsupportedEncodingException " + e)
                返回response
            }
        否则
            执行response.setCode(1)                                                       // SYSTEM_ERROR: 1
            执行response.setRemark("No consumer offset in this broker")
            返回response
        执行response.setCode(0)                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response
    // 获取所有延迟队列消费情况
    private RemotingCommand getAllDelayOffset(ChannelHandlerContext ctx, RemotingCommand request)
         执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
         序列化brokerController.messageStore.scheduleMessageService为JSON格式的字符串，设置给content变量
         如果content变量不等于null并且长度大于0
             try {
                 执行response.setBody(content.getBytes("UTF-8"))
             } catch (UnsupportedEncodingException e) {
                 执行response.setCode(1)                                                   // SYSTEM_ERROR: 1
                 执行response.setRemark("UnsupportedEncodingException " + e)
                 返回response
             }
         否则
             执行response.setCode(1)                                                       // SYSTEM_ERROR: 1
             执行response.setRemark("No delay offset in this broker")
             返回response
         执行response.setCode(0)                                                           // SUCCESS: 0
         执行response.setRemark(null)
         返回response
    // 获取所有订阅组信息
    private RemotingCommand getAllSubscriptionGroup(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
          执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
          序列化brokerController.subscriptionGroupManager为JSON格式的字符串，设置给content变量
          如果content变量不等于null并且长度大于0
              try {
                  执行response.setBody(content.getBytes("UTF-8"))
              } catch (UnsupportedEncodingException e) {
                  执行response.setCode(1)                                                   // SYSTEM_ERROR: 1
                  执行response.setRemark("UnsupportedEncodingException " + e)
                  返回response
              }
          否则
              执行response.setCode(1)                                                       // SYSTEM_ERROR: 1
              执行response.setRemark("No subscription group in this broker")
              返回response
          执行response.setCode(0)                                                           // SUCCESS: 0
          执行response.setRemark(null)
          返回response
    // 重新设置group消费组下所有消费了topic的consumer的所有消息队列的offset（按照时间对应的offset和当前offset设置重置offset）
    public RemotingCommand resetOffset(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行brokerController.broker2Client.resetOffset(requestHeader.topic, requestHeader.group, requestHeader.timestamp, requestHeader.isForce)并返回
    // 对指定的clientAddr发送GET_CONSUMER_STATUS_FROM_CLIENT请求，获取group消费组下所有主题对应的消息队列的的offset信息
    public RemotingCommand getConsumerStatus(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行brokerController.broker2Client.getConsumeStatus(requestHeader.topic, requestHeader.group, requestHeader.clientAddr)并返回
    // 注册filterserver信息到brokerServer，关系由filterServerManager进行维护
    private RemotingCommand registerFilterServer(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(RegisterFilterServerResponseHeader.class)返回RemotingCommand类型的实例，设置给response变量
        设置responseHeader变量等于response.readCustomHeader()
        解序列化request为RegisterFilterServerRequestHeader类型的实例，设置给requestHeader变量
        执行brokerController.filterServerManager.registerFilterServer(ctx.channel(), requestHeader.filterServerAddr)
        执行responseHeader.setBrokerId(brokerController.brokerConfig.brokerId)
        执行responseHeader.setBrokerName(brokerController.brokerConfig.brokerName)
        执行response.setCode(0)                                                           // SUCCESS: 0
        执行response.setRemark(null)
        返回response
    // 对consumerGroup消费组下的指定clientId对应的Consumer发送GET_CONSUMER_RUNNING_INFO请求
    private RemotingCommand getConsumerRunningInfo(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        解序列化request为GetConsumerRunningInfoRequestHeader类型的实例，设置给requestHeader变量
        执行callConsumer(307, request, requestHeader.consumerGroup, requestHeader.clientId)并返回                  // GET_CONSUMER_RUNNING_INFO: 307
    // 根据消息ID对应的物理offset位置获取消息，直接发送给consumerGroup消费组下的指定clientId对应的Consumer，让其消费
    private RemotingCommand consumeMessageDirectly(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException
        解序列化request为ConsumeMessageDirectlyResultRequestHeader类型的实例，设置给requestHeader变量
        执行request.extFields.put("brokerName", brokerController.brokerConfig.brokerName)
        设置selectMapedBufferResult变量等于null
        try {
            设置messageId变量等于MessageDecoder.decodeMessageId(requestHeader.msgId)
            设置selectMapedBufferResult变量等于brokerController.messageStore.selectOneMessageByOffset(messageId.offset)
            设置body变量等于new byte[selectMapedBufferResult.size]
            执行selectMapedBufferResult.byteBuffer.get(body)
            执行request.setBody(body)

        } catch (UnknownHostException e) {
        } finally {
            如果selectMapedBufferResult变量不等于null
                执行selectMapedBufferResult.release()
        }
        执行callConsumer(309, request, requestHeader.consumerGroup, requestHeader.clientId)     // CONSUME_MESSAGE_DIRECTLY: 309
    private RemotingCommand callConsumer(int requestCode, RemotingCommand request, String consumerGroup, String clientId) throws RemotingCommandException
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        设置clientChannelInfo变量等于brokerController.consumerManager.findChannel(consumerGroup, clientId)
        如果clientChannelInfo变量等于null
            执行response.setCode(1)                                                                   // SYSTEM_ERROR: 1
            执行response.setRemark("The Consumer not online")
            返回response
        try {
            执行RemotingCommand.createRequestCommand(requestCode, null)返回RemotingCommand类型的实例，设置给newRequest变量
            执行newRequest.setExtFields(request.extFields)
            执行newRequest.setBody(request.body)
            执行brokerController.broker2Client.callClient(clientChannelInfo.channel, newRequest)并返回
        } catch (RemotingTimeoutException e) {
            执行response.setCode(207)                                                                 // CONSUME_MSG_TIMEOUT: 207
            执行response.setRemark("consumer Timeout")
            返回response
        } catch (Exception e) {
            执行response.setCode(1)                                                                   // SYSTEM_ERROR: 1
            执行response.setRemark("invoke consumer Exception")
            返回response
        }
