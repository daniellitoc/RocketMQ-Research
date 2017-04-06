# com.alibaba.rocketmq.broker.client.net.Broker2Client

    public Broker2Client(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    public RemotingCommand callClient(Channel channel, RemotingCommand request) throws RemotingSendRequestException, RemotingTimeoutException, InterruptedException
        执行brokerController.remotingServer.invokeSync(channel, request, 10000)
    // 当添加或停止Consumer时，通知指定Consumer，consumerGroup下的clientId列表发生变化，需要重新均衡消息队列
    public void notifyConsumerIdsChanged(Channel channel, String consumerGroup)
        如果consumerGroup参数等于null
            退出方法
        设置requestHeader变量等于new NotifyConsumerIdsChangedRequestHeader()
        执行requestHeader.setConsumerGroup(consumerGroup)
        执行RemotingCommand.createRequestCommand(40, requestHeader)返回RemotingCommand类型的实例，设置给request变量           // NOTIFY_CONSUMER_IDS_CHANGED: 40
        try {
            执行brokerController.remotingServer.invokeOneway(channel, request, 10)
        } catch (Exception e) {
        }
    // 获取timestamp对应每个消息队列的offset，和现有的正在消息的offset比较，谁小用谁，如果isForce参数等于true，表示一定使用timestamp获取的，最后发送请求到group对应的所有Consumer
    public RemotingCommand resetOffset(String topic, String group, long timeStamp, boolean isForce)
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给response变量
        设置topicConfig变量等于brokerController.topicConfigManager.selectTopicConfig(topic)
        如果topicConfig变量等于null
            执行response.setCode(1)                                                                                       // SYSTEM_ERROR: 1
            执行response.setRemark("[reset-offset] reset offset failed, no topic in this broker")
            返回response变量
        设置offsetTable变量等于new HashMap<MessageQueue, Long>()
        循环topicConfig.writeQueueNums次，设置i等于循环索引                                                                  // ??此处貌似应该用readQueueNums，消费者用的是读消息队列，生产者用的是写消息队列
            设置mq变量等于new MessageQueue()实例
            执行mq.setBrokerName(brokerController.brokerConfig.brokerName)
            执行mq.setTopic(topic)
            执行mq.setQueueId(i)
            设置consumerOffset变量等于brokerController.consumerOffsetManager.queryOffset(group, topic, i)
            如果consumerOffset变量等于-1
                执行response.setCode(1)                                                                                   // SYSTEM_ERROR: 1
                执行response.setRemark("THe consumer group not exist")
                返回response变量
            设置timeStampOffset变量等于brokerController.messageStore.getOffsetInQueueByTime(topic, i, timeStamp)
            如果isForce参数等于true或者timeStampOffset变量小于consumerOffset变量
                执行offsetTable.put(mq, timeStampOffset)
            否则
                执行offsetTable.put(mq, consumerOffset)
        设置requestHeader变量等于new ResetOffsetRequestHeader()实例
        执行requestHeader.setTopic(topic);
        执行requestHeader.setGroup(group);
        执行requestHeader.setTimestamp(timeStamp);
        执行RemotingCommand.createRequestCommand(220, requestHeader)返回RemotingCommand类型的实例，设置给request变量          // RESET_CONSUMER_CLIENT_OFFSET: 220
        设置body变量等于new ResetOffsetBody()实例
        执行body.setOffsetTable(offsetTable)
        执行request.setBody(body.encode())
        设置consumerGroupInfo变量等于brokerController.consumerManager.getConsumerGroupInfo(group)
        如果consumerGroupInfo变量不等于null并且consumerGroupInfo.getAllChannel()元素个数大于0
            设置channelInfoTable变量等于consumerGroupInfo.channelInfoTable，并进行遍历，设置entry等于当前元素
                try {
                    执行brokerController.remotingServer.invokeOneway(entry.key, request, 5000);
                } catch (Exception e) {
                }
        否则
            执行response.setCode(206)                                                                                     // CONSUMER_NOT_ONLINE: 206
            执行response.setRemark("Consumer not online, so can not reset offset")
            返回response变量
        执行response.setCode(0)                                                                                           // SUCCESS: 0
        设置resBody变量等于new ResetOffsetBody()
        执行resBody.setOffsetTable(offsetTable)
        执行response.setBody(resBody.encode())
        返回response变量
    // 如果originClientId等于null，获取所有Consumer中topic下的group的所有消息队列的offset信息，否则获取originClientId和之前的Consumer的消息队列消费情况
    public RemotingCommand getConsumeStatus(String topic, String group, String originClientId)
        执行RemotingCommand.createResponseCommand(null)返回RemotingCommand类型的实例，设置给result变量
        设置requestHeader变量等于new GetConsumerStatusRequestHeader()实例
        执行requestHeader.setTopic(topic)
        执行requestHeader.setGroup(group)
        执行RemotingCommand.createRequestCommand(221, requestHeader)返回RemotingCommand类型的实例，设置给request变量          // GET_CONSUMER_STATUS_FROM_CLIENT: 221
        设置channelInfoTable变量等于brokerController.consumerManager.getConsumerGroupInfo(group).channelInfoTable
        如果channelInfoTable变量等于null或者元素个数等于0
            执行response.setCode(1)                                                                                       // SYSTEM_ERROR: 1
            执行response.setRemark("No Any Consumer online in the consumer group")
            返回response变量
        设置consumerStatusTable变量等于new HashMap<String, Map<MessageQueue, Long>>()
        遍历channelInfoTable变量，设置entry变量等于当前元素
            设置clientId变量等于entry.value.clientId
            如果originClientId参数等于null或者等于clientId变量
                try {
                    执行brokerController.remotingServer.invokeSync(channel, request, 5000)返回RemotingCommand类型的实例，设置给response变量
                    如果response.code等于0                      // SUCCESS: 0
                        如果response.body不等于null
                            解序列化response.body为GetConsumerStatusBody类型的实例，设置给body变量
                            执行consumerStatusTable.put(clientId, body.messageQueueTable)
                } catch (Exception e) {
                }
                如果originClientId参数不等于null并且等于clientId变量
                    退出循环
        执行result.setCode(0)                             // SUCCESS: 0
        设置resBody变量等于new GetConsumerStatusBody()实例
        执行resBody.setConsumerTable(consumerStatusTable)
        执行result.setBody(resBody.encode())
        返回result变量

## com.alibaba.rocketmq.broker.client.ClientHousekeepingService

    public ClientHousekeepingService(BrokerController brokerController)
        设置brokerController属性等于brokerController参数
    public void start()
        开启定时任务，延迟10秒执行，任务执行间隔为10秒钟
            try {
                执行scanExceptionChannel()
            } catch (Exception e) {
            }
    private void scanExceptionChannel()
        执行brokerController.producerManager.scanNotActiveChannel()                                // 关闭所有超时的Channel，依赖Producer的定时心跳请求
        执行brokerController.consumerManager.scanNotActiveChannel()                                // 关闭所有超时的Channel，依赖Consumer的定时心跳请求
        执行brokerController.filterServerManager.scanNotActiveChannel()                            // 关闭所有超时的Channel，依赖FilterServer的定时注册请求
    public void onChannelClose(String remoteAddr, Channel channel)                                // 连接关闭，Channel可能是FilterServer的Channel，也可能是Producer和Consumer的Channel
        执行brokerController.producerManager.doChannelCloseEvent(remoteAddr, channel)              // 删除Producer对应的Channel
        执行brokerController.consumerManager.doChannelCloseEvent(remoteAddr, channel)              // 删除Consumer对应的Channel
        执行brokerController.filterServerManager.doChannelCloseEvent(remoteAddr, channel)          // 删除FilterServer对应的Channel
    public void onChannelException(String remoteAddr, Channel channel)                            // 出现异常，Channel可能是FilterServer的Channel，也可能是Producer和Consumer的Channel
        执行brokerController.producerManager.doChannelCloseEvent(remoteAddr, channel)              // 删除Producer对应的Channel
        执行brokerController.consumerManager.doChannelCloseEvent(remoteAddr, channel)              // 删除Consumer对应的Channel
        执行brokerController.filterServerManager.doChannelCloseEvent(remoteAddr, channel)          // 删除FilterServer对应的Channel
    public void onChannelIdle(String remoteAddr, Channel channel)                                 // 触发连接空闲，Channel可能是FilterServer的Channel，也可能是Producer和Consumer的Channel
        执行brokerController.producerManager.doChannelCloseEvent(remoteAddr, channel)              // 删除Producer对应的Channel
        执行brokerController.consumerManager.doChannelCloseEvent(remoteAddr, channel)              // 删除Consumer对应的Channel
        执行brokerController.filterServerManager.doChannelCloseEvent(remoteAddr, channel)          // 删除FilterServer对应的Channel
    public void shutdown()
        关闭定时任务

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
    public void registerProcessor(int requestCode, NettyRequestProcessor processor, ExecutorService executor)
        如果executor参数等于null
            设置executor参数等于publicExecutor属性
        执行processorTable.put(requestCode, new Pair<NettyRequestProcessor, ExecutorService>(processor, executor))              // 注册处理器和执行该请求的业务线程池
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
        执行定时任务，延迟3秒钟执行，执行间隔为1秒钟
            执行scanResponseTable()
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
        关闭eventLoopGroupBoss属性和eventLoopGroupWorker属性
        执行nettyEventExecuter.shutdown()
        关闭defaultEventExecutorGroup属性
        关闭publicExecutor属性
    public RemotingCommand invokeSync(Channel channel, RemotingCommand request, long timeoutMillis) throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException
        执行invokeSyncImpl(channel, request, timeoutMillis)并返回
    public void invokeOneway(Channel channel, RemotingCommand request, long timeoutMillis) throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException
        执行invokeOnewayImpl(channel, request, timeoutMillis)
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
                执行responseFuture.putResponse(cmd)                                                                            // 释放countDownLatch

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
        执行processMessageReceived(ctx, msg)
        