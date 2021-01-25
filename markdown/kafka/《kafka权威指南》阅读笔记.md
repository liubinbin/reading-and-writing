# 《kafka权威指南》阅读笔记

## 安装kafka

* num.recovery.threads.per.data.dir 配置每个目录使用线程数。
* broker 的 CPU 在有压缩的情况下可能会用于解压数据获取设置偏移量。
* vm.dirty_ratio 参数控制被内核精华曾刷新到磁盘之前的脏页数量，可以监测 cat /proc/vmstat | egrep "dirty|writeback"。
* 操作系统 noatime 可以减少磁盘访问。

## kafka生产者

* max.in.flight.requests.per.connection 参数指定了生产者在收到服务器响应之前可以发送多少个消息。

## kafka消费者

* 消费者通过向群组协调器的broker 发送心跳来维持它们和群组的从属关系。
* 和 consumer.poll 在一个循环内的操作需要尽快结束，因为心跳也是在这个轮训发送出去的。
* fetch.min.bytes、fetch.max.wait.ms和max.partition.fetch.bytes 来控制的 broker 给消费者返回数据。
* session.timeout.ms 和 heartbeat.interval.ms 关系消费者和服务器之间的连接。

## 深入kafka

* 控制器使用 epoch 来避免“脑裂”。脑裂指两个节点同时认为自己是当前的控制器。
* follower 想 leader TP 发送请求时完全有序的。
* 当前正在写入数据的分片为活跃分片，不会被删除。
* broker 会为每一个分片打开一个句柄。
* 磁盘保存格式与生产者发送过来的格式一致，这样可以直接写入磁盘，不必再做其他操作。
* bin/kafka-run-class.sh kafka.tools.DumpLogSegments 使用 --deep-iteration 参数。
* kafka 的清理是做 compact ，这个功能感觉用处不大。这个功能和墓碑消息匹配。

## 可靠的数据传递

* 消息只有在被写入所有的同步副本之后才被认为是已提交的。
* broker 和主题级别可以设置 min.insync.replicas 来设置最少同步副本数。
* VerifiableProducer 和 VerifiableConsumer 关注一下。
* 故障测试最好包括1、客户端从服务器断开连接。2、首领选举。3、一次重启 broker。4、一次重启消费者。5、一次重启生产者。
* 客户端有 jmx 度量指标。重要生产者治标包括 error-rate和retry-rate。
* Burrow 监测 comsumer-log。
* 每条消息有消息生成时间，消费者可以通过检测当前时间和此时间的差来判断。

## 构建数据管道

* kafka connect

## 跨集群数据镜像

* MirrorMaker 在数据中心之间镜像数据。
* 其他方案有 uReplicator 和 Replicator。

## 管理kafka

* bin/kafka-topics.sh 有 --under-replicated-partitions 和 --unavailable-partitions 参数。
* bin/kafka-config.sh --entity-type clients -entity-name <client-id>
* kafka-preferred-replica-election.sh 启动的首选的副本选举，一般不建议。
* kafka-reassign-partitions.sh 来修改 topic 的分区，包括 -generate、-execute、-verify。
* kafka-run-class.sh kafka.tools.DumpLogSegments --file **.log --print-data-log。--index-sanity-check 检查无用的索引，--verify-index-only 检查索引的匹配度。
* bin/kafka-replica-verification.sh 检查副本是否一致。
* zk 中 /controller 保存控制器，/admin/reassign_partitions 来保存分区重分配。


## 监控kafka

* kafka.server.type=ReplicaManager,name=UnderReplicatePartitionis 获取作为首领的broker 有多少分区处于非同步分区，此治标为 broker 级别。
* kafka-assigner 工具可以做分区的分配。
* kafka.controller:type=KafkaController,name=ActiveControllerCount 获取活跃控制器度量指标。
* kafka.server:type=KfkaRequestHandlerPool,name=RequestHandlerAvgIdlePercent 获取请求处理器空闲率。小于 20% 可能有问题。
* kafka 0.10 版本引入了一种新的格式，偏移量可以直接附加在消息批次里，可以避免 broker 执行大量的解压缩和重新压缩。
* 主题流入字节度量指标：kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec。是客户端发往 broker 的流量。
* 主题流入字节度量指标：kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec，流出包括副本流量。
* 主题流入消息度量治标：kafka.serverLtype+BrokerTopicMetric,name=MessageInPerSec。
* 离线分区（有 controller 提供）数量度量指标：kafka.controller:type=KafkaController,name=OfflinePartitionsCount。
* java.lang:type=GarbageCollector,name=G1 Old Generation 获取 fgc 信息。
* java.lang:type=OperatingSystem 中的 MaxFileDescriptorCount 和 OpenFileDescritproCount 分别代表 fd 要的最大值和目前的值，需要关注。
* 生产者里的 record-error-rate （无法发送的消息）和 request-latency-avg （生产者请求道 broker 所需要的平均时间）。
* 需要捋清楚字节、记录、请求和批次之间的关系。
* 生产者里的指标：kafka.product:type=producer-metrics,client-id=CLIENTID；kafka.producer:type=product-node-metrics,client=id=CLIENTID,node-id=BROKERID；kafka.product:type=product-topic-metrics,client.id=CLIENTID,topic=TOPICNAME。
* 消费者里的指标：kafka.consumer:type=consumer-metrics,client=CLIENTID；
* 延时可以试试 LinkedIn 的 burrow 监控系统。
* 端到端监控可以看看 kafka Monitor 进行监控。

## 流式处理

* ​无
