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
* 





























