#实现一个object-store

想要达到的目的

- 使用netty等项目支撑
- 使用range，hash等找设备
- 能达到比较高的性能
- 持续更新和重构
- 能学习在jvm中自己管理内存
- 不要过度设计
- 不要过多为小kv设计


## 接口设计

- 接口设计需要简单
- 不应该让用户感受到confused
- 接口应该尽量对称
- 客户端不应该接入过多的逻辑

## 底层文件格式的设计

- HFile中数据格式和ambry 的格式，都使用了block的概念。
- 需要使用bf先去筛选
- 需要index记录key和的value的offset，保持最新的数据来对付多版本。如果是hash值的话，还需要解决冲突。




## Multipart

- 将大的object分成的小的object发送。



## 目标

- done is better than perfect
- 2019年元旦发布第一版



##疑问

- ambry 写文件是以什么级别阻塞的







##最终设计

- fe通过netty去接受请求，装置不同的handler。
- 对于object的请求，寻求policy，寻找device（ip：dev）。hash和range两种方式。
- 对于bucket请求，应该是所有人都需要存储。
- be也通过netty去接受请求。






jetty + 一个object一个文件 =》 快速实现

netty + 文件格式 





##实现需要

- 一个key到file+offset的hashmap。
- 一个文件按offset来写入value值。





====== 从HttpContent获取bytebuf，转成bytebuffer写入file的中，可以参考netty源码中的AbstractDiskHttpData.java的相关实现。实现这个文件的接口其实也可以。

##实现思路

- 如果直接写入传递给decode的offer的文件的话，有一些头和尾需要去掉，否则写入的文件会有问题。
- 可以向filechannel传递position来设置。
- 直接修改netty的源代码，自己写替换。
- 重写实现Decoder和FileUpload，将文件名和offset的传入到fileupload中，然后在创建channel时设置offset。
- https://blog.csdn.net/yhl_jxy/article/details/79335384 fileChannel的用法。







读取思路

- 从hashmap中获去file+offset。
- 往ctx中write一个带offset的DefaultFileRegion，参考HttpStaticFileServerHandler的实现。

