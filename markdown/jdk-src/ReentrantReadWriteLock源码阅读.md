# ReentrantReadWriteLock源码阅读



## 用法





## 主要结构





##Sync主要方法

1. 将一个int用分成两部分来











### NonfairSync

final boolean writerShouldBlock()

​	直接返回false。

final boolean readerShouldBlock()

​	TODO



###FairSync

final boolean writerShouldBlock()

​	判断是否有前驱节点在排队

final boolean readerShouldBlock()

​	判断是否有前驱节点在排队



### ReadLock

lock方法：





unlock方法：

















### WriteLock

lock方法：





unlock方法：



















## 主要思想

利用好AQS