[TOC]

文件：`src/kudu/cfile/cfile_util.h`

本文件定义了一些“工具类”和“类型别名”。

# `enum ImcompatibleFeatures`

参见`message CFileFooterPB`(文件：`src/kudu/cfile/cfile.proto`)

虽然在`protobuf`结构(`CFileFooterPB`)中（每个打开的`CFile`，这个结构都会保存在内存中），使用一个`uint32_t`类型来描述这个“位图”，但是在代码中，使用了一个枚举类型`enum`来表示它。

```
// Used to set the CFileFooterPB bitset tracking incompatible features
enum IncompatibleFeatures {
  NONE = 0,

  // Write a crc32 checksum at the end of each cfile block
  CHECKSUM = 1 << 0,

  SUPPORTED = NONE | CHECKSUM
};
```

由此可以看出，目前“不兼容的特性”，只有一个：就是文件中是否包含`checksum`.

# `ValidxKeyEncoder`  -- 类型别名，是一个函数
```
typedef std::function<void(const void*, faststring*)> ValidxKeyEncoder;
```

在`WriteOptions`中，有一个该类型的成员(`validx_key_encoder`)。

在代码中，该成员都是被赋值为一个`lamda表达式`。

# `struct WriterOptions`

`CFileWriter`中使用这个结构来写数据。

## `index_block_size`
`index block`的大小。 默认值是`32KB`。

## `block_restart_interval`
向`RocksDB`那样的“前缀编码”(`Prefix Encoding`)中，需要进行“重启”的区间。

即，每隔这么一段长度，就需要“重启”编码。

默认值是`16`。

注意1：这个值是可以随时进行“动态调整”的。

注意2：大部分客户端，都不需要关系这个配置项。

## `write_posidx`
当前`CFile`是否需要一个“位置索引”。默认值为`false`;

## `write_validx`
当前`CFile`是否需要一个“值索引”。默认值为`false`;

## `optimize_index_keys`
是否进行一个优化：存储最短的前缀，而不是完整的`key`。默认值为`true`。

>> 问题：有什么用？

## `storage_attributes`
列存引擎的属性。

默认值：在`schema.h`中的所有默认值。该类的定义参见`src/kudu/common/schema.h`

## `validx_key_encoder`

>> 问题：干嘛用的？

```
struct WriterOptions {
  size_t index_block_size;

  int block_restart_interval;

  // Whether the file needs a positional index.
  bool write_posidx;

  // Whether the file needs a value index
  bool write_validx;

  // Whether to optimize index keys by storing shortest separating prefixes
  // instead of entire keys.
  bool optimize_index_keys;

  ColumnStorageAttributes storage_attributes;

  // An optional value index key encoder. If not set, the default encoder
  // encodes the entire value.
  boost::optional<ValidxKeyEncoder> validx_key_encoder;

  WriterOptions();
};
```

# `struct ReaderOptions`

读选项，`CFileReader`会使用。

```
struct ReaderOptions {
  ReaderOptions();

  // The IO context of this reader.
  //
  // Default: nullptr
  const fs::IOContext* io_context = nullptr;

  // The MemTracker that should account for this reader's memory consumption.
  //
  // Default: the root tracker.
  std::shared_ptr<MemTracker> parent_mem_tracker;
};

```

