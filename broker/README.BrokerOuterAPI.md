# com.alibaba.rocketmq.broker.out.BrokerOuterAPI
    private TopAddressing topAddressing = new TopAddressing("http://{rocketmq.namesrv.domain:jmenv.tbsite.net}:8080/rocketmq/{rocketmq.namesrv.domain.subgroup:nsaddr}")
    private String nameSrvAddr = null

    public String fetchNameServerAddr()                                                     // 获取和设置namserver列表
        设置addrs变量等于topAddressing.fetchNSAddr()                                          // 请求url获取对应的响应文本，每次调用，都会重新请求
        如果addrs变量不等于null并且不等于nameSrvAddr属性
            设置nameSrvAddr属性等于addrs变量
            执行updateNameServerAddressList(addrs)
        返回nameSrvAddr属性
    public void updateNameServerAddressList(String addrs)                                   // 用;分隔成列表设置nameserver
        使用;分隔addrs参数成List，设置给lst变量
        执行remotingClient.updateNameServerAddressList(lst)

    public BrokerOuterAPI(NettyClientConfig nettyClientConfig, RPCHook rpcHook)
        设置remotingClient属性等于new NettyRemotingClient(nettyClientConfig)实例
        执行remotingClient.registerRPCHook(rpcHook)
    public void start()
        执行remotingClient.start()
    public void shutdown()
        执行remotingClient.shutdown()
    // 对namesrvAddr发送同步REGISTER_BROKER请求，用于注册BrokerServer信息，会返回masterAddr和haServerAddr，以及所有顺序主题配置信息，超时时间为3000毫秒，返回的responseCode不等于0，抛异常
    private RegisterBrokerResult registerBroker(String namesrvAddr, String clusterName, String brokerAddr, String brokerName, long brokerId, String haServerAddr, TopicConfigSerializeWrapper topicConfigWrapper, List<String> filterServerList, boolean oneway) throws RemotingCommandException, MQBrokerException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException, InterruptedException
        设置requestHeader变量等于new RegisterBrokerRequestHeader()实例
        执行requestHeader.setBrokerAddr(brokerAddr)
        执行requestHeader.setBrokerId(brokerId)
        执行requestHeader.setBrokerName(brokerName)
        执行requestHeader.setClusterName(clusterName)
        执行requestHeader.setHaServerAddr(haServerAddr)
        执行RemotingCommand.createRequestCommand(103, requestHeader)返回RemotingCommand类型的实例，设置给request变量                // REGISTER_BROKER: 103
        设置requestBody变量等于new RegisterBrokerBody()实例
        执行requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper)
        执行requestBody.setFilterServerList(filterServerList)
        执行request.setBody(requestBody.encode())
        如果oneway参数等于true
            try {
                remotingClient.invokeOneway(namesrvAddr, request, 3000)
            } catch (Exception e) {
            }
            返回null
        执行remotingClient.invokeSync(namesrvAddr, request, 3000)返回RemotingCommand类型的实例，设置给response变量
        如果response.code等于0                                                                                                  // SUCCESS: 0
            解序列化response参数为RegisterBrokerResponseHeader类型的实例，设置给responseHeader变量
            设置result变量等于new RegisterBrokerResult()
            执行result.setMasterAddr(responseHeader.masterAddr)
            执行result.setHaServerAddr(responseHeader.haServerAddr)
            如果response.body不等于null
                执行result.setKvTable(KVTable.decode(response.getBody(), KVTable.class))
            返回result变量
        抛异常MQClientException(response.code, response.remark)
    // 注册BrokerServer信息到所有的NameServer
    public RegisterBrokerResult registerBrokerAll(String clusterName, String brokerAddr, String brokerName, long brokerId, String haServerAddr, TopicConfigSerializeWrapper topicConfigWrapper, List<String> filterServerList, boolean oneway)
        设置registerBrokerResult变量等于null
        如果remotingClient.getNameServerAddressList()不等于null，进行遍历，设置namesrvAddr变量等于当前元素
            try {
                执行registerBroker(namesrvAddr, clusterName, brokerAddr, brokerName, brokerId, haServerAddr, topicConfigWrapper, filterServerList, oneway)返回RegisterBrokerResult类型的实例，设置给result变量
                如果result变量不等于null
                    设置registerBrokerResult变量等于result变量
            } catch (Exception e) {
            }
        返回registerBrokerResult变量
    // 对namesrvAddr发送同步UNREGISTER_BROKER请求，用于解注册BrokerServer信息，超时时间为3000毫秒，返回的responseCode不等于0，抛异常
    public void unregisterBroker( String namesrvAddr, String clusterName, String brokerAddr, String brokerName, long brokerId) throws RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException, InterruptedException, MQBrokerException
        设置requestHeader变量等于new UnRegisterBrokerRequestHeader()实例
        执行requestHeader.setBrokerAddr(brokerAddr)
        执行requestHeader.setBrokerId(brokerId
        执行requestHeader.setBrokerName(brokerName)
        执行requestHeader.setClusterName(clusterName)
        执行RemotingCommand.createRequestCommand(104, requestHeader)返回RemotingCommand类型的实例，设置给request变量                // UNREGISTER_BROKER: 104
        执行remotingClient.invokeSync(addr, request, 3000)返回RemotingCommand类型的实例，设置给response变量
        如果response.code等于0                                                                                                  // SUCCESS: 0
            退出方法
        抛异常MQClientException(response.code, response.remark)
    // 从所有的NameServer解注册BrokerServer信息
    public void unregisterBrokerAll(String clusterName, String brokerAddr, String brokerName, long brokerId)
        如果remotingClient.getNameServerAddressList()不等于null，进行遍历，设置namesrvAddr变量等于当前元素
            try {
                执行unregisterBroker(namesrvAddr, clusterName, brokerAddr, brokerName, brokerId)
            } catch (Exception e) {
            }
    // slave专用，对指定masterAddr发送同步GET_ALL_TOPIC_CONFIG请求，获取主题配置，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public TopicConfigSerializeWrapper getAllTopicConfig(String addr) throws RemotingConnectException,  RemotingSendRequestException, RemotingTimeoutException, InterruptedException, MQBrokerException
        执行RemotingCommand.createRequestCommand(21, null)返回RemotingCommand类型的实例，设置给request变量                         // GET_ALL_TOPIC_CONFIG: 21
        执行remotingClient.invokeSync(addr, request, 3000)返回RemotingCommand类型的实例，设置给response变量
        如果response.code等于0                                                                                                 // SUCCESS: 0
            解序列化response.body为TopicConfigSerializeWrapper类型的实例并返回
        抛异常MQClientException(response.code, response.remark)
    // slave专用，对指定masterAddr发送同步GET_ALL_CONSUMER_OFFSET请求，获取所有主题下的group级别中，每个消息队列的offset，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public ConsumerOffsetSerializeWrapper getAllConsumerOffset(String addr) throws InterruptedException, RemotingTimeoutException, RemotingSendRequestException, RemotingConnectException, MQBrokerException
        执行RemotingCommand.createRequestCommand(43, null)返回RemotingCommand类型的实例，设置给request变量                         // GET_ALL_CONSUMER_OFFSET: 43
        执行remotingClient.invokeSync(addr, request, 3000)返回RemotingCommand类型的实例，设置给response变量
        如果response.code等于0                                                                                                 // SUCCESS: 0
            解序列化response.body为ConsumerOffsetSerializeWrapper类型的实例并返回
        抛异常MQClientException(response.code, response.remark)
    // slave专用，对指定masterAddr发送同步GET_ALL_DELAY_OFFSET请求，获取所有延迟级别的offset和timestamp，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public String getAllDelayOffset(String addr) throws InterruptedException, RemotingTimeoutException, RemotingSendRequestException, RemotingConnectException, MQBrokerException
        执行RemotingCommand.createRequestCommand(45, null)返回RemotingCommand类型的实例，设置给request变量                         // GET_ALL_DELAY_OFFSET: 45
        执行remotingClient.invokeSync(addr, request, 3000)返回RemotingCommand类型的实例，设置给response变量
        如果response.code等于0                                                                                                 // SUCCESS: 0
            返回new String(response.body)
        抛异常MQClientException(response.code, response.remark)
    // slave专用，对指定masterAddr发送同步GET_ALL_SUBSCRIPTIONGROUP_CONFIG请求，获取订阅组配置，超时时间为timeoutMillis毫秒，返回的responseCode不等于0，抛异常
    public SubscriptionGroupWrapper getAllSubscriptionGroupConfig(String addr) throws InterruptedException, RemotingTimeoutException, RemotingSendRequestException, RemotingConnectException, MQBrokerException
        执行RemotingCommand.createRequestCommand(201, null)返回RemotingCommand类型的实例，设置给request变量                        // GET_ALL_SUBSCRIPTIONGROUP_CONFIG: 201
        执行remotingClient.invokeSync(addr, request, 3000)返回RemotingCommand类型的实例，设置给response变量
        如果response.code等于0                                                                                                 // SUCCESS: 0
            解序列化response.body为SubscriptionGroupWrapper类型的实例并返回
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