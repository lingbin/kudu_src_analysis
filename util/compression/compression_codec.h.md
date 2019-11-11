[TOC]

文件: `src/kudu/util/compression/compression_codec.h`

# `class CompressionCodec`

该类是 “压缩器”，用来对“指定的数据”进行“压缩”和“解压缩”。

该类是一个“抽象类”，提供了“压缩”和“解压缩”的接口，不同的“压缩类型”作为它的子类，分别提供自己的实现。  
该类的子类，都在是`.cpp`文件中定义和实现。

目前支持的压缩类型，参加`enum CompressionType`(文件：`src/kudu/util/compression/compression.proto`)

其中只有`3`个有效的压缩类型：
1. `SNAPPY`:  -- `SnappyCodec`类
2. `LZ4`:     -- `Lz4Codec`类
3. `ZLIB`:    -- `ZlibCodec`类

**注意：所有的子类都是“无状态”的，所有进程中每个类型的“压缩器”都各有一个即可。**  
所以，在每个子类中，都会提供一个`GetSingleton()`函数，用来返回一个“单例对象”。  
并且会封装一个全局函数(`GetCompressionCodec`)，根据传入的类型，获取对应类型的“压缩器”。

## 成员
该类没有成员。

## 接口列表

### `Compress()`

有两个重载函数。

两者的区别类似于`Read()`和`ReadV()`的区别，即一个是“一次压缩一个`Slice`对象”，另一个是“一次压缩多个`Slice`对象”。

说明1：参数中的`uint8_t* compressed`指针，“压缩的结果”就保存在它“所指向的地址”中。

说明2: 该函数的作用，就是对`input`中的数据进行压缩，将结果保存在`compressed`中。

注意：`compressed`所指向的内存，长度至少要为`MaxCompressedLength(input_length)`。（这样才能保证，有足够的空间来容纳“压缩的结果”）。

通过参数`compressed_length`返回在‘压缩后’数据的长度。

### `Uncompress()`
对“压缩过的数据（`Slice compressed`）”进行解压缩，解压的结果放在`uncompressed`中。

注意：和“压缩”不同，在解压的时候，并不能区分有多少个“数据段”（即在压缩的时候，是否被一次性压缩的多个`Slice`对象），所以这里和“压缩”时不同，只有一个函数（没有重载函数，即没有`UncompressToManySlices()`函数）.

传递进来的参数`UNcompressed_length`，表示‘解压缩后’数据的预期长度。  
也就是说，如果“解压后”得到的数据，其长度不等于这个值，那么就是发生了错误。

### `MaxCompressedLength()`
给定一个长度（需要进行压缩的数据的长度），返回经过压缩后，最大的数据长度。（各个子类，会根据自己的压缩方式，计算出一个值）。

### `type()`
每个子类，返回各自的“压缩类型”。

```
  virtual Status Compress(const Slice& input,
                          uint8_t *compressed, size_t *compressed_length) const = 0;

  virtual Status Compress(const std::vector<Slice>& input_slices,
                          uint8_t *compressed, size_t *compressed_length) const = 0;

  virtual Status Uncompress(const Slice& compressed,
                            uint8_t *uncompressed, size_t uncompressed_length) const = 0;

  virtual size_t MaxCompressedLength(size_t source_bytes) const = 0;

  virtual CompressionType type() const = 0;
```

# `SlicesSource`

工具类，用来辅助进行`Snappy`压缩。

在`snappy`库中，提供了一个对一组对象，同时进行压缩的方法。但是需要传入的对象是一个`snappy::Source`类型。  

所以，在这里，使用`SliceSource`类来封装一个`std::vector<Slice>`，并让它继承自`snappy::Source`类。 这样，就可以使用`snappy`库的接口，对齐进行压缩（一次性压缩一组`Slice`对象）。

因为继承`snappy::Source`类，所以也需要提供它所需要的接口。

```
class SlicesSource : public snappy::Source {
    ...
}
```

## 成员
### `available_`
需要进行压缩的“源数据”的总长度。

### `slice_index_`和`slice_offset_`

因为本类是封装的一组`Slice`列表，如果把一个`Slice`对象看成一个“一维数组”，那么所有`Slice`对象加在一起，就组成了一个“二维数组”。

使用`slice_index_`和`slice_offset_`一起，来标识已经压缩过的数据的“位置”。即在此位置之前的，都是已经压缩过的。后续的压缩，从它们当前指定的“位置”开始继续压缩。

```
图中的`M`表示当前的“位置”。

### 初始状态
    slice_index_ = 0；
    slice_offset_ = 0;
                      0 1 2 3 4 5 6 7 8 9 
        slice[0]:    |M x x x x x x x |
        slice[1]:    |x x x x |
        slice[2]:    |x x x x x x |
        slice[3]:    |x x x x x x x x x |
        
### 某个中间状态
    slice_index_ = 2；
    slice_offset_ = 3;
                      0 1 2 3 4 5 6 7 8 9 
        slice[0]:    |x x x x x x x x |
        slice[1]:    |x x x x |
        slice[2]:    |x x x M x x |
        slice[3]:    |x x x x x x x x x |

### 压缩结束时
    slice_index_ = 3；
    slice_offset_ = 8;
                      0 1 2 3 4 5 6 7 8 9 
        slice[0]:    |x x x x x x x x |
        slice[1]:    |x x x x |
        slice[2]:    |x x x M x x |
        slice[3]:    |x x x x x x x x x |
```

### `const vector<Slice>& slices_`

类型为`const vector<Slice>& `。

注意1：这是一个引用，所以是构建时，不会进行“内存拷贝”。

注意2：这是一个`const`的引用，所以它的值不会被修改。

```
  size_t available_;
  size_t slice_index_;
  size_t slice_offset_;
  const vector<Slice>& slices_;
```

## 接口列表

### `Available()`
剩余的、还没有被压缩的数据大小。

在一开始的时候，该值是所有需要被压缩的数据大小总和（即所封装的所有`Slice`的大小之和）。

随着压缩的进行，该值主键减小。在压缩完成后，该值变为0；

### `Peek()`
从“源数据”中取出一段数据。

去除从“当前位置”到“当前`Slice`对象末尾”的这一段数据。

返回的结果的指针，自己作为返回值。 这段数据的长度，通过传入参数`len`来返回。

注意：这个函数，并没有修改当前`SliceSource`对象的任何成员（具体的修改，是在`Skip()`方法中修改的。）。

### `Skip()`
调过长度为`n`的一段数据。

注意：可能会跨过多个`Slice`对象。

说明：在上层使用时，首先调用`Peek()`来获取需要压缩的“数据指针”和“长度”，在进行压缩以后，调用`Skip()`方法来跳过这段数据。

```
使用方法：(伪代码)
    
    size_t len = 0;
    // 1. 获取 带压缩 的数据
    char* data_to_compress = SliceSource::Peek(&len);
    // 2. 执行压缩
    Compress(data_to_compress, len);
    // 3. 在源数据中跳过这段“已压缩的”数据
    SliceSource::Skip(len);
```

### `Dump()`
将封装的`Slice`数组的内容导出到一个`faststring`对象中。

```
class SlicesSource : public snappy::Source {
 public:
  explicit SlicesSource(const std::vector<Slice>& slices)
    : slice_index_(0),
      slice_offset_(0),
      slices_(slices) {
    available_ = TotalSize();
  }

  size_t Available() const OVERRIDE {
    return available_;
  }

  const char* Peek(size_t* len) OVERRIDE {
    if (available_ == 0) {
      *len = 0;
      return nullptr;
    }

    const Slice& data = slices_[slice_index_];
    *len = data.size() - slice_offset_;
    return reinterpret_cast<const char *>(data.data()) + slice_offset_;
  }

  void Skip(size_t n) OVERRIDE {
    DCHECK_LE(n, Available());
    if (n == 0) return;

    available_ -= n;
    if ((n + slice_offset_) < slices_[slice_index_].size()) {
      slice_offset_ += n;
    } else {
      n -= slices_[slice_index_].size() - slice_offset_;
      slice_index_++;
      while (n > 0 && n >= slices_[slice_index_].size()) {
        n -= slices_[slice_index_].size();
        slice_index_++;
      }
      slice_offset_ = n;
    }
  }

  void Dump(faststring *buffer) {
    buffer->reserve(buffer->size() + TotalSize());
    for (const Slice& block : slices_) {
      buffer->append(block.data(), block.size());
    }
  }

 private:
  size_t TotalSize(void) const {
    size_t size = 0;
    for (const Slice& data : slices_) {
      size += data.size();
    }
    return size;
  }
};
```

# `class SnappyCodec`

`Snappy`格式的“压缩器”，继承自`CompressionCodec`。

该类内部的接口函数，都是调用相应的`snappy`库进行实现的。

## 成员
该类是“无状态”的，所以该类不需要任何数据成员。

## 接口列表

### `static GetSingleton()`
在整个进程中，每种类型的“压缩器”都只有一个对象。 使用该方法获取该对象的“单例”。

### 继承的接口

接口和父类相同，本类的实现是通过调用`snappy`类库的接口来实现“压缩”和“解压缩”功能。

```
class SnappyCodec : public CompressionCodec {
 public:
  static SnappyCodec *GetSingleton() {
    return Singleton<SnappyCodec>::get();
  }

  Status Compress(const Slice& input,
                  uint8_t *compressed, size_t *compressed_length) const OVERRIDE {
    snappy::RawCompress(reinterpret_cast<const char *>(input.data()), input.size(),
                        reinterpret_cast<char *>(compressed), compressed_length);
    return Status::OK();
  }

  Status Compress(const vector<Slice>& input_slices,
                  uint8_t *compressed, size_t *compressed_length) const OVERRIDE {
    SlicesSource source(input_slices);
    snappy::UncheckedByteArraySink sink(reinterpret_cast<char *>(compressed));
    if ((*compressed_length = snappy::Compress(&source, &sink)) <= 0) {
      return Status::Corruption("unable to compress the buffer");
    }
    return Status::OK();
  }

  Status Uncompress(const Slice& compressed,
                    uint8_t *uncompressed,
                    size_t uncompressed_length) const OVERRIDE {
    bool success = snappy::RawUncompress(reinterpret_cast<const char *>(compressed.data()),
                                         compressed.size(), reinterpret_cast<char *>(uncompressed));
    return success ? Status::OK() : Status::Corruption("unable to uncompress the buffer");
  }

  size_t MaxCompressedLength(size_t source_bytes) const OVERRIDE {
    return snappy::MaxCompressedLength(source_bytes);
  }

  CompressionType type() const override {
    return SNAPPY;
  }
};

```

# `class Lz4Codec`

`Lz4`格式的“压缩器”，继承自`CompressionCodec`。

该类内部的接口函数，都是调用相应的`lz4`库进行实现的。

## 成员
该类是“无状态”的，所以该类不需要任何数据成员。

## 接口列表

### `static GetSingleton()`
在整个进程中，每种类型的“压缩器”都只有一个对象。 使用该方法获取该对象的“单例”。

### 继承的接口
接口和父类相同，本类的实现是通过调用`lz4`类库的接口来实现“压缩”和“解压缩”功能。

说明：在对一组`Slice`对象进行编码时（参见`Compress()`），会先将所有的`Slice`对象拷贝到一个`faststring`对象中，然后再进行压缩。   
原因是：`Lz4`库，并没有提供一次性压缩多个`Slice`对象的方法。

```
class Lz4Codec : public CompressionCodec {
 public:
  static Lz4Codec *GetSingleton() {
    return Singleton<Lz4Codec>::get();
  }

  Status Compress(const Slice& input,
                  uint8_t *compressed, size_t *compressed_length) const OVERRIDE {
    int n = LZ4_compress(reinterpret_cast<const char *>(input.data()),
                         reinterpret_cast<char *>(compressed), input.size());
    *compressed_length = n;
    return Status::OK();
  }

  Status Compress(const vector<Slice>& input_slices,
                  uint8_t *compressed, size_t *compressed_length) const OVERRIDE {
    if (input_slices.size() == 1) {
      return Compress(input_slices[0], compressed, compressed_length);
    }

    SlicesSource source(input_slices);
    faststring buffer;
    source.Dump(&buffer);
    return Compress(Slice(buffer.data(), buffer.size()), compressed, compressed_length);
  }

  Status Uncompress(const Slice& compressed,
                    uint8_t *uncompressed,
                    size_t uncompressed_length) const OVERRIDE {
    int n = LZ4_decompress_fast(reinterpret_cast<const char *>(compressed.data()),
                                reinterpret_cast<char *>(uncompressed), uncompressed_length);
    if (n != compressed.size()) {
      return Status::Corruption(
        StringPrintf("unable to uncompress the buffer. error near %d, buffer", -n),
                     KUDU_REDACT(compressed.ToDebugString(100)));
    }
    return Status::OK();
  }

  size_t MaxCompressedLength(size_t source_bytes) const OVERRIDE {
    return LZ4_compressBound(source_bytes);
  }

  CompressionType type() const override {
    return LZ4;
  }
};
```

# `class ZlibCodec`

`Zlib`格式的“压缩器”，继承自`CompressionCodec`。

该类内部的接口函数，都是调用相应的`Zlib`库进行实现的。

## 成员
该类是“无状态”的，所以该类不需要任何数据成员。

## 接口列表

### `static GetSingleton()`
在整个进程中，每种类型的“压缩器”都只有一个对象。 使用该方法获取该对象的“单例”。

### 继承的接口
接口和父类相同，本类的实现是通过调用`Zlib`类库的接口来实现“压缩”和“解压缩”功能。

说明：和`Lz4Codec`一样，在对一组`Slice`对象进行编码时（参见`Compress()`），会先将所有的`Slice`对象拷贝到一个`faststring`对象中，然后再进行压缩。   
原因也是一样的：`Zlib`库，并没有提供一次性压缩多个`Slice`对象的方法。

```
class ZlibCodec : public CompressionCodec {
 public:
  static ZlibCodec *GetSingleton() {
    return Singleton<ZlibCodec>::get();
  }

  Status Compress(const Slice& input,
                  uint8_t *compressed, size_t *compressed_length) const OVERRIDE {
    *compressed_length = MaxCompressedLength(input.size());
    int err = ::compress(compressed, compressed_length, input.data(), input.size());
    return err == Z_OK ? Status::OK() : Status::IOError("unable to compress the buffer");
  }

  Status Compress(const vector<Slice>& input_slices,
                  uint8_t *compressed, size_t *compressed_length) const OVERRIDE {
    if (input_slices.size() == 1) {
      return Compress(input_slices[0], compressed, compressed_length);
    }

    // TODO: use z_stream
    SlicesSource source(input_slices);
    faststring buffer;
    source.Dump(&buffer);
    return Compress(Slice(buffer.data(), buffer.size()), compressed, compressed_length);
  }

  Status Uncompress(const Slice& compressed,
                    uint8_t *uncompressed, size_t uncompressed_length) const OVERRIDE {
    int err = ::uncompress(uncompressed, &uncompressed_length,
                           compressed.data(), compressed.size());
    return err == Z_OK ? Status::OK() : Status::Corruption("unable to uncompress the buffer");
  }

  size_t MaxCompressedLength(size_t source_bytes) const OVERRIDE {
    // one-time overhead of six bytes for the entire stream plus five bytes per 16 KB block
    return source_bytes + (6 + (5 * ((source_bytes + 16383) >> 14)));
  }

  CompressionType type() const override {
    return ZLIB;
  }
};
```

# 几种类库的比较

“压缩后的长度”

类型     | 功能   |     函数                |   获取方式             | 
---------|--------| ----------------------- | ---------------------- |
`snappy` | 压缩1  | `snappy::RawCompress()` | 直接通过“传入参数”返回 |
`snappy` | 压缩2  | `snappy::Compress()`    | 函数的“返回值”         |
`Lz4`    | 压缩   | `LZ4_compress()`        | 函数的“返回值”         |
`Zlib`   | 压缩   | `::compress()`          | 函数的“返回值”         |

“解压缩后的长度”


类型     | 功能   |     函数                    |   获取方式             | 
---------|--------| --------------------------- | ---------------------- |
`snappy` | 解压缩1  | `snappy::RawUncompress()` | 直接通过“传入参数”返回 |
`Lz4`    | 解压缩   | `LZ4_compress()`          |        |
`Zlib`   | 解压缩   | `::compress()`            | 无法获取         |

# 全局函数

## `GetCompressionCodec()`
给定一个`CompressType`，返回对应的“压缩器”。

说明：因为每种类型的实现类中，都提供了`GetSingleton()`方法，所以只要分别调用对应的方法即可。

```
Status GetCompressionCodec(CompressionType compression,
                           const CompressionCodec** codec) {
  switch (compression) {
    case NO_COMPRESSION:
      *codec = nullptr;
      break;
    case SNAPPY:
      *codec = SnappyCodec::GetSingleton();
      break;
    case LZ4:
      *codec = Lz4Codec::GetSingleton();
      break;
    case ZLIB:
      *codec = ZlibCodec::GetSingleton();
      break;
    default:
      return Status::NotFound("bad compression type");
  }
  return Status::OK();
}
```

## `GetCompressionCodecType()`
给定一个“字符串”，返回它所对应的`CompressType`。

注意：这个字符串，可能是“小写形式”，所以需要在该方法中，需要先转化为“大写形式”（`ToUpperCase()`）。

```
CompressionType GetCompressionCodecType(const std::string& name) {
  std::string uname;
  ToUpperCase(name, &uname);

  if (uname == "SNAPPY")
    return SNAPPY;
  if (uname == "LZ4")
    return LZ4;
  if (uname == "ZLIB")
    return ZLIB;
  if (uname == "NONE")
    return NO_COMPRESSION;

  LOG(WARNING) << "Unable to recognize the compression codec '" << name
               << "' using no compression as default.";
  return NO_COMPRESSION;
}
```




