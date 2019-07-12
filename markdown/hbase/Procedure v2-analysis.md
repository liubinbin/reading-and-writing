#Procedure v2学习





hbase的master里包含很多需要









MasterRpcServices接受请求

	getMaster().getMasterCoprocessorHost().preCreateTable(tableDescriptor, newRegions);

	//传入了一个latch，需要等待此latch执行countdown
	ProcedurePrepareLatch latch = ProcedurePrepareLatch.createBlockingLatch();
	submitProcedure(new CreateTableProcedure(
	    procedureExecutor.getEnvironment(), tableDescriptor, newRegions, latch));
	latch.await();
	
	getMaster().getMasterCoprocessorHost().postCreateTable(tableDescriptor, newRegions);


submitProcedure
	一些准备工作

	//写入wal
	store.insert(proc, null);
	
	//调用 scheduler.addBack(proc)，将proc加入各自的队列当中
	pushProcedure(proc);


ProcedureExecutor中的WorkerThread会不停去poll队列里的数据


getInitialState 获取到状态机的第一个状态

StateMachineProcedure里的保存状态

状态机在执行完第一个状态时释放latch

