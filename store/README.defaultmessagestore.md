# com.alibaba.rocketmq.store.DefaultMessageStore
    // 消息过滤
    private MessageFilter messageFilter = new DefaultMessageFilter()
    // 逻辑队列
    private ConcurrentHashMap<String/* topic */, ConcurrentHashMap<Integer/* queueId */, ConsumeQueue>> consumeQueueTable
    // 运行标志位，可以标识磁盘空间不足、是否可读、是否可写
    private RunningFlags runningFlags = new RunningFlags()

    public DefaultMessageStore(MessageStoreConfig messageStoreConfig, BrokerStatsManager brokerStatsManager) throws IOException
        设置messageStoreConfig属性等于messageStoreConfig参数
        设置brokerStatsManager属性等于brokerStatsManager参数
        设置allocateMapedFileService属性等于new AllocateMapedFileService()实例                          // 预分配MapedFile对象服务
        设置commitLog属性等于new CommitLog(this)实例                                                    // 物理队列
        设置consumeQueueTable属性等于new ConcurrentHashMap<String/* topic */, ConcurrentHashMap<Integer/* queueId */, ConsumeQueue>>(32)
        设置flushConsumeQueueService属性等于new FlushConsumeQueueService()实例                          // 逻辑队列的刷盘任务
        设置cleanCommitLogService属性等于new CleanCommitLogService()实例                                // 清理物理队列任务
        设置cleanConsumeQueueService属性等于new CleanConsumeQueueService()实例                          // 清理逻辑队列任务
        设置dispatchMessageService属性等于new DispatchMessageService(messageStoreConfig.putMsgIndexHightWater)            // 分发消息服务
        设置storeStatsService属性等于new StoreStatsService()                                           // 统计服务
        设置indexService属性等于new IndexService(this)                                                 // 索引服务
        设置haService属性等于new HAService(this)                                                       // 负责读取数据到slave和接受数据从master并写入到物理队列
        如果messageStoreConfig.brokerRole等于SLAVE
            设置reputMessageService属性等于new ReputMessageService()                                   // 解析物理队列put到逻辑队列任务
            设置scheduleMessageService属性等于new ScheduleMessageService(this)                         // 处理延迟消费任务，slave需要scheduleMessage的初始化（master的schedule是解析特殊的逻辑队列，匹配到后，使用正常的putMessage逻辑，因此slave依靠reput可以完全实现内容同步，offset是通过定时任务完成的同步）
        否则如果messageStoreConfig.brokerRole等于ASYNC_MASTER或SYNC_MASTER
            设置reputMessageService属性等于null
            设置scheduleMessageService属性等于new ScheduleMessageService(this)                         // 处理延迟消费任务
        否则
            设置reputMessageService属性等于null
            设置scheduleMessageService属性等于null
        // commitLog的recover需要依赖服务
        执行allocateMapedFileService.start()
        执行dispatchMessageService.start()
        执行indexService.start()
    public boolean load()
        设置result变量等于true
        try {
            如果messageStoreConfig.storePathRootDir/abort文件存在，设置lastExitOK变量等于false，否则设置lastExitOK变量等于true            // 根据abort文件是否存在，判断上一次是否正常退出
            如果scheduleMessageService属性不等于null
                设置result变量等于result && scheduleMessageService.load() // 加载延迟队列服务，load过程主要是初始化offsetTable和delayLevelTable，实际的延迟队列只是特别名称的逻辑队列，在正确的时候得到延迟消息后，在做正常的putMessage逻辑完成的，因此还是依赖于Consumeueue和CommitLog
            设置result变量等于result && commitLog.load()                  // 加载物理队列
            设置result变量等于result && loadConsumeQueue()                // 加载所有逻辑队列
            如果result变量等于true
                设置storeCheckpoint属性等于new StoreCheckpoint(messageStoreConfig.storePathRootDir/checkpoint)            // 存储检查点
                执行indexService.load(lastExitOK)                       // 加载索引
                执行recover(lastExitOK)                                 // 恢复操作
        } catch (Exception e) {
            设置result变量等于false
        }
        如果result变量等于false
            执行allocateMapedFileService.shutdown()
        返回result变量
    // 加载逻辑队列
    private boolean loadConsumeQueue()
        如果messageStoreConfig.storePathRootDir/consumequeue文件夹存在子目录，遍历所有子目录，设置fileTopic变量等于当前文件夹
            设置topic变量等于fileTopic.getName()
            如果fileTopic目录下存在子目录，遍历所有子目录，设置fileQueueId变量等于当前文件夹
                设置queueId变量等于Integer.parseInt(fileQueueId.getName())
                设置logic变量等于new ConsumeQueue(topic, queueId, "messageStoreConfig.storePathRootDir/consumequeue", messageStoreConfig.getMapedFileSizeConsumeQueue(), this)            // 对mapedFileSizeConsumeQueue进行ceil处理，确保是20的倍数
                执行putConsumeQueue(topic, queueId, logic)
                如果logic.load()等于false，返回false                      // 加载单个逻辑队列
        返回true
    // 加入逻辑队列到consumerQueueTable中
    private void putConsumeQueue(String topic, int queueId, ConsumeQueue consumeQueue)
        设置map变量等于consumeQueueTable.get(topic)
        如果map变量等于null
            设置map变量等于new ConcurrentHashMap<Integer/* queueId */, ConsumeQueue>()
            执行map.put(queueId, consumeQueue)
            执行consumeQueueTable.put(topic, map)
        否则
            执行map.put(queueId, consumeQueue)
    // 执行恢复逻辑
    private void recover(boolean lastExitOK)
        执行recoverConsumeQueue()
        如果lastExitOK参数等于true
            执行commitLog.recoverNormally()                             // 上一次正常关闭的恢复场景
        否则
            执行commitLog.recoverAbnormally()                           // 上一次非正常关闭的恢复场景
        循环，直到dispatchMessageService.hasRemainMessage()等于false结束  // 等读、写队列为空在继续
            try {
                Thread.sleep(500)
            } catch (InterruptedException e) {
            }
        执行recoverTopicQueueTable()                                   // 根据物理最烈的最小offset纠正所有逻辑队列的最小offset
    // 恢复所有逻辑队列
    private void recoverConsumeQueue()
        遍历consumeQueueTable.values，设置maps变量等于当前元素
            遍历maps.values，设置logic变量等于当前元素
                执行logic.recover()                                    // 恢复单个逻辑队列
    // 根据物理最烈的最小offset纠正所有逻辑队列的最小offset
    private void recoverTopicQueueTable()
        设置table变量等于new HashMap<String, Long>(1024)
        设置minPhyOffset变量等于commitLog.getMinOffset()
        遍历consumeQueueTable.values，设置maps变量等于当前元素
            遍历maps.values变量，设置logic变量等于当前元素
                执行table.put(logic.topic + "-" + logic.queueId, logic.getMaxOffsetInQuque())                           // ??应该是queue
                执行logic.correctMinOffset(minPhyOffset)               // 根据物理最烈的最小offset纠正逻辑队列的最小offset
        执行commitLog.setTopicQueueTable(table)                        // 记录所有逻辑队列的最大offset，用于put到物理队列时计算逻辑队列对应的offset
    public void start() throws Exception
        执行flushConsumeQueueService.start()
        执行commitLog.start()
        执行storeStatsService.start()
        如果scheduleMessageService属性不等于null并且messageStoreConfig.brokerRole不等于SLAVE
            执行scheduleMessageService.start()                        // slave不要启动，reput可以干完
        如果reputMessageService属性不等于null
            执行reputMessageService.setReputFromOffset(commitLog.getMaxOffset())          // slave专用，设置reput读取的初始位置，为当前物理队列最大offset
            执行reputMessageService.start()
        执行haService.start()
        创建messageStoreConfig.storePathRootDir/abort文件
        启动定时任务，延迟1分钟执行，任务执行间隔为messageStoreConfig.cleanResourceInterval毫秒    // 定时清理过期文件
            执行cleanFilesPeriodically()
        设置shutdown属性等于false
    // 清理物理队列和逻辑队列的过期文件
    private void cleanFilesPeriodically()
        执行cleanCommitLogService.run()
        执行cleanConsumeQueueService.run()
    public void shutdown()
        如果shutdown属性等于false
            设置shutdown属性等于true
            执行scheduledExecutorService.shutdown()
            try {
                Thread.sleep(1000 * 3)
            } catch (InterruptedException e) {
            }
            如果scheduleMessageService属性不等于null
                执行scheduleMessageService.shutdown()
            执行haService.shutdown()
            执行storeStatsService.shutdown()
            执行dispatchMessageService.shutdown()
            执行indexService.shutdown()
            执行flushConsumeQueueService.shutdown()
            执行commitLog.shutdown()
            执行allocateMapedFileService.shutdown()
            如果reputMessageService属性不等于null
                执行reputMessageService.shutdown()
            执行storeCheckpoint.flush()
            执行storeCheckpoint.shutdown()
            删除messageStoreConfig.storePathRootDir/abort文件           // 正常关闭会删除该文件
    // 当回当前系统时间，会断不是通过native获取系统时间戳，是通过后台线程定时刷新维护的当前时间
    public long now()
        执行systemClock.now()并返回
    // 更新HA地址，slave专用
    public void updateHaMasterAddress(String newAddr)
        执行haService.updateMasterAddress(newAddr)
    // 获取指定queueId对应的逻辑队列的最大offset，不存在queueId返回0
    public long getMaxOffsetInQuque(String topic, int queueId)                       // ??应该是queue
        设置logic变量等于findConsumeQueue(topic, queueId)
        如果logic变量不等于null
            执行logic.getMaxOffsetInQuque()并返回                                      // ??应该是queue
        返回0
    // 获取指定queueId对应的逻辑队列的最小offset，不存在queueId返回-1
    public long getMinOffsetInQuque(String topic, int queueId)                       // ??应该是queue
        设置logic变量等于findConsumeQueue(topic, queueId)
        如果logic变量不等于null
            执行logic.getMinOffsetInQuque()并返回                                      // ??应该是queue
        返回-1                                                                        // ??不会不存在，而且最小返回0
    // 获取指定queueId对应的逻辑队列中指定时间戳的消息offset，不存在queueId返回0
    public long getOffsetInQueueByTime(String topic, int queueId, long timestamp)
         设置logic变量等于findConsumeQueue(topic, queueId)
         如果logic变量不等于null
             执行logic.getOffsetInQueueByTime(timestamp)并返回
         返回0
    // 获取指定的逻辑队列，不存在就创建一个
    public ConsumeQueue findConsumeQueue(String topic, int queueId)
        设置map变量等于consumeQueueTable.get(topic)
        如果map变量等于null
            设置newMap变量等于new ConcurrentHashMap<Integer, ConsumeQueue>(128)
            执行consumeQueueTable.putIfAbsent(topic, newMap)，设置返回值给oldMap变量
            如果oldMap变量不等于null
                设置map变量等于oldMap变量
            否则
                设置map变量等于newMap变量
        设置logic变量等于map.get(queueId)
        如果logic变量等于null
            设置newLogic变量等于new ConsumeQueue(topic, queueId, "messageStoreConfig.storePathRootDir/consumequeue", messageStoreConfig.getMapedFileSizeConsumeQueue(), this)     // 对mapedFileSizeConsumeQueue进行ceil处理，确保是20的倍数
            执行map.putIfAbsent(queueId, newLogic)，设置返回值给oldLogic变量
            如果oldLogic变量不等于null
                设置logic变量等于oldMap变量
            否则
                设置logic变量等于newLogic变量
        返回logic变量
    // 删除所有topics（删除对应的逻辑队列，删除commitLog维护的逻辑队列最大offset映射）
    public int cleanUnusedTopic(Set<String> topics)
        遍历consumeQueueTable属性，设置entry变量等于当前元素
            设置topic变量等于entry.key
            如果topics.contains(topic)等于true并且topic变量不等于"SCHEDULE_TOPIC_XXXX"
            遍历entry.value.values，设置cq变量等于当前元素
                执行cq.destroy()
                执行commitLog.removeQueurFromTopicQueueTable(cq.topic, cq.queueId)                    // ??应该是queue
            删除entry变量
        返回0
    //检测topic中的queueId逻辑队列中，consumerOffset对应的物理offset位置和物理队列的最大offset的差值是否大于物理队列可使用的内存容量，如果大于表示在磁盘上，其他情况都不在
    public boolean checkInDiskByConsumeOffset(String topic, int queueId, long consumeOffset)
        设置maxOffsetPy变量等于commitLog.getMaxOffset()
        设置consumeQueue变量等于findConsumeQueue(topic, queueId)
        如果consumeQueue变量不等于null
            设置bufferConsumeQueue变量等于consumeQueue.getIndexBuffer(consumeOffset)
            如果bufferConsumeQueue变量不等于null
                try {
                    for (int i = 0; i < bufferConsumeQueue.size;) {
                        i += 20
                        设置offsetPy变量等于bufferConsumeQueue.byteBuffer.getLong()
                        执行checkInDiskByCommitOffset(offsetPy, maxOffsetPy)并返回
                    }
                } finally {
                    执行bufferConsumeQueue.release()
                }
            否则
                返回false
        返回false
    // 检测当前offsetPy和maxOffsetPy的差值，是否超过了可使用的内存容量
    private boolean checkInDiskByCommitOffset(long offsetPy, long maxOffsetPy)
        设置memory变量等于(long) (StoreUtil.TotalPhysicalMemorySize * (messageStoreConfig.accessMessageInMemoryMaxRatio / 100.0))
        返回(maxOffsetPy - offsetPy) > memory
    // 获取物理队列中，指定commitLogOffset位置的消息对应的ByteBuffer
    public SelectMapedBufferResult selectOneMessageByOffset(long commitLogOffset)
        设置sbr变量等于commitLog.getMessage(commitLogOffset, 4)
        如果sbr变量不等于null
            try {
                设置size变量等于sbr.byteBuffer.getInt()
                执行commitLog.getMessage(commitLogOffset, size)并返回
            } finally {
                执行sbr.release()
            }
        返回null
    // 获取物理队列中，指定commitLogOffset位置的消息
    public MessageExt lookMessageByOffset(long commitLogOffset)
        设置sbr变量等于commitLog.getMessage(commitLogOffset, 4)
        如果sbr变量不等于null
            try {
                设置size变量等于sbr.byteBuffer.getInt()
                执行lookMessageByOffset(commitLogOffset, size)并返回
            } finally {
                执行sbr.release()
            }
        返回null
    // 获取物理队列中，指定commitLogOffset位置的消息对应的ByteBuffer，并解序列化为消息
    public MessageExt lookMessageByOffset(long commitLogOffset, int size)
        设置sbr变量等于commitLog.getMessage(commitLogOffset, size)
        如果sbr变量不等于null
            try {
                执行MessageDecoder.decode(sbr.byteBuffer, true, false)并返回
            } finally {
                执行sbr.release()
            }
        返回null
    // 返回topic中的queueId逻辑队列中，minOffset到maxOffset之间的所有messageId
    public Map<String, Long> getMessageIds(String topic, int queueId, long minOffset, long maxOffset, SocketAddress storeHost)
        设置messageIds变量等于new HashMap<String, Long>()实例
        如果shutdown属性等于true
            返回messageIds变量
        设置consumeQueue变量等于findConsumeQueue(topic, queueId)
        如果consumeQueue变量不等于null
            设置minOffset参数等于Math.max(minOffset, consumeQueue.getMinOffsetInQuque())                  // 取最大的 ??应该是queue
            设置maxOffset参数等于Math.min(maxOffset, consumeQueue.getMaxOffsetInQuque())                  // 取最小的 ??应该是queue
            如果maxOffset参数等于0
                返回messageIds变量
            设置nextOffset变量等于minOffset参数
            while (nextOffset < maxOffset) {
                设置bufferConsumeQueue变量等于consumeQueue.getIndexBuffer(nextOffset)
                如果bufferConsumeQueue变量不等于null
                    try {
                        // 逻辑队列的每个消息占用20个byte（不存在body，存储的是指向物理队列的相关标识信息）
                        // size标识的是数据大小
                        for (int i = 0; i < bufferConsumeQueue.size; i += 20) {
                            设置offsetPy变量等于bufferConsumeQueue.byteBuffer.getLong()
                            // messageId占用16个byte，其中8个byte标识当前Socket（host:port），8个byte标识物理队列对应的offset
                            设置msgId变量等于MessageDecoder.createMessageId(ByteBuffer.allocate(16), MessageExt.SocketAddress2ByteBuffer(storeHost), offsetPy)
                            执行messageIds.put(msgId, nextOffset++)
                            如果nextOffset变量大于maxOffset参数
                                返回messageIds变量
                        }
                    } finally {
                        bufferConsumeQueue.release()
                    }
                否则
                    返回messageIds变量
            }
        返回messageIds变量
    public GetMessageResult getMessage(String group, String topic, int queueId, long offset, int maxMsgNums, SubscriptionData subscriptionData)
        如果shutdown属性等于true
            返回null
        如果runningFlags.isReadable()等于false                                  // 验证是否开启了读限流
            返回null
        设置beginTime变量等于systemClock.now()
        设置status变量等于NO_MESSAGE_IN_QUEUE
        设置nextBeginOffset变量等于offset
        设置minOffset变量和maxOffset变量等于0
        设置getResult变量等于new GetMessageResult()实例
        设置maxOffsetPy变量等于commitLog.getMaxOffset()
        设置consumeQueue变量等于findConsumeQueue(topic, queueId)                 // 获取对应的逻辑队列
        如果consumeQueue变量不等于null
            设置minOffset变量等于consumeQueue.getMinOffsetInQuque()              // ??应该是queue
            设置maxOffset变量等于consumeQueue.getMaxOffsetInQuque()              // ??应该是queue
            如果maxOffset变量等于0                                              // 逻辑队列刚创建，没有任务消息，此时maxOffset/minOffset/nextBeginOffset都是0
                设置status变量等于NO_MESSAGE_IN_QUEUE
                设置nextBeginOffset变量等于0
            否则如果offset变量小于minOffset变量                                  // offset过期了，对应逻辑队列中的文件都被删了，此时nextBeginOffset等于minOffset
                设置status变量等于OFFSET_TOO_SMALL
                设置nextBeginOffset变量等于minOffset变量
            否则如果offset变量等于maxOffset变量                                  // 如果和逻辑队列的最大offset相等，可能两种情况（一种是确实没消息，第二种是逻辑队列是异步从物理队列构建的，此时消息还没构建完），此时nextBeginOffset等于offset
                设置status变量等于OFFSET_OVERFLOW_ONE
                设置nextBeginOffset变量等于offset变量
            否则如果offset变量大于maxOffset变量                                  // 如果大于逻辑队列的最大offset，这种情况，开始纠正
                设置status变量等于OFFSET_OVERFLOW_BADLY
                如果minOffset变量等于0                                         // 如果是新建的逻辑队列，设置nextBeginOffset等于0
                    设置nextBeginOffset变量等于minOffset变量
                否则                                                          // 如果不是新建的，设置nextBeginOffset等于maxOffset
                    设置nextBeginOffset变量等于maxOffset变量
            否则                                                              // 正常情况下
                设置bufferConsumeQueue变量等于consumeQueue.getIndexBuffer(offset) 返回指定offset处的ByteBuffer
                如果bufferConsumeQueue变量不等于null
                    try {
                        设置status变量等于NO_MATCHED_MESSAGE
                        设置nextPhyFileStartOffset变量等于Long.MIN_VALUE
                        设置maxPhyOffsetPulling变量等于0
                        设置diskFallRecorded变量等于false
                        for (int i = 0; i < bufferConsumeQueue.size && i < 16000; i += 20)              // 逻辑队列里，每个消息占用20byte，每次最多检测16000条消息
                            设置offsetPy变量等于bufferConsumeQueue.byteBuffer.getLong()                   // 对应物理队列的offset
                            设置sizePy变量等于bufferConsumeQueue.byteBuffer.getInt()                      // 对应物理队列的消息大小
                            设置tagsCode变量等于bufferConsumeQueue.byteBuffer.getLong()                   // 特殊标识
                            设置maxPhyOffsetPulling变量等于offsetPy变量                                   // 设置为物理队列的offset
                            如果nextPhyFileStartOffset变量不等于Long.MIN_VALUE
                                如果offsetPy变量小于nextPhyFileStartOffset变量                           // 如果之前发生过轮转，此处会一直跳过
                                    继续下一次循环
                            设置isInDisk变量等于checkInDiskByCommitOffset(offsetPy, maxOffsetPy)         // 检测offsetPy对应的消息是否在磁盘中而不是内存中
                            // 是否达到上限了，如果达到上限了，直接返回
                            如果isTheBatchFull(sizePy, maxMsgNums, getResult.bufferTotalSize, getResult.messageMapedList.size(), isInDisk)等于true
                                退出循环
                            如果messageFilter.isMessageMatched(subscriptionData, tagsCode)等于true      // 过滤通过
                                设置selectResult变量等于commitLog.getMessage(offsetPy, sizePy)           // 从物理队列中获取对应的消息
                                如果selectResult变量不等于null
                                    执行storeStatsService.getMessageTransferedMsgCount.incrementAndGet()
                                    执行getResult.addMessage(selectResult)                             // 添加到getResult中
                                    设置status变量等于FOUND
                                    设置nextPhyFileStartOffset变量等于Long.MIN_VALUE                    // 不需要跳过了
                                    如果diskFallRecorded变量等于true                                     // ??不过走下边的逻辑，diskFallRecorded一直为true，是不是应该根据isInDisk设置为diskFallRecorded为true
                                        设置diskFallRecorded变量等于true
                                        设置fallBehind变量等于consumeQueue.maxPhysicOffset - offsetPy
                                        执行brokerStatsManager.recordDiskFallBehind(group, topic, queueId, fallBehind)
                                否则                                                                    // 从物理队列拿不到消息，说明文件发生了轮转，设置开始位置等于当前offsetPy对应的文件的下一个文件的开头
                                    如果getResult.bufferTotalSize等于0
                                        设置status变量等于MESSAGE_WAS_REMOVING
                                    设置nextPhyFileStartOffset变量等于commitLog.rollNextFile(offsetPy)
                            否则                                                                        // 过滤不通过
                                如果getResult.bufferTotalSize等于0
                                    设置status变量等于NO_MATCHED_MESSAGE
                        设置nextBeginOffset变量等于offset + (i / 20)                                    // 下一次消息的开始位置
                        设置diff变量等于getMaxPhyOffset() - maxPhyOffsetPulling                         // 最后一条消息的物理队列对应的offset和物理队列最大offset的差值
                        // 如果超过了内存使用限制（有堆积），使用slave，否则使用master
                        设置memory变量等于(long) (StoreUtil.TotalPhysicalMemorySize * (messageStoreConfig.accessMessageInMemoryMaxRatio / 100.0))
                        执行getResult.setSuggestPullingFromSlave(diff > memory)
                    } finally {
                        bufferConsumeQueue.release()
                    }
                否则
                    // 逻辑队列中，不存在对应的offset，但是前面的验证通过了，此时表示在验证过程中，逻辑文件发生了轮转
                    // offset并不是表示文件级别的offset，而是逻辑offset（每条消息占用20byte，下一条消息的offset比上一条消息的offset大1，实际文件级别大20byte）
                    // 此处返回的是offset对应的文件的下一个文件的开头
                    设置status变量等于OFFSET_FOUND_NULL
                    设置nextBeginOffset变量等于consumeQueue.rollNextFile(offset)
        否则                                                  // 没有找到对应的逻辑队列，此时maxOffset/minOffset/nextBeginOffset都是0
            设置status变量等于NO_MATCHED_LOGIC_QUEUE
            设置nextBeginOffset变量等于0
        如果status变量等于FOUND
            执行storeStatsService.getMessageTimesTotalFound.incrementAndGet()
        否则
            执行storeStatsService.getMessageTimesTotalMiss.incrementAndGet()
        设置eclipseTime变量等于systemClock.now() - beginTime
        执行storeStatsService.setGetMessageEntireTimeMax(eclipseTime)
        执行getResult.setStatus(status)
        执行getResult.setNextBeginOffset(nextBeginOffset)
        执行getResult.setMaxOffset(maxOffset)
        执行getResult.setMinOffset(minOffset)
        返回getResult变量
    // 获取物理队列的最大offset
    public long getMaxPhyOffset()
        返回commitLog.getMaxOffset()并返回
    // 如果消息个数 + 1超过maxMsgNums，表示达到上限了
    // 如果需要拉取磁盘
    //      如果当前getMessage已有的byte大小 + 当前消息的大小超过了maxTransferBytesOnMessageInDisk，表示达到上限了
    //      如果当前getMessage已有的消息个数 + 1超过了maxTransferCountOnMessageInDisk，表示达到上限了
    // 如果不需要拉取磁盘
    //      如果当前getMessage已有的byte大小 + 当前消息的大小超过了maxTransferBytesOnMessageInMemory，表示达到上限了
    //      如果当前getMessage已有的消息个数 + 1超过了maxTransferCountOnMessageInMemory，表示达到上限了
    private boolean isTheBatchFull(int sizePy, int maxMsgNums, int bufferTotal, int messageTotal, boolean isInDisk)
        如果bufferTotal参数等于0或者messageTotal参数等于0
            返回false
        如果messageTotal参数 + 1大于等于maxMsgNums参数
            返回true
        如果isInDisk参数等于true
            如果bufferTotal参数 + sizePy参数大于messageStoreConfig.maxTransferBytesOnMessageInDisk
                返回true
            如果messageTotal参数 + 1大于messageStoreConfig.maxTransferCountOnMessageInDisk
                返回true
        否则
            如果bufferTotal参数 + sizePy参数大于messageStoreConfig.maxTransferBytesOnMessageInMemory
                返回true
            如果messageTotal参数 + 1大于messageStoreConfig.maxTransferCountOnMessageInMemory
                返回true
        返回false
    // 添加消息到物理队列中
    public PutMessageResult putMessage(MessageExtBrokerInner msg)
        如果shutdown属性等于true
            返回new PutMessageResult(SERVICE_NOT_AVAILABLE, null)实例
        如果messageStoreConfig.brokerRole等于SLAVE
            返回new PutMessageResult(SERVICE_NOT_AVAILABLE, null)实例
        如果runningFlags.isWriteable()属性等于false                               // 验证是否开启了写限流
            返回new PutMessageResult(SERVICE_NOT_AVAILABLE, null)实例
        如果msg.topic.length()大于Byte.MAX_VALUE
            返回new PutMessageResult(MESSAGE_ILLEGAL, null)实例
        如果msg.propertiesString不等于null并且长度大于Short.MAX_VALUE
            返回new PutMessageResult(MESSAGE_ILLEGAL, null)实例
        设置beginTime变量等于systemClock.now()
        设置result变量等于commitLog.putMessage(msg)
        设置eclipseTime变量等于systemClock.now() - beginTime
        执行storeStatsService.setPutMessageEntireTimeMax(eclipseTime)
        执行storeStatsService.getSinglePutMessageTopicTimesTotal(msg.topic).incrementAndGet()
        如果result变量等于null或者result.isOk()等于false
            执行storeStatsService.putMessageFailedTimes.incrementAndGet()
        返回result
    // 添加分发请求到分发服务中
    public void putDispatchRequest(DispatchRequest dispatchRequest)
        执行dispatchMessageService.putRequest(dispatchRequest)
    // 根据物理队列的offset，删除所有逻辑队列中的无效逻辑文件
    public void truncateDirtyLogicFiles(long phyOffset)
        遍历consumeQueueTable.values，设置maps变量等于当前元素
            遍历maps.values，设置logic变量等于当前元素
                执行logic.truncateDirtyLogicFiles(phyOffset)
    // 废弃所有逻辑队列（删除对应的文件，offset重新置位）
    public void destroyLogics()
        遍历consumeQueueTable.values，设置maps变量等于当前元素
            遍历maps.values，设置logic变量等于当前元素
                执行logic.destroy()
    // 读取文件级别offset之后内容
    public SelectMapedBufferResult getCommitLogData(long offset)      // master使用
        如果shutdown属性等于true
            返回null
        返回commitLog.getData(offset)
    // 写入data到文件级别startOffset对应的文件，触发reput
    public boolean appendToCommitLog(long startOffset, byte[] data)         // slave使用
        如果shutdown属性等于true
            返回false
        设置result变量等于commitLog.appendData(startOffset, data)
        如果result变量等于true
            执行reputMessageService.wakeup()
        返回result变量

# com.alibaba.rocketmq.store.AllocateMapedFileService
    // 申请MapedFile服务，每次申请，会申请两次，一次是当前需要用的，一个是预留的，预留的不返回给给客户端，因此创建操作有开销，这样下一次申请的时候，直接返回上一次预留的

    private ConcurrentHashMap<String, AllocateRequest> requestTable = new ConcurrentHashMap<String, AllocateRequest>()
    private PriorityBlockingQueue<AllocateRequest> requestQueue =  new PriorityBlockingQueue<AllocateRequest>()

    public void start()
    public void run()
        循环，知道stop属性等于true或者mmapOperation()等于false
    private boolean mmapOperation()
        设置req变量等于null
        try {
            设置req变量等于requestQueue.take()
            如果requestTable.get(req.filePath)等于null                              // 已经处理过了
                返回true
            如果req.mapedFile等于null                                               // 创建对应的实例，填充到请求中，如果出了问题，设置hasException等于true，如果创建成功，设置hasException等于false
                设置mapedFile变量等于new MapedFile(req.filePath, req.fileSize)
                执行req.setMapedFile(mapedFile)
                设置hasException属性等于false
        } catch (InterruptedException e) {
            设置hasException属性等于true
            返回false
        } catch (IOException e) {
            设置hasException属性等于true
        } finally {
            如果req变量不等于null
                执行req.countDownLatch.countDown()                                 // 标识当前请求处理完了
        }
        返回true
    public void shutdown()
        设置stoped变量等于true
        终止和等待线程结束
        遍历requestTable.values，设置req变量等于当前元素
            如果req.mapedFile不等于null
                执行req.mapedFile.destroy(1000)
    public MapedFile putRequestAndReturnMapedFile(String nextFilePath, String nextNextFilePath, int fileSize)
        // 创建两个请求，并加入到requestTable和requestQueue中
        设置nextReq变量等于new AllocateRequest(nextFilePath, fileSize)实例
        设置nextNextReq变量等于new AllocateRequest(nextNextFilePath, fileSize)实例
        设置nextPutOK变量等于requestTable.putIfAbsent(nextFilePath, nextReq) == null
        设置nextNextPutOK变量等于requestTable.putIfAbsent(nextNextFilePath, nextNextReq) == null
        如果nextPutOK变量等于true
            执行requestQueue.offer(nextReq)
        如果nextNextPutOK变量等于true
            执行requestQueue.offer(nextNextReq)
        如果hasException等于true                                    // 如果之前的申请产生了错误，直接返回null
            返回null
        设置result变量等于requestTable.get(nextFilePath)             // 只拿当前的，在上一次申请的时候，已经预留了，所以应该很快
        try {
            如果result变量不等于null
                执行result.countDownLatch.await(5 * 1000, TimeUnit.MILLISECONDS)
                执行requestTable.remove(nextFilePath)
                返回result.mapedFile
        } catch (InterruptedException e) {
        }
        返回null

# com.alibaba.rocketmq.store.CommitLog

    public CommitLog(DefaultMessageStore defaultMessageStore)
        设置defaultMessageStore属性等于defaultMessageStore参数
        // mapedFileSizeCommitLog默认值: 1G
        设置mapedFileQueue属性等于new MapedFileQueue(defaultMessageStore.messageStoreConfig.storePathCommitLog, defaultMessageStore.messageStoreConfig.mapedFileSizeCommitLog, defaultMessageStore.allocateMapedFileService)
        // flushDiskType默认值: ASYNC_FLUSH
        如果defaultMessageStore.messageStoreConfig.flushDiskType等于SYNC_FLUSH
            设置flushCommitLogService变量等于new GroupCommitService()
        否则
            设置flushCommitLogService变量等于new FlushRealTimeService()
        // 真正干活的
        设置appendMessageCallback变量等于new DefaultAppendMessageCallback(defaultMessageStore.messageStoreConfig.maxMessageSize)
    // 加载物理队列
    public boolean load()
        执行mapedFileQueue.load()并返回
    // 正常关闭的恢复，最多恢复最后3个文件
    public void recoverNormally()
        设置checkCRCOnRecover变量等于defaultMessageStore.messageStoreConfig.checkCRCOnRecover                     // 是否校验CRC，默认值: true
        设置mapedFiles变量等于mapedFileQueue.mapedFiles
        如果mapedFiles变量元素个数大于0
            设置index变量等于mapedFiles.size() - 3                    // 从最后3个开始还原
            如果index变量小于0
                设置index变量等于0
            设置mapedFile变量等于mapedFiles.get(index)                // 获取对应的mapedFile
            设置byteBuffer变量等于mapedFile.sliceByteBuffer()
            设置processOffset变量等于mapedFile.fileFromOffset         // 获取开始位置
            设置mapedFileOffset变量等于0                              // 记录解析得到的消息总长度
            while (true) {
                设置dispatchRequest变量等于checkMessageAndReturnSize(byteBuffer, checkCRCOnRecover)               // 验证消息格式是否合理
                设置size变量等于dispatchRequest.msgSize
                如果size变量大于0                                                                                 // 获取消息大小，增加mapedFileOffset
                    设置mapedFileOffset变量等于mapedFileOffset + size
                否则如果size变量等于-1
                    退出循环
                否则如果size变量等于0                                                                             // 如果为0，标识当前文件解析完毕，开始解析下一个文件
                    设置index++
                    如果index变量大于等于mapedFiles.size()
                        跳出循环
                    否则
                        设置mapedFile变量等于mapedFiles.get(index)
                        设置byteBuffer变量等于mapedFile.sliceByteBuffer()
                        设置processOffset变量等于mapedFile.fileFromOffset
                        设置mapedFileOffset变量等于0
            }
            // 基于文件开始位置加上解析得到的消息总长度，设置commited位置，同时，删除所有文件结束位置超过processOffset的文件
            设置processOffset变量等于processOffset + mapedFileOffset
            执行mapedFileQueue.setCommittedWhere(processOffset)
            执行mapedFileQueue.truncateDirtyFiles(processOffset)
    // 尝试正常解析 + 校验消息体的crc，如果解析出现问题，返回-1标识，否则返回消息（正常消息和文件结束标识）
    public DispatchRequest checkMessageAndReturnSize(ByteBuffer byteBuffer, boolean checkCRC)
        执行checkMessageAndReturnSize(byteBuffer, checkCRC, true)并返回
    // 尝试正常解析，如果解析出现问题，返回-1长度，否则返回消息（正常消息和文件结束标识）
    public DispatchRequest checkMessageAndReturnSize(ByteBuffer byteBuffer, boolean checkCRC, boolean readBody)
        try {
            设置byteBufferMessage变量等于appendMessageCallback.msgStoreItemMemory         // 返回Callback持有的byteBuffer，用作临时变量
            设置bytesContent变量等于byteBufferMessage.array()
            设置totalSize变量等于byteBuffer.getInt()                                      // 消息大小
            设置magicCode变量等于byteBuffer.getInt()                                      // MAGICCODE
            如果magicCode变量等于MessageMagicCode                                         // 正常的消息标识

            否则如果magicCode变量等于BlankMagicCode                                        // 文件结束标识，对于此标识，其他前面的4个字节表示的不是消息大小，而是有多少空位
                返回new DispatchRequest(0)实例
            否则                                                                        // ??其他情况
                返回new DispatchRequest(-1)实例
            设置bodyCRC变量等于byteBuffer.getInt()                                        // 消息体的CRC
            设置queueId变量等于byteBuffer.getInt()                                        // 逻辑队列ID
            设置flag变量等于byteBuffer.getInt()                                           // 客户端设定的标识
            设置flag变量等于flag + 0
            设置queueOffset变量等于byteBuffer.getLong()                                   // 逻辑队列offset（类似增量ID）
            设置physicOffset变量等于byteBuffer.getLong()                                  // 物理队列offset（文件级别的position）
            设置sysFlag变量等于byteBuffer.getInt()                                        // 系统标识（是否压缩、是否是MultiTag，事物类型（非事物、事物提交型、事物回滚型、预提交型））
            设置bornTimeStamp变量等于byteBuffer.getLong()                                 // 消息创建时间
            设置bornTimeStamp变量等于bornTimeStamp + 0
            执行byteBuffer.get(bytesContent, 0, 8)                                       // 发送消息的Socket
            设置storeTimestamp变量等于byteBuffer.getLong()                                // 消息存储时间
            执行byteBuffer.get(bytesContent, 0, 8)                                      // 存储消息的Socket
            设置reconsumeTimes变量等于byteBuffer.getInt()                                // 重消费次数
            设置preparedTransactionOffset变量等于byteBuffer.getLong()                    // 实际存储的是物理队列offset（文件级别的position）
            设置bodyLen变量等于byteBuffer.getInt()                                       // 消息体长度
            如果bodyLen变量大于0                                                         // 如果消息体长度大于0且readBody等于true，读取消息体，根据配置决定是否和存消息时的crc进行校验，如果不等，返回-1表示文件有问题
                如果readBody变量等于true
                    执行byteBuffer.get(bytesContent, 0, bodyLen)
                    如果checkCRC变量等于true
                        设置crc变量等于UtilAll.crc32(bytesContent, 0, bodyLen)
                        如果crc变量不等于bodyCRC变量
                            返回new DispatchRequest(-1)实例
                否则
                    执行byteBuffer.position(byteBuffer.position() + bodyLen)
            设置topicLen变量等于byteBuffer.get()                                         // 主题长度
            执行byteBuffer.get(bytesContent, 0, topicLen)
            设置topic变量等于new String(bytesContent, 0, topicLen)                       // 生成消息主题
            设置tagsCode变量等于0
            设置keys变量等于""
            设置propertiesLength变量等于byteBuffer.getShort()                            // properties长度
            如果propertiesLength变量大于0
                // 读取properties并解序列化，获取KEYS和TAGS属性值，根据sysFlag和tags设置tagsCode变量
                执行byteBuffer.get(bytesContent, 0, propertiesLength)
                设置properties变量等于new String(bytesContent, 0, propertiesLength)
                设置propertiesMap变量等于MessageDecoder.string2messageProperties(properties)
                设置keys变量等于propertiesMap.get("KEYS")
                设置tags变量等于propertiesMap.get("TAGS")
                如果tags变量不等于null并且tags变量元素个数大于0
                    设置tagsCode变量等于MessageExtBrokerInner.tagsString2tagsCode(MessageExt.parseTopicFilterType(sysFlag), tags)     // 转成tagsCode
                // 如果是延迟消息，根据延迟级别和消息存储时间，重新设置tagCode
                {
                    设置t变量等于propertiesMap.get("DELAY")
                    如果topic变量等于"SCHEDULE_TOPIC_XXXX"并且t变量不等于null              // 如果是延迟消息，
                        设置delayLevel变量等于Integer.parseInt(t)
                        如果delayLevel变量大于defaultMessageStore.scheduleMessageService.maxDelayLevel
                            设置delayLevel变量等于defaultMessageStore.scheduleMessageService.maxDelayLevel
                        如果delayLevel变量大于0
                            设置tagsCode变量等于defaultMessageStore.scheduleMessageService.computeDeliverTimestamp(delayLevel, storeTimestamp)
                }
            // 返回带有正常消息长度的信息
            返回new DispatchRequest(topic, queueId, physicOffset, totalSize, tagsCode, storeTimestamp, queueOffset, keys, sysFlag, preparedTransactionOffset)实例
        } catch (BufferUnderflowException e) {
            执行byteBuffer.position(byteBuffer.limit())
        } catch (Exception e) {
            执行byteBuffer.position(byteBuffer.limit())
        }
        返回new DispatchRequest(-1)实例                                                 // 解析出错，返回-1长度
    // 非正常关闭的恢复
    public void recoverAbnormally()
        设置checkCRCOnRecover变量等于defaultMessageStore.messageStoreConfig.checkCRCOnRecover                     // 是否校验CRC，默认值: true
        设置mapedFiles变量等于mapedFileQueue.mapedFiles
        如果mapedFiles变量元素个数大于0
            设置index变量等于mapedFiles.size() - 1                    // 从最后1个开始还原
            设置mapedFile变量等于null
            for (; index >= 0; index--) {
                设置mapedFile变量等于mapedFiles.get(index)
                如果isMapedFileMatchedRecover(mapedFile)等于true     // 如果最后一个文件合理，即从最后的文件开始还原，否则倒退到前一个文件，一次进行
                    退出循环
            }
            如果index变量小于0
                设置index变量等于0
                设置mapedFile变量等于mapedFiles.get(index)           // 获取对应的MapedFile
            设置byteBuffer变量等于mapedFile.sliceByteBuffer()
            设置processOffset变量等于mapedFile.fileFromOffset        // 获取开始位置
            设置mapedFileOffset变量等于0                             // 记录解析得到的消息总长度
            while (true) {
                // 验证消息格式是否合理
                设置dispatchRequest变量等于checkMessageAndReturnSize(byteBuffer, checkCRCOnRecover)
                设置size变量等于dispatchRequest.msgSize
                如果size变量大于0                                    // 获取消息大小，增加mapedFileOffset
                    设置mapedFileOffset变量等于mapedFileOffset + size
                    执行defaultMessageStore.putDispatchRequest(dispatchRequest)
                否则如果size变量等于-1
                    退出循环
                否则如果size变量等于0                                 // 如果为0，标识当前文件解析完毕，开始解析下一个文件
                    设置index++
                    如果index变量大于等于mapedFiles.size()
                        跳出循环
                    否则
                        设置mapedFile变量等于mapedFiles.get(index)
                        设置byteBuffer变量等于mapedFile.sliceByteBuffer()
                        设置processOffset变量等于mapedFile.fileFromOffset
                        设置mapedFileOffset变量等于0
            }
            // 基于文件开始位置加上解析得到的消息总长度，设置commited位置，同时，删除所有文件结束位置超过processOffset的文件
            设置processOffset变量等于processOffset + mapedFileOffset
            执行mapedFileQueue.setCommittedWhere(processOffset)
            执行mapedFileQueue.truncateDirtyFiles(processOffset)
            // 根据物理队列的processOffset，删除所有逻辑队列中的无效逻辑文件
            执行defaultMessageStore.truncateDirtyLogicFiles(processOffset)
        否则
            // 重头开始了
            执行mapedFileQueue.setCommittedWhere(0)
            // 废弃所有逻辑队列（删除对应的文件，offset重新置位）
            执行defaultMessageStore.destroyLogics()
    // 根据文件中第一条消息的MagicCode是否合理，以及第一条消息的存储时间是否匹配checkpoint，判断文件是否正常
    private boolean isMapedFileMatchedRecover(MapedFile mapedFile)
        设置byteBuffer变量等于mapedFile.sliceByteBuffer()
        设置magicCode变量等于byteBuffer.getInt(4)                     // MessageMagicCodePostion: 4
        如果magicCode变量不等于MessageMagicCode
            返回false
        设置storeTimestamp变量等于byteBuffer.getLong(56)              // MessageStoreTimestampPostion: 56
        如果storeTimestamp变量等于0
            返回false
        // messageIndexEnable表示是否开启消息索引，默认值等于true
        // messageIndexSafe表示是否使用安全的消息索引功能，即可靠模式。可靠模式下，异常宕机恢复慢。非可靠模式下，异常宕机恢复快，默认值等于false
        如果defaultMessageStore.messageStoreConfig.messageIndexEnable等于true并且defaultMessageStore.messageStoreConfig.messageIndexSafe等于true
            如果storeTimestamp小于等于defaultMessageStore.storeCheckpoint.getMinTimestampIndex()      // 小于等于最后的索引时间
                返回true
        否则
            如果storeTimestamp小于等于defaultMessageStore.storeCheckpoint.getMinTimestamp()           // 小于等于最小时间戳(物理队列或逻辑队列的时间，谁小用谁，同时减去3秒)
                返回true
        返回false
    public void setTopicQueueTable(HashMap<String, Long> topicQueueTable)
        设置topicQueueTable属性等于topicQueueTable参数
    // 如果物理队列的第一个文件可用，返回第一个文件的开始文件，否则返回第二个文件的开始位置
    public long getMinOffset()
        设置mapedFile变量等于mapedFileQueue.getFirstMapedFileOnLock()
        如果mapedFile变量不等于null
            如果mapedFile.isAvailable()等于true
                执行mapedFile.getFileFromOffset()并返回
            否则
                执行rollNextFile(mapedFile.getFileFromOffset())并返回
        返回-1
    // offset对应的文件的下一个文件的开始位置
    public long rollNextFile(long offset)
        设置mapedFileSize变量等于defaultMessageStore.messageStoreConfig.mapedFileSizeCommitLog
        返回offset + mapedFileSize - offset % mapedFileSize
    // 返回物理队列中，最后一个文件的开始位置 + 最后一个文件当前write的位置
    public long getMaxOffset()
        执行mapedFileQueue.getMaxOffset()
    // 启动刷盘服务
    public void start() throws Exception
        执行flushCommitLogService.start()
    // 关闭刷盘服务
    public void shutdown()
        执行flushCommitLogService.shutdown()
    // 移除逻辑队列和对应的maxOffset信息
    public void removeQueurFromTopicQueueTable(String topic, int queueId)
        synchronized (this)
            执行topicQueueTable.remove(topic + "-" + queueId)
        }
    // 根据offset获取对应的物理队列文件中，offset % 文件大小的位置后边size个byte内容
    public SelectMapedBufferResult getMessage(long offset, int size)
        设置mapedFileSize变量等于defaultMessageStore.messageStoreConfig.mapedFileSizeCommitLog
        设置mapedFile变量等于mapedFileQueue.findMapedFileByOffset(offset, (0 == offset ? true : false))
        如果mapedFile变量不等于null
            设置pos变量等于offset % mapedFileSize
            执行mapedFile.selectMapedBuffer(pos, size)并返回
        返回null
    // 存消息
    public PutMessageResult putMessage(MessageExtBrokerInner msg)
        执行msg.setStoreTimestamp(System.currentTimeMillis())                         // 设置存储时间
        执行msg.setBodyCRC(UtilAll.crc32(msg.getBody()))                              // 设置消息体的crc
        设置result变量等于null
        设置topic变量等于msg.topic
        设置queueId变量等于msg.queueId
        设置tagsCode变量等于msg.tagsCode
        设置tranType变量等于MessageSysFlag.getTransactionValue(msg.sysFlag)
        // 如果是非事务性和事物提交型消息，如果是延迟消息，根据延迟级别重新设置queueId和topic，设置tagsCode等于延迟到期时间
        如果tranType变量等于0或者8                      // TransactionNotType: 0        TransactionCommitType: 8
            如果msg.getDelayTimeLevel()变量大于0
                如果msg.getDelayTimeLevel()大于defaultMessageStore.scheduleMessageService.maxDelayLevel
                    执行msg.setDelayTimeLevel(defaultMessageStore.scheduleMessageService.maxDelayLevel)
                设置topic变量等于SCHEDULE_TOPIC_XXXX
                设置queueId变量等于ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel())
                设置tagsCode变量等于defaultMessageStore.scheduleMessageService.computeDeliverTimestamp(msg.getDelayTimeLevel(), msg.storeTimestamp)
                执行msg.properties.put("REAL_TOPIC", msg.topic)
                执行msg.properties.put("REAL_QID", msg.queueId)
                执行msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.properties))
                执行msg.setTopic(topic)
                执行msg.setQueueId(queueId)
        设置eclipseTimeInLock变量等于0
        synchronized (this) {
            设置beginLockTimestamp变量等于defaultMessageStore.systemClock.now()
            执行msg.setStoreTimestamp(beginLockTimestamp)                             // 设置存储时间
            设置mapedFile变量等于mapedFileQueue.getLastMapedFile()
            如果mapedFile变量等于null
                返回new PutMessageResult(CREATE_MAPEDFILE_FAILED, null)实例
            设置result变量等于mapedFile.appendMessage(msg, appendMessageCallback)      // 存消息到最后一个MapedFile
            如果result.status等于PUT_OK

            否则如果result.status等于END_OF_FILE                                       // 如果END_OF_FILE，重新存消息到最后一个MapedFile
                设置mapedFile变量等于mapedFileQueue.getLastMapedFile()
                如果mapedFile变量等于null
                    返回new PutMessageResult(CREATE_MAPEDFILE_FAILED, null)实例
                设置result变量等于mapedFile.appendMessage(msg, appendMessageCallback)
            否则如果result.status等于MESSAGE_SIZE_EXCEEDED                            // 消息太大
                返回new PutMessageResult(MESSAGE_ILLEGAL, result)实例
            否则如果result.status等于UNKNOWN_ERROR
                返回new PutMessageResult(UNKNOWN_ERROR, result)实例
            否则
                返回new PutMessageResult(UNKNOWN_ERROR, result)实例
            // 创建分发请求，put到分发服务里
            设置dispatchRequest变量等于new DispatchRequest(topic, queueId, result.wroteOffset, result.wroteBytes, tagsCode, msg.storeTimestamp, result.logicsOffset, msg.properties.get("KEYS"), msg.sysFlag, msg.preparedTransactionOffset)
            执行defaultMessageStore.putDispatchRequest(dispatchRequest)
        }
        设置putMessageResult变量等于new PutMessageResult(PUT_OK, result)
        执行defaultMessageStore.storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.wroteBytes)
        设置request变量等于null
        // 触发刷盘任务，对于flushDiskType等于SYNC_FLUSH场景且isWaitStoreMsgOK()等于true场景，进行等到
        // flushDiskType默认值等于ASYNC_FLUSH
        // 正常消息的isWaitStoreMsgOK()等于true
        如果defaultMessageStore.messageStoreConfig.flushDiskType等于SYNC_FLUSH
            如果msg.isWaitStoreMsgOK()等于true
                设置request变量等于new GroupCommitRequest(result.wroteOffset + result.wroteBytes)
                执行flushCommitLogService.putRequest(request)
                设置flushOK变量等于request.waitForFlush(defaultMessageStore.messageStoreConfig.syncFlushTimeout)
                如果flushOK变量等于false
                    执行putMessageResult.setPutMessageStatus(FLUSH_DISK_TIMEOUT)
            否则
                执行flushCommitLogService.wakeup()
        否则
            执行flushCommitLogService.wakeup()
        // 如果brokerRole等于SYNC_MASTER，对于isWaitStoreMsgOK()场景，会等待slave接收消息（maser端针对每个slave会启动一个线程频繁的读取物理队列内容发送到slave）
        // brokerRole默认值等于ASYNC_MASTER
        如果defaultMessageStore.messageStoreConfig.brokerRole等于SYNC_MASTER
            设置service变量等于defaultMessageStore.haService
            如果msg.isWaitStoreMsgOK()等于true
                如果service.isSlaveOK(result.wroteOffset + result.wroteBytes)变量等于true
                    如果request变量等于null
                        设置request变量等于new GroupCommitRequest(result.wroteOffset + result.wroteBytes)
                    执行service.putRequest(request)
                    执行service.waitNotifyObject.wakeupAll()
                    设置flushOK变量等于request.waitForFlush(defaultMessageStore.messageStoreConfig.syncFlushTimeout)
                    如果flushOK变量等于false
                        执行putMessageResult.setPutMessageStatus(FLUSH_SLAVE_TIMEOUT)
                否则
                    执行putMessageResult.setPutMessageStatus(SLAVE_NOT_AVAILABLE)
        返回putMessageResult变量
    // 如果物理队列中，第一个文件不可用，则废弃和删除
    public boolean retryDeleteFirstFile(long intervalForcibly)
        执行mapedFileQueue.retryDeleteFirstFile(intervalForcibly)并返回
    // 获取物理队列中offset对应的mapedFile，以offset % 文件大小作为起始位置，文钱mapedFile的写位置作为结束位置的byteBuffer
    public SelectMapedBufferResult getData(long offset)
        执行getData(offset, (0 == offset ? true : false))并返回
    // 获取物理队列中offset对应的mapedFile，以offset % 文件大小作为起始位置，文钱mapedFile的写位置作为结束位置的byteBuffer
    public SelectMapedBufferResult getData(long offset, boolean returnFirstOnNotFound)
        设置mapedFileSize变量等于messageStoreConfig.mapedFileSizeCommitLog
        设置mapedFile变量等于mapedFileQueue.findMapedFileByOffset(offset, returnFirstOnNotFound)
        如果mapedFile变量不等于null
            设置pos变量等于offset % mapedFileSize
            执行mapedFile.selectMapedBuffer(pos)并返回
        返回null
    // 追加data数据到最后文件
    public boolean appendData(long startOffset, byte[] data)
        synchronized (this) {
            设置mapedFile变量等于mapedFileQueue.getLastMapedFile(startOffset)
            如果mapedFile变量等于null
                返回false
            执行mapedFile.appendMessage(data)并返回
        }
    // 根据过期时间，删除物理队列文件
    public int deleteExpiredFile(long expiredTime, int deleteFilesInterval, long intervalForcibly, boolean cleanImmediately)
        执行mapedFileQueue.deleteExpiredFileByTime(expiredTime, deleteFilesInterval, intervalForcibly, cleanImmediately)并返回
    // 返回物理队列offset对应的消息的存储时间
    public long pickupStoretimestamp(long offset, int size)
        如果offset参数大于getMinOffset()
            设置result变量等于getMessage(offset, size)
            如果result变量不等于null
                try {
                    执行result.byteBuffer.getLong(56)并返回
                } finally {
                    执行result.release()
                }
        返回-1

## com.alibaba.rocketmq.store.CommitLog.FlushRealTimeService
    // 提供了异步实时刷盘机制

    public void start()
    public void shutdown()
    public void run()
        循环，直到stop属性等于true
            设置flushCommitLogTimed变量等于defaultMessageStore.messageStoreConfig.flushCommitLogTimed                             // 是否定时方式刷盘，默认值: false，标识实时刷盘
            设置interval变量等于defaultMessageStore.messageStoreConfig.flushIntervalCommitLog                                     // 刷盘间隔时间，默认值: 1000
            设置flushPhysicQueueLeastPages变量等于defaultMessageStore.messageStoreConfig.flushCommitLogLeastPages                 // 至少刷几个PAGE，默认值: 4
            设置flushPhysicQueueThoroughInterval变量等于defaultMessageStore.messageStoreConfig.flushCommitLogThoroughInterval     // 彻底刷盘间隔时间，默认值: 1000 * 10
            设置currentTimeMillis变量等于系统时间戳
            如果lastFlushTimestamp属性 + flushPhysicQueueThoroughInterval变量小于currentTimeMillis变量      // 类似于每隔flushPhysicQueueThoroughInterval时间进行一次输盘，不是绝对的
                设置lastFlushTimestamp属性等于currentTimeMillis变量
                设置flushPhysicQueueLeastPages变量等于0
            try {
                如果flushCommitLogTimed变量等于true                                                       // 定时刷盘，wakeup不起作用
                    Thread.sleep(interval)
                否则                                                                                     // 实时刷盘，可以依赖于wakeup
                    如果hasNotified属性等于true
                        设置hasNotified属性等于false
                    否则
                        wait(interval)
                        设置hasNotified属性等于false
                执行mapedFileQueue.commit(flushPhysicQueueLeastPages)                                     // 提交
                设置storeTimestamp变量等于mapedFileQueue.storeTimestamp
                如果storeTimestamp变量大于0
                    执行defaultMessageStore.storeCheckpoint.setPhysicMsgTimestamp(storeTimestamp)
            } catch (Exception e) {
            }
        循环3次
            如果mapedFileQueue.commit(0)等于true
                退出循环
    public void wakeup()
        如果hasNotified属性等于false
            设置hasNotified属性等于true
            执行notify()

## com.alibaba.rocketmq.store.CommitLog.GroupCommitService
    // 也是异步实时刷盘机制，比FlushRealTimeService的区别，在于1.几乎间隔，2.每次都是全刷，3.等待请求完成

    private volatile List<GroupCommitRequest> requestsWrite = new ArrayList<GroupCommitRequest>()
    private volatile List<GroupCommitRequest> requestsRead = new ArrayList<GroupCommitRequest>()

    // 加入刷盘请求到写列表，然后立即触发执行读请求
    public void putRequest(GroupCommitRequest request)
        synchronized (this) {
            执行requestsWrite.add(dispatchRequest)
            如果hasNotified属性等于false
                设置hasNotified属性等于true
                执行notify()
        }
    public void start()
    public void shundown()
    public void run()
        循环，直到stop属性等于true
            try {
                如果hasNotified属性等于true
                    设置hasNotified属性等于false
                    执行swapRequests()
                否则
                    try {
                        wait(0)
                    } catch (InterruptedException e) {
                    } finally {
                        设置hasNotified属性等于false
                        执行swapRequests()
                    }
                执行doCommit()                            // 处理请求
            } catch (Exception e) {
            }
        try {
            Thread.sleep(5 * 1000)
        } catch (InterruptedException e) {
        }
        synchronized (this) {
            执行swapRequests()
        }
        执行doCommit()
    // 写请求列表和读请求列表互换
    private void swapRequests()
        设置tmp变量等于requestsWrite属性
        设置requestsWrite属性等于requestsRead属性
        设置requestsRead属性等于tmp变量
    // 处理所有读请求
    private void doCommit()
        如果requestsRead属性元素个数大于0
            遍历requestsRead属性，设置req变量等于当前元素
                设置flushOK变量等于false
                for (int i = 0; (i < 2) && !flushOK; i++) {                                 // 最多flush两次
                    设置flushOK变量等于mapedFileQueue.committedWhere >= req.nextOffset        // 大于等于re.nextOffset表示之前或当前已经flush成功
                    如果flushOK变量等于false
                        执行mapedFileQueue.commit(0)                                        // 不是刷几个PAGE，是全刷
                }
                执行req.wakeupCustomer(flushOK)
            设置storeTimestamp变量等于mapedFileQueue.storeTimestamp
            如果storeTimestamp变量大于0
                defaultMessageStore.storeCheckpoint.setPhysicMsgTimestamp(storeTimestamp)   // 更新物理队列最后存储时间
            执行requestsRead.clear()                                                        // 清空读请求列表
        否则
            执行mapedFileQueue.commit(0)                                                    // 不是刷几个PAGE，是全刷
    public void wakeup()
        如果hasNotified属性等于false
            设置hasNotified属性等于true
            执行notify()

## com.alibaba.rocketmq.store.CommitLog.DefaultAppendMessageCallback
    // 存储消息，根据分配的地址存储消息

    DefaultAppendMessageCallback(int size)
        设置msgIdMemory属性等于ByteBuffer.allocate(16)                        // 消息ID的byteBuffer，8个byte存储当前host:port，8个byte存储在物理队列的offset
        设置msgStoreItemMemory属性等于ByteBuffer.allocate(size + 8)           // size参数表示的是最大消息大小，默认值: 1024 * 512，+8是因为每个文件都有结束标识，占用8个byte
        设置maxMessageSize属性等于size参数
    public AppendMessageResult doAppend(long fileFromOffset, ByteBuffer byteBuffer, int maxBlank, Object msg)
        设置msgInner变量等于(MessageExtBrokerInner) msg
        设置wroteOffset变量等于fileFromOffset + byteBuffer.position()
        设置msgId变量等于MessageDecoder.createMessageId(msgIdMemory, msgInner.getStoreHostBytes(), wroteOffset)           // 根据当前BrokerServer和物理队列offset创建messageId
        设置key变量等于msgInner.topic + "-" + msgInner.queueId
        设置queueOffset变量等于topicQueueTable.get(key)                                                                   // 设置逻辑队列offset
        如果queueOffset变量等于null
            设置queueOffset变量等于0
            执行topicQueueTable.put(key, queueOffset)
        设置tranType变量等于MessageSysFlag.getTransactionValue(msgInner.sysFlag)                                          // 对于回滚型事物消息和预提交型事物，设置逻辑队列offset等于0（这俩类型不参与分发逻辑，所以逻辑队列offset没意义）
        如果tranType变量等于4或12                         // TransactionPreparedType: 4       TransactionRollbackType: 12
            设置queueOffset变量等于0
        计算消息字节长度设置给msgLen变量
        如果msgLen变量大于maxMessageSize属性                                                                               // 验证消息长度
            返回new AppendMessageResult(MESSAGE_SIZE_EXCEEDED)实例
        如果msgLen变量 + 8大于maxBlank变量                                                                                 // 如果+8超过了文件长度，写入文件结束标识，返回END_OF_FILE，下一次执行getLastMapedFile时会返回新的
            执行msgStoreItemMemory.flip()
            执行msgStoreItemMemory.limit(maxBlank)
            执行msgStoreItemMemory.putInt(maxBlank)
            执行msgStoreItemMemory.putInt(CommitLog.BlankMagicCode)
            执行byteBuffer.put(msgStoreItemMemory.array(), 0, maxBlank)
            返回new AppendMessageResult(END_OF_FILE, wroteOffset, maxBlank, msgId, msgInner.storeTimestamp, queueOffset)实例
        执行msgStoreItemMemory.flip()
        执行msgStoreItemMemory.limit(msgLen)
        写入消息到msgStoreItemMemory属性
        执行byteBuffer.put(msgStoreItemMemory.array(), 0, msgLen)                                                        // 写入消息
        设置result变量等于new AppendMessageResult(PUT_OK, wroteOffset, msgLen, msgId, msgInner.storeTimestamp, queueOffset)实例
        如果tranType变量等于0或者8                         // TransactionNotType: 0          TransactionCommitType: 8      // 对于非事务性和提交型事物消息，逻辑队列offset++
            执行topicQueueTable.put(key, ++queueOffset)
        返回result变量

# com.alibaba.rocketmq.store.MapedFileQueue

    public void setCommittedWhere(long committedWhere)

    public MapedFileQueue(String storePath, int mapedFileSize, AllocateMapedFileService allocateMapedFileService)
        设置storePath属性等于storePath参数
        设置mapedFileSize属性等于mapedFileSize参数
        设置allocateMapedFileService属性等于allocateMapedFileService参数
    // 加载文件
    public boolean load()
        获取storePath目录下的所有子文件，设置给files变量
        如果files变量不等于null，重新排序，并进行遍历，设置file变量等于当前元素
            如果file.length()不等于mapedFileSize属性
                返回true
            // 最新的文件writePosition和commitedPosition可能不是mapedFileSize，会调动recover的时候执行truncateDirtyFiles进行恢复
            try {
                设置mapedFile变量等于new MapedFile(file.getPath(), mapedFileSize)实例
                执行mapedFile.setWrotePostion(mapedFileSize)
                执行mapedFile.setCommittedPosition(mapedFileSize)
                执行mapedFiles.add(mapedFile)
            } catch (IOException e) {
                return false
            }
        返回true
    // 删除所有fileFromOffset大于offset的队列文件，设置最后一个文件的writePosition和commitedPosition
    public void truncateDirtyFiles(long offset)
        设置willRemoveFiles变量等于new ArrayList<MapedFile>()实例
        遍历mapedFiles属性，设置file变量等于当前元素
            设置fileTailOffset变量等于file.fileFromOffset + mapedFileSize
            如果fileTailOffset变量大于offset参数
                如果offset参数大于等于file.fileFromOffset
                    执行file.setWrotePostion((int) (offset % mapedFileSize))
                    执行file.setCommittedPosition((int) (offset % mapedFileSize))
                否则
                    执行file.destroy(1000)                        // 废弃文件
                    执行willRemoveFiles.add(file)
        执行deleteExpiredFile(willRemoveFiles)
    // 删除过期文件从mapedFiles中
    private void deleteExpiredFile(List<MapedFile> files)
        如果files参数元素个数大于0
            try {
                readWriteLock.writeLock().lock()
                遍历fiels参数，设置file变量等于当前元素
                    如果mapedFiles.remove(file)等于false
                        退出循环
            } catch (Exception e) {
            } finally {
                readWriteLock.writeLock().unlock()
            }
    // 返回第一个mapedFile文件
    public MapedFile getFirstMapedFileOnLock()
        try {
            readWriteLock.readLock().lock()
            执行getFirstMapedFile()并返回
        } finally {
            readWriteLock.readLock().unlock()
        }
    // 返回第一个mapedFile文件
    private MapedFile getFirstMapedFile()
        如果mapedFiles属性元素个数等于0
            返回null
        执行mapedFiles.get(0)
    // 文件最后一个文件的开始offset + 最后一个文件的写入位置
    public long getMaxOffset()
        try {
            readWriteLock.readLock().lock()
            如果mapedFiles属性元素个数大于0
                设置mapedFile变量等于mapedFiles.get(mapedFiles.size() - 1)
                返回mapedFile.fileFromOffset + mapedFile.wrotePostion.get()
        } catch (Exception e) {
        } finally {
            readWriteLock.readLock().unlock()
        }
        返回0
    // 返回offset对应的文件，如果找不到且returnFirstOnNotFound参数等于true，返回第一个文件
    public MapedFile findMapedFileByOffset(long offset, boolean returnFirstOnNotFound)
        try {
            readWriteLock.readLock().lock()
            设置mapedFile变量等于getFirstMapedFile()
            如果mapedFile变量不等于null
                设置index变量等于(int) ((offset / mapedFileSize) - (mapedFile.getFileFromOffset() / mapedFileSize))
                try {
                    返回mapedFiles.get(index)
                } catch (Exception e) {
                    如果returnFirstOnNotFound等于true
                        返回mapedFile变量
                }
        } catch (Exception e) {
        } finally {
            readWriteLock.readLock().unlock()
        }
        返回null
    // 返回最新的文件
    public MapedFile getLastMapedFile()
        执行getLastMapedFile(0)并返回
    // 如果文件列表不为空，并且最后一个文件isFull()等于false，则返回最后一个文件
    // 如果为见列表为空，获取startOffset对应的文件(startOffset - startOffset % mapedFileSize)，创建当前文件和下一个文件，添加当前文件到mapedFiles列表中
    // 如果最后一个文件isFull()等于true，创建下一个文件，和下下一个文件，添加下一个文件到mapedFiles列表中
    public MapedFile getLastMapedFile(long startOffset)
        设置createOffset变量等于-1
        设置mapedFileLast变量等于null
        {
            readWriteLock.readLock().lock()
            如果mapedFiles属性元素个数等于0
                设置createOffset变量等于startOffset - (startOffset % mapedFileSize)
            否则
                设置mapedFileLast变量等于mapedFiles.get(mapedFiles.size() - 1)
            readWriteLock.readLock().unlock()
        }
        如果mapedFileLast变量不等于null并且mapedFileLast.isFull()等于true
            设置createOffset变量等于mapedFileLast.fileFromOffset + mapedFileSize
        如果createOffset变量不等于-1
            设置nextFilePath变量等于storePath/UtilAll.offset2FileName(createOffset)
            设置nextNextFilePath变量等于storePath/UtilAll.offset2FileName(createOffset + mapedFileSize)
            设置mapedFile变量等于null
            如果allocateMapedFileService不等于null
                设置mapedFile变量等于allocateMapedFileService.putRequestAndReturnMapedFile(nextFilePath, nextNextFilePath, mapedFileSize)
            否则
                try {
                    设置mapedFile变量等于new MapedFile(nextFilePath, mapedFileSize)实例
                } catch (IOException e) {
                }
            如果mapedFile变量不等于null
                readWriteLock.writeLock().lock()
                如果mapedFiles属性元素个数等于0
                    执行mapedFile.setFirstCreateInQueue(true)
                执行mapedFiles.add(mapedFile)
                readWriteLock.writeLock().unlock()
            返回mapedFile变量
        返回mapedFileLast变量
    // 如果第一个文件不可用，则废弃和删除
    public boolean retryDeleteFirstFile(long intervalForcibly)
        设置mapedFile变量等于getFirstMapedFileOnLock()
        如果mapedFile变量不等于null
            如果mapedFile.isAvailable()等于false
                设置result变量等于mapedFile.destroy(intervalForcibly)
                如果result变量等于true
                    执行deleteExpiredFile(Arrays.asList(mapedFile))
                返回result变量
        返回false
    // 执行刷盘，获取刷盘位置对应的mapedFile，如果flushLeastPages大于0且write - commit < flushLeastPages * 4096，则不需要刷盘，此时，返回true，否则进行刷盘，最终会返回false
    public boolean commit(int flushLeastPages)
        设置result变量等于true
        设置mapedFile变量等于findMapedFileByOffset(committedWhere, true)
        如果mapedFile变量不等于null
            设置tmpTimeStamp变量等于mapedFile.storeTimestamp
            设置offset变量等于mapedFile.commit(flushLeastPages)
            设置where变量等于mapedFile.fileFromOffset + offset
            设置result变量等于where == committedWhere             // ??是不是应该用不等进行判断
            设置committedWhere属性等于where
            如果flushLeastPages变量等于0
                设置storeTimestamp属性等于tmpTimeStamp变量
        返回result变量
    // 废弃和删除最后一个文件
    public void deleteLastMapedFile()
        如果mapedFiles属性元素个数大于0
            设置mapedFile变量等于mapedFiles.get(mapedFiles.size() - 1)
            执行mapedFile.destroy(1000)
            执行mapedFiles.remove(mapedFile)
    // 返回最后一个文件
    public MapedFile getLastMapedFile2()
        如果mapedFiles属性元素个数等于0
            返回null
        返回mapedFiles.get(mapedFiles.size() - 1)
    // 返回所有文件，依次进行判断，如果发现某个文件的最近更改时间大于等于timestamp，则返回，否则返回最后一个文件
    public MapedFile getMapedFileByTime(long timestamp)
        设置mfs变量等于copyMapedFiles(0)
        如果mfs变量等于null
            返回null
        for (int i = 0; i < mfs.length; i++) {
            设置mapedFile变量等于mfs[i]
            如果mapedFile.getLastModifiedTimestamp()大于等于timestamp参数
                返回mapedFile变量
        }
        返回mfs[mfs.length - 1]
    // unitSize基本可以认定是20，改方法是逻辑队列专用的
    // 返回所有文件，遍历到倒数第二个，如果发现某个文件的最后一条逻辑消息的offset小于offset，则删除过期文件从mapedFiles中
    public int deleteExpiredFileByOffset(long offset, int unitSize)
        设置mfs变量等于copyMapedFiles(0)
        设置deleteCount变量等于0
        设置files变量等于new ArrayList<MapedFile>()实例
        如果mfs变量不等于null
            设置mfsLength变量等于mfs.length - 1
            for (int i = 0; i < mfsLength; i++) {
                设置destroy变量等于true
                设置mapedFile变量等于mfs[i]
                设置result变量等于mapedFile.selectMapedBuffer(mapedFileSize - unitSize)
                如果result变量不等于null
                    设置maxOffsetInLogicQueue变量等于result.byteBuffer.getLong()
                    执行result.release()
                    设置destroy变量等于maxOffsetInLogicQueue < offset
                否则
                    退出循环
                如果destroy变量等于true并且mapedFile.destroy(1000 * 60)等于true
                    执行files.add(mapedFile)
                    执行deleteCount++
                否则
                    退出循环
            }
        执行deleteExpiredFile(files)
        返回deleteCount变量
    // 返回所有文件，依次遍历，如果cleanImmediately变量等于true或者文件的上次修改和当前系统时间戳的差值大于expiredTime，废弃文件和删除过期文件从mapedFiles中
    // 最多删除10个
    public int deleteExpiredFileByTime(long expiredTime, int deleteFilesInterval, long intervalForcibly, boolean cleanImmediately)
        设置mfs变量等于copyMapedFiles(0)
        如果mfs变量等于null
            返回0
        设置mfsLength变量等于mfs.length - 1
        设置deleteCount变量等于0
        设置files变量等于new ArrayList<MapedFile>()实例
        for (int i = 0; i < mfsLength; i++) {
            设置mapedFile变量等于mfs[i]
            设置liveMaxTimestamp变量等于mapedFile.getLastModifiedTimestamp() + expiredTime
            如果当前系统时间戳大于等于liveMaxTimestamp变量或者cleanImmediately属性等于true
                如果mapedFile.destroy(intervalForcibly)等于true
                    执行files.add(mapedFile)
                    执行deleteCount++
                    如果files.size()大于等于10
                        退出循环
                    如果deleteFilesInterval变量大于0并且i + 1小于mfsLength变量
                        try {
                            Thread.sleep(deleteFilesInterval);
                        } catch (InterruptedException e) {
                        }
                否则
                    退出循环
        }
        执行deleteExpiredFile(files)
        返回deleteCount变量
    // 如果文件个数大于reservedMapedFiles，返回所有文件，否则，返回null
    private Object[] copyMapedFiles(int reservedMapedFiles)
        设置mfs变量等于null
        try {
            readWriteLock.readLock().lock()
            如果mapedFiles属性元素个数小于等于reservedMapedFiles参数
                返回null
            设置mfs变量等于mapedFiles.toArray()
        } catch (Exception e) {
        } finally {
            readWriteLock.readLock().unlock()
        }
        返回mfs变量
    // 废弃所有文件
    public void destroy()
        执行readWriteLock.writeLock().lock()
        遍历mapedFiles属性，设置mf变量等于当前元素
            执行mf.destroy(1000 * 3)
        执行mapedFiles.clear()
        设置committedWhere属性等于0
        删除storePath目录
        执行readWriteLock.writeLock().unlock()

# com.alibaba.rocketmq.store.ConsumeQueue

    public ConsumeQueue(String topic, int queueId, String storePath, int mapedFileSize, DefaultMessageStore defaultMessageStore)
        设置storePath属性等于storePath参数
        设置mapedFileSize属性等于mapedFileSize参数
        设置defaultMessageStore属性等于defaultMessageStore参数
        设置topic属性等于topic参数
        设置queueId属性等于queueId参数
        设置mapedFileQueue属性等于new MapedFileQueue(storePath/topic/queueId, mapedFileSize, null)实例
        设置byteBufferIndex属性等于ByteBuffer.allocate(20)实例                  // 逻辑队列存储的是对物理队列的应用，每个消息消耗20个byte
    // 加载队列文件
    public boolean load()
        执行mapedFileQueue.load()并返回
    // 恢复操作
    public void recover()
        设置mapedFiles变量等于mapedFileQueue.mapedFiles
        如果mapedFiles变量元素个数大于0
            设置index变量等于mapedFiles.size() - 3                // 从最后3个开始恢复
            如果index变量小于0
                设置index变量等于0
            设置mapedFile变量等于mapedFiles.get(index)                        // 获取对应文件
            设置byteBuffer变量等于mapedFile.sliceByteBuffer()
            设置processOffset变量等于mapedFile.fileFromOffset
            设置mapedFileOffset变量等于0                                      // 记录当前文件解析的消息长度
            while (true) {
                for (int i = 0; i < mapedFileSize; i += 20) {
                    设置offset变量等于byteBuffer.getLong()                    // 物理队列offset
                    设置size变量等于byteBuffer.getInt()                       // 消息大小
                    设置tagsCode变量等于byteBuffer.getLong()                  // 延迟消息使用，记录了最终应该消费的时间

                    如果offset变量大于等于0并且size变量大于0
                        设置mapedFileOffset变量等于i + 20                     // +20
                        设置maxPhysicOffset属性等于offset变量                 // 设置最大物理队列offset
                    否则
                        退出循环                                             // 文件解析完了
                }
                如果mapedFileOffset变量等于mapedFileSize变量                  // 如果完全都解析了，说明当前文件可能不是最新的
                    执行index++
                    如果index变量大于等于mapedFiles.size()                    // 但是，还真是最新的
                        退出循环
                    否则                                                    // 不是最新的，确定了
                        设置mapedFile变量等于mapedFiles.get(index)
                        设置byteBuffer变量等于mapedFile.sliceByteBuffer()
                        设置processOffset变量等于mapedFile.fileFromOffset     // 设置processOffset
                        设置mapedFileOffset变量等于0                          // 重置为0
                否则
                    退出循环
            }
            设置processOffset变量等于processOffset + mapedFileOffset          // 之前的文件最大位置 + 当前文件解析得到的长度
            执行mapedFileQueue.truncateDirtyFiles(processOffset)            // 删除所有fileFromOffset大于processOffset的队列文件，设置最后一个文件的writePosition和commitedPosition
    public SelectMapedBufferResult getIndexBuffer(long startIndex)
        设置mapedFileSize变量等于mapedFileSize属性
        设置offset变量等于startIndex * 20
        如果offset变量大于getMinLogicOffset()
            设置mapedFile变量等于mapedFileQueue.findMapedFileByOffset(offset)
            如果mapedFile变量不等于null
                执行mapedFile.selectMapedBuffer((int) (offset % mapedFileSize))并返回
        返回null
    // 返回逻辑队列中的最小offset
    public long getMinOffsetInQuque()                                       // ??应该是queue
        返回minLogicOffset / 20
    // 返回逻辑队列中的最大offset
    public long getMaxOffsetInQuque()                                       // ??应该是queue
        执行mapedFileQueue.getMaxOffset() / 20
    // 获取第一个文件，并进行解析，获取大于等于物理offset的消息，设置对应文件的起始位置 + 对于当前文件的position为minLogicOffset
    public void correctMinOffset(long phyMinOffset)
        设置mapedFile变量等于mapedFileQueue.getFirstMapedFileOnLock()
        如果mapedFile变量不等于null
            设置result变量等于mapedFile.selectMapedBuffer(0)
            如果result变量不等于null
                try {
                    for (int i = 0; i < result.size; i += 20) {
                        设置offsetPy变量等于result.byteBuffer.getLong()
                        result.byteBuffer.getInt()
                        result.byteBuffer.getLong()
                        如果offsetPy变量大于等于phyMinOffset参数
                            设置minLogicOffset属性等于result.mapedFile.getFileFromOffset() + i
                            退出循环
                    }
                } catch (Exception e) {
                } finally {
                    result.release()
                }
    // 查找消息的存储时间最接近timestamp的消息
    public long getOffsetInQueueByTime(long timestamp)
        设置mapedFile变量等于mapedFileQueue.getMapedFileByTime(timestamp)             // 返回所有文件，依次进行判断，如果发现某个文件的最近更改时间大于等于timestamp，则返回，否则返回最后一个文件
        如果mapedFile变量不等于null
            设置offset变量等于0
            设置low变量等于minLogicOffset > mapedFile.fileFromOffset ? (int) (minLogicOffset - mapedFile.fileFromOffset) : 0          // 设置有消息的开始位置
            设置high变量等于0
            设置midOffset变量、targetOffset变量、leftOffset变量、midOffset变量、rightOffset变量等于-1
            设置leftIndexValue变量等于-1L
            设置rightIndexValue变量等于-1L
            设置sbr变量等于mapedFile.selectMapedBuffer(0)
            如果sbr变量不等于null
                设置byteBuffer变量等于sbr.byteBuffer
                设置high变量等于byteBuffer.limit() - 20               // 最后一条消息的offset
                try {
                    while (high >= low) {
                        设置midOffset变量等于(low + high) / (2 * 20) * 20                                 // 获取中间位置
                        执行byteBuffer.position(midOffset)
                        设置phyOffset变量等于byteBuffer.getLong()                                         // 读取消息的逻辑offset
                        设置size变量等于byteBuffer.getInt()                                               // 读取消息大小
                        设置storeTime变量等于defaultMessageStore.commitLog.pickupStoretimestamp(phyOffset, size)      // 从物理队列获取消息的存储时间
                        如果storeTime变量小于0                                                            //
                            返回0
                        否则如果storeTime变量等于timestamp变量                                            // 恰好等于，不需要再找了
                            设置targetOffset变量等于midOffset变量
                            退出循环
                        否则如果storeTime变量大于timestamp变量                                            // 大于timestamp，调低high
                            设置high变量等于midOffset - 20
                            设置rightOffset变量等于midOffset变量
                            设置rightIndexValue变量等于storeTime变量
                        否则                                                                           // 小于timestamp，调高low
                            设置low变量等于midOffset + 20
                            设置leftOffset变量等于midOffset变量
                            设置leftIndexValue变量等于storeTime变量
                    }

                    如果targetOffset变量不等于-1                                           // 刚好找到匹配的
                        设置offset变量等于targetOffset变量
                    否则
                        如果leftIndexValue变量等于-1                                      // 一直是右右的查找方式
                            设置offset变量等于rightOffset变量
                        否则如果rightIndexValue变量等于-1                                  // 一直是左左的查找方式
                            设置offset变量等于leftOffset变量
                        否则
                            // 谁离的近用谁
                            设置offset变量等于Math.abs(timestamp - leftIndexValue) > Math.abs(timestamp - rightIndexValue) ? rightOffset : leftOffset
                    返回(mapedFile.fileFromOffset + offset) / 20                          // (开始位置 + 位置位置) / 20
                } finally {
                    sbr.release();
                }
        返回0
    // 根据物理队列的最大offset，删除所有无效的逻辑文件
    public void truncateDirtyLogicFiles(long phyOffet)
        设置maxPhysicOffset变量属性等于phyOffet - 1
        while (true) {
            设置mapedFile变量等于mapedFileQueue.getLastMapedFile2()           // 获取最后一个MapedFile
            如果mapedFile变量不等于null
                设置byteBuffer变量等于mapedFile.sliceByteBuffer()
                执行mapedFile.setWrotePostion(0)              // 先设置为0
                执行mapedFile.setCommittedPosition(0)         // 先设置为0
                for (int i = 0; i < mapedFileSize; i += 20)                 // 遍历文件
                    设置offset变量等于byteBuffer.getLong()                    // 返回消息物理队列offset
                    设置size变量等于byteBuffer.getInt()                       // 返回消息大小
                    执行byteBuffer.getLong()
                    如果i变量等于0
                        如果offset变量大于等于phyOffet变量                    // 第一条消息比物理队列的最大offset还大，删除文件
                            执行mapedFileQueue.deleteLastMapedFile()
                            退出循环                                        // 退出解析当前文件的循环了，还有上一个循环呢
                        否则                                               // 比物理队列的最大offset小，设置writePosition和committedPosition
                            设置pos变量等于i + 20
                            执行mapedFile.setWrotePostion(pos)
                            执行mapedFile.setCommittedPosition(pos)
                            设置maxPhysicOffset属性等于offset变量            // 设置最大物理offset属性
                    否则
                        如果offset变量大于等于0并且size变量大于0
                            如果offset变量大于等于phyOffet变量               // 超过物理队列的最大offset，退出方法
                                退出方法
                            // 比物理队列的最大offset小，设置writePosition和committedPosition
                            设置pos变量等于i + 20
                            执行mapedFile.setWrotePostion(pos)
                            执行mapedFile.setCommittedPosition(pos)
                            设置maxPhysicOffset属性等于offset变量           // 设置最大物理offset属性
                            如果pos变量等于mapedFileSize属性                // 下一个位置已经超过文件了，当前文件解析完了，退出方法
                                退出方法
                        否则                                             // 解析完了，退出方法
                            退出方法
                }
            否则
                退出循环
        }
    // 刷盘
    public boolean commit(int flushLeastPages)
        执行mapedFileQueue.commit(flushLeastPages)并返回
    // 存消息
    public void putMessagePostionInfoWrapper(long offset, int size, long tagsCode, long storeTimestamp, long logicOffset)
        设置canWrite变量等于defaultMessageStore.runningFlags.isWriteable()                        // 判断是否有写入权限
        // 在有写权限的前提下，最多试5次
        for (int i = 0; i < 5 && canWrite; i++) {
            设置result变量等于putMessagePostionInfo(offset, size, tagsCode, logicOffset)
            如果result变量等于true
                执行defaultMessageStore.storeCheckpoint.setLogicsMsgTimestamp(storeTimestamp)
                退出方法
            否则
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                }
        }
        执行defaultMessageStore.runningFlags.makeLogicsQueueError()
    // 存消息
    private boolean putMessagePostionInfo(long offset, int size, long tagsCode, long cqOffset)
        如果offset参数小于等于maxPhysicOffset属性
            返回true
        执行byteBufferIndex.flip()
        执行byteBufferIndex.limit(20)                                 // 占用20个字节
        执行byteBufferIndex.putLong(offset)                           // 物理offset 8个byte
        执行byteBufferIndex.putInt(size)                              // 消息大小   4个byte
        执行byteBufferIndex.putLong(tagsCode)                         // 如果是延迟队列，存储的是最终应该被消费的时间   8个byte
        设置expectLogicOffset变量等于cqOffset * 20                     // 实际的逻辑队列offset（文件级别的）
        // 根据expectLogicOffset返回最新的mapedFile，如果不等于null
        // 如果mapedFile队列中第一个并且分配的逻辑offset不等于0并且mapedFile的写位置等于0，对应的场景可能是，比如前面的恢复操作，一个都没恢复成，这种情况，首先设置最小逻辑offset等于expectLogicOffset，同时填充文件expectLogicOffset之前的部分
        // 设置物理offset属性等于offset，追加byteBuffer到mapedFile
        设置mapedFile变量等于mapedFileQueue.getLastMapedFile(expectLogicOffset)
        如果mapedFile变量不等于null
            如果mapedFile.firstCreateInQueue等于true并且cqOffset参数不等于0并且mapedFile.wrotePostion.get()等于0
                设置minLogicOffset属性等于expectLogicOffset变量
                执行fillPreBlank(mapedFile, expectLogicOffset)
            设置maxPhysicOffset属性等于offset
            执行mapedFile.appendMessage(this.byteBufferIndex.array())并返回
        返回false
    // 添加untilWhere之前的空位
    private void fillPreBlank(MapedFile mapedFile, long untilWhere)
        设置byteBuffer变量等于ByteBuffer.allocate(20)
        执行byteBuffer.putLong(0L)
        执行byteBuffer.putInt(Integer.MAX_VALUE)
        执行byteBuffer.putLong(0L)
        设置until变量等于untilWhere % mapedFileQueue.mapedFileSize
        for (int i = 0; i < until; i += 20) {
            执行mapedFile.appendMessage(byteBuffer.array());
        }
    // 返回当前文件下一个文件的开头位置
    public long rollNextFile(long index)
        设置mapedFileSize变量等于mapedFileSize属性
        设置totalUnitsInFile变量等于mapedFileSize / 20
        返回index + totalUnitsInFile - index % totalUnitsInFile
    // 废弃逻辑队列
    public void destroy()
        设置maxPhysicOffset属性等于-1
        设置minLogicOffset属性等于0
        执行mapedFileQueue.destroy()
    // 获取mapedFileQueue中的返回所有文件，遍历到倒数第二个，如果发现某个文件的最后一条逻辑消息的offset小于offset，则删除过期文件从mapedFiles中
    // 获取第一个文件，并进行解析，获取大于等于物理offset的消息，设置对应文件的起始位置 + 对于当前文件的position为minLogicOffset
    // 返回删除个数
    public int deleteExpiredFile(long offset)
        设置cnt变量等于mapedFileQueue.deleteExpiredFileByOffset(offset, 20)
        执行correctMinOffset(offset)
        返回cnt变量

## com.alibaba.rocketmq.store.DefaultMessageStore.FlushConsumeQueueService
    // 逻辑队列的刷盘任务

    public void start()
    public void shundown()
    public void run()
        循环，直到stop属性等于true
            try {
                如果hasNotified属性等于true
                    设置hasNotified属性等于false
                否则
                    wait(messageStoreConfig.flushIntervalConsumeQueue)
                    设置hasNotified属性等于false
                执行doFlush(1)
            } catch (Exception e) {
            }
        执行doFlush(3)
    // 对所有逻辑队列进行刷盘
    private void doFlush(int retryTimes)
        // 如果大于0，则标识这次刷盘必须刷多少个page，如果=0，则有多少刷多少
        设置flushConsumeQueueLeastPages变量等于messageStoreConfig.flushConsumeQueueLeastPages             // flushConsumeQueueLeastPages默认值: 2
        如果retryTimes参数等于3                                   // 如果重试次数等于3，设置为0
            设置flushConsumeQueueLeastPages变量等于0
        设置logicsMsgTimestamp变量等于0
        设置flushConsumeQueueThoroughInterval变量等于messageStoreConfig.flushConsumeQueueThoroughInterval // flushConsumeQueueThoroughInterval默认值: 1000 * 60
        设置currentTimeMillis变量等于系统当前时间戳
        如果currentTimeMillis变量大于等于lastFlushTimestamp + flushConsumeQueueThoroughInterval           // 上一次全刷的和当前时间的间隔超过了指定时间，设置为0
            设置lastFlushTimestamp变量等于currentTimeMillis变量
            设置flushConsumeQueueLeastPages变量等于0
            设置logicsMsgTimestamp变量等于storeCheckpoint.logicsMsgTimestamp           // 获取最近一次存储到逻辑队列的消息的存储时间
        遍历consumeQueueTable.values，设置maps变量等于当前元素
            遍历maps.values，设置cq变量等于当前元素
                设置result变量等于true
                for (int i = 0; i < retryTimes && !result; i++) {
                    设置result变量等于cq.commit(flushConsumeQueueLeastPages)
                }
        如果flushConsumeQueueLeastPages变量等于0
            如果logicsMsgTimestamp变量大于0
                执行storeCheckpoint.setLogicsMsgTimestamp(logicsMsgTimestamp)         // 设置逻辑队列
            执行storeCheckpoint.flush()

## com.alibaba.rocketmq.store.DefaultMessageStore.CleanCommitLogService
    // 清理物理队列任务

    // 警告阀值
    private double DiskSpaceWarningLevelRatio = Double.parseDouble(System.getProperty("rocketmq.broker.diskSpaceWarningLevelRatio", "0.90"))
    // 强制清理阀值
    private double DiskSpaceCleanForciblyRatio = Double.parseDouble(System.getProperty("rocketmq.broker.diskSpaceCleanForciblyRatio", "0.85"))
    private volatile boolean cleanImmediately = false
    // 立刻开始强制删除文件
    public void run()
        try {
            执行deleteExpiredFiles()
            执行redeleteHangedFile()
        } catch (Exception e) {
        }
    // 清理物理队列过期文件
    private void deleteExpiredFiles()
        设置fileReservedTime变量等于messageStoreConfig.fileReservedTime                                       // fileReservedTime默认值: 72
        设置deletePhysicFilesInterval变量等于messageStoreConfig.fileReservedTime                              // deleteCommitLogFilesInterval默认值: 100
        设置destroyMapedFileIntervalForcibly变量等于messageStoreConfig.destroyMapedFileIntervalForcibly       // destroyMapedFileIntervalForcibly默认值: 120 * 1000
        设置timeup变量等于isTimeToDelete()
        设置spacefull变量等于isSpaceToDelete()
        设置manualDelete变量等于manualDeleteFileSeveralTimes > 0
        如果timeup变量等于true或者spacefull变量等于true或者manualDelete变量等于true
            如果manualDelete变量等于true
                执行manualDeleteFileSeveralTimes--
            // 是否立刻强制删除文件
            设置cleanAtOnce变量等于messageStoreConfig.cleanFileForciblyEnable && cleanImmediately             // cleanFileForciblyEnable默认值: true
            设置fileReservedTime变量等于fileReservedTime * 60 * 60 * 1000
            执行commitLog.deleteExpiredFile(fileReservedTime, deletePhysicFilesInterval, destroyMapedFileIntervalForcibly, cleanAtOnce)
    // 根据配置的清理时间决定是否清理
    private boolean isTimeToDelete()
        设置when变量等于messageStoreConfig.deleteWhen                                                         // deleteWhen默认值: 04
        返回UtilAll.isItTimeToDo(when)            // 使用;对when进行分割，获取当前小时，判断是否在分割后的列表中，如果是返回true，否则最后返回false
    // 根据磁盘空间使用率决定是否清理
    private boolean isSpaceToDelete()
        设置ratio变量等于messageStoreConfig.getDiskMaxUsedSpaceRatio() / 100.0                                // diskMaxUsedSpaceRatio默认值: 75
        设置cleanImmediately属性等于false
        {
            设置storePathPhysic变量等于messageStoreConfig.storePathCommitLog
            设置physicRatio变量等于UtilAll.getDiskPartitionSpaceUsedPercent(storePathPhysic)                 // 获取物理队列目录下文件占整个磁盘的百分比
            如果physicRatio变量大于DiskSpaceWarningLevelRatio                                                // 超过警告阀值
                设置diskok变量等于runningFlags.getAndMakeDiskFull()                                          // 标记磁盘满了，开启强制清理
                如果diskok变量等于true
                    执行System.gc()
                设置cleanImmediately属性等于true
            否则如果physicRatio变量大于DiskSpaceCleanForciblyRatio                                            // 超过了清理阀值，开启强制清理
                设置cleanImmediately属性等于true
            如果physicRatio变量小于0或者大于ratio变量                                                          // 超过最大使用阀值
                返回true
        }
        {
            设置storePathLogics变量等于messageStoreConfig.storePathRootDir/consumequeue
            设置logicsRatio变量等于UtilAll.getDiskPartitionSpaceUsedPercent(storePathPhysic)                 // 获取逻辑队列目录下文件占整个磁盘的百分比

            如果logicsRatio变量大于DiskSpaceWarningLevelRatio                                                // 超过警告阀值
                设置diskok变量等于runningFlags.getAndMakeDiskFull()                                          // 标记磁盘满了，开启强制清理
                如果diskok变量等于true
                    执行System.gc()
                设置cleanImmediately变量等于true
            否则如果logicsRatio变量大于DiskSpaceCleanForciblyRatio                                            // 超过了清理阀值，开启强制清理
                设置cleanImmediately变量等于true
            如果logicsRatio变量小于0或者大于ratio变量                                                          // 超过最大使用阀值
                返回true
        }
        返回false
    private void redeleteHangedFile()
        设置interval变量等于messageStoreConfig.redeleteHangedFileInterval
        设置currentTimestamp变量等于当前时间戳
        如果currentTimestamp - lastRedeleteTimestamp属性大于interval
            设置lastRedeleteTimestamp属性等于currentTimestamp变量
            设置destroyMapedFileIntervalForcibly变量等于messageStoreConfig.destroyMapedFileIntervalForcibly
            执行commitLog.retryDeleteFirstFile(destroyMapedFileIntervalForcibly)

## com.alibaba.rocketmq.store.DefaultMessageStore.CleanConsumeQueueService

    public void run()
        try {
            执行deleteExpiredFiles()
        } catch (Exception e) {
        }
    // 根据当前物理队列的最小offset，删除所有逻辑队列的过期文件，删除索引的过期文件，如果删除成功，根据配置决定是否睡眠
    private void deleteExpiredFiles()
        设置deleteLogicsFilesInterval变量等于messageStoreConfig.deleteConsumeQueueFilesInterval           // deleteConsumeQueueFilesInterval默认值: 100
        设置minOffset变量等于commitLog.getMinOffset()
        如果minOffset变量大于lastPhysicalMinOffset属性
            设置lastPhysicalMinOffset属性等于minOffset变量
            遍历consumeQueueTable.values，设置maps变量等于当前元素
                遍历maps.values，设置logic变量等于当前元素
                    设置deleteCount变量等于logic.deleteExpiredFile(minOffset)
                    如果deleteCount变量大于0并且deleteLogicsFilesInterval变量大于0
                        try {
                            Thread.sleep(deleteLogicsFilesInterval)
                        } catch (InterruptedException e) {
                        }
            执行indexService.deleteExpiredFile(minOffset)

## com.alibaba.rocketmq.store.DefaultMessageStore.DispatchMessageService
    // 针对请求中的非事务型消息和已提交型事物消息，构建逻辑队列和索引

    private volatile List<DispatchRequest> requestsWrite
    private volatile List<DispatchRequest> requestsRead

    public DispatchMessageService(int putMsgIndexHightWater)
        设置putMsgIndexHightWater参数等于putMsgIndexHightWater * 1.5
        设置requestsWrite属性等于new ArrayList<DispatchRequest>(putMsgIndexHightWater)
        设置requestsRead属性等于new ArrayList<DispatchRequest>(putMsgIndexHightWater)
    // 判断读请求列表和写请求列表是否还有元素
    public boolean hasRemainMessage()
        设置reqs变量等于requestsWrite属性
        如果reqs变量不等于null并且元素个数大于0
            返回true
        设置reqs变量等于requestsRead属性
        如果reqs变量不等于null并且元素个数大于0
            返回true
        返回false
    // 添加请求到写请求列表
    public void putRequest(DispatchRequest dispatchRequest)
        设置requestsWriteSize变量等于0
        synchronized (this) {
            执行requestsWrite.add(dispatchRequest)
            设置requestsWriteSize变量等于requestsWrite.size()
            如果hasNotified属性等于false
                设置hasNotified属性等于true
                执行notify()
        }
        执行storeStatsService.setDispatchMaxBuffer(requestsWriteSize)
        如果requestsWriteSize变量大于messageStoreConfig.putMsgIndexHightWater         // 超过阀值，睡1毫秒，主要是让出当前线程，让其他线程可被执行
            try {
                Thread.sleep(1)
            } catch (InterruptedException e) {
            }
    public void start()
    public void shundown()
    public void run()
        循环，直到stop属性等于true
            try {
                如果hasNotified属性等于true
                    设置hasNotified属性等于false
                    执行swapRequests()
                否则
                    try {
                        wait(0)
                    } catch (InterruptedException e) {
                    } finally {
                        设置hasNotified属性等于false
                        执行swapRequests()
                    }
                执行doDispatch()
            } catch (Exception e) {
            }
        try {
            Thread.sleep(5 * 1000)
        } catch (InterruptedException e) {
        }
        synchronized (this) {
            执行swapRequests()
        }
        执行doDispatch()
    // 读请求列表和写请求列表转换
    private void swapRequests()
        设置tmp变量等于requestsWrite属性
        设置requestsWrite属性等于requestsRead属性
        设置requestsRead属性等于tmp变量
    // 针对非事务型消息和已提交型事物消息，构建逻辑队列以及根据配置，决定是否构建索引
    private void doDispatch()
        如果requestsRead属性元素个数大于0
            遍历requestsRead属性，设置req变量等于当前元素
                设置tranType变量等于MessageSysFlag.getTransactionValue(req.getSysFlag())
                如果tranType变量等于TransactionNotType或者TransactionCommitType
                    执行findConsumeQueue(req.topic, req.queueId).putMessagePostionInfoWrapper(req.commitLogOffset, req.msgSize, req.tagsCode, req.storeTimestamp, req.consumeQueueOffset)
            如果messageStoreConfig.messageIndexEnable等于true                  // messageIndexEnable默认值: true
                执行indexService.putRequest(requestsRead.toArray())
            执行requestsRead.clear()                                          // 清空读请求列表

## com.alibaba.rocketmq.store.DefaultMessageStore.ReputMessageService
    // Slave专用，master和slave的复制，实际是复制物理队列，因此当前任务读取物理队列，执行putDispatchRequest模拟正常存消息后的效果，比如构建逻辑队列和索引等

    // 设置构建逻辑队列的物理队列的开始位置
    public void setReputFromOffset(long reputFromOffset)
    public void start()
    public void shundown()
    public void run()
        循环，直到stop属性等于true
            try {
                如果hasNotified属性等于true
                    设置hasNotified属性等于false
                否则
                    wait(messageStoreConfig.flushIntervalConsumeQueue)
                    设置hasNotified属性等于false
                执行doReput()
            } catch (Exception e) {
            }
    // 基于reputFromOffset读取物理队列中的新消息，模拟发送消息成功后的操作（putDispatcherRequest）
    private void doReput()
        for (boolean doNext = true; doNext;) {
            设置result变量等于commitLog.getData(reputFromOffset)
            如果result变量不等于null
                try {
                    设置reputFromOffset属性等于result.startOffset
                    for (int readSize = 0; readSize < result.getSize() && doNext;) {
                        设置dispatchRequest变量等于commitLog.checkMessageAndReturnSize(result.byteBuffer, false, false)
                        设置size变量等于dispatchRequest.msgSize
                        如果size变量大于0
                            执行putDispatchRequest(dispatchRequest)
                            设置reputFromOffset属性等于reputFromOffset + size
                            设置readSize变量等于readSize + size
                            执行storeStatsService.getSinglePutMessageTopicTimesTotal(dispatchRequest.topic)
                            执行storeStatsService.getSinglePutMessageTopicSizeTotal(dispatchRequest.topic).addAndGet(dispatchRequest.msgSize)
                        否则如果size变量等于-1
                            设置doNext变量等于false
                        否则如果size变量等于0
                            设置reputFromOffset属性等于commitLog.rollNextFile(reputFromOffset)
                            设置readSize变量等于result.size
                    }
                } finally {
                    result.release()
                }
            否则
                设置doNext变量等于false
        }
    // 立即触发reput操作
    public void wakeup()
        如果hasNotified属性等于false
            设置hasNotified属性等于true
            执行notify()

# com.alibaba.rocketmq.store.schedule.ScheduleMessageService
    // 延迟队列服务，master启动，slave不需要启动

    // Key: Level, Value:  Delay timeMillis
    private ConcurrentHashMap<Integer, Long> delayLevelTable = new ConcurrentHashMap<Integer, Long>(32)
    // Key: Level, Value: Offset
    private ConcurrentHashMap<Integer, Long> offsetTable = new ConcurrentHashMap<Integer, Long>(32)

    public ScheduleMessageService(DefaultMessageStore defaultMessageStore)
        设置defaultMessageStore变量等于defaultMessageStore参数
    // 初始化offsetTable和delayLevelTable
    public boolean load()
        设置result变量等于false
        try {
            读取defaultMessageStore.messageStoreConfig.storePathRootDir/config/delayOffset.json文件内容作为String类型，设置给jsonString变量
            如果jsonString变量等于null
                设置result变量等于loadBak()
            否则
                解序列化jsonString变量作为DelayOffsetSerializeWrapper类型，设置给delayOffsetSerializeWrapper变量
                如果delayOffsetSerializeWrapper变量不等于null
                    执行offsetTable.putAll(delayOffsetSerializeWrapper.offsetTable)
                设置result变量等于true
        } catch (Exception e) {
            设置result变量等于loadBak()
        }
        如果result变量等于true
            设置result变量等于parseDelayLevel()
        返回result变量
    private boolean loadBak()
        try {
            读取defaultMessageStore.messageStoreConfig.storePathRootDir/config/delayOffset.json.bak文件内容作为String类型，设置给jsonString变量
            如果jsonString变量不等于null
                解序列化jsonString变量作为DelayOffsetSerializeWrapper类型，设置给delayOffsetSerializeWrapper变量
                如果delayOffsetSerializeWrapper变量不等于null
                    执行offsetTable.putAll(delayOffsetSerializeWrapper.offsetTable)
                返回true
        } catch (Exception e) {
            返回false
        }
        返回true
    // 解析延迟级别，初始化delayLevelTable
    public boolean parseDelayLevel()
        HashMap<String, Long> timeUnitTable = new HashMap<String, Long>()
        timeUnitTable.put("s", 1000L)
        timeUnitTable.put("m", 1000L * 60)
        timeUnitTable.put("h", 1000L * 60 * 60)
        timeUnitTable.put("d", 1000L * 60 * 60 * 24)
        try {
            设置levelArray变量等于defaultMessageStore.messageStoreConfig.messageDelayLevel.split(" ")
            for (int i = 0; i < levelArray.length; i++) {
                设置value变量等于levelArray[i]
                设置level变量等于i + 1
                如果level变量大于maxDelayLevel属性
                    设置maxDelayLevel属性等于level变量
                执行delayLevelTable.put(level, timeUnitTable.get(value.substring(value.length() - 1)) * Long.parseLong(value.substring(0, value.length() - 1))
            }
        } catch (Exception e) {
            return false
        }
        return true
    // 只有master会启动，slave不会启动
    public void start()
        遍历delayLevelTable属性，设置entry变量等于当前元素
            设置offset变量等于offsetTable.get(entry.key)
            如果offset变量等于null
                设置offset变量等于0
            如果entry.value不等于null
                延迟1秒钟执行new DeliverDelayedMessageTimerTask(entry.key, offset)任务      // 对每个延迟级别启动一个任务
        开启定时任务，延迟10秒执行，任务执行间隔为defaultMessageStore.messageStoreConfig.flushDelayOffsetInterval毫秒          // 开启定时任务持久化offset
            try {
                执行persist()
            } catch (Exception e) {
            }
    // 持久化offset
    public synchronized void persist()
        设置delayOffsetSerializeWrapper变量等于new DelayOffsetSerializeWrapper()实例
        执行delayOffsetSerializeWrapper.setOffsetTable(offsetTable)
        序列化delayOffsetSerializeWrapper变量为JSON格式的字符串，设置给jsonString变量
        如果jsonString变量不等于null
            try {
                写入jsonString变量值到defaultMessageStore.messageStoreConfig.storePathRootDir/config/delayOffset.json文件中
            } catch (Exception e) {
            }
    public void shutdown()
        去掉定时任务
    // 根据延迟级别和存储时间，返回应该消费的时间
    public long computeDeliverTimestamp(int delayLevel, long storeTimestamp)
        设置time变量等于delayLevelTable.get(delayLevel)
        如果time变量不等于null
            返回time + storeTimestamp
        返回storeTimestamp + 1000
    // 根据延迟级别返回逻辑队列id
    public static int delayLevel2QueueId(int delayLevel)
        返回delayLevel - 1
    // 更新延迟级别的offset
    private void updateOffset(int delayLevel, long offset)
        执行offsetTable.put(delayLevel, offset)

## com.alibaba.rocketmq.store.schedule.ScheduleMessageService.DeliverDelayedMessageTimerTask

    // 设置延迟级别和开始offset
    public DeliverDelayedMessageTimerTask(int delayLevel, long offset)
        设置delayLevel属性等于delayLevel参数
        设置offset属性等于offset参数
    public void run()
        try {
            执行executeOnTimeup()
        } catch (Exception e) {
            执行timer.schedule(new DeliverDelayedMessageTimerTask(delayLevel, offset), 10 * 1000)
        }
    public void executeOnTimeup()
        // 获取对应的逻辑延迟队列
        设置cq变量等于defaultMessageStore.findConsumeQueue("SCHEDULE_TOPIC_XXXX", delayLevel2QueueId(delayLevel))
        设置failScheduleOffset等于offset属性
        如果cq变量不等于null
            // 获取对应逻辑队列offset及以后的ByteBuffer
            设置bufferCQ变量等于cq.getIndexBuffer(offset)
            如果bufferCQ变量不等于null
                try {
                    设置nextOffset变量等于offset属性
                    设置i变量等于0
                    for (; i < bufferCQ.size; i += 20)
                        设置offsetPy变量等于bufferCQ.byteBuffer.getLong()
                        设置sizePy变量等于bufferCQ.byteBuffer.getInt()
                        设置tagsCode变量等于bufferCQ.byteBuffer.getLong()

                        设置now变量等于System.currentTimeMillis()
                        设置deliverTimestamp变量等于correctDeliverTimestamp(now, tagsCode)
                        设置nextOffset变量等于offset + (i / 20)
                        设置countdown变量等于deliverTimestamp - now
                        如果countdown变量小于等于0      // 时间到了，需要消费
                            设置msgExt变量等于defaultMessageStore.lookMessageByOffset(offsetPy, sizePy)                   // 获取对应的消息
                            如果msgExt变量不等于null
                                try {
                                    设置msgInner变量等于messageTimeup(msgExt)                                             // 转换成正常消息并存储，如果存储不OK，继续重新开启任务，然后更新offset
                                    设置putMessageResult变量等于defaultMessageStore.putMessage(msgInner)
                                    如果putMessageResult变量不等于null并且putMessageResult.putMessageStatus等于PUT_OK
                                        继续循环
                                    timer.schedule(new DeliverDelayedMessageTimerTask(delayLevel, nextOffset), 10 * 1000)
                                    执行updateOffset(delayLevel, nextOffset)
                                    退出方法
                                } catch (Exception e) {
                                }
                        否则                          // 当前消息没到消费时间，等到countdown毫秒执行
                            执行timer.schedule(new DeliverDelayedMessageTimerTask(delayLevel, nextOffset), countdown)
                            执行updateOffset(delayLevel, nextOffset)          // 更新offset
                            退出方法
                    设置nextOffset变量等于offset + (i / 20)
                    执行timer.schedule(new DeliverDelayedMessageTimerTask(delayLevel, nextOffset), 100L)
                    执行updateOffset(delayLevel, nextOffset)
                    退出方法
                } finally {
                    bufferCQ.release()
                }
            否则
                如果对应的offset不存在，设置为最小offset，然后重新执行任务
                设置cqMinOffset变量等于cq.getMinOffsetInQuque()                // ??应该是queue
                如果offset属性小于cqMinOffset变量
                    设置failScheduleOffset变量等于cqMinOffset变量
        执行timer.schedule(new DeliverDelayedMessageTimerTask(delayLevel, failScheduleOffset), 100L)
    // 如果应该消费的时间大于当前时间 + 延迟时间，返回当前时间 + 延迟时间，否则返回应该消费的时间
    private long correctDeliverTimestamp(long now, long deliverTimestamp)
        设置result变量等于deliverTimestamp参数
        设置maxTimestamp变量等于now参数 + delayLevelTable.get(delayLevel)
        如果deliverTimestamp参数大于maxTimestamp变量
            设置result变量等于now
        返回result变量
    private MessageExtBrokerInner messageTimeup(MessageExt msgExt)
        转换msgExt实例为MessageExtBrokerInner实例，还原真实主题、队列ID

# com.alibaba.rocketmq.store.StoreCheckpoint
    // checkpoint服务，用来存储物理队列最后一条消息的存储时间，逻辑队列最后一条消息的存储时间，最近满的索引文件中最后一条消息的存储时间
    public void setPhysicMsgTimestamp(long physicMsgTimestamp)
    public void setLogicsMsgTimestamp(long logicsMsgTimestamp)
    public void setIndexMsgTimestamp(long indexMsgTimestamp)

    public StoreCheckpoint(String scpPath) throws IOException
        设置file变量等于new File(scpPath)实例
        设置randomAccessFile属性等于new RandomAccessFile(file, "rw")
        设置fileChannel属性等于randomAccessFile.getChannel()
        设置mappedByteBuffer属性等于fileChannel.map(MapMode.READ_WRITE, 0, MapedFile.OS_PAGE_SIZE)
        如果file.exists()等于true
            设置physicMsgTimestamp属性等于mappedByteBuffer.getLong(0)
            设置logicsMsgTimestamp属性等于mappedByteBuffer.getLong(8)
            设置indexMsgTimestamp属性等于mappedByteBuffer.getLong(16)
    public void shutdown()
        执行flush()
        执行MapedFile.clean(mappedByteBuffer)
        try {
            执行fileChannel.close()
        } catch (IOException e) {
            e.printStackTrace()
        }
    public void flush()
        执行mappedByteBuffer.putLong(0, physicMsgTimestamp)
        执行mappedByteBuffer.putLong(8, logicsMsgTimestamp)
        执行mappedByteBuffer.putLong(16, indexMsgTimestamp)
        执行mappedByteBuffer.force()

# com.alibaba.rocketmq.store.ha.HAService

    // Server
    //  开启defaultMessageStore.messageStoreConfig.haListenPort端口
    //  处理ACCEPTOR事件，针对每个接入的连接，启动一个线程，初始化nextTransferFromWhere属性，循环执行haService.defaultMessageStore.getCommitLogData(nextTransferFromWhere)并写入到channel和累加nextTransferFromWhere属性

    // Client
    //  连接messageStoreConfig.haMasterAddress
    //  循环从服务端读取数据，并调用defaultMessageStore.appendToCommitLog(masterPhyOffset, bodyData)

# com.alibaba.rocketmq.store.index.IndexService

    private ArrayList<IndexFile> indexFileList = new ArrayList<IndexFile>()
    private ReadWriteLock readWriteLock = new ReentrantReadWriteLock()
    private LinkedBlockingQueue<Object[]> requestQueue = new LinkedBlockingQueue<Object[]>(300000)

    public IndexService(DefaultMessageStore store)
        设置defaultMessageStore属性等于store参数
        设置hashSlotNum属性等于store.messageStoreConfig.maxHashSlotNum
        设置indexNum属性等于store.messageStoreConfig.maxIndexNum
        设置storePath属性等于store.messageStoreConfig.storePathRootDir/index
    public void start()
    public void shutdown()
    public boolean load(boolean lastExitOK)
        如果storePath目录下存在子文件，排序并进行遍历，设置file变量等于当前元素
            try {
                设置f变量等于new IndexFile(file.getPath(), hashSlotNum, indexNum, 0, 0)
                执行f.load()
                如果lastExitOK变量等于false
                    如果f.getEndTimestamp()大于defaultMessageStore.storeCheckpoint.indexMsgTimestamp
                        执行f.destroy(0)
                        继续下一次循环
                执行indexFileList.add(f)
            } catch (IOException e) {
                return false
            }
        返回true
    public void putRequest(Object[] reqs)
        执行requestQueue.offer(reqs)
    public void deleteExpiredFile(long offset)
        设置files变量等于null
        try {
            readWriteLock.readLock().lock()
            如果indexFileList属性元素个数为空
                退出方法
            设置endPhyOffset变量等于indexFileList.get(0).getEndPhyOffset()
            如果endPhyOffset变量小于offset参数
                设置files变量等于indexFileList.toArray()
        } catch (Exception e) {
        } finally {
            readWriteLock.readLock().unlock()
        }
        如果files变量不等于null
            设置fileList变量等于new ArrayList<IndexFile>()实例
            遍历files变量，设置f变量等于当前元素
                如果f.getEndPhyOffset()小于offset
                    执行fileList.add(f)
                否则
                    退出循环
            执行deleteExpiredFile(fileList)
    private void deleteExpiredFile(List<IndexFile> files)
        如果files参数元素个数大于0
            try {
                readWriteLock.writeLock().lock()
                遍历files参数，设置file变量等于当前元素
                    设置destroyed变量等于file.destroy(3000)
                    如果destroyed变量等于true
                        设置destroyed变量等于indexFileList.remove(file)
                    如果destroyed变量等于false
                        退出循环
            } catch (Exception e) {
            } finally {
                readWriteLock.writeLock().unlock()
            }

# com.alibaba.rocketmq.store.StoreStatsService
