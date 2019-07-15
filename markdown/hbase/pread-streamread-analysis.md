#pread vs stream read





















## HBase 里的代码

HFileBlock.java 

```
protected int readAtOffset(FSDataInputStream istream, byte[] dest, int destOffset, int size,
    boolean peekIntoNextBlock, long fileOffset, boolean pread)
    throws IOException {
  if (peekIntoNextBlock && destOffset + size + hdrSize > dest.length) {
    // We are asked to read the next block's header as well, but there is
    // not enough room in the array.
    throw new IOException("Attempted to read " + size + " bytes and " + hdrSize +
        " bytes of next header into a " + dest.length + "-byte array at offset " + destOffset);
  }

  if (!pread) {
    // Seek + read. Better for scanning.
    HFileUtil.seekOnMultipleSources(istream, fileOffset);
    // TODO: do we need seek time latencies?
    long realOffset = istream.getPos();
    if (realOffset != fileOffset) {
      throw new IOException("Tried to seek to " + fileOffset + " to " + "read " + size +
          " bytes, but pos=" + realOffset + " after seek");
    }

    if (!peekIntoNextBlock) {
      IOUtils.readFully(istream, dest, destOffset, size);
      return -1;
    }

    // Try to read the next block header.
    if (!readWithExtra(istream, dest, destOffset, size, hdrSize)) {
      return -1;
    }
  } else {
    // Positional read. Better for random reads; or when the streamLock is already locked.
    int extraSize = peekIntoNextBlock ? hdrSize : 0;
    if (!positionalReadWithExtra(istream, fileOffset, dest, destOffset, size, extraSize)) {
      return -1;
    }
  }
  assert peekIntoNextBlock;
  return Bytes.toInt(dest, destOffset + size + BlockType.MAGIC_LENGTH) + hdrSize;
}
```

pread调用 

```
先 istream.seek(offset);
后 in.read(buf, bufOffset, bytesRemaining);
```

stream read调用

```
ret = in.read(position, buf, bufOffset, bytesRemaining);
```



streamWrapper包装了一个DFSInputStream



##HDFS部分代码

DFSInputStream.java 

pread调用

```
public int read(long position, byte[] buffer, int offset, int length)
```

stream read

```
public synchronized void seek(long targetPos) throws IOException 
public synchronized int read(@Nonnull final byte buf[], int off, int len)
```





















































































