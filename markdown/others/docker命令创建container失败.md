# docker命令创建container失败

今天在通过 --name 参数指定容器名创建 container 时失败了。提示“service endpoint with name xxx already exists.”。

1、通过 docker ps -a 无法找到的 xxx。

2、尝试通过命令 docker rm -f xxx 强制删除 xxx，发现无效。

3、依据 https://cloud.tencent.com/developer/article/1386135 提示查找 docker network inspect bridge，发现了 xxx。

4、通过命令 docker network disconnect --force bridge xxx 删除网络信息。

5、至此创建操作便可执行成功。