#AbstractQueuedSynchronizer源码阅读

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