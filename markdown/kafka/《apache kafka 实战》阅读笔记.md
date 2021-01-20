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
* 





























