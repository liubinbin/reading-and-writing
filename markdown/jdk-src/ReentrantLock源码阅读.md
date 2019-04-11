# ReentrantLock源码阅读



## 用法

ReentrantLock lock = new ReentrantLock();

lock.lock();

try{

​	....

} finally {

​	lock.unlock();

}

从中，我们可以看到基本就俩方法，lock和unlock。

lock实际调用了sync.lock();

unlock实际调用了sync.release(1);

此处需要注意，这俩方法本省并不会自己阻塞直到能获取锁。



## 主要结构

private final Sync sync;

Sync是ReentrantLock的内部类，其中NonfairSync和FairSync继承自Sync，而Sync继承自AQS。



##Sync主要方法

final boolean nonfairTryAcquire(int acquires):

   	1. 判断本进程是否是设置的进程，有一些检查。
     2. 使用AQS里state，将state的数字增加acquires。
      3. 如果state为0，则使用CAS将0改为acquires，更改state，state变量为volatile。

protected final boolean tryRelease(int releases) :

1. 判断本进程是否是设置的进程，有一些检查。
2. 如state减为0，则设置ExclusiveOwnerThread为null。
3. 将state的值减少releases。

###主要思想

state从0设置为正数时使用CAS，并且保留设置Thread到的ExclusiveOwnerThread中，acquire和release则只是更改state。



## NonfairSync和FairSync

默认为NonfairSync。

两者的lock方法主要就是调用AQS的acquire(1);

### NonfairSync.tryAcquire(int acquires)  

调用了Sync的nonfairTryAcquire方法

### FairSync.tryAcquire(int acquires)

1. 首先会判断state，如果没有比自己等待时间更长线程（hasQueuedPredecessors），则进入设置cas和setExclusiveOwnerThread的过程。
2. 如果自己是则ExclusiveOwnerThread则更改state。
3. 否则失败。

### 是否fair区别

如果是fair会判断等待时间，由AQS来实现，保存等待的线程的列表。此处十分重要。



## AbstractQueuedSynchronizer



### 主要变量

private transient volatile Node head;

private transient volatile Node tail;

private transient int state;

### 主要方法

public final boolean hasQueuedPredecessors()：

​	判断是否有比自己等待时间更长线程

public final void acquire(int arg):

​	先调用tryAcquire，如果失败则调用acquireQueued将自己添加到等待队列里。

private Node addWaiter(Node mode):

​	将本节点加入到有head和tail的构成的队列里。使用CAS操作更改head和tail。



## 主要思想

利用好AQS