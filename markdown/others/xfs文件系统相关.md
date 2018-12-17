# xfs文件系统属性查看

xfs文件系统可允许为文件写attr。swift特殊选用了xfs文件系统，在swift中，object的属性信息被encode在attr中。

##安装

apt-get install attr

##使用

get $filePath

获取属性，将获取到的属性带入如下命令，获取对应的值

get -n name $filePath