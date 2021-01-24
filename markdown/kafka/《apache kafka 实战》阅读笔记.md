# 《apache kafka 实战》阅读笔记

## 认识Apache Kafka

* kafka 直接使用了 ByteBuffer，减少 java 对象造成的字节膨胀。
* 使用页缓存在 broker 崩溃后重启后仍可以使用。
* kafka 使用 topic-partition-message 三级结构。
* 使用<topic，partition，offset> 可以唯一确定消息。
* ISR 全称为 in-sync replica，只有这个集合内才会被选举为 leader，也只有集合中所有的 replica 接收到同一条消息，才能讲此消息称为“已提交”。

## Kafka 发展历史

* 《The world beyond batch：streaming》
* kafkaConsumer.poll 方法采用了类似 Linux epoll 的轮询机制，使得 consumer 只使用了一个线程就可以管理连向不同 broker 的多个 socket，减少了线程间的开销成本。
* kafkaConsumer 有后台心跳线程。
* kafka 消费者由 coordinator 来协调对 group 的管控。

## Kafka线上环境部署

* epoll 相较于 select 取消了轮训机制，取而代之是回调机制。新版的 client 使用了 Java 的Selector 机制，在 linux 就是 epoll 实现。
* 在 linux 通过 FileChannel.transferTo 方法实现，使用了 sendfile，采用了零拷贝。
* kafka 只将数据写入 page cache，刷入磁盘由操作系统完成。
* Kafka 在 broker 免除了解压缩的消耗，需要确认 2.1 和 2.5 的情况。
* tickTime：最小时间单位。
* initLimit：指定 flollower 节点初识时连接 leader 节点的最大 tick 次数。
* syncLimit：设定了 follower 节点与 leader 节点进行同步的最大 tick 次数。
* advertiesd.listeners 为发布给 clients 的监听器，可以使用此参数绑定公网 IP 给客户端用。

## producer 开发

* https://cwiki.apache.org/confluence/display/KAFKA/Clients

* buffer.memory 指定了 producer 端用于缓存消息的缓冲区大小。
* batch.size 和 linger.ms 影响 batch 大小。
* producer 的拦截器包括三个开放接口（onSend，onAcknowledgement 和 close）。
* kafka 的压缩时在 producer 和 consumer 端执行压缩和解压缩。
* kafkaProducer 为线程安全。

##consumer开发

* kafka consumer 内部使用一个 map 来保存其订阅 topic 所属分区的 offset。
* __consumer_offsets 内的记录为三元组，group.id+topic+ partition, offset，在内部定期执行 compact 保存最新的 offset。
* session.timeout.ms 代表 coordinator 检测失败时间，max.poll.interval.ms 代表处理消息的最大时间。heartbeat.interval.ms 通知 coordinator 保持 group 状态。
* consumer 采用了 poll 的设计，使用一个线程来同时管理多个 socket 的连接。一个 KafkaConsumer 有两个线程，一个时 poll 主线程，一个是心跳线程。
* KafkaConsumer 不是线程安全的，wakeup 方法可以在另外的线程调用。
* 上次提交位置 <= 当前位置 <= 水位 <= 日志最新位移
* consumer 提交位移通过向所属的 coordinator 发送位移提交请求来实现的。
* commitSync 是在 poll 里现实的，非单独线程实现。
* rebalance generation 来标识第几届 rebalance。
* consumer group 的 leader 为整个 group 分配方案。整个的 rebalance 的操作有两个步骤，分别是加入组（确定 leader ）和同步更新分配方案。
* 可以使用独立的 consumer 通过 assing 来分配 TP 的消费问题。

## kafka 设计原理

* v2 版本的消息格式回保存 PID 和 epoch 和序列号等，用于实现幂等性和事务。
* 剔出 ISR 的只有 replica.lag.time.max.ms，用于检测持续落后的情况，可以兼容瞬间峰值流量。
* follower 发送过来的 FETCH 请求因为无数据而暂时被寄存到 leader 的 purgatory 中。
* follower 的 leo 是在下一轮 fetch 中向 leader 回报 leo。
* leader epoch 和 offset 一起组成 （ epoch， offset）来解决数据问题。代码主要在LeaderEpochCache 和 LeaderEpochCheckpointFile 类中。
* 位移索引文件保存的是<相对位移，文件物理位置>，时间戳索引文件保存的是<时间戳，相对位移>。相对位移指的是索引文件起始位置的差值。
* https://kafka.apache.org/protocol 包含了 kafka 使用到的协议。
* meta 的信息是保存在每个 broker 中，而且是相同的元数据。
* bin/kafka-broker-api-versions.sh 来查看每个请求的版本。
* controller 把更新元数据请求封装好（UpdateMetadataRequests）发给每个 broker。
*  受控关闭时 broker 的发送 ControlledShutdownRequest 给 controller 来控制。
* kafka 里重要的数据组件 ControllerContext，在 1.0.0 版本，使用单线程的基于事件的模型，使用了 LinkedBlockingQueue 保护 controller 的状态。
* broker 中，processor 线程负责将请求放入队列，一个 processor 负责多个 socket，内部使用 Selector 来管理，并且有与 processor 一一对应的处理 response 的线程，KafkaRequestHandler 处理具体的请求。
* broker 处理请求： 1. 启动 acceptor。2. acceptor 监听到有 socket 连接。3. 讲请求发送给 processor。4. processor 获取到后将请求给 KafkaRequestHandler。5. 请求处理完返回到 processor。6. processor 会给给客户端。
* consumer group 分配策略是 consumer 端实现的。
* PreparingRebalance 等待新成员进入（耗时 max.poll.interval.ms）为 AwaitingSync 状态开始分配消费方案。
* 对于实现幂等性，producer 会得到一个<PID + 分区号，序列号>，所以只能实现当个 producer EOS。
* 对于事务，有了 TransactionalId 可以保证多会话之间的事务。使用了 generation 来隔离分代事务。
* 对于原子性写入多个，使用了一类特殊消息，即控制消息，在消息属性字段（attribute field）中专门使用1来表征他是控制消息，卸载多个。

## 管理 kafka 集群

* 升级 kafka。1. 在 server.properties 里指定 inter.broker.protocol.version 和 log.message.format.version 为旧版本。2. 更新 jar 包，重启 broker。3. 在 server.properties 里指定 inter.broker.protocol.version 和 log.message.format.version 为新版本。4. 一次重启 broker。
* 新增 broker 节点，旧的 topic 不会自动均衡，需要认为操作。
* kafka-consumer-groups 设置的 conusmer group 必须为 inactive。
* 消息者组信息保留时间为 offset.retention.minutes，此参数计算为最后一个成员退出组开始算。
* kafka-preferred-replica-election.sh 来调整 preferred leader。auto.leader.rebalance.enable 是帮组用户自动执行此操作。
* kafka-reassign-paritions.sh 来调整分区的分配，有 generate，execute 和 verify。也可以用于增加副本。
* 使用 kafka-run-class.sh kafka.tools..DumpLogSegments 来查看信息。关注 baseOffset 和 lastOffset 是否相等，可以查看被压缩进一条wrap消息的多条消息，也可以使用 --deep-iteration 来查看详细信息。
* 使用 GetOffsetShell 来获取总消息数。
* kafka-simple-consumer-shell.sh --topic __consumer_offsets --partition 25 --broker-list localhost:9092 --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" 读取内部 topic 的信息。
* 在 API 层可以使用 AdminClient 来管理集群。

## 监控 kafka 集群

* Kafka JMX MBean 监控kafka 的信息，主要包括 1. kafka broker 2. clients 3. producer 4. consmer。详细信息见 http://kafka.apache.org/documentation/#monitoring。
* 入站消息费率：kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec 
* 出战消息费率：kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec 
* 当前集群中 active controller 个数：kafka.controller.type=KafkaController,name=ActiveControllerCount 
* 备份不足的分区数：kafka.server:type=ReplicateManager,name=UnderReplicatedPartitions
* leader分区数：kafka.server:type=ReplicateManager,name=LeaderCount
* ISR变化速率：kafka.server:type=ReplicateManager,name=IsrExpandsPerSec，kafka.server:type=ReplicateManager,name=IsrShrinksPerSec
* broker I/O 工作处理线程空闲率：kafka.server:type=SocketServer,name=NetWorkProcessorAvgIdlePercent
* broker 网络处理线程空闲率：kafka.server:type=KafkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent
* producer 的 MBean 有 kafka.producer:type=producer-type-metrics,client-id=<CLINET_ID>,node-id=<BROKER_ID> 和 kafka.producer:type=producer-topic-metrics,client-id=<CLIENT_ID>,topic=<TOPIC>
* consumer 的 MBean 有 fetcher 类，coordinator 类和 consumer 类。
* JVM 进程状态主要是 java.lang:type=OperatingSystem、MaxFileDescriptionCount 和 java.lang:type=OperatingSystem、OpenFileDescritporCount
* GC性能主要是 java.lang:typeGarbageCollector...等， 包括老年代和新生代的 gc 次数和时长，还有一个指标是 LastGcInfo 记录上一次 GC 的详情信息。
* 主流监控框架包括 JmxTool，kafka-manger、kafka monitor（使用了一个 topic 去检测信息）、Kafka Offset Monitor（不更新，不建议使用）、CruiseControl 
* kafka-run-class.sh kafka.tools.JmxTool 加参数可以用于监控 jmx。

## 调优 kafka 集群

* 主意一下 num.replica.fetchers 这个参数，该值控制 broker 端 follower 副本从的 leader 副本拉取消息的最大线程数。
* min.insync.replicas 来制定了若要成功写入某条消息则必须要等待响应完成的最少 ISR 服本数。比较建议为 replication.factor -1，也可以为 1。
*  num.recovery.threads.per.data.dir 在多盘的情况下，适当变大来加快从崩溃中恢复。

## Kafka Connect 与 Kafka Streams

* 





















