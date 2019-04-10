# CopyOnWriteArrayList源码阅读



## 主要结构

transient ReentrantLock lock = new ReentrantLock();

private transient volatile Object[] array;



## 主要思想

1. 所有的更改（包括添加，删除）互斥，在CopyOnWriteArrayList中有一个ReentrantLock类型的变量。但是get操作不够互斥锁的保护。
2. 所有的更改操作都在新建一个elements来做为存储修改后的数据的地方。最后才将此地址更新到this.array中
3. array变量用volatile 来修饰，保持的可见性，如果在get时获取到旧的array其实也没事，可能会造成数据不是最新的（更新array非原子操作，但是不影响大局）。



