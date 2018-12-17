#swift-smallfiles里的锁

swift-smallfiles版本将多个object写成大的volume放在磁盘中，将hash值大volume和offset的信息写入leveldb中。为了防止多个请求同时对一个volume进行操作，此patch使用fcntl里的锁来保证，只有一个请求能排他的写一个volume。

fcntl使用

fcntl.flock(lock_file, operation)

  operation : 包括：

​    fcntl.LOCK_UN 解锁  解锁也可以通过os.close(lock_file)实现

​    fcntl.LOCK_EX  排他锁

​    fcntl.LOCK_SH  共享锁

​    fcntl.LOCK_NB  非阻塞锁

LOCK_SH 共享锁:所有进程没有写访问权限，即使是加锁进程也没有。所有进程有读访问权限。

LOCK_EX 排他锁:除加锁进程外其他进程没有对已加锁文件读写访问权限。

LOCK_NB 非阻塞锁:



operation可以多个在一起使用，比如fcntl.flock(lock_file,  fcntl.LOCK_EX| fcntl.LOCK_NB)

