# ConcurrentSkipListMap源码阅读及CCSmap讨论



## 主要类

```java
class Node<K, V> {
  K key;
  Object value;
  Node<K, V> next;
}

class Index<K, V> {
  Node<K, V> node;
  Index<K, V> down;
  Index<K, V> right;
}

HeadIndex<K, V> extends Index<K, V> {
  int level;
}
```



## Marker作用

key为null， value为this，来区别其与正常node。

从ConcurrentSkipListMap的注释中可以看到相关内容。注释里解释这marker是为了避免insert和delete的冲突，具体过程如下：

假设有 b -> n -> f 三个节点，想把n删除。

1. 设置n的value字段为null（CAS操作）。改变之后，n就已被考虑为删除了。

2. 在n后面添加一个marker（CAS操作）。改变之后结构如下：

   b -> n -> marker -> f 

3. 将n删除，删除之后结构如下：

   b  -> f 

   ​

## CAS操作

```java
n = b.next;

.....

z = new Node<K, V>(key, value, n);

if (!b.casNext(n, z))

	break;

```

上述代码显示了如下原子插入操作，并且操作可以保证一定是在b和n之间插入z，而不是在其他并发操作可能修改的值：

b -> n 到 n -> z -> n



## 添加Index

1. 获取rnd来决定添加多少层Index。
2. 添加Index，有可能会扩展层数（扩展层数只能一次扩展一层），在此时，即将插入index都已经完成node和down的设置，从最上面的index开始寻找插入地址。
3. q为游走的节点，在每次寻找可以插入的位置，如果插入则q向下走，如果大小不合适则向右走。其中通过link函数将index节点插入。



## 主要函数

1. findPredecessor：在base level的node里寻找小于指定值的Node。
2. helpDelete：通过调用n.helpDelete(b, f) 将n删除。第一种情况是在n后面增加marker，第二种情况是将n和后续的marker一并删除。
3. link：通过调用q.link(r, t)将t插入到q和r之间，主要是通过将t的后继设置为r，然后通过cas将q的后继从r改为t。



## 阿里的CCSMap的优化点

patch：HBASE-20312

1. index node直接encode到Node里，用long[] levelIndex来标识。

2. 用一个long来指代的地址，原文：We use a primitive long to record the offset of each Key-Value object in
   the chunk pool (the first 4 bytes is chunk id, and the second 4 bytes is
   inner offset of this chunk)

3. 不需要实现delete操作，这样的话减少了marker的相关逻辑，能够在设置next节点时通过cas执行成功。

4. boolean helpCasUpdateNextNode(int level, long b, long expected, long update)，当level为0时，这代表更新node链条，level不为0时，则代表更新index节点。 这个设计来自于getNextNode和levelIndex是相邻存储。

   ​

