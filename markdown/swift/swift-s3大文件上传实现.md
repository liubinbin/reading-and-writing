# swift 大文件上传实现

参考：https://lingxiankong.github.io/2017-07-28-s3curl-swift.html

* 在container里获取uploadId
* 按文件顺序制定参数partNumber将文件上传
* 写一个描述文件，说明我们要上传的文件是怎么样的。
* 将此文件上传到对应object的uploadId中。
* 完成。