# swift的replicator

简单记录几个重要的点。

## handoff_first模式

在处理job之前会将job排序，handoff的part排到列表前面。在顺序处理一个个请求job时，如果有handoff的job失败并且遇到非delete的job的话，就结束此轮replicate操作，同时打印“Handoffs first mode still has handoffs remaining.  Aborting current replication pass”的日志。所以出现此日志的时候，可以说明所有handoff的partition都已经被处理过了。

handoff的job失败逻辑：

	Error syncing handoff partition: Timeout (480s) => 
	handoff_partition_deleted = False => 
	handoffs_remaining +=1 =>
	遇到非delete的job就会报错 => 
	有Error报错，但是没有退出说明，需要做的非delete类型job不多。
##判断是否是delete的job

delete=len(nodes) > len(part_nodes) - 1

part_nodes是从ring文件从获取到的此partition对应的所有primary的device。

nodes是从part_nodes筛选的和本device的id不同的nodes。

如果delete为true，则代表本device为handoff，因为根据之前的nodes的筛选可以得出本device不在primary中。如果delete为false，则相反。