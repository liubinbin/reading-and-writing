# kafka源码-producer

Producer 的代码作用是把客户端传入记录按 topic 和 partition 写入到 kafka 的 broker 端。由此可以猜测 producer 的源码有几个需要注意的点。

## 获取元数据

```
private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long maxWaitMs)
```

NetWorkClient.java 里的 DefaultMetadataUpdater 的 maybeUpdate 方法获取元数据。

由 public List<ClientResponse> poll(long timeout, long now) 调用。

poll 方法由 Sender 线程不停的调用，或 wakeup 调用。

## 把数据按 topic + partition 组织

每个 KafkaProducer 有一个类型为 RecordAccumulator 的成员变量。在 RecordAccumulator  有一个类型为 ConcurrentMap<TopicPartition, Deque<ProducerBatch>> 的成员变量 batches。

ProducerBatch 能暂存记录形成一个 batch。

## 发送数据到 broker 端

通过 this.sender.wakeup(); 来触发发送程序运行。
