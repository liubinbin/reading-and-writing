# 《Ceph之RADOS设计原理与实现》阅读笔记

## 一生万物-RADOS导论

* 对象演进与排序，用于实现 Backfill、Scrub。
* PGID = 0X4977FA mod 256 = 18 -> 1.18 （1 代表存储池标志）
* 在PG分裂与集群扩容时，分裂之前祖先PG的数量，PGP等保存数据。

## 计算寻址之美与数据平衡之殇-CRUSH

* Straw算法考虑了3个因素：固定输入、元素编号和元素权重。
* 单OSD承载100PG。
* 定制 CRUSH 使用 crushtool -i。
* 数据重平衡 balancer（reweight、weight-set和upmap）

## 集群的大脑-Monitor

* 使用了 paxos简化版，只有leader能处理请求。
* OSDMap集群管理只传递更改（Incremental）。
* 由强一致性协议保证Ceph。

##存储的基石-OSD

* OSD中由一个Monclient与Monitor通信。
* 网络通信AsyncMessager。
* OSD上电需要使用预留存储空间 100M。
* 故障检测由up、down、in、out四种状态。
* 粒度：1. 文件系统为 4K（块大小）；2. ceph 为 4MB。

## 高效本地对象存储引擎-BlueStore

* RMW + COW。
* 读可并发，写是排他。
* 写PG有OpSequencer。
* 带日志的写请求：写日志 + 覆盖写数据。
* extent {offset、length、data}
* 逻辑层有 onode（lextent blob）；物理层（pextent）。
* 同一对象所 extent 组成 extent-map。
* 磁盘空间块是最小单位（4K），改进为段管理（offset，length）。
* BlueStore使用位图，将空间列表存盘。
* BitmapAllocator -> BitmapAreaInternal -> BitmapAreaLeaf -> BitmapAreaZone -> BitmapEntry。
* BitmapEntry 为 64 位无符号整数，记录 64个物理上相邻的块状态。
* BlueFS中BlueStore的原数据有RocketsDB 管理。
* dir_map 找到文件夹、通过 file_map 找到文件。

## 移动的对象载体-PG

* Primary + 副本的形式。
* Authority History为权威日志。
* Everion（Epoch + version）由primary生成，单调递增。
* 读：客户端 -> 哈希值 -> PGID -> OSD -> OP -> 读。
* 写：PG事务 -> PGBackend -> 副本写入应答 -> Primary 回复完成。
* do_request -> do_op -> execute_ctx。
* Primary 不断通过 eval_repop 评估 RepGather 的进展。
* Peering：1. past_interval 2. GetInfo(权威日志) 3. GetLog 4. GetMissing 5. Activate(固化 Peering 结果)。

## 在线数据恢复-Recovery 和 Backfill

* 根据日志可以恢复：Recoery，不可恢复：Backfill。
* 对象修复分为 Pull 和 Push。
* Peering时为每个副本生成完整的missing列表。
* CRUSH尽可能以Primary能走日志恢复就让其坐Primary（不是选举产生）。
* Backfill被调度，Primary需获取一个对象的严格有序列表。使用backfill_info结构来记录信息。

## 数据正确性与一致性的守护者-Scrub

* scrub需要PG里对象有序，从而分批操作。
* PG粒度排序时时使用RocksDB排序。
* 对象扫描由Primary发起，汇报给Primary。
* Chunky-scrub会阻塞部分对象。

## 基于dmClock的分布式流控策略

* dmClock 基本权利为 Qos模版：预留时间、权重和上限。
* dmClock实现为两级映射（client和IO请求队列），三个二叉树（预留、权重和上限）
* mgr 是将集群中的一些指标暴露给外界使用。

## 纠删码原理与实践

* n个输入，构造m个等式，时对应的m元一次方程组由唯一解。

