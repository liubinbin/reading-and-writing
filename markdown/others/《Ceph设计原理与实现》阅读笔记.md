# 《Ceph设计原理与实现》阅读笔记

## 计算为王-基于可扩展哈希的受控副本分布策略CRUSH

* 暂时调整reweight为0可以使节点暂时失效，迁回过程只需同步该OSD不在线期间产生的新数据即可。

## 性能之巅-新型对象存储引擎BlueStore

* PG有唯一的PGID，由Monitor统一管理，Monitor时采用paxos实现一致性。

## 时空博弈—纠删码原理与overwrites支持

* 无

## 迁移之美-PG 读写流程与状态迁移详解

* Log 用于解决 PG 副本之间的的数据一致性问题。
* 消息接受与分发 -> do_request -> do_op -> execute_ctx。
* 创建 PG、Peering（past_intevals -> GetInfo -> GetLog -> GetMissing -> Activate -> Recovery ->  Backfill）
* Backfil 是 Recovery 的一种特殊场景，指 Peering完成后，对 PG 实例实施增量同步。如果是新疆入，可以完全拷贝 Primary 的数据做全量同步。
* OSD 收到消息会封装为 op，通过函数 ms_fast_dispatch 分发。 
* 所有日志在PG中使用一个公共的日志队列进行管理。
* IO操作有来自客户端的ClientOp、OSD之间的SubOp，快照数据删除的SnapTrim、静默错误的Scrub、数据恢复和迁移的Recovery。

## 控制先行-存储服务质量QoS

* dmClock队列，一个为客户端的client队列，第二级为真实的请求队列。

## 无心插柳—分布式块存储RBD

* 一个 pool 由 rbd_directory 、rbd_children 和一个个 image 组成。
* rbd_directory 所记录的原数据由两个作用，一是列出所有的image，二是通过 image 名称和 image_id 互查。
## 应云而生—对象存储网关RGW

* 对象太大会分段上传。


## 经典重现—分布式文件系统 CephFS

* 使用了动态子树。
* MDS原数据存储由日志保证。
* MDS恢复通过segments划分日志。