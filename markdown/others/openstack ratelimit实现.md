# openstack swift ratelimit实现

## 参数

1. rate_buffer_seconds：和缓存同步时间
2. ratelimit(各种类型)：确定的限速
3. clock_accuracy：时钟准确，一般为1000（1s）

## 实现

1. 计算每次请求可以使用到的时间，clock_accuracy除以ratelimit中的值，设为time_per_request。
2. 将缓存中的时间加上加上time_per_request，将此值于现在的时间比较，若前者小，则sleep差值。
3. 每隔rate_buffer_seconds同步一次缓存中的时间。此处为了解决通过时间判断的问题。

# 问题

1. 依赖时间的方式会导致proxy-server时间没有完全同步的集群的某些节点变的很慢，一直让慢的节点处于判断为需要sleep一段时间。