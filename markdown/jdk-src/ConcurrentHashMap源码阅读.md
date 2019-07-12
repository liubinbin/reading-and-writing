# ConcurrentHashMap源码阅读

# 1.7版本

分段锁实现

##主要类

```java
static final class HashEntry<K, V> {
  final int hash;
  final K key;
  volatile V value;
  volatile HashEntry<K, V> next;
}

static finall class Segment<K, V> extends ReentrantLock implelements Serializable{
  tranisent volatile HashEntry<K, V>[] table;
  tranisent int count;
  tranisent int modCount;
  
}
```



## 主要结构

Segment<K, V>[] segments：



## 值得关注方法和变量

1. Segment 继承了ReentrantLock。

2. 使用putOrderedObject和getObjectVolatile来保证性能和原子性及非常高可见性，putOrderedObject使用了StoreStore屏障而不是LoadStore屏障。

3. scanAndLock和scanAndLockForPut方法遍历了并且在逻辑中添加了自旋逻辑，减少直接lock造成的性能损失。

4. 在put时，每次操作都是添加到链表的头节点，虽然数据存在数组的最后一个，这样可以保证get请求不受影响。

5. size计算时，引入了一个modCount，如果在segment统计的前后都modCount没变，则直接返回统计结果，否则再次尝试，如果尝试次数过多则直接进行lock。

6. remove和put操作前是会执行lock操作，但put没有。主要是由于“增删都是原子操作+voliate读取”。

7. 在rehash操作时，会重用部分不需要重新组织的链表（从 lastRun 到尾节点的所有数据，在新链表中的哈希位置一样），以此来提高效率。

   ​



# 1.8版本



##主要类

```java
static class Node<K, V> impelements Map.Entry<K, V> {
  final int hash; //如果是个负值，则代表是有特殊使用
  final K key;
  volatile V val;
  volatile Node<K, V> next;
}
```



## 主要结构

transient volatile Node<K, V>[] table;

​	存储数据使用，大小一直是2的幂。

transient volatile Node<K, V>[] nextTable;

​	只有在resize中会使用。	





## 值得关注方法和变量

1. Node中有一个Node<K, V> find(int h, Object k) 的方法，是从this开始搜索为k的值。
2. table这个数组里的值用cas更新。
3. 在put和remove操作中，对table[hash & (length - 1)] 使用synchronized来保护。
4. 如果链表长度大于TREEIFY_THRESHOLD的话，就转化成红黑树。
5. Node的hash为-1时为MOVED等等。
6. size计算使用了baseCount + counterCells来求值。

