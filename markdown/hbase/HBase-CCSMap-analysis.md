# HBase In-Memory Memstore Compaction




## 阿里的CCSMap的优化点

patch：HBASE-20312

1. index node直接encode到Node里，用long[] levelIndex来标识。
2. 用一个long来指代的地址，原文：We use a primitive long to record the offset of each Key-Value object in
   the chunk pool (the first 4 bytes is chunk id, and the second 4 bytes is
   inner offset of this chunk)
3. 不需要实现delete操作，这样的话减少了marker的相关逻辑，能够在设置next节点时通过cas执行成功。
4. boolean helpCasUpdateNextNode(int level, long b, long expected, long update)，当level为0时，这代表更新node链条，level不为0时，则代表更新index节点。 这个设计来自于getNextNode和levelIndex是相邻存储。
5. 在concurrentSkipList的close操作中直接把整个chunk回收。
6. 在chunk内有个变量nextFreeOffset，在此左侧则已被占用，在此后侧则未被占用。此变量通过cas来变更，回收时重置。
7. usedChunkQueue保存的chunk按未使用的空间排序。



