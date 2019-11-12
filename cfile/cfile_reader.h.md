[TOC]

文件：`src/kudu/cfile/cfile_reader.h`

# `class CFileReader`

本文件的职责：去读取一个`cfile`.

## `enum CFileReader::CacheControl`

```
  enum CacheControl {
    CACHE_BLOCK,
    DONT_CACHE_BLOCK
  };
```

## 成员

### 构造时传入的参数

说明：在构建本对象的时候，是已经构建好`ReadableBlock`和`file_size_`了，所以在构建该对象的时候，就会直接初始化`block_`和`file_size_`成员。

#### `block_`
当前`cfile`对象的`kudu::fs block`（因为是读取场景，所以是`kudu::fs::ReadableBlock`）。

#### `file_size_`
`cfile`的大小。

### 需要读取文件，才能获取到的成员
#### `cfile_version_`
`cfile`的版本：`"kuducfl1"`或`"kuducfl2"`

#### `header_`
`cfile`的header信息

#### `footer_`
`cfile`的footer信息

#### `codec_`
“压缩器”

#### `type_info_`
当前`cfile`对应的 列类型信息。

#### `type_encoding_info_`
“编码器”的工厂对象。（可以创建“编码器”和“解码器”对象）

### 其它成员
#### `init_once_`
`KuduOnceLambda`类型，用来保证本类的初始化函数（`InitOnce()`）只会被执行一次。

因为`InitOnce()`中会去读取文件（`cfile header`和`cfile footer`），还有一些其它的初始化工作，所以只需要执行一次即可。

#### `mem_consumption_`
封装了一个`MemTracker`对象，用来记录“当前对象”所占用的内存总大小。

```
  const std::unique_ptr<fs::ReadableBlock> block_;
  const uint64_t file_size_;

  uint8_t cfile_version_;

  gscoped_ptr<CFileHeaderPB> header_;
  gscoped_ptr<CFileFooterPB> footer_;
  const CompressionCodec* codec_;
  const TypeInfo *type_info_;
  const TypeEncodingInfo *type_encoding_info_;

  KuduOnceLambda init_once_;

  ScopedTrackedConsumption mem_consumption_;
```

## 接口方法

### 静态方法

#### `static Open()`
使用一个给定的`ReadableBlock`对象，“充分打开”一个`cfile`。


>> 名字解释：  
>> “充分打开” VS “延迟打开” 
>> 说明：在打开一个`cfile`的时候，会进行一些IO操作（至少要去读`cfile header`和`cfile footer`），并且会进行一些验证操作。
>> 1. “充分打开”： `fully open`，是指“完全打开”，即上面所有的操作（“IO读取操作”和“验证”）都会做。
>> 2. “延迟打开”： `lazily open`，即不进行上面的操作。
>> 
>> 说明：这里的“延迟”，指的就是将“IO操作”进行延迟。

对应的`ReadableBlock`对象，会赋值给传入的参数`block`中。

说明：在该函数成功返回后，当前`CFileReader`就已经是可以使用了（即可以用来读取数据了）。

```
Status CFileReader::Open(unique_ptr<ReadableBlock> block,
                         ReaderOptions options,
                         unique_ptr<CFileReader>* reader) {
  unique_ptr<CFileReader> reader_local;
  const IOContext* io_context = options.io_context;
  RETURN_NOT_OK(OpenNoInit(std::move(block),
                           std::move(options),
                           &reader_local));
  RETURN_NOT_OK(reader_local->Init(io_context));

  *reader = std::move(reader_local);
  return Status::OK();
}
```

说明：该方法的实现，其实就是调用`OpenNoInit()`，然后再进行初始化（`Init()`）。

> 备注：具体的IO操作（读取`cfile header`和`cfile footer`），是在`Init()`方法中。

#### `static OpenNoInit()`
给定一个`ReadableBlock`对象，延迟地（`lazily open`）打开一个`cfile`。

在这个方法中，既不会进行任何的`IO`操作，也不会进行任何验证。（正常情况下，在打开一个`cfile`以后，要根据该`cfile header`和`cfile footer`进行一些列的验证）。

所以，在该方法成功返回后，并不能直接使用它返回的`CFileReader`对象去读取数据。（如果要进行读取数据，使用这还需要先调用下`Init()`方法。）

不过，也是有一小部分方法可以调用的。

**问题：为什么“打开”区分为`Open()`和`OpenNoInit()`？什么场景下使用？**  

```
Status CFileReader::OpenNoInit(unique_ptr<ReadableBlock> block,
                               ReaderOptions options,
                               unique_ptr<CFileReader>* reader) {
  uint64_t block_size;
  RETURN_NOT_OK(block->Size(&block_size));
  const IOContext* io_context = options.io_context;
  unique_ptr<CFileReader> reader_local(
      new CFileReader(std::move(options), block_size, std::move(block)));
  if (!FLAGS_cfile_lazy_open) {
    RETURN_NOT_OK(reader_local->Init(io_context));
  }

  *reader = std::move(reader_local);
  return Status::OK();
}
```

### `Init()`

“充分打开”一个之前被“lazily open”的`cfile`。

实际上就是调用`InitOnce()`方法来进行初始化。

只不过这里使用了`KuduOnceLambda`来保证了，即使多次调用这个`Init()`方法，也最多只会调用`InitOnce()`函数一次。

说明1： 上面说的“直接打开”和“延迟打开”，在实现上的区别就是：是否

```
Status CFileReader::Init(const IOContext* io_context) {
  RETURN_NOT_OK_PREPEND(init_once_.Init([this, io_context] { return InitOnce(io_context); }),
                        Substitute("failed to init CFileReader for block $0",
                                   block_id().ToString()));
  return Status::OK();
}
```

### `NewIterator()`
创建一个“迭代器”。

```
Status CFileReader::NewIterator(unique_ptr<CFileIterator>* iter,
                                CacheControl cache_control,
                                const IOContext* io_context) {
  iter->reset(new CFileIterator(this, cache_control, io_context));
  return Status::OK();
}
```

### `ReadBlock()`

根据指定的`BlockPointer`，读取对应位置的`cfile block`。

有两种情况：
1. 如果在`BlockCache`中存在，那么从`BlockCache`中读取；
2. 如果在`BlockCache`中不存在，那么从“文件系统”中读取。

获取到的`cfile block`的数据，在参数`ret`中返回。

```
Status CFileReader::ReadBlock(const IOContext* io_context, const BlockPointer &ptr,
                              CacheControl cache_control, BlockHandle *ret) const {
  DCHECK(init_once_.init_succeeded());
  CHECK(ptr.offset() > 0 &&
        ptr.offset() + ptr.size() < file_size_) <<
    "bad offset " << ptr.ToString() << " in file of size "
                  << file_size_;
  BlockCacheHandle bc_handle;
  Cache::CacheBehavior cache_behavior = cache_control == CACHE_BLOCK ?
      Cache::EXPECT_IN_CACHE : Cache::NO_EXPECT_IN_CACHE;
  BlockCache* cache = BlockCache::GetSingleton();
  BlockCache::CacheKey key(block_->id(), ptr.offset());
  if (cache->Lookup(key, cache_behavior, &bc_handle)) {
    TRACE_COUNTER_INCREMENT("cfile_cache_hit", 1);
    TRACE_COUNTER_INCREMENT(CFILE_CACHE_HIT_BYTES_METRIC_NAME, ptr.size());
    *ret = BlockHandle::WithDataFromCache(&bc_handle);
    // Cache hit
    return Status::OK();
  }

  // Cache miss: need to read ourselves.
  // We issue trace events only in the cache miss case since we expect the
  // tracing overhead to be small compared to the IO (even if it's a memcpy
  // from the Linux cache).
  TRACE_EVENT1("io", "CFileReader::ReadBlock(cache miss)",
               "cfile", ToString());
  TRACE_COUNTER_INCREMENT("cfile_cache_miss", 1);
  TRACE_COUNTER_INCREMENT(CFILE_CACHE_MISS_BYTES_METRIC_NAME, ptr.size());

  uint32_t data_size = ptr.size();
  if (has_checksums()) {
    if (PREDICT_FALSE(kChecksumSize > data_size)) {
      return Status::Corruption("invalid data size for block pointer",
                                ptr.ToString());
    }
    data_size -= kChecksumSize;
  }

  ScratchMemory scratch;
  // If we are reading uncompressed data and plan to cache the result,
  // then we should allocate our scratch memory directly from the cache.
  // This avoids an extra memory copy in the case of an NVM cache.
  if (codec_ == nullptr && cache_control == CACHE_BLOCK) {
    scratch.TryAllocateFromCache(cache, key, data_size);
  } else {
    scratch.AllocateFromHeap(data_size);
  }
  uint8_t* buf = scratch.get();
  Slice block(buf, data_size);
  uint8_t checksum_scratch[kChecksumSize];
  Slice checksum(checksum_scratch, kChecksumSize);

  // Read the data and checksum if needed.
  Slice results_backing[] = { block, checksum };
  bool read_checksum = has_checksums() && FLAGS_cfile_verify_checksums;
  ArrayView<Slice> results(results_backing, read_checksum ? 2 : 1);
  RETURN_NOT_OK_PREPEND(block_->ReadV(ptr.offset(), results),
                        Substitute("failed to read CFile block $0 at $1",
                                   block_id().ToString(), ptr.ToString()));

  if (read_checksum) {
    Status s = VerifyChecksum(ArrayView<const Slice>(&block, 1), checksum);
    if (!s.ok()) {
      RETURN_NOT_OK_HANDLE_CORRUPTION(
          s.CloneAndPrepend(Substitute("checksum error on CFile block $0 at $1",
                                       block_id().ToString(), ptr.ToString())),
          HandleCorruption(io_context));
    }
  }

  // Decompress the block
  if (codec_ != nullptr) {
    // Init the decompressor and get the size required for the uncompressed buffer.
    CompressedBlockDecoder uncompressor(codec_, cfile_version_, block);
    Status s = uncompressor.Init();
    if (!s.ok()) {
      LOG(WARNING) << "Unable to validate compressed block " << block_id().ToString()
                   << " at " << ptr.offset() << " of size " << block.size() << ": "
                   << s.ToString();
      return s;
    }
    int uncompressed_size = uncompressor.uncompressed_size();

    // If we plan to put the uncompressed block in the cache, we should
    // decompress directly into the cache's memory (to avoid a memcpy for NVM).
    ScratchMemory decompressed_scratch;
    if (cache_control == CACHE_BLOCK) {
      decompressed_scratch.TryAllocateFromCache(cache, key, uncompressed_size);
    } else {
      decompressed_scratch.AllocateFromHeap(uncompressed_size);
    }
    s = uncompressor.UncompressIntoBuffer(decompressed_scratch.get());
    if (!s.ok()) {
      LOG(WARNING) << "Unable to uncompress block " << block_id().ToString()
                   << " at " << ptr.offset()
                   << " of size " <<  block.size() << ": " << s.ToString();
      return s;
    }

    // Now that we've decompressed, we don't need to keep holding onto the original
    // scratch buffer. Instead, we have to start holding onto our decompression
    // output buffer.
    scratch.Swap(&decompressed_scratch);

    // Set the result block to our decompressed data.
    block = Slice(buf, uncompressed_size);
  } else {
    // Some of the File implementations from LevelDB attempt to be tricky
    // and just return a Slice into an mmapped region (or in-memory region).
    // But, this is hard to program against in terms of cache management, etc,
    // so we memcpy into our scratch buffer if necessary.
    block.relocate(scratch.get());
  }

  // It's possible that one of the TryAllocateFromCache() calls above
  // failed, in which case we don't insert it into the cache regardless
  // of what the user requested. The scratch memory includes both the
  // generated key and the data read from disk.
  if (cache_control == CACHE_BLOCK && scratch.IsFromCache()) {
    cache->Insert(scratch.mutable_pending_entry(), &bc_handle);
    *ret = BlockHandle::WithDataFromCache(&bc_handle);
  } else {
    // We get here by either not intending to cache the block or
    // if the entry could not be allocated from the block cache.
    // Since we allocate memory to include the key for the cache entry
    // we must reset the block.
    DCHECK_EQ(block.data(), buf);
    DCHECK(!scratch.IsFromCache());
    *ret = BlockHandle::WithOwnedData(scratch.as_slice());
  }

  // The cache or the BlockHandle now has ownership over the memory, so release
  // the scoped pointer.
  ignore_result(scratch.release());

  return Status::OK();
}
```

说明1： 首先会从`BlockCache`中查找一下，如果能找到，那么直接使用`Cache`中的条目来构建对应的`BlockHandle`对象，并返回。

说明2：只会在没有命中`Cache`的时候，才会进行`TRACE_EVENT`。  
原因是：因为相比于IO操作，`trace`的开销是非常小的（即使只需要从`linux page cache`中拷贝一下）；但是相对于命中缓存时的操作，那就比较大了。

说明3：如果在读取“**未压缩**”的数据，并且打算对结果进行“缓存”，那么我们应该直接从`Cache`中申请内存。  
这样的好处是：对于`NVM`类型的`Cache`，可以避免一次拷贝。

注意：这里必须是“未压缩”的数据，才会直接从`Cache`中申请内存。  

如果是读取“经过压缩”的数据，那么是从“堆内存”中申请内存。

说明4：这里在读取的时候，在指定“数据目标的`Slice`”时，利用了一个技巧。  
1. 先构建一个含有`2`个元素的`Slice`数组；
2. 然后构建`ArrayView`的时候，根据是否需要读取`checksum`，动态的决定这个`ArrayView`的长度。

这样，最终传递个`ReadableBlock`的`ArrayView`对象：
1. 如果需要读取`checksum`，就会含有`2`个`Slice`；
2. 如果不需要读取，那么其中就只含有`1`个`Slice`；

说明5：如果当前`cfile block`是“没有进行压缩”的，那么就会构建对应的`CompressedBlockDecoder`对象，进行解压缩。

### `private`方法

#### `private InitOnce()`

```
Status CFileReader::InitOnce(const IOContext* io_context) {
  VLOG(1) << "Initializing CFile with ID " << block_->id().ToString();
  TRACE_COUNTER_INCREMENT("cfile_init", 1);

  // Parse Footer first to find unsupported features.
  RETURN_NOT_OK_HANDLE_CORRUPTION(ReadAndParseFooter(), HandleCorruption(io_context));

  RETURN_NOT_OK_HANDLE_CORRUPTION(ReadAndParseHeader(), HandleCorruption(io_context));

  if (PREDICT_FALSE(footer_->incompatible_features() & ~IncompatibleFeatures::SUPPORTED)) {
    return Status::NotSupported(Substitute(
        "cfile uses features from an incompatible bitset value $0 vs supported $1 ",
        footer_->incompatible_features(),
        IncompatibleFeatures::SUPPORTED));
  }

  type_info_ = GetTypeInfo(footer_->data_type());

  RETURN_NOT_OK(TypeEncodingInfo::Get(type_info_,
                                      footer_->encoding(),
                                      &type_encoding_info_));

  VLOG(2) << "Initialized CFile reader. "
          << "Header: " << SecureDebugString(*header_)
          << " Footer: " << SecureDebugString(*footer_)
          << " Type: " << type_info_->name();

  // The header/footer have been allocated; memory consumption has changed.
  mem_consumption_.Reset(memory_footprint());

  return Status::OK();
}
```

说明1： 在实现的时候，先解析`cfile footer`，然后解析`cfile header`。  
原因有两个：
1. 当前`cfile`的描述信息，都是在`cfile footer`中的。
2. 当前`cfile`中，是否有`checksum`，是在`CFileFooterPB`结构中的`incompatible_features`中标识的。所以必须先解析出来`footer`，才能去解析`header`(即才能判断`header`中是否有`checksum`信息)

说明2： 要检查“不兼容的特性”。 如果其中有当前运行的`kudu`版本不认识的“特性”（有可能是一个“低版本的`kudu`”在尝试读“高版本的`kudu`写下的数据”），那么就直接退出。  
  参见：`src/kudu/cfile/cfile.proto`中的`CFileFooterPB`说明。

说明3： 在函数的最后，更新下当前对象使用的“总内存”。

#### `private ReadAndParseHeader()`

```
Status CFileReader::ReadAndParseHeader() {
  TRACE_EVENT1("io", "CFileReader::ReadAndParseHeader",
               "cfile", ToString());
  DCHECK(!init_once_.init_succeeded());

  // First read and parse the "pre-header", which lets us know
  // that it is indeed a CFile and tells us the length of the
  // proper protobuf header.
  uint8_t mal_scratch[kMagicAndLengthSize];
  Slice mal(mal_scratch, kMagicAndLengthSize);
  RETURN_NOT_OK_PREPEND(block_->Read(0, mal),
                        "failed to read CFile pre-header");
  uint32_t header_size;
  RETURN_NOT_OK_PREPEND(ParseMagicAndLength(mal, &cfile_version_, &header_size),
                        "failed to parse CFile pre-header");

  // Quick check to ensure the header size is reasonable.
  if (header_size >= file_size_ - kMagicAndLengthSize) {
    return Status::Corruption("invalid CFile header size", std::to_string(header_size));
  }

  // Setup the data slices.
  uint64_t off = kMagicAndLengthSize;
  uint8_t header_scratch[header_size];
  Slice header(header_scratch, header_size);
  uint8_t checksum_scratch[kChecksumSize];
  Slice checksum(checksum_scratch, kChecksumSize);

  // Read the header and checksum if needed.
  vector<Slice> results = { header };
  if (has_checksums() && FLAGS_cfile_verify_checksums) {
    results.push_back(checksum);
  }
  RETURN_NOT_OK(block_->ReadV(off, results));

  if (has_checksums() && FLAGS_cfile_verify_checksums) {
    Slice slices[] = { mal, header };
    RETURN_NOT_OK(VerifyChecksum(slices, checksum));
  }

  // Parse the protobuf header.
  header_.reset(new CFileHeaderPB());
  if (!header_->ParseFromArray(header.data(), header.size())) {
    return Status::Corruption("invalid cfile pb header",
                              header.ToDebugString());
  }

  VLOG(2) << "Read header: " << SecureDebugString(*header_);

  return Status::OK();
}
```

说明1：在解析了`kudu cfile version`和“`CFileHeaderPB`的长度”以后，会进行一个快速验证。  
该文件中的剩余部分，是否大于“`CFileHeaderPB`的长度”。  

如果不大于，说明当前`cfile`的剩余部分，已经小于 只有一个`CFileHeaderPB`或者 不足一个`CFileHeaderPB`结构了，也就是说，该`cfile`的剩余内容，是没有任何内容的，这是不正确的，返回`Status::Corruption`错误。



#### `private ReadAndParseFooter()`

```
Status CFileReader::ReadAndParseFooter() {
  TRACE_EVENT1("io", "CFileReader::ReadAndParseFooter",
               "cfile", ToString());
  DCHECK(!init_once_.init_succeeded());
  CHECK_GT(file_size_, kMagicAndLengthSize) <<
    "file too short: " << file_size_;

  // First read and parse the "post-footer", which has magic
  // and the length of the actual protobuf footer.
  uint8_t mal_scratch[kMagicAndLengthSize];
  Slice mal(mal_scratch, kMagicAndLengthSize);
  RETURN_NOT_OK(block_->Read(file_size_ - kMagicAndLengthSize, mal));
  uint32_t footer_size;
  RETURN_NOT_OK(ParseMagicAndLength(mal, &cfile_version_, &footer_size));

  // Quick check to ensure the footer size is reasonable.
  if (footer_size >= file_size_ - kMagicAndLengthSize) {
    return Status::Corruption(Substitute(
        "invalid CFile footer size $0 in block of size $1",
        footer_size, file_size_));
  }

  uint8_t footer_scratch[footer_size];
  Slice footer(footer_scratch, footer_size);

  uint8_t checksum_scratch[kChecksumSize];
  Slice checksum(checksum_scratch, kChecksumSize);

  // Read both the header and checksum in one call.
  // We read the checksum position in case one exists.
  // This is done to avoid the need for a follow up read call.
  Slice results[2] = {checksum, footer};
  uint64_t off = file_size_ - kMagicAndLengthSize - footer_size - kChecksumSize;
  RETURN_NOT_OK(block_->ReadV(off, results));

  // Parse the protobuf footer.
  // This needs to be done before validating the checksum since the
  // incompatible_features flag tells us if a checksum exists at all.
  footer_.reset(new CFileFooterPB());
  if (!footer_->ParseFromArray(footer.data(), footer.size())) {
    return Status::Corruption("invalid cfile pb footer", footer.ToDebugString());
  }

  // Verify the footer checksum if needed.
  if (has_checksums() && FLAGS_cfile_verify_checksums) {
    // If a checksum exists it was pre-read.
    Slice slices[2] = {footer, mal};
    RETURN_NOT_OK(VerifyChecksum(slices, checksum));
  }

  // Verify if the compression codec is available.
  if (footer_->compression() != NO_COMPRESSION) {
    RETURN_NOT_OK_PREPEND(GetCompressionCodec(footer_->compression(), &codec_),
                          "failed to load CFile compression codec");
  }

  VLOG(2) << "Read footer: " << SecureDebugString(*footer_);

  return Status::OK();
}
```
说明1：在现在的实现中，即使当前的`cfile`是在没有`checksum`的情况下写入的，那么在读取的时候也是会按照有`checksum`的情况进行读取（对应上面读取的`ReadV()`时，会传入`checksum_slice`）。

#### `private VerifyChecksum()`

```
Status CFileReader::VerifyChecksum(ArrayView<const Slice> data, const Slice& checksum) const {
  uint32_t expected_checksum = DecodeFixed32(checksum.data());
  uint32_t checksum_value = 0;
  for (auto& d : data) {
    checksum_value = crc::Crc32c(d.data(), d.size(), checksum_value);
  }
  if (PREDICT_FALSE(checksum_value != expected_checksum ||
                    MaybeTrue(FLAGS_cfile_inject_corruption))) {
    return Status::Corruption(
        Substitute("Checksum does not match: $0 vs expected $1",
                   checksum_value, expected_checksum));
  }
  return Status::OK();
}
```

问题：对于一段数据，为什么一起计算`crc`，和分成几段然后依次计算`crc`，是一样的？ 这里的`crc`是逐个字节计算的吗？

#### `private memory_footprint()`

```
size_t CFileReader::memory_footprint() const {
  size_t size = kudu_malloc_usable_size(this);
  size += block_->memory_footprint();
  size += init_once_.memory_footprint_excluding_this();

  // SpaceUsed() uses sizeof() instead of malloc_usable_size() to account for
  // the size of base objects (recursively too), thus not accounting for
  // malloc "slop".
  if (header_) {
    size += header_->SpaceUsed();
  }
  if (footer_) {
    size += footer_->SpaceUsed();
  }
  return size;
}
```

# `class ScratchMemory`

在`.cpp`中定义和实现，所以只能本文件使用。

**本类的作用：在读取`cfile block`的过程中，用该类来封装“从文件中读取到的数据”。**  

在这个类中，会记录一个`cfile block`的内存、大小、以及它的来源（是“堆内存”、还是“来自`Cache`”）。  

注意1：在本类中所维护的数据，都是在读取场景的。 但是，对于`Cache`来说，却可能是写入操作（因为是将读取结果，保存在`Cache`中，即写入到`Cache`中）

注意2：是否写入`Cache`，用户是否指定的“是否进行缓存”的配置：
即用户会指定“是否需要对`cfile block`进行缓存”：如果值为`CACHE_BLOCK`，那么就会写入`Cache`（参见`enum CacheControl`）。

注意3：即使用户指定的是`CACHE_BLOCK`，但是在运行过程中，即使用户指定了需要对`cfile block`进行缓存，但是如果在向`Cache`申请内存的时候，失败了。    
   这时，不会直接返回失败，而是尝试直接从“堆内存”中申请。  
    注意：这个时候，虽然用户指定了`CACHE_BLOCK`，但是运行过程相当于`CACHE_NO_BLOCK`，即“不进行缓存”。

注意4：在写入`Cache`时，“其中的数据”会封装成一个`BlockCache::PendingEntry`。  如果不写入`Cache`，那么数据就是“裸”的`uint8_t* prt_`和`int size_`。

也就是说，在该对象所封装的数据，有两种可能：
1. 可能来自“堆内存”；
2. 可能来自于`pending block cache`。

> 扩展：因为这里的数据，是从文件读取的数据，所以一定不会从`Cache`读取的数据，所以一定不是`BlockCacheHandle`。

注意5：
1. 如果`cache`是基于`DRAM`来实现（因为），那么在这个`ScratchMemory`中的内容，就是在“堆内存”的内容。  
但是，这里仍然将“cache”和“”
2. 如果`Cache`是基于`NVM`来实现的，那么这里就有很大的区别了。这里会将“`cfile block`的数据”直接放入到`NMV`中。

注意6：释放内存的方式：  
在析构函数中，会释放相关的内存。有两种方式：
1. 如果是“堆内存”，那么使用`delete[]`;
2. 如果来自`Cache`，那么使用`Cache::Free()`;  

说明7：正常情况下，数据都是通过`release()`来“转移数据的拥有权”。

无论数据在“堆内存”，还是在“pending block cache”，都可以使用该方法 转移“数据的拥有权”。

## 成员

### `from_cache_`
数据是否在`pending block cache`中。

注意：类型是`BlockCache::PendingEntry`对象，而不是指针。 

**在析构一个`ScratchMemory`对象的时候，如果它的数据没有被`release()`，那么它所引用的内存，会被释放。**

### `ptr_`
指向具体的内存。

1. 如果数据在“堆内存”中，那么`ptr_`指向具体的数据地址；
2. 如果数据在`pending block cache`中，那么`ptr_`指向具体的`cache`中的数据内容；

也就是说，无论任何时候，`ptr_`指向的都是“数据的实际地址”。（也正是这个原因，仅仅凭借`ptr_`是无法判断当前指向的数据，是来自于“堆内存”、还是来自于`pending block cache`，参见`IsFromCache()`）

另外，如果`ptr_ == nullptr`，那么就可以说明当前`ScratchMemory`对象，尚未被赋予一段有效的数据。  
参见`析构函数`，如果`ptr_ == nullptr`，那么就可以直接返回了（因为当前对象没有维护任何数据）

### `size_`

```
  BlockCache::PendingEntry from_cache_;
  uint8_t* ptr_;
  int size_;
```

**说明1：根据上面的成员，如何去判断当前对象中的数据来源？**  
根据 `from_cache_`中是否为`valid`的。参见`IsFromCache()`方法。

实际上`BlockCache::PendingEntry::valid()`中，就是判断`BlockCache::PendingEntry`中的`std::unique_ptr<>`成员是否为`nullptr`。

## 接口列表

### 构造函数

```
  ScratchMemory() : ptr_(nullptr), size_(-1) {}
  ~ScratchMemory() {
    if (!ptr_) return;
    if (!from_cache_.valid()) {
      delete[] ptr_;
    }
  }
```

说明：如果`ptr_ == null`，那么说明当前`ScratchMemory`对象，没有维护任何数据，所以就可以直接返回；

### 获取内存
#### `TryAllocateFromCache()`

```
  // Try to allocate 'size' bytes from the cache. If the cache has
  // no capacity and cannot evict to make room, this will fall back
  // to allocating from the heap. In that case, IsFromCache() will
  // return false.
  void TryAllocateFromCache(BlockCache* cache, const BlockCache::CacheKey& key, int size) {
    DCHECK(!ptr_);
    from_cache_ = cache->Allocate(key, size);
    if (!from_cache_.valid()) {
      AllocateFromHeap(size);
      return;
    } else {
      ptr_ = from_cache_.val_ptr();
    }
    size_ = size;
  }
```

**强调：如果从`Cache`中申请内存失败，那么会转化为“从堆内存进行申请”**  
这时，执行以后，当前`cfile block`的数据，不会被添加到`Cache`中。

注意：`ptr_`并不都是直接从“堆”上申请的内存。 见`AllocateFromHeap()`。  
如果是从`Cache`中申请的内存，那么`ptr_`就会指向`Cache`中的内存。

#### `AllocateFromHeap()`
```
  void AllocateFromHeap(int size) {
    DCHECK(!ptr_);
    from_cache_.reset();
    ptr_ = new uint8_t[size];
    size_ = size;
  }
```

### 信息获取方法

#### `IsFromCache()`
```
  // Return true if the current scratch memory was allocated from the cache.
  bool IsFromCache() const {
    return from_cache_.valid();
  }
```

#### `mutable_pending_entry()`

获取在`pending cache`中的“内存”。
```
  BlockCache::PendingEntry* mutable_pending_entry() {
    return &from_cache_;
  }
```
说明：使用者在调用这个方法之前，应该先确保，数据确实是在`pending block cache`中的。

#### `get()`

获取在“堆内存”中的数据。
```
  uint8_t* get() {
    return DCHECK_NOTNULL(ptr_);
  }
```
说明：见上面当前类的概述，无论数据是在“堆内存”中，还是在`pending block cache`中，都是可以通过调用`get()`获取到“相应数据的地址”。

### 其它操作
#### `as_slice()`
```
  Slice as_slice() {
    return Slice(ptr_, size_);
  }
```

这个方法的使用场景：  
当数据是来自于“堆内存”时，使用这个方法构造一个`Slice`对象，这样使用该`Slice`对象的结构，就开始负责要对其中的内存进行释放（实际上是一个`BlockHandle`对象），然后就可以通过`ScratchMemory::release()`放弃“数据拥有权”。

#### `release()`

强调：当前对象中的数据，一定是来自“堆内存”。

**当前函数的作用：就是将当前对象中的内存的“维护权”，移交出去。**   

在该函数返回后，本对象不再引用“堆内存”。 **“调用者”应该负责析构掉 该函数返回的内存。**  

```
  uint8_t* release() {
    uint8_t* ret = ptr_;
    ptr_ = nullptr;
    size_ = -1;
    return ret;
  }
```

> 扩展：这种情况，一般情况下，调用者都是将被函数返回的`uint8_t*`封装在一个`std::unique_ptr<uint8_t[]>`中。

在现在kudu的实现中，参见上面的`as_slice()`，
1. 是先通过转化为一个`Slice`对象，上层通过`BlockHandle`对象来管理这个`Slice`指向的内存； 
2. 然后，调用`release()`，从而让当前`ScratchMemory`对象放弃对“这段内存”的“拥有权”；

#### `Swap()`
```
  void Swap(ScratchMemory* other) {
    std::swap(from_cache_, other->from_cache_);
    std::swap(ptr_, other->ptr_);
    std::swap(size_, other->size_);
  }
```

# `GFLAGS`变量列表

## `FLAGS_cfile_lazy_open`

表示在打开一个`cfile`时，是否允许“延迟打开”。

默认值为`true`.

注意：如果该变量的值为`false`，即不允许进行“延迟打开”，也就是，所有的打开操作，都会变成“完全打开”。 那么即使调用的是`OpenNoInit()`方法，也会进行“完全打开”操作。

# `.cpp`文件中的常量

## `kMagicAndLengthSize`
在每个`cfile header`和`cfile footer`中，都会包含如下两个属性：
1. “`magic`信息”  --- 长度为`8`
2. “相应`protobuf`结构的长度”  --- 长度为`4`

## `kMaxHeaderFooterPBSize`
表示`CFileHeaderPB`和`CFileFooterPB`结构的最大长度。

当前是“硬编码”的值为`64KB`.

```
// Magic+Length: 8-byte magic, followed by 4-byte header size
static const size_t kMagicAndLengthSize = 12;
static const size_t kMaxHeaderFooterPBSize = 64*1024;
```

# 工具函数

在`.cpp`文件中定义，所以只能在本文件中使用

## `ParseMagicAndLength()`

将“`kudu cfile version` + `protobuf len`”解析出来。

参见`cfile design doc`。在`cfile header`和`cfile footer`的“开始部分”，都是这样在一个组合。

总长度为`12`个字节：其中 `kudu cfile version`占用`8`个字节； `protobuf len`占`4`个字节。

```
static Status ParseMagicAndLength(const Slice &data,
                                  uint8_t* cfile_version,
                                  uint32_t *parsed_len) {
  if (data.size() != kMagicAndLengthSize) {
    return Status::Corruption("Bad size data");
  }

  uint8_t version;
  if (memcmp(kMagicStringV1, data.data(), kMagicLength) == 0) {
    version = 1;
  } else if (memcmp(kMagicStringV2, data.data(), kMagicLength) == 0) {
    version = 2;
  } else {
    return Status::Corruption("bad CFile header magic", data.ToDebugString());
  }

  uint32_t len = DecodeFixed32(data.data() + kMagicLength);
  if (len > kMaxHeaderFooterPBSize) {
    return Status::Corruption("invalid data size for header",
                              std::to_string(len));
  }

  *cfile_version = version;
  *parsed_len = len;

  return Status::OK();
}
```

说明：这里对于`CFileHeaderPB`和`CFileFooter`的长度，限制了“最大长度”。  
不能超过`kMaxHeaderFooterPBSize`，它的值是“硬编码”的`64KB`。

