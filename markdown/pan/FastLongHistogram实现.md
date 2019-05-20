#FastLongHistogram实现

##主要思想

	如果我们要去计算百分位，有如下两种方法：
	1. 保存所有的数据，然后排序计算。
	2. 保存最大值和最小值，然后按百分位计算。
	我们从这俩方法中可以直观看到，我们要么很慢，要么误差很大。所以我们看到hbase源码中对百分位计算的优化方法。
	我们给一个预估最小值和最大值，将这个区间，在这之外增加仨区间，最小值到预计最小值，预计最大到10倍预计最大值，10倍预计最大值到最大值。在加入数据时，只记录每个区间的数量。这样我们减小需要记录的数据，并且也减小误差。具体实现如下。

##成员变量

    private final LongAdder[] counts;
    // inclusive
    private final long binsMin;
    // exclusive
    private final long binsMax;
    private final long bins10XMax;
    private final AtomicLong min = new AtomicLong(Long.MAX_VALUE);
    private final AtomicLong max = new AtomicLong(0L);
    
    private final LongAdder count = new LongAdder();
    private final LongAdder total = new LongAdder();
    
    // set to true when any of data has been inserted to the Bins. It is set after the counts are
    // updated.
    private volatile boolean hasData = false;

##初始化

​	counts为长度为numBins+3的LongAdder数组，numBins默认为255。

##添加

	更新min, max, count, total。
	int pos = getIndex(value);
	this.counts[pos].add(count);
##获取分位

	寻找百分位落在哪个区间，然后通过此区间的最大值和最小值计算预估相关的值，然后返回。
	在寻找落入的区间时，使用总数和百分比相乘的数值寻找，因为我们只保存了各个区间的值，通过总数和值来计算是比较符合实际的。
