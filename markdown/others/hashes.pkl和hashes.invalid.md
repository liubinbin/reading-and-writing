# hashes.pkl和hashes.invalid

swift的object-server中保存了很多object的文件，而object-server之间相互的同步需要知道各自有什么object，此时如果去文件系扫描的话，由于数据很多可能会非常慢。此时swift通过引入一个hashes值的缓存和一套缓存机制来加速此过程。

此机制包括两个文件，一个保存hashes值的hashes.pkl，另一个是记录由插入，删除等操作可能引起的hash值变更的hashes.invalid。两者组合来加速hashes值的获取。有点类似存量+增加变更的感觉。

HASH_FILE = 'hashes.pkl'
HASH_INVALIDATIONS_FILE = 'hashes.invalid'

##def read_hashes(partition_dir)

读取相应partition下的HASH_FILE，返回结果hashes，读取结束后会设置('valid', True)和('updated', -1)。

在读取之前会设置('valid', False)，如果读取文件和解析时抛出不被捕获的异常可能发现valid为False

##def write_hashes(partition_dir, hashes)

将hashes写入到相应partition下HASH_FILE，在写入之前会设置('valid', False)，'updated'为当前时间。

##def consolidate_hashes(partition_dir)

此方法会结合HASH_FILE和HASH_INVALIDATIONS_FILE给出hashes值。

1. 获取partition目录的锁。
2. 调用read_hashes获取hashes值。
3. 读取HASH_INVALIDATIONS_FILE文件获取其中的suffix。将hashes中对应的suffix的值设置为None。
4. 如果HASH_INVALIDATIONS_FILE文件有值，则在HASH_FILE 写入新和成hashes。将HASH_INVALIDATIONS_FILE 文件置为空。

## def invalidate_hash(suffix_dir)

在相对应partition下的HASH_INVALIDATIONS_FILE文件中写入指定的suffix。

## def __get_hashes(self, device, partition, policy, recalculate=None, do_listdir=False)

此方法用来获取当前的系统中的hashes值，会被replicator.py和server.py的REPLICATE方法调用。

1. 通过consolidate_hashes获取当前的hashes值保存在orig_hashes中。
2. 如果valid为False，则重新获取hashes，并且设置hashes为('valid', True)。
3. 在hashes中设置recalculate对应的suffix为None。
4. 重新计算hashes中对应的值为None的suffix，每计算一个hashed+1，并且设置modified为True。
5. 如果modified为True，获取partition目录的锁，如果partition对应的HASH_FILE还是orig_hashes，则调用write_hashes重新写入hashes值，再次调用本方法。否则返回hashed, hashes。此方法估计是为了保证hashes不在调用的期间发生更改。