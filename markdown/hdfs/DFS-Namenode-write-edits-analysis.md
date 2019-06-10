# NameNode写edits log

ref: https://juejin.im/post/5bec278c5188253e64332c76

使用机制：分段加锁 + 内存双缓冲



写Edits的主要方法是FSEditLog的 void logEdit(final FSEditLogOp op) ，此方法会被各种操作调用，操作都记录在FSEditLogOp中，再往上走的话，我们可以看到的是NameNodeRpcServer，此类是处理namenode的RPC请求。

具体实现如下：

```java
  void logEdit(final FSEditLogOp op) {
    boolean needsSync = false;
    // 第一次加锁
    synchronized (this) {
      assert isOpenForWrite() :
        "bad state: " + state;
      
      // wait if an automatic sync is scheduled
      // 等待，直到isAutoSyncScheduled变为false
      waitIfAutoSyncScheduled();

      // check if it is time to schedule an automatic sync
      // 获取一个txid，调用editLogStream.shouldForceSync() 返回是否需要sync
      needsSync = doEditTransaction(op);
      if (needsSync) {
        isAutoSyncScheduled = true;
      }
    }

    // Sync the log if an automatic sync is required.
    if (needsSync) {
      // 调用sync
      logSync();
    }
  }
```

* FSEditLog里有一个成员变量txid，是一个递增的数字。
* FSEditLog里有一个isAutoSyncScheduled来继续是否有自动的sync被调用。

