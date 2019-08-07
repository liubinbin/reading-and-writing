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

Segment<K, V>[] segments

1. 集成自 ReentrantLock。
2. 有类型为 HashEntry 数组的变量 table，我们从中可以看到其实 Segment 本身就可以成为一个 HashMap，并且可以用 ReentrantLock 的特性将一些操作包在锁内。



## 值得关注方法和变量

1. Segment  继承了 ReentrantLock，可以很方便的实现加锁和解锁。
2. get 请求时，先根据 hash 值的高位获取到在 Segments 里的下标，获取到对应的 Segment，然后通过的hash 值的地位获取到在 table 里的下标，table 某一个下标对应的 HashEntry 其实就是对应 hash 值的所在链表的头，不停的往后搜索就可以了。
3. size 请求时，每个Segment 引入了一个 modCount 记录每次变化，如果在对所有 segment 的size统计的前后两次都 modCount 没变，则直接返回统计结果。否则再次尝试，如果尝试次数超过RETRIES_BEFORE_LOCK，则直接将每个 Segment 进行 lock，然后获取size的值。
4. put 请求时，每次操作都是添加到链表的头节点，虽然数据存在数组的最后一个，这样可以保证 get 请求不受影响。
5. remove 请求时，对对应的 Segment 进行加锁，然后寻找对应的值删除，最后解锁。
6. 使用 putOrderedObject 和 getObjectVolatile 来保证性能和原子性及非常高可见性，putOrderedObject 使用了 StoreStore 屏障而不是 LoadStore 屏障。
7. scanAndLock 和 scanAndLockForPut 方法用于获取锁，在获取过程中，添加了自旋逻辑，减少直接 lock 造成的性能损失，并且还顺带便利了链表。
8. remove 和 put 操作前是会执行lock操作，但 get 没有。主要是由于“增删都是原子操作+ voliate 读取”。
9. 当 Segment 中的 Entry 的个数超过阈值的话怎会进行 rehash 操作，在 rehash 操作时，会重用部分不需要重新组织的链表（从 lastRun 到尾节点的所有数据，在新链表中的哈希位置一样），以此来提高效率。







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

Node 的 hash 值有几种情况：

1. 如果是证书则是正常的节点。
2. 如果是 cMOVED，则说明正在转移。



sizeCtl 的值有好几种情况：

1. -1 代表在初始化
2. 否则就是参与 resize 的线程的加 1 的负数。
3. 否则记录下次 resize 的阈值。
4. 默认为0 



## 值得关注方法和变量

1. 在 get 请求时，根据 hash 值定位到 table 相应的下标的 Node，如果 第一个的 hash 值为负数，则直接调用find 方法获取对应的值，否则就类似链表一样向下搜索。
2. 在 put 和 remove 操作中，对 table[hash & (length - 1)] 使用 synchronized 来保护。
3. 在 put 请求时，也分为正常的在队列尾添加，或者是一个 TreeBin 节点则直接调用 putTreeVal。在添加结束之后，如果发现个数超过一定值，则会把此节点转化成树结构。
4. Node 中有一个Node<K, V> find(int h, Object k) 的方法，是从 this 开始搜索为k的值。
5. table 这个数组里的值用 cas 更新。
6. 如果链表长度大于 TREEIFY_THRESHOLD 的话，就转化成红黑树。
7. Node 的 hash 为-1时为 MOVED 等等。
8. size计算使用了 baseCount + counterCells 来求值。
9. transfer 方法其实是现在使用先通过 transferIndex 来竞争获取一个区间，然后去处理这个区间内的桶的节点。
10. 注意那个 helpTransfer 函数，在很多地方的实现都用了这种 help 操作。就是在操作的时候只给一个标记，可以别的线程看到或者能帮忙处理（在 ConcurrenSkipList 就有）。
11. 在执行 treeifyBin 如果 table 的长度小于 MIN_TREEIFY_CAPACITY 的话，则先执行 tryPresize，否则就改到TreeNode。




## 关键方法

transfer 方法注释如下：

```java
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            // 寻找分配到本线程的可以迁移的节点下标 i
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                	// 可以操作 i 的那个节点
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                	// 已分配完
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                	// 寻找一个桶，找到了 [nextBound， nextIndex]， 也就是 [bound，i] 的区间，各个线程通过 TRANSFERINDEX 控制。
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            // 检查是否需要结束
            // n 为旧的 table 的长度，nextn 为新的 table 的长度
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                	// 完成此线程参与的扩容
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    	// 说明还有线程在协助迁移
                        return;
                    // 说明会到最开始的第一个迁移的状态结束了
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
            	// 标记此下标对应的节点
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                	// 处理节点 f 
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

tryPresize方法注释如下：

```java
    private final void tryPresize(int size) {
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            if (tab == null || (n = tab.length) == 0) {
            	// 初始化
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
            	// 到最大的size，不能再扩容
                break;
            else if (tab == table) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    Node<K,V>[] nt;
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        // 参与写入迁移
                    	transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                	// 主动第一个迁移
                    transfer(tab, null);
            }
        }
    }
```

