# 入口

* com.alibaba.rocketmq.namesrv.NamesrvStartup
* com.alibaba.rocketmq.broker.BrokerStartup
* com.alibaba.rocketmq.filtersrv.FiltersrvStartup


# Producer

* com.alibaba.rocketmq.client.producer.DefaultMQProducer
* com.alibaba.rocketmq.client.producer.TransactionMQProducer

# Consumer

* com.alibaba.rocketmq.example.quickstart.Consumer
* com.alibaba.rocketmq.example.filter.Consumer
* com.alibaba.rocketmq.example.broadcast.PushConsumer
* com.alibaba.rocketmq.example.ordermessage.Consumer
* com.alibaba.rocketmq.example.simple.PullConsumer
* com.alibaba.rocketmq.example.simple.PullConsumerTest
* com.alibaba.rocketmq.example.simple.PullScheduleService

# 物理队列

文件位置：defaultMessageStore.messageStoreConfig.storePathCommitLog
文件大小：defaultMessageStore.messageStoreConfig.mapedFileSizeCommitLog

    store/commitlog
        00000000000000000000
            正常消息格式
                消息大小|MAGICCODE|消息体的CRC|逻辑队列ID|客户端设定的标识|逻辑队列offset|物理队列offset|系统标识|消息创建时间|发送消息的Socket|消息存储时间|存储消息的Socket|重消费次数|事物预写offset|消息体长度|消息体|主题长度|主题|属性长度|属性
                4       4         4         4         4             8             8             4       8          8              8          8               4        8             4              1           2
                注意事项
                    MAGICCODE等于MessageMagicCode的表示正常消息
                    逻辑队列offset类似于增量ID（topic-queueId作为key，不存在用0，存在+1；启动时会根据各个逻辑队列的最大offset进行设置）
                    物理队列offset等于文件的开始位置 + 消息在文件中的position
                    系统标识分别为是否压缩、是否是MultiTag，事物类型（非事物、事物提交型、事物回滚型、预提交型）
                    事物预写offset实际存储的是物理队列offset
            文件结束标识
                空位个数|MAGICCODE
                4       4
                注意事项
                    存储消息时，至少要保证有（消息大小 + 8）个byte可是使用，最后8个byte存储文件结束标识
                    空位个数表示文件大小 - 当前位置
        00000000001073741824

        00000000002147483648

# 逻辑队列

文件位置：messageStoreConfig.storePathRootDir/consumequeue
文件大小：messageStoreConfig.mapedFileSizeConsumeQueue                   // 20的倍数

    store
        consumequeue
            topic1

                1
                    00000000000000000000
                        填充消息格式
                            物理队列offset|消息大小|tagCode
                            8             4      8
                            注意事项
                                物理队列offset等于0
                                消息大小等于long的最大值
                                tagCode等于0
                        正常消息格式
                            物理队列offset|消息大小|tagCode
                            8             4      8
                            注意事项
                                tagCode对于延迟消息，存储的消息实际应该被消费的时间，其他情况存储消息的properties["TAGS"]的hashCode值，如果不存在，存储0
                                逻辑队列offset表示(文件的开始位置 + 逻辑消息在文件中的position) / 20              // 每个消息占用20个byte，所以逻辑队列offset也可视为增量数字
                                逻辑消息的位置等于逻辑队列offset * 20 + 文件的开始位置

                    00000000000006000000


                    00000000000012000000

                2

                3

            topic2

                1

                2

                3

                4

# 消息ID

    host + port + 物理队列offset


