org.apache.lucene.codecs.compressing

Class [CompressingStoredFieldsIndexWriter](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/compressing/CompressingStoredFieldsIndexWriter.html)

- [java.lang.Object](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html?is-external=true)

  org.apache.lucene.codecs.compressing.CompressingStoredFieldsIndexWriter

- All Implemented Interfaces:

  [Closeable](https://docs.oracle.com/javase/8/docs/api/java/io/Closeable.html?is-external=true), [AutoCloseable](https://docs.oracle.com/javase/8/docs/api/java/lang/AutoCloseable.html?is-external=true)

  ------

  ```java
  public final class CompressingStoredFieldsIndexWriter
  extends Object
  implements Closeable
  ```



基于block 的 [`Codec`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/Codec.html)s 的高效索引格式。

该 writer 生成一个文件，该文件可以使用 内存有效数据结构 加载到内存，快速找 包含了任何文档的 block。

 为了有紧凑的内存表示形式，对于每个 1024 chunks 的block，该索引计算了 每个 chuck 的  平均字节数，对于每个 chunk，仅存储了

-  ${chunk number} * ${average length of a chunk}
- 和 实际 chunk 起始偏移量 

的差值。

写入的数据如下：PackedIntsVersion, <Block>^BlockCount, BlocksEndMarker

![](.\pics\Block.png)

### 1. PackedIntsVersion

 [PackedInts.VERSION_CURRENT`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/util/packed/PackedInts.html#VERSION_CURRENT) as a [`VInt`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/store/DataOutput.html#writeVInt-int-)

### 2. BlocksEndMarker

 `0` as a [`VInt`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/store/DataOutput.html#writeVInt-int-), 这个标记了block 的结束，因为 blocks 不允许以 0 开始。

### 3. Block 

包含了 BlockChunks, <DocBases>, <StartPointers> 3个部分。

### 4. BlockChunks 

在 block 中 以[`VInt`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/store/DataOutput.html#writeVInt-int-) 编码的 chucks 个数。

### 5. DocBases 

包含了 DocBase, AvgChunkDocs, BitsPerDocBaseDelta, DocBaseDeltas 四部分。

### 6. DocBase 

chunks 的 block 的第一个 文档 ID，as a  [`VInt`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/store/DataOutput.html#writeVInt-int-)

### 7. AvgChunkDocs 

一个单独的 chunk中，文档数平均值，as a  [`VInt`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/store/DataOutput.html#writeVInt-int-)

### 8. BitsPerDocBaseDelta 

从 平均值 表示 增量 所需要的位数，使用 [ZigZag encoding](https://developers.google.com/protocol-buffers/docs/encoding#types)

### 9. DocBaseDeltas 

 [`packed`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/util/packed/PackedInts.html)  数组的每个元素 BitsPerDocBaseDelta 位数，表示 从 平均 doc base 的增量，使用 [ZigZag encoding](https://developers.google.com/protocol-buffers/docs/encoding#types)

### 10. StartPointers 

包含了  StartPointerBase, AvgChunkSize, BitsPerStartPointerDelta, StartPointerDeltas 四个部分

### 11. StartPointerBase 

block 中的 第一个 开始的 pointer，as a [`VLong`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/store/DataOutput.html#writeVLong-long-)

### 12. AvgChunkSize

压缩文档的平均大小，as a [`VLong`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/store/DataOutput.html#writeVLong-long-)

### 13. BitsPerStartPointerDelta 

从平均值的增量需要的位数，使用 [ZigZag encoding](https://developers.google.com/protocol-buffers/docs/encoding#types)

### 14. StartPointerDeltas 

[`packed`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/util/packed/PackedInts.html) 数组的每个元素 BitsPerStartPointerDelta 位数，表示 从 平均 start pointer 的增量，使用 [ZigZag encoding](https://developers.google.com/protocol-buffers/docs/encoding#types)

### 15. Footer

[`CodecFooter`](http://lucene.apache.org/core/7_5_0/core/org/apache/lucene/codecs/CodecUtil.html#writeFooter-org.apache.lucene.store.IndexOutput-)

Notes

- For any block, the doc base of the n-th chunk can be restored with `DocBase + AvgChunkDocs * n + DocBaseDeltas[n]`.
- For any block, the start pointer of the n-th chunk can be restored with `StartPointerBase + AvgChunkSize * n + StartPointerDeltas[n]`.
- Once data is loaded into memory, you can lookup the start pointer of any document chunk by performing two binary searches: a first one based on the values of DocBase in order to find the right block, and then inside the block based on DocBaseDeltas (by reconstructing the doc bases for every chunk).

**NOTE: This API is for internal purposes only and might change in incompatible ways in the next release.**