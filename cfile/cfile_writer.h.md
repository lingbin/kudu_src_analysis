[TOC]

文件：`src/kudu/cfile/cfile_writer.h`

该文件中的类，表示要去写一个`cfile`到文件中的过程。

# `class NullBitmapBuilder`

创建一个“位图”，是否是否为`null`。

## 成员
### `nitems_`
该位图中已经添加元素的个数。

### `bitmap_`
用来保存“位图”的数据内容。

会用该结构去构建`RleEncoder`对象，会将编码的结果保存在其中。

## 使用方法：

```
NullBitmapBuilder builder(xxx);
builder.AddRun(xxx, xxx);
builder.AddRun(yyy, yyy);
builder.AddRun(zzz, zzz);
Slice result = builder.Finish();
```
## 接口列表

### `AddRun()`
增加一段数据。这段数据的值都是一样的，长度为`run_length`。

说明：这个方法的“名字来源”：因为对于当前“null位图”，采用的编码方式是`run-length encoding`。即对于一段连续的“数”，就会认为是一个`run`。

这个方法的作用，就是添加“一段连续的数”，就是添加一个`run`，所以方法的名字是`AddRun()`。

### `Finish()`
结束编码。

说明：返回的`Slice`对象，就是编码的结果。

注意：因为返回的是`Slice`对象，它所引用的数据，是在`NullBitmapBuilder`对象中的成员（`faststring bitmap_`）。  
所以在该`Slice`对象的有效期内，`NullBitmapBuilder`对象必须是有效的（这样才能保证`Slice`中间接引用的内存地址是有效的）。

```
class NullBitmapBuilder {
 public:
  explicit NullBitmapBuilder(size_t initial_row_capacity)
    : nitems_(0),
      bitmap_(BitmapSize(initial_row_capacity)),
      rle_encoder_(&bitmap_, 1) {
  }

  size_t nitems() const {
    return nitems_;
  }

  void AddRun(bool value, size_t run_length = 1) {
    nitems_ += run_length;
    rle_encoder_.Put(value, run_length);
  }

  // the returned Slice is only valid until this Builder is destroyed or Reset
  Slice Finish() {
    int len = rle_encoder_.Flush();
    return Slice(bitmap_.data(), len);
  }

  void Reset() {
    nitems_ = 0;
    rle_encoder_.Clear();
  }

 private:
  size_t nitems_;
  faststring bitmap_;
  RleEncoder<bool> rle_encoder_;
};
```
# 一些常量

## 公开的
```
const char kMagicStringV1[] = "kuducfil";
const char kMagicStringV2[] = "kuducfl2";

const int kMagicLength = 8;
const size_t kChecksumSize = sizeof(uint32_t);
```

## 私有的
在`.cpp`文件中定义的

```
static const size_t kMinBlockSize = 512;
```

# `class CFileWriter`

本类的作用就是：要磁盘上写入一个`CFile`。

## `private enum CFileWriter::State`

当前`CFileWriter`的状态：1) 刚初始化； 2) 正在写； 3) 已经写完；

```
  enum State {
    kWriterInitialized,
    kWriterWriting,
    kWriterFinished
  };
```

一个`CFileWriter`对象，都要经过这些状态的流转。  
即 `kWriterInitialized   -->  kWriterWriting  --> kWriterFinished`

## 成员

### `block_`
当前`CFile`对应的`WritableBlock`。当前`CFile`的数据，就是要写到这个`WritableBlock`中。

注意区分：`block_`和`data_block_`成员
1. `block_`：对应的底层文件，即`WritableBlock`对象。
2. `data_block_`: 是对应的`BlockBuilder`，即对应的`CFileBlock`的“构建器”。

### `off_`
当前已经写入的数据，在`WritableBlock`中的偏移量；也是写到`WritableBlock`中的数据总量。

初始值为`0`。

### `value_count_`
已经被写的“值”的总数量。 

注意：这个值，是没有经过“去重”的(不是`select distinct count(*)`)。也就是说，比如连续写了2个`aa`，那么这个值为`2`。

### `option_`
当前`CFileWriter`对象对应的“写选项”；

类型为`WriterOptions`，参见：`src/kudu/cfile/cfile_util.h`.

### `is_nullable_`
当前列是否“可以为空”。来自于`Schema`。

### `compression_`
压缩方式。

### `typeinfo_`
当前列的类型信息。

### `type_encoding_info_`
编码方式。

### `last_key_`
最后一个编码的`值`;

>> 问题：什么时间使用？怎么用？

### `tmp_buf_`
一个临时的缓存，在“编码”时使用。

### `unflushed_metadata_`
已经被添加到`CFileWriter`中、但是尚未写到磁盘上的“元数据”。

> 备注：这里的“flush”，只是表述“是否已经写到磁盘”，并不包含`fsync`的含义。

说明1: 这里“元数据”的含义：  
上层用户会通过`AddMetadataPair()`方法，向本对象添加一些“元信息”（本质上，就是一些“键值对”）。

说明2：在`CFileHeaderPB`和`CFileFooterPB`中，都有一个`repeated FileMetadataPairPB metadata`的成员。

被添加到`unflushed_metadata_`中的“键值对”，最终都会分别添加到`CFileHeaderPB`或`CFileFooterPB`中的`metadata`成员中。

说明3：通过在代码中执行`grep AddMetadataPair`，可以看到，在目前的实现中，被添加的“键值对”有`3`种：
1. `DiskRowSet::kMinKeyMetaEntryName`：
2. `DiskRowSet::kMaxKeyMetaEntryName`：
3. `DeltaFileReader::kDeltaStatsEntryName`：

说明4：一个“键值对”，要么被加入到`CFileHeaderPB::metadata`中，要么被加入到`CFileFooterPB::metadata`中，不会两个都加入。

1. 如果在调用`Start()`方法**之前**，调用`AddMetadataPari()`添加的“键值对”，会被写入到`CFileHeaderPB`中(在`Start()`中写入)。
2. 如果在调用`Start()`方法**之后**，调用`AddMetadataPari()`添加的“键值对”，会被写入到`CFileFooterPB`中（在`Finish()`中写入）。

### `data_block_`
是对应的`BlockBuilder`对象，即对应的`CFileBlock`的“构建器”。

### `posidx_builder`
“位置索引”的构建器.（`pos - idx - bulder`）

### `validx_builder_`
“值索引”的构建器。(`val - idx - builder`)

### `null_bitmap_builder_`
表示“空值”的位图构建器。

### `block_compressor_`
用来压缩一个`CFile Block`。

### `state_`
当前`CFileWriter`所处的状态：1) 刚初始化； 2) 正在写； 3) 已经写完；

```
  std::unique_ptr<fs::WritableBlock> block_;

  uint64_t off_;

  rowid_t value_count_;

  WriterOptions options_;

  bool is_nullable_;
  CompressionType compression_;
  const TypeInfo* typeinfo_;
  const TypeEncodingInfo* type_encoding_info_;

  // The last key written to the block.
  // Only set if the writer is writing an embedded value index.
  faststring last_key_;

  faststring tmp_buf_;

  std::vector<std::pair<std::string, std::string> > unflushed_metadata_;

  gscoped_ptr<BlockBuilder> data_block_;
  gscoped_ptr<IndexTreeBuilder> posidx_builder_;
  gscoped_ptr<IndexTreeBuilder> validx_builder_;
  gscoped_ptr<NullBitmapBuilder> null_bitmap_builder_;
  gscoped_ptr<CompressedBlockBuilder> block_compressor_;

  State state_;
```

## 写一个`cfile`的执行流程

### 如果是一个`Rowset`中的“数据块”对应的`cfile block`

```
//// 1. 构建对象
CFileWriter cfw;

//// 2. （可选）添加一些“键值对”
cfw.AddMetadataPair();

//// 3. 开始进行写入 （会先把`CFileHeaderPB`写入到文件中）
cfs.Start();

//// 4. 添加数据
if (nullable) {
    cfs.AppendNullableEntries();
} else {
    cfs.AppendEntries();
}

//// 5. （可选）添加一些“键值对”
cfw.AddMetadataPair();

//// 6. 结束 
cfs.Finish();
```

### 其它特殊类型的`cfile block`

比如：`BloomFileWriter`(参见`src/kudu/cfile/bloomfile.h`).  

在这个类中，使用一个`CFileWriter`对象作为成员，用来写一个`bloomfilter`对应的`cfile`。

它会直接调用，在`CFileWriter`中的一些其它的公开接口。  

直接调用一些详细接口的方式，好处是可以提供更精确的控制。  
比如上层用户在写多个文件(`fs::WritableBlock`)的时候(可能会对应多个文件)，可以自己将所有的`WritableBlock`封装在一个`BlockCreateTransaction`中，然后一次性提交，从而可以减少`fsync`的开销。

## 接口列表
### 构造函数

说明1：因为在一个`DiskRowSet`中，“每个列的数据”和“DeltaFile”的数据，都会对应一个`CFile`。所以，在构造函数中，要传入一个“列的schema”信息：包括 1) `列类型`; 2) 是否`nullable`;

说明2：因为要将数据写入到磁盘，所以，也会传入一个“`WritableFile`”作为参数，数据会写入到这个文件中。

说明3：本类有很多的成员属性，在“构造函数”中，主要就是初始化这些成员属性。

```
CFileWriter::CFileWriter(WriterOptions options,
                         const TypeInfo* typeinfo,
                         bool is_nullable,
                         unique_ptr<WritableBlock> block)
  : block_(std::move(block)),
    off_(0),
    value_count_(0),
    options_(std::move(options)),
    is_nullable_(is_nullable),
    typeinfo_(typeinfo),
    state_(kWriterInitialized) {
  EncodingType encoding = options_.storage_attributes.encoding;
  Status s = TypeEncodingInfo::Get(typeinfo_, encoding, &type_encoding_info_);
  if (!s.ok()) {
    // TODO: we should somehow pass some contextual info about the
    // tablet here.
    WARN_NOT_OK(s, "Falling back to default encoding");
    s = TypeEncodingInfo::Get(typeinfo,
                              TypeEncodingInfo::GetDefaultEncoding(typeinfo_),
                              &type_encoding_info_);
    CHECK_OK(s);
  }

  compression_ = options_.storage_attributes.compression;
  if (compression_ == DEFAULT_COMPRESSION) {
    compression_ = GetDefaultCompressionCodec();
  }

  if (options_.storage_attributes.cfile_block_size <= 0) {
    options_.storage_attributes.cfile_block_size = FLAGS_cfile_default_block_size;
  }
  if (options_.storage_attributes.cfile_block_size < kMinBlockSize) {
    LOG(WARNING) << "Configured block size " << options_.storage_attributes.cfile_block_size
                 << " smaller than minimum allowed value " << kMinBlockSize
                 << ": using minimum.";
    options_.storage_attributes.cfile_block_size = kMinBlockSize;
  }

  if (options_.write_posidx) {
    posidx_builder_.reset(new IndexTreeBuilder(&options_, this));
  }

  if (options_.write_validx) {
    if (!options_.validx_key_encoder) {
      auto key_encoder = &GetKeyEncoder<faststring>(typeinfo_);
      options_.validx_key_encoder = [key_encoder] (const void* value, faststring* buffer) {
        key_encoder->ResetAndEncode(value, buffer);
      };
    }

    validx_builder_.reset(new IndexTreeBuilder(&options_, this));
  }
}
```

说明1：在获取针对“当前列”的“编码方法工厂类”时（即`TypeEncodingInfo::Get()`），先按照当前列的`DataType`和`EncodingType`来进行获取，如果获取不到，就使用该类类型(`DataType`)所对应的“默认编码方式”，重新进行获取。  

注意：经过上面的两轮的获取，要求一定能够获取到 一个“编解码工厂对象”。  
否则，说明对于给定的`DataType`，并没有设置“默认的编码方式”，这是不允许的，会退出进程。

说明2：当前`CFileWriter`类，并没有把`EncodingType`作为自己的成员属性。  
但是`TypeEncodingInfo`类，提供了`encoding_type()`方法，来获取相应的`EncodingType`。  
如果本类中，如果需要获取对应的`EncodingType`时，应该使用`type_encoding_info_->encoding_type()`去获取。

在本类中，在写`CFileFooterPB`的时候，会使用到这个。 因为在`CFileFooterPB`中，需要包含当前`cfile block`的编码方法（从而，在读取的时候，才能知道如何进行解码）。

说明3：如果`options_.storage_attributes.cfile_block_size <= 0`，说明对于当前`CFile Block`没有设置`cfile_block_size`属性。

>> 问题：这种情况的发生场景是？ 兼容旧版本？


### `private`方法

#### `AddBlock()`

向“文件”中写入一个`CFile Block`。

注意：这里添加的是`CFile Block`，不是`FileWritableBlock`。

说明1：使用`std::vector<Slice>`来表示`CFile Block`的内容。这个`CFile Block`的内容，是在调用这个方法时，已经组织好了的。

说明2：该方法的作用：将`Block`的数据写入到文件中，这些数据在文件中的位置信息，通过传入参数`block_ptr`返回。

说明3：参数`name_for_log`，用来在打印日志的时候，打印在日志中。

注意：这个是最终向一个文件中，添加`cfile block`数据的地方（即直接向文件中写入数据）。  
所以有两件事都是在这里做的：
1. 进行“数据压缩”；
2. 计算`checksum`;

```
Status CFileWriter::AddBlock(const vector<Slice> &data_slices,
                             BlockPointer *block_ptr,
                             const char *name_for_log) {
  uint64_t start_offset = off_;
  vector<Slice> out_slices;

  if (block_compressor_ != nullptr) {
    // Write compressed block
    Status s = block_compressor_->Compress(data_slices, &out_slices);
    if (!s.ok()) {
      LOG(WARNING) << "Unable to compress block at offset " << off_
                   << ": " << s.ToString();
      return s;
    }
  } else {
    out_slices = data_slices;
  }

  // Calculate and append a data checksum.
  uint8_t checksum_buf[kChecksumSize];
  if (FLAGS_cfile_write_checksums) {
    uint32_t checksum = 0;
    for (const Slice &data : out_slices) {
      checksum = crc::Crc32c(data.data(), data.size(), checksum);
    }
    InlineEncodeFixed32(checksum_buf, checksum);
    out_slices.emplace_back(checksum_buf, kChecksumSize);
  }

  RETURN_NOT_OK(WriteRawData(out_slices));

  uint64_t total_size = off_ - start_offset;

  *block_ptr = BlockPointer(start_offset, total_size);
  VLOG(1) << "Appended " << name_for_log
          << " with " << total_size << " bytes at " << start_offset;
  return Status::OK();
}
```

步骤1：如果配置了“压缩方法”，先将`CFile Block`进行压缩（`CompressedBlockBuilder::Compress()`）

> 注意：压缩结果，在`CompressedBlockBuilder`对象中。

步骤2：如果配置了要“计算`checksum`”，那么就计算`checksum`。

> 注意：这里是对“压缩后的数据”，计算`checksum`。

> 说明：`checksum`是一个`uint32_t`类型的整型。直接将“内存形式”写入到文件。

步骤3：将“所有内容”（包括“压缩后的数据”、“`checksum`值”）都写入到“文件”中。

步骤4：构建`BlockPointer`对象。    
之所以要返回这个对象，是因为上层，如果需要构建“位置索引”的话，需要这个信息。

#### `FlushMetadataToPB()`

将`unflushed_metadata_`中的键值对，填充到`CFileHeaderPB`或`CFileFooterPB`的`metadata`成员中。

也就是说，这个函数只是在修改“内存结构”，没有任何磁盘操作。

```
void CFileWriter::FlushMetadataToPB(RepeatedPtrField<FileMetadataPairPB> *field) {
  typedef pair<string, string> ss_pair;
  for (const ss_pair &entry : unflushed_metadata_) {
    FileMetadataPairPB *pb = field->Add();
    pb->set_key(entry.first);
    pb->set_value(entry.second);
  }
  unflushed_metadata_.clear();
}
```

#### `WriteRawData()`
将“数据”写入到文件中。 数据来自参数中的“`Slice`数组”。

```
Status CFileWriter::WriteRawData(const vector<Slice>& data) {
  size_t data_size = accumulate(data.begin(), data.end(), static_cast<size_t>(0),
                                [&](int sum, const Slice& curr) {
                                  return sum + curr.size();
                                });
  Status s = block_->AppendV(data);
  if (!s.ok()) {
    LOG(WARNING) << "Unable to append data of size "
                 << data_size << " at offset " << off_
                 << ": " << s.ToString();
  }
  off_ += data_size;
  return s;
}
```

#### `FinishCurDataBlock()`
结束当前`CFile Block`，并将它的写入到“文件”中。

```
Status CFileWriter::FinishCurDataBlock() {
  uint32_t num_elems_in_block = data_block_->Count();
  if (is_nullable_) {
    num_elems_in_block = null_bitmap_builder_->nitems();
  }

  if (PREDICT_FALSE(num_elems_in_block == 0)) {
    return Status::OK();
  }

  rowid_t first_elem_ord = value_count_ - num_elems_in_block;
  VLOG(1) << "Appending data block for values " <<
    first_elem_ord << "-" << (first_elem_ord + num_elems_in_block);

  // The current data block is full, need to push it
  // into the file, and add to index
  Slice data = data_block_->Finish(first_elem_ord);
  VLOG(2) << " actual size=" << data.size();

  uint8_t key_tmp_space[typeinfo_->size()];
  if (validx_builder_ != nullptr) {
    // If we're building an index, we need to copy the first
    // key from the block locally, so we can write it into that index.
    RETURN_NOT_OK(data_block_->GetFirstKey(key_tmp_space));
    VLOG(1) << "Appending validx entry\n" <<
      kudu::HexDump(Slice(key_tmp_space, typeinfo_->size()));
  }

  vector<Slice> v;
  faststring null_headers;
  if (is_nullable_) {
    Slice null_bitmap = null_bitmap_builder_->Finish();
    PutVarint32(&null_headers, num_elems_in_block);
    PutVarint32(&null_headers, null_bitmap.size());
    v.emplace_back(null_headers.data(), null_headers.size());
    v.push_back(null_bitmap);
  }
  v.push_back(data);
  Status s = AppendRawBlock(v, first_elem_ord,
                            reinterpret_cast<const void *>(key_tmp_space),
                            Slice(last_key_),
                            "data block");

  if (is_nullable_) {
    null_bitmap_builder_->Reset();
  }

  if (validx_builder_ != nullptr) {
    RETURN_NOT_OK(data_block_->GetLastKey(key_tmp_space));
    (*options_.validx_key_encoder)(key_tmp_space, &last_key_);
  }
  data_block_->Reset();

  return s;
}
```

说明1：如果当前列是`nullable`的，那么获取当前已经添加的元素时，必须从`null_bitmap_builder_`中获取（是不可以从`data_block_`中进行获取的）。

只有当“当前列”不是`nullable`的时候，才可以从`data_block_`中进行获取。

原因是：对于一个`nullable == true`的列，如果一个‘值’为`null`，那么是不会向`data_block_`中添加数据的。 也就是说，如果一个列是`nullable`的，那么它的`data_block_`中的数据行数，并不表示当前已经添加的“数据行数”（因为不包括值为`null`的数据行）。

而如果不是`nullable`的，那么所有的数据都会被添加到`data_block_`中，所以`data_block_`中的数据行数，就是“已经添加的数据行数”。

说明2：`first_elem_ord` 是 `first element ordinary`的缩写。即当前`CFile Block`中的第一行数据，在整个`RowSet`中的“行号”。

说明3：因为调用当前方法，就说明当前`CFile Block`是已经“满”了，所以要将当前`CFile Block`的数据写入到磁盘上，并且在“索引”中添加相应的记录。

说明4：如果需要有“值索引”，那么在从当前`CFile Block`的“第`1`个”数据，进行编码，添加到“值索引”中。

说明5：参见`cfile的design-doc`，“不可空的”和“不可空”的`CFile Block`，它们的“内容”略有不同。

说明6：调用`AppendRawBlock()`来将数据写入到磁盘，并且在对应的索引(“值索引”和“位置索引”)中添加相应的条目。

说明7：如果需要“值索引”，那么记录下当前`data_block_`中的“最后一个‘值’”。 用来构建“值索引”。


### `public`方法

#### `Start()`

打开一个`CFileWriter`对象，并将`CFile Header`写入到文件中。

```
Status CFileWriter::Start() {
  TRACE_EVENT0("cfile", "CFileWriter::Start");
  CHECK(state_ == kWriterInitialized) <<
    "bad state for Start(): " << state_;

  if (compression_ != NO_COMPRESSION) {
    const CompressionCodec* codec;
    RETURN_NOT_OK(GetCompressionCodec(compression_, &codec));
    block_compressor_.reset(new CompressedBlockBuilder(codec));
  }

  CFileHeaderPB header;
  FlushMetadataToPB(header.mutable_metadata());

  uint32_t pb_size = header.ByteSize();

  faststring header_str;
  // First the magic.
  header_str.append(kMagicStringV2);
  // Then Length-prefixed header.
  PutFixed32(&header_str, pb_size);
  pb_util::AppendToString(header, &header_str);

  vector<Slice> header_slices;
  header_slices.emplace_back(header_str);

  // Append header checksum.
  uint8_t checksum_buf[kChecksumSize];
  if (FLAGS_cfile_write_checksums) {
    uint32_t header_checksum = crc::Crc32c(header_str.data(), header_str.size());
    InlineEncodeFixed32(checksum_buf, header_checksum);
    header_slices.emplace_back(checksum_buf, kChecksumSize);
  }

  RETURN_NOT_OK_PREPEND(WriteRawData(header_slices), "Couldn't write header");

  BlockBuilder *bb;
  RETURN_NOT_OK(type_encoding_info_->CreateBlockBuilder(&bb, &options_));
  data_block_.reset(bb);

  if (is_nullable_) {
    size_t nrows = ((options_.storage_attributes.cfile_block_size + typeinfo_->size() - 1) /
                    typeinfo_->size());
    null_bitmap_builder_.reset(new NullBitmapBuilder(nrows * 8));
  }

  state_ = kWriterWriting;

  return Status::OK();
}
```

步骤1：首先构建出“压缩器”对象。

步骤2：构建`CFileHeaderPB`对象，并将它写入到“文件”。

其中，对于1个`protobuf`对象的大小，使用的是`ProtoObject.ByteSize()`方法，去获取它的大小。

步骤3：创建`BlockBuilder`对象，后续任何添加的数据，都会用该对象进行编码。

步骤4：如果当前列是“可空的”，那么会初始化`null_bitmap_builder_`成员。

说明：构建的`NullBitmapBuilder`对象时，它的长度，是取决于当前`cfile_block_size`的大小。

首先，因为对于任何一个列，表示它的一个“值”的大小是确定的。比如：一个`INT32`类型的“值”占用`4`个字节；一个`String`类型的“值”，占用`sizeof(Slice)`个字节。

所以，根据`cfile_block_size`的大小，就可以计算出，在一个`CFile Block`中，可以容纳的“值”的“数量上限”。即在一个`CFile Block`中，可以表示的“行”的“数量上限”。

> 说明：对于“可以为空”(`nullable == true`)的列，因为“`null`位图”也需要占用一部分空间，但是这里并没有考虑，所以这里计算出来的只是一个“上限”。  
> 在真正进行存储的时候，真正存储的“值”的数量，是小于这里计算出来的`nrows`的。

>> 问题：这里在构建`NullBitmapBuilder`对象的时候，传入的大小为什么要乘以`8`，即（`nrows * 8`）?

步骤5：这个方法的最后，会将当前`CFileWriter`对象的状态，修改为`kWriterWriting`。

#### `Finish()`
关闭当前`CFile`,并且关闭底层的`WritableBlock`。

#### `FinishAndReleaseBlock()`

步骤1：先结束当前`CFile Block`(这是最后一个`CFile Block`，它的大小可能还没有“满”)；

这个`CFile Block`相关的索引，也会被添加。

步骤2：向文件中添加`CFileFooterPB`。（包括组织其中的各个成员，以及按需计算`checksum`等）

步骤3：调用`WriableBlock::Finalize()`，通知底层文件系统，已经不会再写入数据了。

步骤4：向将当前`WritableBlock`添加到`BlockCreationTransaction`中。（后续在该“事务”进行提交时，会修改相应的内存结构。）

注意：从这个流程可以看出来，对于一个`RowSet`的所有`WritableBlock`，**是“先写好文件，然后再修改元数据”的**。

#### `AddMetadataPair()`
添加一个“键值对”作为“元信息”。

参见`unflushed_metadata_`的说明信息。

#### `GetMetaValueOrDie()`
从`unflushed_metadata_`中查找指定的`key`。

注意：如果找不到，这里会触发`FATAL`错误。  
之所以这么严格，是因为本身需要添加“元信息”的场景并不多。如果没有找到，那么一定是出现了某种未知的错误。

#### `Appendentries()`
向`BlockBuilder`中添加一组“值”。

注意1：当前列，是“不可为空”的。 即`nullable == false`.

注意2：因为是添加一组值，但是当前`CFile Block`的剩余空间，可能不够容纳。这个时候，就需要结束当前`CFile Block`，并新建一个新的`CFile Block`。

```
Status CFileWriter::AppendEntries(const void *entries, size_t count) {
  DCHECK(!is_nullable_);

  int rem = count;
  const uint8_t *ptr = reinterpret_cast<const uint8_t *>(entries);
  while (rem > 0) {
    int n = data_block_->Add(ptr, rem);
    DCHECK_GE(n, 0);

    ptr += typeinfo_->size() * n;
    rem -= n;
    value_count_ += n;

    if (data_block_->IsBlockFull()) {
      RETURN_NOT_OK(FinishCurDataBlock());
    }
  }

  DCHECK_EQ(rem, 0);
  return Status::OK();
}
```

#### `AppendNullableEntries()`

和`AppendEntries()`类似，只不过当前`CFile Block`所对应的列，是“可以为空”的。

注意1：因为是添加一组值，但是当前`CFile Block`的剩余空间，可能不够容纳。这个时候，就需要结束当前`CFile Block`，并新建一个新的`CFile Block`。

注意2：因为“当前列”可以为空，所以还需要对“`null`位图”进行修改，从而表示某一个“值”是否为空。

注意3：参数`entries`是一个数据，其中的元素并不是“紧凑的”。  
也就是说，如果添加了`10`行数据，但是其中`9`行都是`null`，在`entries`数组中仍然是有`10`个元素。

```
Status CFileWriter::AppendNullableEntries(const uint8_t *bitmap,
                                          const void *entries,
                                          size_t count) {
  DCHECK(is_nullable_ && bitmap != nullptr);

  const uint8_t *ptr = reinterpret_cast<const uint8_t *>(entries);

  size_t nitems;
  bool is_null = false;
  BitmapIterator bmap_iter(bitmap, count);
  while ((nitems = bmap_iter.Next(&is_null)) > 0) {
    if (is_null) {
      size_t rem = nitems;
      do {
        int n = data_block_->Add(ptr, rem);
        DCHECK_GE(n, 0);

        null_bitmap_builder_->AddRun(true, n);
        ptr += n * typeinfo_->size();
        value_count_ += n;
        rem -= n;

        if (data_block_->IsBlockFull()) {
          RETURN_NOT_OK(FinishCurDataBlock());
        }

      } while (rem > 0);
    } else {
      null_bitmap_builder_->AddRun(false, nitems);
      ptr += nitems * typeinfo_->size();
      value_count_ += nitems;
    }
  }

  return Status::OK();
}
```

说明1：这里根据`not_null`的进行分支判断，是因为在`not_null == true`的时候，是需要向`data_block_`中添加数据，这就有可能遇到当前`CFile Block`满了的情况，这时就需要进行切换为新的`CFile Block`。

而`not_null == false`的时候，只是设置下“位图”，是不会向`data_block_`中添加数据的，所以也就不需要切换`CFile Block`。

说明2: 虽然当前列的值是可能为空的，但是传入的参数`bitmap`和`entries`仍然是“一一对应”的。  
如果某一行的值为`null`，那么它在`bitmap`中对应的bit位是`1`，在`entries`也是有值的，只不过我们不去考虑这段值的内容。  

也就是说，只要处理了一行数据，无论是否为`null`，那么都需要同时递增 `bitmap`和`entries`的“序号”。

#### `AppendRawBlock()`

将一个`CFile Block`添加到“文件”中，并且添加相应的“索引”。

说明：需要写入文件的内容，在参数`data_slices`中。

说明：如果当前`CFile Block`需要写“值索引”，那么`valid_prev`参数的值，是上一个`Block`的“最后一个值”。

使用这个值，可以用来优化“值索引”中的条目：减少其中的条目的长度。

```
如果： valid_prev == abce"
       valid_curr == "abcedfghijklmnopqrst"
      
      那么，在向“值索引”中添加条目时，只需要将当前`block`的开始边界设置为 "abcde"即可。
      
      参见`GetSeparatingKey()`的实现。（文件：src/kudu/cfile/cfile_util.h）
      
      它会通过对`valid_curr`进行truncate，找到一个 既大于等于`valid_prev`，又小于等于 valid_curr的 最小的值。
```

```
Status CFileWriter::AppendRawBlock(const vector<Slice>& data_slices,
                                   size_t ordinal_pos,
                                   const void *validx_curr,
                                   const Slice& validx_prev,
                                   const char *name_for_log) {
  CHECK_EQ(state_, kWriterWriting);

  BlockPointer ptr;
  Status s = AddBlock(data_slices, &ptr, name_for_log);
  if (!s.ok()) {
    LOG(WARNING) << "Unable to append block to file: " << s.ToString();
    return s;
  }

  // Now add to the index blocks
  if (posidx_builder_ != nullptr) {
    tmp_buf_.clear();
    KeyEncoderTraits<UINT32, faststring>::Encode(ordinal_pos, &tmp_buf_);
    RETURN_NOT_OK(posidx_builder_->Append(Slice(tmp_buf_), ptr));
  }

  if (validx_builder_ != nullptr) {
    CHECK(validx_curr != nullptr) <<
      "must pass a key for raw block if validx is configured";

    (*options_.validx_key_encoder)(validx_curr, &tmp_buf_);
    Slice idx_key = Slice(tmp_buf_);
    if (options_.optimize_index_keys) {
      GetSeparatingKey(validx_prev, &idx_key);
    }
    VLOG(1) << "Appending validx entry\n" <<
            kudu::HexDump(idx_key);
    s = validx_builder_->Append(idx_key, ptr);
    if (!s.ok()) {
      LOG(WARNING) << "Unable to append to value index: " << s.ToString();
      return s;
    }
  }

  return s;
}
```

# `GFLAGS`列表

## `FLAGS_cfile_default_compression_codec`

默认的“压缩方法”。

默认值为`none`，即不进行压缩。

可以取的“值域范围”是（不区分大小写）：`"SNAPPY"`/`"LZ4"`/`"ZLIB"`/`"NONE"`。

如果取值不是这4个值，相当于它的值为`none`，这时会打印一条`WARNING`日志（提示设置了一个无效的“压缩方法”）。

## `FLAGS_cfile_default_block_size`

默认的`CFile Blokc`的大小。

默认值是`256KB`。

注意：一个`CFile Block`，必须大于`512`字节。（参见：`kMinBlockSize`）。

如果设置到值小于`512`，那么就会将`CFile Block`的大小设置为`512`。


## `FLAGS_cfile_write_checksums`

标识在写`CFile Block`的时候，是否要计算并写入`checksum`。

默认值为`true`，即要进行`checksum`。


