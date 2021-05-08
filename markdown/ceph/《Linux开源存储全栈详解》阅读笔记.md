# 《Linux 开源存储全栈详解》阅读笔记

## 前言

* 在脑海里形成它的拓扑，并不断细化。

## Linux 开源存储

* 去除重复数据时，很多需要计算 FingerPrint。
* 原数据如果是基于内存需要启用日志来持久化数据。

## 存储硬件与协议

* IO服务时间包括的：1. 磁头寻道时间。2. 盘片旋转时间。3. 数据传输时间。
* 一般磁头寻道平均时间为 3-15ms。平均旋转时间为3ms。

## Linux 存储栈

* 用户角度读取数据为流的表现形式；存储设备角度为块的表现形式。文件系统则在两者之间。
* /proc/<pid>/maps 可查看映射信息，第1列为内存区域地质，第2列为内存区域大小，第3列为属性，第4列为内存映射文件。
* 与read/write相比，mmap的方式可以减少一次用户空间到内核空间的复制。
* 虚拟文件系统架在系统调用接口和不同的文件系统实现之间，让用户可以使用同一个接口操作不同文件系统。
* Btrfs 的原数据通过 B-Tree 管理。
* Page Cache是以物理页为单位对磁盘文件进行缓存的。通常为4K。
* 对文件缓存的写操作，采用了写时复制，等到要同步或清理缓存时再同步，一般会有延迟。
* 内核以块做为单位，大小一般为512Byte、1KB或4KB。
* 2.4开始，Buffer 不再是个的独立的缓存，包含在Page cache里。
* Direct I/O 传输不经过 Page Cache，减小内核缓冲区和用户空间的数据复制此处，降低对文件读写时所带来的CPU负载能力以及内存带宽的占用率。
* IO请求在经过通用块层之后会进入IO调度层。
* IO调度算法有的 noop、deadline和CFQ。
* 每个进程有自己私有的 Plug（做蓄流）队列的，在这里页会做合并。
* bcache 时内核使用固态硬盘做为缓存，有writeback、writethrough和writeaound。bucket是bcache的关键的结构，通过LRU算法替换。

## 存储加速

* SPDK 是用于与加速 NVMe SSD 作为后端存储使用的应用软件加速哭。核心是用户态、异步、轮询方式的 NVMe驱动。
* 扇区针对磁盘，块针对扇区。

## 存储安全

* MTTR、MTTF、MTBF（Mean Time Between Failure）。
* 在纠删码问题上，对数据分组，恢复也在组内进行。

## 存储管理与软件定义存储

* OpenSDS 为 Linux 下一个子项目，旨在通过软件定义存储参考架构和API标准的全球化推行。
* Libvirt 提供一个软件库来管理节点上的虚拟机。通过管理存储池和卷来为虚拟机提供存储资源。

## 分布式存储与Ceph

*  ceph 底层 RADOS 是对象存储系统，librados 基于此。

* OSD 层里每个 object 有标记符、二进制数据和原数据。

* OSD 向 monitor 集群汇报故障，客户端通过 OSD 连接 IO。

* 每个文件有 ino、文件由 object组成（oid）；hash（oid）% mask -> pgid；crush（pgid） -> （OSD1、OSD2）

* PG 和 PGP 的区别。

* 数据为 primary-replica 模型，强一致性，默认从 primary 读取，也可配置从 replica 读取。

* cache Tiering（写回、forward、readonly、readforward、readproxy、proxy）

* 块存储主要是需要做条带化，由 kernel/librbd 两种形式。

* cephFS 里有 MDS 根据目录长度、访问频率等分割；使用动态子树方法；quota可以针对目录。

* BlueStore 重要分为数据管理、原数据管理和空间管理。

  kvDB：RocketDB

  Allocator：最小单元格式化，使用 Bitmap 分配。

  BlueFS： for RocketDB

  元数据：共享 BlueStore 的块设备

  BlockDevice

* SeaStore 为新一代 ObjectStore。

* CRUSH 由 CRUSH map、CRUSH Rule 和 OSD Map 组成。

* 可以使用 curshTool 来查看信息。

* RBD 的 mirror 主要通过 journal 来实现。

* RBD snapshot 使用 COW 来做。

* ceph 通过 pglog 来数据恢复，pglog 包含对象信息和 pg 版本号，主要通过 last_complete 和 last_update 来控制。

* pglog 由 PG 锁。

* ceph 的一致性由 PG 层的 pglock、Store 层的 OpSequencer。

* RBDCache 可认为是块设备的内部缓存。

* barrier_passing 在事务基本，可提高效率。

* QoS 主要是设置规则、共享资源。

* 后端 QoS 在 OSD 内部按 PG 分组。

* mClock 主要由预留值、最小值和权重。

* 测试方法：fio -> 块；bonnie++ -> 文件；CoSBench 或 Swift-Bench -> 对象

* 集群性能数据可通过 Ceph Perf Counters 获取，可通过命令 ceph daemon <> perf dump。

* 综合测试分析工具由 CBT 和 CeTune。

* ObjectStore 的 fio 代码在 src/test/fio 中。

* OSD 层实现了 op-tracker 机制。

## OpenStack存储

* 无

## 容器存储

* 临时存储，有可写层，联合文件系统。
* k8s 里的 Label 机制，Deployment 概念。
* PV：持久化存储卷；PVC：向平台对于存储资源的请求。
* CSI 用于 k8s 和存储方解耦。