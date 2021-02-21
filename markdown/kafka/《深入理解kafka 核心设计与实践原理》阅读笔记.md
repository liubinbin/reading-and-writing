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

* 

