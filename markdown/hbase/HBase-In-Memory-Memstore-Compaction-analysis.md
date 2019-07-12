# HBase In-Memory Memstore Compaction















参考资料：https://yq.aliyun.com/articles/573702/



使用

```
<property>
 <name>hbase.hregion.compacting.memstore.type</name>
 <value><none|basic|eager|adaptive></value>
</property>
```

也可以单独设置某个列族的级别：

```
create ‘<tablename>’, 
{NAME => ‘<cfname>’, IN_MEMORY_COMPACTION => ‘<NONE|BASIC|EAGER|ADAPTIVE>’}
```



内存结构







测试

























































