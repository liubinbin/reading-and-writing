# AtomicLong & LongAdder源码阅读

在多线程准确计数的情况下，一般使用能实现原子操作的AtomicLong来作为计数变量类型，可以帮我们省去很多的麻烦。

## AtomicLong

AtomicLong的主要成员变量为volatile修饰的value。其中getAndIncrement，getAndAdd和incrementAndGet等方法都是通过unsafe的getAndAddLong方法实现。而getAndAddLong方法具体实现如下：

```java
	public final long getAndAddLong(Object var1, long var2, long var4) {
        long var6;
        do {
            var6 = this.getLongVolatile(var1, var2);
        } while(!this.compareAndSwapLong(var1, var2, var6, var6 + var4));

        return var6;
    }
```

通过compareAndSwapLong不停的重试知道成功。此方法在极度高的并发可能存在很大的竞争失败。而LongAdder就解决其中一些极高并发的场景。

##LongAdder

从类名和注释看，此类主要是为了提供高并发的add和update用的，用了一些空间来获取更高吞吐。从提供的API看，主要就是increment，decrement和sum。

从sum方法看，LongAdder类其实是牺牲了一些一致性来获取更高吞吐。sum的最终结果的获取其实就是对base和cells数组里的值求和的结果。由此我们可以看到LongAdder的大概思路应该是将add操作分布到base和cells数组来获取更高的并发。

LongAdder的increment和decrement内部是add方法实现。所以主要分析一下add方法。分三步：

1. 尝试cas修改base的值，如果成功则返回。
2. 尝试cas修改cells数组中getProbe()获取到的下标的值，如果成功则返回。
3. 调用Striped64里的longAccumulate方法。 

longAccumulate方法具体实现如下：

```java
    final void longAccumulate(long var1, LongBinaryOperator var3, boolean var4) {
        int var5;
        if((var5 = getProbe()) == 0) {
            ThreadLocalRandom.current();
            var5 = getProbe();
            var4 = true;
        }

        boolean var6 = false;

        while(true) {
            Striped64.Cell[] var7 = this.cells;
            int var9;
            long var10;
            if(this.cells != null && (var9 = var7.length) > 0) {
                Striped64.Cell var8;
                if((var8 = var7[var9 - 1 & var5]) == null) {
                    if(this.cellsBusy == 0) {
                        Striped64.Cell var32 = new Striped64.Cell(var1);
                        if(this.cellsBusy == 0 && this.casCellsBusy()) {
                            // 初始化对应的下标的值，并且以var1初始化，成功就可以退出了。
                            boolean var33 = false;

                            try {
                                Striped64.Cell[] var14 = this.cells;
                                int var15;
                                int var16;
                                if(this.cells != null && (var15 = var14.length) > 0 && var14[var16 = var15 - 1 & var5] == null) {
                                    var14[var16] = var32;
                                    var33 = true;
                                }
                            } finally {
                                this.cellsBusy = 0;
                            }

                            if(var33) {
                                break;
                            }
                            continue;
                        }
                    }

                    var6 = false;
                } else if(!var4) {
                    var4 = true;
                } else {
                    // 对对应下标的Cell尝试更改，如果成功则退出。
                    if(var8.cas(var10 = var8.value, var3 == null?var10 + var1:var3.applyAsLong(var10, var1))) {
                        break;
                    }

                    if(var9 < NCPU && this.cells == var7) {
                        if(!var6) {
                            var6 = true;
                        } else if(this.cellsBusy == 0 && this.casCellsBusy()) {
                            // 发现数组长度小于processor个数时，执行扩容。
                            try {
                                if(this.cells == var7) {
                                    Striped64.Cell[] var34 = new Striped64.Cell[var9 << 1];

                                    for(int var35 = 0; var35 < var9; ++var35) {
                                        var34[var35] = var7[var35];
                                    }

                                    this.cells = var34;
                                }
                            } finally {
                                this.cellsBusy = 0;
                            }

                            var6 = false;
                            continue;
                        }
                    } else {
                        var6 = false;
                    }
                }

                var5 = advanceProbe(var5);
            } else if(this.cellsBusy == 0 && this.cells == var7 && this.casCellsBusy()) {
                // cells为空时初始化。
                boolean var12 = false;

                try {
                    if(this.cells == var7) {
                        Striped64.Cell[] var13 = new Striped64.Cell[2];
                        var13[var5 & 1] = new Striped64.Cell(var1);
                        this.cells = var13;
                        var12 = true;
                    }
                } finally {
                    this.cellsBusy = 0;
                }

                if(var12) {
                    break;
                }
            } else if(this.casBase(var10 = this.base, var3 == null?var10 + var1:var3.applyAsLong(var10, var1))) {
                break;
            }
        }

    }
```

getProbe方法是从Thread.currentThread()里获取 threadLocalRandomProbe。

cellsBusy用来标记Cells数组正在扩容获取数组上的值正在数值化。cellsBusy是个volatile变量，并且提供了一个casCellsBusy的方法来原子性的更改。

## 性能比较

测试代码例子：https://github.com/liubinbin/pan/tree/master/src/main/java/cn/liubinbin/experiment/atomiclongvsaddr