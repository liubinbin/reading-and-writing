#查看cpu高的线程

通过jps获取线程号 



top -Hp 25606 



printf '%x\n' 25608 



jstack 25606 | grep 6408