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