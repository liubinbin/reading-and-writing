# java process的使用注意事项

在java中如果要去启动另一个进程执行某一个任务，可以使用java中的process。但是在使用的时候需要注意一点。

## 流的正确对应

如果父进程中使用getErrorStream，那子进程中的输出需要以System.err.print的形式出现。

如果父进程中使用getInputStream，那子进程中的输出需要以System.out.print的形式出现。

如果出现混乱的情况，则可能出现流混乱导致子进程和父进程在流的输入输出都卡住。

##解决办法

1. 有推荐使用ProcessBuilder里面，可以使用redirectErrorStream方法将标准输出流和错误输出流合二为一，然后使用getInputStream去获取流。
2. 在子进程中统一使用一种形式。
3. 使用线程方式。