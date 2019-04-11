# 基本命令

##创建topic

```
./bin/kafka-topics.sh --create --zookeeper zk1:2181,zk2:2181,zk3:2181 --replication-factor 3 --partitions 3 --topic liubb
```

##查看topic信息

```
./bin/kafka-topics.sh --describe --zookeeper zk1:2181,zk2:2181,zk3:2181 --topic liubb
```

##列出topic信息

```
./bin/kafka-topics.sh --list --zookeeper zk1:2181,zk2:2181,zk3:2181
```

##生产消息

```
./bin/kafka-console-producer.sh --broker-list broker1:9092,broker2:9092,broker3:9092 --topic  liubb
```

##消费消息

```
./bin/kafka-console-consumer.sh --bootstrap-server broker1:9092,broker2:9092,broker3:9092 --from-beginning --topic liubb
```

##消费者性能测试

```
./bin/kafka-consumer-perf-test.sh --broker-list broker1:9092,broker2:9092,broker3:9092 --topic liubb
```

##生产者性能测试

```
./bin/kafka-producer-per-test.sh --topic liubb --num-records 100000000 --throughput 100000 --record-size 100 --producer.config ../config/producer.properties
```

##解析log数据文件

```
./bin/kafka-run-class.sh kafka.tools.DumpLogSegments --files filePath --print-data-log
filePath可以指log，index和timeindex文件。
```

##读取元数据topic

```
./bin/kafka-simple-consumer-shell.sh --topic __consumer_offsets --partition 11 --broker-list localhost:9092,localhost:9093,localhost:9094 --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter"
```

#工具参考网址

https://cwiki.apache.org/confluence/display/KAFKA/System+Tools

































