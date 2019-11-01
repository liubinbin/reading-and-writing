# mybatis 使用经验

##　＃和$的使用区别

* "#" 会解析传入的变量值，适合 limit where 子句。
* "$" 不解析传入的变量值，适合 order by 子句。

## 提取公共部分

参考 https://www.cnblogs.com/EnzoDin/p/6445281.html

## order 部分

在 page 里实现一个针对 order 的 sql 语句的拼接，在 xml 中使用此字符串。