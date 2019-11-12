[TOC]

文件： `src/kudu/cfile/block_compression.h`

# 经过压缩的`CFile Block`的格式

## `CFile version 1`
```
// 4-byte little-endian: compressed_size
//    the size of the compressed data, not including the 8-byte header.
// 4-byte little-endian: uncompressed_size
//    the expected size of the data after decompression
// <compressed data>
//
```

说明：从上面可以看出，除了“压缩以后”的数据以外，有`2`个`4字节`的“头信息”，即`header`的长度为`8`个字节。

注意区分：这里的`header`是指“每个`CFile Block`的`header`信息”，而不是`CFile`自身的`header`（对应的是`CFileHeaderPB`结构）。

## `CFile version2`

```
// 4-byte little-endian: uncompressed_size
//    The size of the data after decompression.
//
//    NOTE: if uncompressed_size is equal to the remaining size of the
//    block (i.e. the uncompressed and compressed sizes are equal)
//    then the block is assumed to be uncompressed, and the codec should
//    not be executed.
// <compressed data>
```

说明：在采用`kudu v2`版本时，如果`uncompressed_size`的大小是等于“当前`CFile Block`”的剩余大小（也就是说，“压缩过的数据”和“未压缩的数据”是相同大小的），那么这个`CFile Block`会被假定为“**没有进行过压缩**”，也就是说，遇到这种情况，“是不需要进行‘解压缩’”的。  

# 注意事项
从本文件的两个类实现可以看出：

1. `CFile Block`是执行“压缩”和“解压缩”的最小单位。
2. 在`Block Cache`中，也是以`CFile Block`为单位的，而且保存的是“已经解压过的”`CFile Block`(这样从`Block Cache`中取出来的数据，就不需要再次进行解压缩)。

# `class CompressedBlockBuilder`

这个类，是用来对`CFile Block`的原始数据进行压缩。

在`kudu`目前的实现中：
1. “写数据”一定是按照`v2`版本进行；
2. “读数据”要按照版本分别进行。因为要兼容旧版本，如果遇到旧版本的数据，也要能够进行读取。

注意：本类是有状态的，在每次调用`Compress()`以后，压缩的结果保存在`buffer_`中。直到本类析构，或者下一次调用`Compress()`，才会将`buffer_`中的内容清理掉。


## 成员

### `static kHeaderLength`
因为现在已经始终使用`v2`版本进行写数据。在`kudu v2`版本的`CFile Block`的结构中，`header`的长度为`4`。（参见前面介绍的`kudu v2`版本的组织结构）。

### `const codec_`
对应的“压缩器”对象。

**注意：这是一个`常量指针`。（即每次构建这个对象，相应的“压缩器”对象只是传递一个指针，不需要新建出来对象）**  

按照`kudu`现在的实现，在每个进程中，只有`3`个“压缩器”对象(`SnappyCodec`, `Lz4Codec`, `ZlibCodec`)。这3个“压缩器”对象都是无状态的，可以被任意多个线程同时使用。

### `buffer_`
用来保存 对`CFile Block`的数据进行压缩以后的结果。

```
public:
  static const size_t kHeaderLength = 4;

private:
  const CompressionCodec* codec_;
  faststring buffer_;
```
## 接口列表

### 构造函数
传入一个“压缩器”对象的指针。 

注意：在当前`CompressedBlockBuilder`的生命周期内，该“压缩器”应该是一直有效的。

### `Compress()`
本类唯一的接口。

说明1：因为本类就是为了对`Block`进行压缩，所以唯一的接口就是“压缩”(`Compress()`)。

说明2：被压缩的数据，通过参数`data_slices`进行传入。压缩的结果，在参数`result`中。

说明3：压缩结果`result`中所“间接引用”的数据，和“压缩率”有关。
即在对当前数据进行压缩以后，会计算一下“压缩率”。   
根据压缩的比率，在`result`中所“间接引用”的数据，有两种可能的结果：
1. 如果压缩率符合要求，那么`result`所“间接引用”的数据，在当前对象的“缓存池”（`buffer_`）中。
2. 如果压缩率比较低，那么不再进行压缩，即在`result`中保存的，就是传入的`data_slices`。（注意：这时`result[0]`的长度为4，表示`header`信息，这个表示`header`的`Slice`所间接引用的数据，是在`buffer_`中的）

```
// 'codec' is expected to remain alive for the lifetime of this object.
  explicit CompressedBlockBuilder(const CompressionCodec* codec);

  // Sets "*data_slices" to the compressed version of the given input data.
  // The input data is formed by concatenating the elements of 'data_slices'.
  //
  // The slices inside the result may either refer to data owned by this instance,
  // or to slices of the input data. In the former case, the slices remain valid
  // until the class is destructed or until Compress() is called again. In the latter
  // case, it's up to the user to ensure that the original input data is not
  // modified while the elements of 'result' are still being used.
  //
  // If an error was encountered, returns a non-OK status.
  Status Compress(const std::vector<Slice>& data_slices,
                  std::vector<Slice>* result);

Status CompressedBlockBuilder::Compress(const vector<Slice>& data_slices, vector<Slice>* result) {
  size_t data_size = 0;
  for (const Slice& data : data_slices) {
    data_size += data.size();
  }

  // On the read side, we won't read any data which uncompresses larger than the
  // configured maximum. So, we should prevent writing any data which would later
  // be unreadable.
  if (data_size > FLAGS_max_cfile_block_size) {
    return Status::InvalidArgument(Substitute(
        "uncompressed block size $0 is greater than the configured maximum "
        "size $1", data_size, FLAGS_max_cfile_block_size));
  }

  // Ensure that the buffer for header + compressed data is large enough
  // for the upper bound compressed size reported by the codec.
  size_t ub_compressed_size = codec_->MaxCompressedLength(data_size);
  buffer_.resize(kHeaderLength + ub_compressed_size);

  // Compress
  size_t compressed_size;
  RETURN_NOT_OK(codec_->Compress(data_slices,
                                 buffer_.data() + kHeaderLength, &compressed_size));

  // If the compression was not effective, then store the uncompressed data, so
  // that at read time we don't need to waste CPU executing the codec.
  // We use a user-provided threshold, but also guarantee that the compression saves
  // at least one byte using integer math. This way on the read side we can assume
  // that the compressed size can never be >= the uncompressed.
  double ratio = static_cast<double>(compressed_size) / data_size;
  if (compressed_size >= data_size || // use integer comparison to be 100% sure.
      ratio > FLAGS_min_compression_ratio) {
    buffer_.resize(kHeaderLength);
    InlineEncodeFixed32(&buffer_[0], data_size);
    result->clear();
    result->reserve(data_slices.size() + 1);
    result->push_back(Slice(buffer_.data(), kHeaderLength));
    for (const Slice& orig_data : data_slices) {
      result->push_back(orig_data);
    }
    return Status::OK();
  }

  // Set up the header
  InlineEncodeFixed32(&buffer_[0], data_size);
  *result = { Slice(buffer_.data(), compressed_size + kHeaderLength) };

  return Status::OK();
}
```

说明1： 如果“需要压缩的数据”大小超过了`FLAGS_max_cfile_block_size`，那么返回错误。

> 备注：在读取的时候，如果一个`CFile Block`的“未压缩的”数据大小超过了`FLAGS_max_cfile_block_size`，那么也是不进行读取的。  
>  所以，在这里（在写入的时候），如果发现一个`CFile Block`的大小，超过了`FLAGS_max_cfile_block_size`，那么就应该直接拒绝掉。

说明2: 通过计算`CompressionCodec::MaxCompressedLength()`方法，获取到在当前压缩方法下、当前要压缩的数据，在压缩以后的 数据量的最大值，并确保“用来保存压缩结果”的“缓存池”中有足够的空间（`faststring::resize()`）。

说明3：缓存中，既要容纳压缩后的数据，还有`block header`。所以在`faststring::resize()`的时候，传入的大小是`(MaxCompresssedLength() + kHeaderLength)`。

说明4：调用`CompressionCodec::Compress()`，执行压缩。  注意，在“缓存池”中的“前`4`个字节”，是用来保存`CFile Block Header`的。  
所以，压缩后的数据，是从“第`5`个”字节开始的。

说明5：如果压缩率比较低（“压缩后的数据量”/“压缩前的数据量” > `FLAGS_min_compression_ratio == 0.9`），那么不进行压缩。  
注意：这时，仍然会用到“缓存池”。  
在“缓存池”中的内容是`header`信息（即“数据在压缩前”的大小，即“源数据的大小”）。

也就是说，如果压缩率比较低，那么在`std::vector<Slice>* result`中，这个“列表的大小”比“传入的`data_slices`的大小” 大 `1`。（因为在`result[0]`是`header`所对应的`Slice`对象。）  

因为`header`的长度为`4`，所以`result[0]`这个`Slice`对象的长度为`4`。 即这时，`buffer_.size()`也是`4`。

> 备注：这个优化是在`commit_id: d9bd5b8752efd4ae407043e491b7fd1e0b885a92`时引入的。

说明6：如果“压缩率”符合要求，那么在`result`中，只有一个`Slice`对象。它所间接引用的数据，在当前对象的“缓存池”（`buffer_`）中。

说明7：这里在比较“压缩率”的时候，用了两个比较条件

```
double ratio = static_cast<double>(compressed_size) / data_size;
  if (compressed_size >= data_size || // use integer comparison to be 100% sure.
      ratio > FLAGS_min_compression_ratio) {
  ...          
}
```

**使用条件：`compressed_size >= data_size` 的原因：**

首先，这里要实现的**真正目的是： 只要“压缩后”的数据量没有变小，（即“压缩后”的数据量，不小于 “压缩前”的数据量），那么就存储的就一定是“未压缩”的数据**。

其次，对于`FLAGS_min_compression_ratio`，在该`flags`的赋值是，并没有严格要求它的值必须是“小于`1`”的。也就是说，如果用户误用，是可以赋值为一个`大于等于1`的值的。所以只用`ratio > FLAGS_min_compression_ratio`这一个条件是达不到要求的。

直接使用“压缩前”和“压缩后”的数据大小进行比较，才能达到这个目的。

# `class CompressedBlockDecoder`
本类的作用是：在读取的时候，对`CFile Block`进行解压缩。

同时支持读取`kudu v1`和`kudu v2`版本的`CFile Block`。通过在构建该对象的时候，在“构造函数”中传入`cfile_version`来指定。

## 成员

注意：`codec_`/`cfile_version_`/`data_`这3个变量是`const`的，也就是必须在“构造函数”中对它们进行初始化。

注意1：从这些成员可以看出，这个类和`CompressedBlockBuilder`是不同的，在解压的过程中，不会将“解压后的数据”保存在本类中。  
**也就是说，本类是无状态的**。

>> 备注：在`commit_id: d9bd5b8752efd4ae407043e491b7fd1e0b885a92`之前，本类(`CompressedBlockDecoder`）是作为`CFileReader`的成员属性的。在每次新建`CFileReader`对象的时候，就会新建本对象。  
>> 现在已经把对本类的使用方式，修改为：仅作为“栈对象”来使用（即不再是`CFileReader`的成员）。即使用的时候，直接在“栈”上分配一个，用完就会析构掉（因为本类是“无状态的”，所以可以这样使用）。    
>> 这种“栈对象”的方式，相对于原来的方式，优点在于：  
>> 1. 减少了一次“堆分配”，并且在访问的时候，减少了“指针的间接访问”。   
>> 2. 将来可以在`CompressedBlockDecoder`中添加更多的属性，而不必担心“线程安全”的问题。

因为本类是“无状态”的，并且是“栈对象”的方式，所以本类的使用方法如下：

```
////// 参见`CFileReader::ReadBlock()`(文件`src/kudu/cfile/cfile_reader.cc`)

    //// 1. 在“栈”上创建该对象，并进行`Init()`
    CompressedBlockDecoder uncompressor(codec_, cfile_version_, block);
    Status s = uncompressor.Init();
    if (!s.ok()) {
      LOG(WARNING) << "Unable to validate compressed block " << block_id().ToString()
                   << " at " << ptr.offset() << " of size " << block.size() << ": "
                   << s.ToString();
      return s;
    }
    
    //// 2. 获取“未压缩时数据的大小”，即“数据在解压后的预期大小”
    int uncompressed_size = uncompressor.uncompressed_size();

    //// 3. 分配内存，用来存放解压后的数据
    // If we plan to put the uncompressed block in the cache, we should
    // decompress directly into the cache's memory (to avoid a memcpy for NVM).
    ScratchMemory decompressed_scratch;
    if (cache_control == CACHE_BLOCK) {
      decompressed_scratch.TryAllocateFromCache(cache, key, uncompressed_size);
    } else {
      decompressed_scratch.AllocateFromHeap(uncompressed_size);
    }
    
    //// 4. 执行“解压缩”
    s = uncompressor.UncompressIntoBuffer(decompressed_scratch.get());
    if (!s.ok()) {
      LOG(WARNING) << "Unable to uncompress block " << block_id().ToString()
                   << " at " << ptr.offset()
                   << " of size " <<  block.size() << ": " << s.ToString();
      return s;
    }
```

### “两个版本”的`header`长度 -- `static`

+ `kudu v1`的`header`长度为`8`;
+ `kudu v1`的`header`长度为`4`;

```
  static const size_t kHeaderLengthV1 = 8;
  static const size_t kHeaderLengthV2 = 4;
```

### `const codec_`
使用的“解压缩器”。

**注意：这是一个`常量的指针`。（即每次构建这个对象，相应的“压缩器”对象只是传递一个指针，不需要新建出来对象）** 

按照`kudu`现在的实现，在每个进程中，只有`3`个对象(`SnappyCodec`, `Lz4Codec`, `ZlibCodec`)来负责各个类型的“压缩和解压缩”。  
这3个“解压缩器”对象都是无状态的，可以被任意多个线程同时使用。

### `const cfile_version_`
当前数据的版本号。

### `cosnt data_`
待解压的数据。

### `uncompressed_size_`
在“解压缩”之前，数据的大小。

```
  const CompressionCodec* const codec_;
  const int cfile_version_;
  const Slice data_;

  int uncompressed_size_ = -1;
```

## 接口列表

### `private header_length()`

```
  size_t header_length() const {
    return cfile_version_ == 1 ? kHeaderLengthV1 : kHeaderLengthV2;
  }
```

>> 问题：如果将当前类改成模板类（模板参数是`kHeaderLength`，即会有两个实例类，一个是`4`，一个是`8`），那么是否可以减少一些“分支判断”？

### `Init()`

解析`CFile Block header`的信息，进行一些检查，并解析出`uncompressed_size_`成员。

注意1：这里解析出来`uncompressed_size_`非常重要，因为只有知道的“解压缩”以后的长度，才能在“真正执行解压缩”（`UncompressIntoBuffer()`）的时候，能够正确的设置“缓存池”的大小，从而安全的进行解压缩。


```
Status CompressedBlockDecoder::Init() {
  // Check that the on-disk size is at least as big as the expected header.
  if (PREDICT_FALSE(data_.size() < header_length())) {
    return Status::Corruption(
        Substitute("data size $0 is not enough to contains the header. "
                   "required $1, buffer: $2",
                   data_.size(), header_length(),
                   KUDU_REDACT(data_.ToDebugString(50))));
  }

  const uint8_t* p = data_.data();
  // Decode the header
  uint32_t compressed_size;
  if (cfile_version_ == 1) {
    // CFile v1 stores the compressed size in the compressed block header.
    // This is redundant, since we already know the block length, but it's
    // an opportunity for extra verification.
    compressed_size = DecodeFixed32(p);
    p += 4;

    // Check that the on-disk data size matches with the buffer.
    if (data_.size() != header_length() + compressed_size) {
      return Status::Corruption(
          Substitute("compressed size $0 does not match remaining length in buffer $1, buffer: $2",
                     compressed_size, data_.size() - header_length(),
                     KUDU_REDACT(data_.ToDebugString(50))));
    }
  } else {
    // CFile v2 doesn't store the compressed size. Just use the remaining length.
    compressed_size = data_.size() - header_length();
  }

  uncompressed_size_ = DecodeFixed32(p);

  // In CFile v2, we ensure that compressed_size <= uncompressed_size,
  // though, as per the file format, if compressed_size == uncompressed_size,
  // this indicates that the data was not compressed.
  if (PREDICT_FALSE(compressed_size > uncompressed_size_ &&
                    cfile_version_ > 1)) {
    return Status::Corruption(
        Substitute("compressed size $0 must be <= uncompressed size $1, buffer",
                   compressed_size, uncompressed_size_),
        KUDU_REDACT(data_.ToDebugString(50)));
  }

  // Check if uncompressed size seems to be reasonable.
  if (uncompressed_size_ > FLAGS_max_cfile_block_size) {
    return Status::Corruption(
      Substitute("uncompressed size $0 overflows the maximum length $1, buffer",
                 uncompressed_size_, FLAGS_max_cfile_block_size),
      KUDU_REDACT(data_.ToDebugString(50)));
  }

  return Status::OK();
}
```

说明1： 因为“待解压”的数据中，是包含`header`的，所以“待解压的数据”的长度，一定要大于等于`header`的长度。（如果是`kudu v1`，那么数据长度应该大于等于`8`；如果是`kudu v2`，那么数据长度应该大于等于`4`；）

说明2：在上面“压缩”的时候，已经说过，对于每个`CFile Block`，“进行压缩过的数据大小” 是一定要小于 “压缩前的数据大小”。（如果压缩以后，数据量大小没有减少，在“压缩”的时候，就直接保存的“未压缩”的数据）。 

所以，这里如果发现“压缩过的数据大小”，比“压缩前的数据”要大，说明出现了问题，直接返回失败。

说明3：对于每个`CFile Block`，它“压缩前的数据大小”，是一定要小于或等于`FLAGS_max_cfile_block_size`的(“压缩”的逻辑保证了这点)。  
如果违反，说明出错了，返回失败。

> 问题：如果是下面这种情况怎么处理？
> 比如进程写按照一个比较大的`FLAGS_max_cfile_block_size`来写入了数据。然后重启进程，并设置了一个比较小的`FLAGS_max_cfile_block_size`，这时进行读取时，就会失败？

### `UncompressIntoBuffer()`

对`data_`成员中的数据 进行“解压缩”，将结果保存在参数`dst`中。

注意1：`dst`一定要预分配好足够的空间，来容纳“解压后的数据”。

注意2：如果“待解压的数据”和“解压以后的预期大小”是一样的，那么说明这些数据是“没有结果压缩的”，所以直接取出来即可，不需要进行解压缩。

```
Status CompressedBlockDecoder::UncompressIntoBuffer(uint8_t* dst) {
  DCHECK_GE(uncompressed_size_, 0);

  Slice compressed = data_;
  compressed.remove_prefix(header_length());
  if (uncompressed_size_ == compressed.size() && cfile_version_ > 1) {
    // TODO(perf): we could potentially avoid this memcpy and instead
    // just use the data in place. However, it's a bit tricky, since the
    // block cache expects that the stored pointer for the block is at
    // the beginning of block data, not the compression header. Copying
    // is simple to implement and at least several times faster than
    // executing a codec, so this optimization is still worthwhile.
    memcpy(dst, compressed.data(), uncompressed_size_);
  } else {
    RETURN_NOT_OK(codec_->Uncompress(compressed, dst, uncompressed_size_));
  }

  return Status::OK();
}
```

说明1：在“数据是未经压缩的”情况下，理论上是可以直接复用“源数据”的内存的（即，不需要将数据从`data_`拷贝到`dst`），但是这样就会有一点诡异，因为`block cache`希望它所保存的指针，都是直接指向“解压后的数据”的，而不是指向“`CFile Block`整体”（包括`header`和“数据”）。  

而且，这里使用“拷贝”也是最容易实现的方式。（如果复用内存，那么需要保证`data_`的内存不会失效）

并且，即使是“拷贝”，也比 去执行“解压缩”的效率快很多，所以使用“拷贝”是可以接受的。

说明2：在调用`CompressionCodec::Uncompress()`的时候，是去掉`header`部分的。（对应代码中的`remove_prefix(header_length())`）.

# `GFlags`变量

## `FLAGS_max_cfile_block_size`
指定一个“未经压缩”的`CFile Block`的“最大容量”。

默认值是`16MB`。

如果一个`CFile Blokc`的大小，超过了该值，那么不会进行数据压缩，也不会写入文件。

>> 备注：这个`flags`是在`commit_id: e1acdfba66913819f9c5d604ad5449e1eca83766`中被添加进来的。添加的原因可以参加`commit msg`。

简单的说：这个`flags`变量，只是用来解决下面这种情况的“变通方法”。

在该提交之前的`kudu`实现中，在代码中“硬编码”限制了每个`CFile Block`的大小为`16MB`。但是有可能，用户在插入数据时，某列的值是超过`16MB`的，这时，在原来的实现中，会触发一个`CHECK()`，导致进程退出。  

而这时，对于旧版本的`kudu`，因为是“硬编码”，所以只能先修改代码，然后重新编译，然后再次重新运行。

所以，为了在遇到这种情况的时候，不重新编译，才将这个值设置为了`gflags`。这样，如果遇到了上述的情况，只需要修改下这个`flags`的值，然后直接重启就可以了（不需要“重新编译”代码。）

## `FLAGS_min_compression_ratio`

指定允许的“最低的压缩率”，默认值是`0.90`

如果为某个列 配置了一种“压缩算法”，但是经过压缩以后，实际的压缩率小于这个值，那么就直接写入“未压缩的数据”（而不是写入压缩以后的数据）。  

这个参数的目的，是为了在压缩效果不好的时候，减小`CPU`的压力。这时，所付出的代价是：使用一小部分空间的开销（因为这时压缩率虽然不好，但是压缩后的数据大小，可能依然是小于“未压缩的数据大小”）。

> 备注：这个优化是在`commit_id: d9bd5b8752efd4ae407043e491b7fd1e0b885a92`时引入的。

注意：
1. 在“写入数据”时，是始终会进行“压缩计算”的。（因为只有压缩之后，才会知道对于本次的数据，“压缩”是否有效果）
2. 在“读取数据”时，会进行检查，如果“待读取的数据大小”和“预期解压后的数据大小”是一样的，那么说明之前写入的数据是没有经过“压缩”的，所以就不再进行“解压缩”计算。













