# 《Ceph设计原理与实现》阅读笔记

## 计算为王-基于可扩展哈希的受控副本分布策略CRUSH

* 暂时调整reweight为0可以使节点暂时失效，迁回过程只需同步该OSD不在线期间产生的新数据即可。

## 性能之巅-新型对象存储引擎BlueStore

* PG有唯一的PGID，由Monitor统一管理，Monitor时采用paxos实现一致性。

