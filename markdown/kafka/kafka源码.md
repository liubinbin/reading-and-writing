# kafka源码

kafka.server.epoch.LeaderEpochCache.scala （leader epoch数据结构）、kafka.server.checkpoints.LeaderEpochCheckpointFile（checkpoint检查点文件操作类）
自身记录在自己High Watermark文件（replication-offset-checkpoint）中的信息
Log.scala L1140
KafkaApis.scala L493
follower Broker 3从同步副本列表中移除或追赶上leader log end offset，最新的消息才会认为提交。





cat /proc/vmstat | egrep "dirty|writeback"







/bin/kafka-consumer-group.sh --boot-server localhost:9092 --group test-group --describe --command-config  config/saslscram-adminclient.properties



/bin/kafka-consumer-group.sh --boot-server localhost:9092 --group test-group --reset-offsets --to-earliest --command-config  config/saslscram-adminclient.properties



/bin/kafka-consumer-group.sh --boot-server localhost:9092 --group test-group --reset-offsets --to-earliest --execute --command-config  config/saslscram-adminclient.properties	

​		--to-offset <offset>

​		--shift-by N

​		--to-datetime <datetime>

​		--by-duration <duration>

https://www.cnblogs.com/huxi2b/p/7284767.html





KafkaController.java 	L890 private[controller] def sendUpdateMetadataRequest(brokers: Seq[Int], partitions: Set[TopicPartition] = Set.empty[TopicPartition])







java -jar cmdline-jmxclient-0.10.3.jar - xx.101.130.1:9999 ObjectName Value

	kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions；
	kafka.controller:type=KafkaController,name=ActiveControllerCount
	kafka.network:type=RequestMetrics,name=TotalTimeMs,request={Produce|FetchConsumer|FetchFollower}
		Request total time
	kafka.network:type=RequestMetrics,name=RequestQueueTimeMs,request={Produce|FetchConsumer|FetchFollower}
		Time the request waits in the request queue
	kafka.network:type=RequestMetrics,name=RemoteTimeMs,request={Produce|FetchConsumer|FetchFollower}
		Time the request waits for the follower

kafkaConfig.scala



内核提供了直接IO的方式，O_DIRECT,可以绕过页缓存，直接把文件内容从堆中写到磁盘文件。

一个页缓存中的页如果被修改，那么会被标记成脏页。脏页需要写回到磁盘中的文件块。有两种方式可以把脏页写回磁盘，也就是flush。
	1. 手动调用sync()或者fsync()系统调用把脏页写回
	2. pdflush进程会定时把脏页写回到磁盘

基本把page cache的特点说了，它是面向内存，面向文件的。这正好说明了页缓存的作用，它位于内存和文件之间，文件IO操作实际上只和页缓存交互，不直接和内存交互。


metadata.fetch.timeout.ms（生产者的maxBlockTimeMs变量，默认值为60秒）：生产者第一次发送消息，如果主题没有分区，它等待元数据更新的最长阻塞时间（第7.3.2节第三小节）。

"max.block.ms";
The configuration controls how long <code>KafkaProducer.send()</code> and <code>KafkaProducer.partitionsFor()</code> will block."
                                                    + "These methods can be blocked either because the buffer is full or metadata unavailable."
                                                    + "Blocking in the user-supplied serializers or partitioner will not be counted against this timeout.";


kafkaProducer.java 	L1077
	public List<PartitionInfo> partitionsFor(String topic) {
	    Objects.requireNonNull(topic, "topic cannot be null");
	    try {
	        return waitOnMetadata(topic, null, maxBlockTimeMs).cluster.partitionsForTopic(topic);
	    } catch (InterruptedException e) {
	        throw new InterruptException(e);
	    }
	}

​	