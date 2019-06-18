# HashMap源码阅读






## 主要关注的问题

1. hashmap的扩容
2. hash函数
3. count计算
4. 数据过大时的处理





## 实现

HashMap有一个数组组成，数组里的每个元数据为一个链表。






