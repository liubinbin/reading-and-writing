# mysql revoke 误操作

## 背景

在跑程序的时候发现mysql连接部分报权限问题。然后就开始针对mysql做权限操作，没想到操作失误，可能是revoke的范围过大，导致root用户无法登陆mysql（由于开发环境，基本就是用mysql连接）。

发现无法连接，立马到mysql服务器备份所有的mysql数据，开始找解决房屋内

## 解决方案

最开始预备方案，以下方案没尝试：

1. 使用 --skip-grant-tables 进入mysql，然后修改mysql.user的内容（不确定能不能操作，以后可以尝试一下）。
2. 用别的mysql的底层库文件替换。

最终解决方案：在等待远程处理问题的时候发现有另外的同事已经连上mysql，并且没有推出，所以在那个客户端执行

```sql
grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option
flush privileges;
```

然后在mysql服务器执行赋权操作。

## 经验教训

1. 权限问题的操作要慎重。

