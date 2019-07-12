HBase 行锁

hbase 行锁

	加锁
		// STEP 1. Try to acquire as many locks as we can and build mini-batch of operations with
		// locked rows
		miniBatchOp = batchOp.lockRowsAndBuildMiniBatch(acquiredRowLocks);
	解锁
		releaseRowLocks(acquiredRowLocks);
