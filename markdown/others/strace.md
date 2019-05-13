#strace命令解决问题

##场景

执行一个python程序，在个人用户下报ImportError，在root下不报错。

## 命令

通过strace -o file $CMD获取到在俩用户下执行程序的strace输出。使用vimdiff找到最开始不同的地方，然后检查那个路径的权限问题，处理掉就好了。

##strace

strace常用来跟踪进程执行时的系统调用和所接收的信号。其他参数有-f -F    -T -tt -e trace=all 等。
