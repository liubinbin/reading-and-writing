# paldb解读

主要分析函数如下：

##StoreWriter的put方法

paldb的数据按key对应的byte数组的长度散列。不同key长度会有不同index file和data file。



```java
  public void put(byte[] key, byte[] value)
      throws IOException {
    int keyLength = key.length;

    //获取indexfile的stream，如果在indexFiles数组和indexStreams数组没有对应key长度索引文件则创建。
    DataOutputStream indexStream = getIndexStream(keyLength);

    // 在stream中写入key。
    indexStream.write(key);

    // 判断此key对应长度的最后一个插入的值是否和目前值一致。不明白此处的判断意义。
    byte[] lastValue = lastValues[keyLength];
    boolean sameValue = lastValue != null && Arrays.equals(value, lastValue);

    // 获取对应的长度已写入的数据的长度，这个值在后续close时会使用到。
    long dataLength = dataLengths[keyLength];
    if (sameValue) {
      dataLength -= lastValuesLength[keyLength];
    }

    // 在indexStream写入的长度（就是的value的offset），获取offset的长度，，并在maxOffsetLengths记录当前key长度的最大offset的长度，此数值在后面的计算slot时会使用到。
    int offsetLength = LongPacker.packLong(indexStream, dataLength);
    System.out.println("offsetLength: " + offsetLength + " maxOffsetLengths[keyLength]: " + maxOffsetLengths[keyLength]);
    maxOffsetLengths[keyLength] = Math.max(offsetLength, maxOffsetLengths[keyLength]);

    // 只有sameValue为false时才执行下面的命令：
    if (!sameValue) {
      // 获取datafile的stream，如果在dataFiles数组和dataStreams数组没有对应key长度索引文件则创建。
      DataOutputStream dataStream = getDataStream(keyLength);

      // 在datastream写入value长度和value值
      int valueSize = LongPacker.packInt(dataStream, value.length);
      dataStream.write(value);

      // 更新数据长度，此处可以认为是在datastream里的offset，在后续close时会使用到。
      dataLengths[keyLength] += valueSize + value.length;

      // 更新最后一个插入值的信息，在lastValues和lastValuesLength中。
      lastValues[keyLength] = value;
      lastValuesLength[keyLength] = valueSize + value.length;
	  // 更新所有插入的value的count计数
      valueCount++;
    }
	// 更新key的count计数, 此数和valueCount不一定一致。
    keyCount++;
    keyCounts[keyLength]++;
  }
```



##StoreWriter的close方法



```java
  public void close()
      throws IOException {
    // 关闭data file和index file的stream
    for (DataOutputStream dos : dataStreams) {
      if (dos != null) {
        dos.close();
      }
    }
    for (DataOutputStream dos : indexStreams) {
      if (dos != null) {
        dos.close();
      }
    }

    // Stats
    LOGGER.log(Level.INFO, "Number of keys: {0}", keyCount);
    LOGGER.log(Level.INFO, "Number of values: {0}", valueCount);

    // 新建一个list变量filesToMerge来存储要merge的文件，包括meta文件，index文件，data文件
    List<File> filesToMerge = new ArrayList<File>();

    try {

      // 写metadata文件（以metadata.dat结尾）
      File metadataFile = new File(tempFolder, "metadata.dat");
      metadataFile.deleteOnExit();
      FileOutputStream metadataOututStream = new FileOutputStream(metadataFile);
      DataOutputStream metadataDataOutputStream = new DataOutputStream(metadataOututStream);
      writeMetadata(metadataDataOutputStream);
      metadataDataOutputStream.close();
      metadataOututStream.close();
      filesToMerge.add(metadataFile);

      // 创建索引文件
      for (int i = 0; i < indexFiles.length; i++) {
        if (indexFiles[i] != null) {
          // 按的key长度索引文件。
          filesToMerge.add(buildIndex(i));
        }
      }

      // Stats collisions
      LOGGER.log(Level.INFO, "Number of collisions: {0}", collisions);

      // 把数据文件加入到filesToMerge文件
      for (File dataFile : dataFiles) {
        if (dataFile != null) {
          filesToMerge.add(dataFile);
        }
      }

      // 检查磁盘空间
      checkFreeDiskSpace(filesToMerge);
      // merge三类文件值一个文件outputStream中。
      mergeFiles(filesToMerge, outputStream);
    } finally {
      // 关闭outputStream，删除临时文件（filesToMerge）
      outputStream.close();
      cleanup(filesToMerge);
      // 最终所有的数据合并在outputStream（也就是我们指出的文件中）中。
    }
  }
```



##StoreWriter的writeMetadata方法



```java
  private void writeMetadata(DataOutputStream dataOutputStream)
      throws IOException {
    // 写入版本信息
    dataOutputStream.writeUTF(FormatVersion.getLatestVersion().name());

    // 写入当前时间
    dataOutputStream.writeLong(System.currentTimeMillis());

    // 获取对应长度有值的个数和目前key长度的最大
    int keyLengthCount = getNumKeyCount();
    int maxKeyLength = keyCounts.length - 1;

    // 写入key个数
    dataOutputStream.writeInt(keyCount);

    // 写入获取到的有值的key长度的个数keyLengthCount
    dataOutputStream.writeInt(keyLengthCount);

    // 写入获取到key长度最大值maxKeyLength
    dataOutputStream.writeInt(maxKeyLength);

    // 初始化datasLength
    long datasLength = 0l;
    // 针对每个key长度有值的长度执行如下命令
    for (int i = 0; i < keyCounts.length; i++) {
      // 判断是否有此长度的key
      if (keyCounts[i] > 0) {
        // 写入长度
        dataOutputStream.writeInt(i);

        // 写入key的个数
        dataOutputStream.writeInt(keyCounts[i]);

        // 写入slot个数（有key的个数除以loadFactor，loadFactor默认为0.75），此slot用于包含数据。
        int slots = (int) Math.round(keyCounts[i] / loadFactor);
        dataOutputStream.writeInt(slots);

        // 以下四步记录各个部分key对应的index和data的offset。
        // 写入slot大小(key长度 + 最大offset的长度)
        int offsetLength = maxOffsetLengths[i];
        dataOutputStream.writeInt(i + offsetLength);

        // 写入index的offset
        dataOutputStream.writeInt((int) indexesLength);

        // 偏移此长度对应的index需要的offset偏移
        indexesLength += (i + offsetLength) * slots;

        // 写入数据长度
        dataOutputStream.writeLong(datasLength);

        // 偏移本长度对应的data需要的offset偏移
        datasLength += dataLengths[i];
      }
    }

    // 写入序列化信息
    try {
      Serializers.serialize(dataOutputStream, config.getSerializers());
    } catch (Exception e) {
      throw new RuntimeException();
    }

    // 写入最终文件中index部分的offset和data部分的offset。index通过目前再加两个值的offset开始，data部分从在index之后再写完所有index的长度开始。
    int indexOffset = dataOutputStream.size() + (Integer.SIZE / Byte.SIZE) + (Long.SIZE / Byte.SIZE);
    dataOutputStream.writeInt(indexOffset);
    dataOutputStream.writeLong(indexOffset + indexesLength);
  }
```







##StoreWriter的buildIndex方法



```java
  private File buildIndex(int keyLength)
      throws IOException {
    // 初始化count（本长度key的个数），slots个数，offsetLength，slotSize(key长度+offset长度)
    long count = keyCounts[keyLength];
    int slots = (int) Math.round(count / loadFactor);
    int offsetLength = maxOffsetLengths[keyLength];
    int slotSize = keyLength + offsetLength;

    // 初始化index文件，此index文件没有temp
    File indexFile = new File(tempFolder, "index" + keyLength + ".dat");
    RandomAccessFile indexAccessFile = new RandomAccessFile(indexFile, "rw");
    try {
      indexAccessFile.setLength(slots * slotSize);
      FileChannel indexChannel = indexAccessFile.getChannel();
      // 使用了一个MappedByteBuffer
      MappedByteBuffer byteBuffer = indexChannel.map(FileChannel.MapMode.READ_WRITE, 0, indexAccessFile.length());

      // 初始化临时index文件的用于读的流
      File tempIndexFile = indexFiles[keyLength];
      DataInputStream tempIndexStream = new DataInputStream(new BufferedInputStream(new FileInputStream(tempIndexFile)));
      try {
        byte[] keyBuffer = new byte[keyLength];
        byte[] slotBuffer = new byte[slotSize];
        byte[] offsetBuffer = new byte[offsetLength];

        // 处理所有的key，由于slot为key个数除以一个系数，系统小于1，所以slot肯定可以放下所有的数据，如果slot过于，冲突会少，但是更多多余的空间会被浪费。
        for (int i = 0; i < count; i++) {
          // 读取key
          tempIndexStream.readFully(keyBuffer);

          // 读取的offset
          long offset = LongPacker.unpackLong(tempIndexStream);

          // 计算hash值
          long hash = (long) hashUtils.hash(keyBuffer);

          boolean collision = false;
          // 用一个探针去试探，检测冲突，这同时也是在散列。
          for (int probe = 0; probe < count; probe++) {
            int slot = (int) ((hash + probe) % slots);
            byteBuffer.position(slot * slotSize);
            byteBuffer.get(slotBuffer);

            long found = LongPacker.unpackLong(slotBuffer, keyLength);
            // 去试探的对应的位置是否有数据
            if (found == 0) {
              // 如果没有值，则说明没有冲突，写入key和offset。
              byteBuffer.position(slot * slotSize);
              byteBuffer.put(keyBuffer);
              int pos = LongPacker.packLong(offsetBuffer, offset);
              byteBuffer.put(offsetBuffer, 0, pos);
              break;
            } else {
              // 如果有值说明有冲突，则标记。
              collision = true;
              // Check for duplicates
              if (Arrays.equals(keyBuffer, Arrays.copyOf(slotBuffer, keyLength))) {
                throw new RuntimeException(
                        String.format("A duplicate key has been found for for key bytes %s", Arrays.toString(keyBuffer)));
              }
            }
          }
		  // 统计冲突个数
          if (collision) {
            collisions++;
          }
        }

        String msg = "  Max offset length: " + offsetLength + " bytes" +
                "\n  Slot size: " + slotSize + " bytes";

        LOGGER.log(Level.INFO, "Built index file {0}\n" + msg, indexFile.getName());
      } finally {
        // 关闭临时index文件输入流
        tempIndexStream.close();

        // 关闭index文件的channel，释放资源。
        indexChannel.close();
        indexChannel = null;
        byteBuffer = null;

        // 删除临时index文件
        if (tempIndexFile.delete()) {
          LOGGER.log(Level.INFO, "Temporary index file {0} has been deleted", tempIndexFile.getName());
        }
      }
    } finally{
      // 关闭index文件
      indexAccessFile.close();
      indexAccessFile = null;
      // 手动执行一次gc，所以在这时候不能在jdk参数中关闭手动gc。
      System.gc();
    }

    return indexFile;
  }
```



##StoreReader的get方法



```java
  public byte[] get(byte[] key)
      throws IOException {
    // 获取key的长度，检测key长度是否有数据在
    int keyLength = key.length;
    if (keyLength >= slots.length || keyCounts[keyLength] == 0) {
      return null;
    }
    // 计算hash值
    long hash = (long) hashUtils.hash(key);
    // 获取对应的key长度的slot个数
    int numSlots = slots[keyLength];
    // 获取对应的key长度的slot大小
    int slotSize = slotSizes[keyLength];
    // 获取index部分的offset
    int indexOffset = indexOffsets[keyLength];
    // 获取data部分的offset
    long dataOffset = dataOffsets[keyLength];
	// 用一个探针去试探对应位置上是否有数据。
    for (int probe = 0; probe < numSlots; probe++) {
      int slot = (int) ((hash + probe) % numSlots);
      // slotBuffer 为slotSize的byte数组, 将所有的key+offset打到一个固定大小slot的好处就是我们可以使用slotSize来定位。
      indexBuffer.position(indexOffset + slot * slotSize);
      indexBuffer.get(slotBuffer, 0, slotSize);
	  // 解析读取出来的对应slot中的数据
      long offset = LongPacker.unpackLong(slotBuffer, keyLength);
      // 判断此slot里是否有数据
      if (offset == 0) {
        return null;
      }
      // 比较slotBuffer和key是否完全相等
      if (isKey(slotBuffer, key)) {
        // mMapData来标记的是否使用MMap
        byte[] value = mMapData ? getMMapBytes(dataOffset + offset) : getDiskBytes(dataOffset + offset);
        return value;
      }
    }
    return null;
  }
```



## StoreReader的getMMapBytes方法



```java
  private byte[] getMMapBytes(long offset)
      throws IOException {
    // 一些bytebuffer的操作，暂不详细分析, 主要的是数据不能一下子读取出来，比较麻烦。
    //Read the first 4 bytes to get the size of the data
    ByteBuffer buf = getDataBuffer(offset);
    int maxLen = (int) Math.min(5, dataSize - offset);
    // 获取的size
    int size;
    if (buf.remaining() >= maxLen) {
      //Continuous read
      int pos = buf.position();
      size = LongPacker.unpackInt(buf);

      // Used in case of data is spread over multiple buffers
      offset += buf.position() - pos;
    } else {
      //The size of the data is spread over multiple buffers
      int len = maxLen;
      int off = 0;
      sizeBuffer.reset();
      while (len > 0) {
        buf = getDataBuffer(offset + off);
        int count = Math.min(len, buf.remaining());
        buf.get(sizeBuffer.getBuf(), off, count);
        off += count;
        len -= count;
      }
      size = LongPacker.unpackInt(sizeBuffer);
      offset += sizeBuffer.getPos();
      buf = getDataBuffer(offset);
    }

    // 初始化输出结果
    byte[] res = new byte[size];

    //Check if the data is one buffer
    if (buf.remaining() >= size) {
      //Continuous read
      buf.get(res, 0, size);
    } else {
      int len = size;
      int off = 0;
      while (len > 0) {
        buf = getDataBuffer(offset);
        int count = Math.min(len, buf.remaining());
        buf.get(res, off, count);
        offset += count;
        off += count;
        len -= count;
      }
    }

    return res;
  }
```



##StoreReader的getDiskBytes方法



```java
  private byte[] getDiskBytes(long offset)
      throws IOException {
    // seek到对应长度的offset
    mappedFile.seek(dataOffset + offset);

    // 从对应长度读取size
    int size = LongPacker.unpackInt(mappedFile);

    // 初始化输出结果
    byte[] res = new byte[size];

    // 读取数据
    if (mappedFile.read(res) == -1) {
      throw new EOFException();
    }

    return res;
  }
```



