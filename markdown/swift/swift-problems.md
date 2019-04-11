# 问题

在架构swift+swift3+keystone的系统中。

##使用python的boto3出现了如下错误

check you key and signing method。

1. 检查配置，是否有和线下集群不一致的地方，经过仔细对比发现没有不一致的地方。
2. 通过sudo tcpdump -vv tcp port 8080 -w abc.curl 获取各种语言的客户端发送到proxy server的包，对比这些包的header。发现有一个叫Authorization的header不一致，出错的header内容带HMAC内容，而正确的内容带的是AWS。
3. 有了以上重大发现之后，拿HMAC和Authorization到网络上搜索，发现有人在有类似内容的网页中踢到了是算sign时的版本。在客户端的构造函数加入一个值config=Config(signature_version='s3')，代码就可以运行通了。
4. 正常运行代码之后，肯定要多花一些时间研究一下为什么，通过debug和阅读boto3的代码，发现两个机器下载到boto3的代码对于默认sign的算法的选择不一致，一个选择s3 一个选择s3v4。
5. 到此，此问题就告一段落了。


## keystone高可用问题

keystone-mange时使用的ip如果为A，其他做HA的ip为B，C
在使用openstack的客户端时，访问A可以，访问B和C不行
在通过接入swift时，访问A，B和C都可以

	此处是个大的风险点
