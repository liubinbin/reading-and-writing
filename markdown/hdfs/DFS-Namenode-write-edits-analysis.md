# NameNode写edits log

ref: https://juejin.im/post/5bec278c5188253e64332c76

使用机制：分段加锁 + 内存双缓冲

## 基础知识

在带QJM的HA模式下，active需要负责写远程的journalnode和本地journalFile；standy负责生成fsimage，然后active从standby那里下载获取。

所以对于namenode而言，每来一个请求，在修改内存之后需要写本地文件和远程写journalnode。

FSEditLog类中有editsDirs，sharedEditsDirs和isSyncRunning的主要成员变量。

* editsDirs：包含本地journal文件和journalnode地址。
* sharedEditsDirs：journalnode地址。
* isSyncRunning代表是否在做sync操作。
* myTransactionId是一个ThreadLocal变量，用来记录当前请求的txid。
* synctxid记录已sync的txid。

## 源码分析

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
      // 1. 获取一个txid，2. 并且写入数据。3. 调用editLogStream.shouldForceSync() 返回是否需要sync，
      needsSync = doEditTransaction(op);
      if (needsSync) {
        // 在双buffer交换完buffer，确定txid才会更改isAutoSyncScheduled
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


logSync继续调用protected void logSync(long mytxid)，txid从myTransactionId获取。

```java
protected void logSync(long mytxid) {
    long syncStart = 0;
    boolean sync = false;
    long editsBatchedInSync = 0;
    try {
      EditLogOutputStream logStream = null;
      synchronized (this) {
        try {
          printStatistics(false);

          // if somebody is already syncing, then wait
          // 如果自己还没被sync，并且有人已经在sync则等待。
          while (mytxid > synctxid && isSyncRunning) {
            try {
              wait(1000);
            } catch (InterruptedException ie) {
            }
          }
  
          //
          // If this transaction was already flushed, then nothing to do
          //
          if (mytxid <= synctxid) {
            return;
          }

          // now, this thread will do the sync.  track if other edits were
          // included in the sync - ie. batched.  if this is the only edit
          // synced then the batched count is 0
          editsBatchedInSync = txid - synctxid - 1;
          // 记录写txid，然后可以确定本轮sync需要执行的内容，其实就是双buffer中一个。
          syncStart = txid;
          isSyncRunning = true;
          sync = true;

          // swap buffers
          try {
            if (journalSet.isEmpty()) {
              throw new IOException("No journals available to flush");
            }
            // 实际调用EditsDoubleBuffer的setReadyToFlush，底层其实交换了bufCurrent和bufReady
            editLogStream.setReadyToFlush();
          } catch (IOException e) {
            final String msg =
                "Could not sync enough journals to persistent storage " +
                "due to " + e.getMessage() + ". " +
                "Unsynced transactions: " + (txid - synctxid);
            LOG.error(msg, new Exception());
            synchronized(journalSetLock) {
              IOUtils.cleanupWithLogger(LOG, journalSet);
            }
            terminate(1, msg);
          }
        } finally {
          // Prevent RuntimeException from blocking other log edit write 
          // 设置isAutoSyncScheduled为false
          doneWithAutoSyncScheduling();
        }
        //editLogStream may become null,
        //so store a local variable for flush.
        logStream = editLogStream;
      }
      
      // do the sync
      // 跳出synchronized，执行sync，此时双buffer中一个可以提供写入。
      long start = monotonicNow();
      try {
        if (logStream != null) {
          logStream.flush();
        }
      } catch (IOException ex) {
        synchronized (this) {
          final String msg =
              "Could not sync enough journals to persistent storage. "
              + "Unsynced transactions: " + (txid - synctxid);
          LOG.error(msg, new Exception());
          synchronized(journalSetLock) {
            IOUtils.cleanupWithLogger(LOG, journalSet);
          }
          terminate(1, msg);
        }
      }
      long elapsed = monotonicNow() - start;
  
      if (metrics != null) { // Metrics non-null only when used inside name node
        metrics.addSync(elapsed);
        metrics.incrTransactionsBatchedInSync(editsBatchedInSync);
        numTransactionsBatchedInSync.addAndGet(editsBatchedInSync);
      }
      
    } finally {
      // Prevent RuntimeException from blocking other log edit sync 
      synchronized (this) {
        if (sync) {
          synctxid = syncStart;
          for (JournalManager jm : journalSet.getJournalManagers()) {
            /**
             * {@link FileJournalManager#lastReadableTxId} is only meaningful
             * for file-based journals. Therefore the interface is not added to
             * other types of {@link JournalManager}.
             */
            if (jm instanceof FileJournalManager) {
              ((FileJournalManager)jm).setLastReadableTxId(syncStart);
            }
          }
          // 修改isSyncRunning
          isSyncRunning = false;
        }
        // 通知在等待线程本次请求已结束。
        this.notifyAll();
     }
    }
  }
```

## 总结

1. 这个设计是为了吞吐量的提升而做的。
2. 此设计可让在某个选中线程发送时，别的请求还可以写入Edit或者在等待sync结束（isSyncRunning变为false）。
3. 此设计和hbase的wal写入线程其实有写类似的目的，参考http://liubinbin.cn/2018/12/30/hbase-src-wal/