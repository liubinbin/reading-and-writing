# 《深入理解kafka 核心设计与实践原理》阅读笔记

## 初识kafka

* Conumser使用拉（pull）模式从服务端拉去消息。
* 服务端里的 listener 和 advertised.listeners 的区别。

## 生产者

* 生产者拦截器（interceptor）可以对消息做定制修改，有onSend、onAcknowledgement 和 close 方法。onAcknowledgement 在 io 线程执行，需要不耗时，可以用于统计等。
* Sender 线程从<分区，Deque<ProducerBatch>>的保存形式转变为<Node,List<ProducerBatch>>的形式。
* ProducerBatch 是按 batch.size 大小组织的，可以利用到 BufferPool 的内存管理机制。
* 发送出去还未收到响应的是在 InFlightRequests，形式是 Map<NodeId, Deque<Request>>，可以通过 max.in.flight.requests.per.connection 来影响（每个链接最左为响应的请求）。
* request.timeout.ms 比 replica.lag.time.max.ms 要大一些，减少因客户端重试而引起的消息重复的概率，可以看看。

## 消费者

* subscribe 可以订阅 topic，assign 可以订阅 TP，partitionFor 可以获取到所有分区信息。
* 推荐使用通用的序列化功能。
* 自动提交位移是在 poll 方法里执行的。
* wakeup 是 KafkaConsumer 唯一从别的线程安全调用的方法的，可用用于退出 poll 逻辑。
* 可以使用 seek 方法指定 offset。
* 使用滑动窗口来实现异步处理消息消费和消息处理。

## 主题与分区

* 创建 topic 时可以使用 replica-assignment 参数手动指定分区副本的分配方案。
* kafka-topic.sh 脚本里的 unavailable-partitions 参数可以看到没有 leader 副本的分区，此处可以和 offlinePartitions MBean 比较。
* topics-with-overrides 来显示有覆盖配置的 topics。
* 可以使用 org.apache.kafka.clients.admin.KafkaAdminClient 来通过代码控制 topic。
* 可以通过 create.topic.policy.class.name 设置服务校验 topic 合法性。 
* 自动分区平衡使用 auto.leader.balance.enable 来控制，具体阈值为 leader.imbalance.per.broker.percentage。通过 kafka-preferred-replica.election.sh 脚本来进行 leader 的重新平衡，有 --path-to-json-file 参数可以指定需要执行的 TP。
* kafka-reassign-partitions.sh 脚本来处理集群扩容和节点失效的场景，生成计划有 --generate --topics-to-move-json-file reassign.json --broker-list 0,1，执行有 --execute --reassignment-json-file。可以同时设置速度。
* 下限 broker 时可以先关闭或重启 broker，这样 broker 就不会有 leader 分区了。
* 副本限流有配置 follower.replication.throttled.rate 和 leader.replication.throttled.rate，可通过 kafka-configs.sh 来设置。
* ls /proc/{$PID}/fd | wc -l 查看占用文件描述符个数。

## 日志存储

* 变长字段使用了 ZigZag 编码，具体内容参见 org.apache.kafka.common.utils.ByteUtils 来实现。
* Record 的字段使用了很多 delta，可以节省空间。
* 日志内容可以使用 bin/kafka-dump-log.sh --files $file-path --print-data-log 来查看。
* 稀疏索引使用 log.index.interval.bytes 来控制，索引文件通过 MappedByteBuffer 将索引映射到内存中。
* 时间戳索引里保存的是消息相对偏移量，消息的文件位置需要从偏移量索引中获取。
* linux 脏页刷写可以查看 vm.dirty_background_ratio、vm.dirty_expire_centisecs 和 vm.dirty_writeback.centisecs 相关内容。
* 作者不建议使用 log.flush.interval.messages 和 log.flush.interval.ms 来强制刷写磁盘，而通过多副本。
* 建议设置 vm.swappiness=1。
* java 内的零拷贝技术是 FileChannel.transferTo 方法。
* 零拷贝通过 DMA 讲内容复制到 Read Buffer，讲 fd 给 Socker Buffer，然后给网卡设备。

## 深入服务器

* fetch.max.wait.ms 默认为 500。
* 在 fetch 稳定后，fetchRequest 会根据 session 和 epoch 来缓存 topic 等信息，这样就可以不用传递 topic 等信息了。
* 时间轮通过多层来实现，第二层时间轮的 tickMs 为第一层时间轮的 interval。interval = tickMs * wheelSize。
* 时间轮是个存储定时任务的环形队列，里面存放了 TimerTaskList。
* 时间轮通过将任务传入 DelayQueue 来推进时间来实现任务处理。
* 延时管理器通过 DelayOperationPurgatory 来做处理，每个操作配置了一个 SystemTimer，底层采用了时间轮实现。
* 通过 controller_epoch 来控制控制器的唯一性。
* kafka 服务内有 kafka-shutdown-hock 的钩子处理，处理资源释放和 leader 分区迁移。
* 在 meta.properties 里会保存 broker.id ，若和其他地方的配置不符合会报 InconsistentBrokerIdExcpetion。
* bootstrap.servers 的地址只用于获取元数据信息，可以将此功能与读写分开。腾讯逻辑集群实现是类似思路。

## 深入客户端

* 客户端有 partition.assignment.strategy 来制定策略，有 RangeAssignor、RoundRobinAssignor 和 StickyAsignor（减少不必要的分区移动）。
* 每个消费组由 GroupCorrdinator 和 ConsumerCorrdinator 协调。
* GroupCoordinator 有 Utils.abs(groupId.hashCode) % groupMetadataTopicPartitionCount 来确定分区号，然后找此节点的 Leader。
* rebalance 包含 FIND_COORDINATOR -> JOIN_GROUP -> SYNC_GROUP -> HEARTBEAT 。
* 在 heartbeat 阶段时，只要保持了心跳，就会被认为是活跃的，是没有问题的。心跳线程是个单独线程。
* 关注 offsets.retention.minutes 的默认值为 10080（即 7 天），超过这个时间点的消费位移信息会被删除。
* kafka-console-consumer.sh --xxxxx  --formatter ‘kafka.coordinator.group.GroupMetadataManager$OffsetsmessageFormatter’ 来查看 __consumer_offsets。
* 在删除主题时，会一并将消费位移信息删除。
* 如果开启幂等（设置 enable.idempotence 为 true），则 broker 会为每个<PID, 分区>维护一个序列号。
* 事务可以保证对多个分区写入操作的原子性。
* transactionalID （客户端指定）获取 PID 的同时，会获取一个单调递增的 producer epoch。
* 消费端有一个参数 isolatiion.level，默认 read_uncommitted，也可以设置为的 read_committed。
* 除了普通消息，还有一种控制消息（ControlBatch），有两种类型（COMMIT 和 ABORT）。
* TransactionCoordinator 来控制，会将消息持久化到 __transaction_state 中。
* 查找 TransactionCoordinator -> 获取 PID -> 开启事务 -> Consume-Transform-Produce -> 提交或终止事务。 

## 可靠性探究

*  数据和服务做副本处理。
* 当 follower 副本将 leader 副本 LEO 之前的日志全部同步时，则认为该 follower 副本已经追上 leader，则更新 lastCaughtUpTimeMs 标识，replica.lag.time.max.ms 参数则用来判断这个值用的。
* 区分 LEO 和 HW，HW = min{LEO}。
* recovery-point-offset-checkpoint、replication-offset-checkpoint 和 log-start-offset-checkpoint 分别记录了 LEO、HW 和 logStartOffset 信息。
* HW 是下一轮 FetchRequest/FetchResponse 请求之后更新的。
* 引入 leader epoch 来解决 Partition 之间数据不一致和丢失的问题。
* 只有在 ISR 里的副本可以进入选举，除非设置了 unclean.leader.election.enable 为 true（判断方法值得看看）。

## Kafka 应用

* 