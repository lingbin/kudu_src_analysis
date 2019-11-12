[TOC]

文件：`src/kudu/cfile/cfile.proto`

# `message FileMetadataPairPB` 

尽管从`CFile`的角度看，这个字段应该是属于“元信息”（即正常情况下，应该不进行`REDACT`）。  
但是这里将`value`进行`REDACT`的原因是因为，在很多场景下，这个字段，经常会保存实际的值（比如会保存`key`的`min/max`值）。

```
message FileMetadataPairPB {
  required string key = 1;

  required bytes value = 2 [ (REDACT) = true ];
}
```

>> 问题：这个结构是干什么用的？ `key`和`value`的含义是？

# `message CFileHeaderPB`

每个`CFile`，都会包含一个`CFileHeaderPB`、一个`CFileFooterPB`、若干的“数据块”。

参见`cfile-design doc`

```
message CFileHeaderPB {
  // These fields were used in Kudu <= 1.2, for the original header cfile format
  // corresponding to the 'kuducfil' magic string.
  // The new format ('kuducfl2' magic) no longer uses them, instead using the
  // "feature flags" in the CFileFooterPB.
  // required int32 major_version = 1;
  // required int32 minor_version = 2;
  repeated FileMetadataPairPB metadata = 3;
}
```

# `message BlockPointerPB`
用来表示一个`CFile Block`在`BlockManager Block`中的“偏移量”和“长度”。

```
message BlockPointerPB {
  required int64 offset = 1;
  required int32 size = 2;
}
```

# `message BTreeInfoPB`

```
message BTreeInfoPB {
  required BlockPointerPB root_block = 1;
}
```

# `message IndexBlockTrailerPB`
用来标识`CFile Block`的索引。

```
message IndexBlockTrailerPB {
  required int32 num_entries = 1;

  enum BlockType {
    UNKNOWN = 999;
    LEAF = 0;
    INTERNAL = 1;
  };
  required BlockType type = 2;
}
```

# `message CFileFooterPB`

一个`CFile`的“页脚”。

**说明：使用`uint32_t`类型的位图，来表示“各种特性”**  

正常情况下，这种常常会使用一个 `enum`的值列表，来表示“特性列表”。

这里之所以使用一个`uint32_t`，是希望节省内存。  
因为对于任何一个打开的`CFile`，它的`CFileFooterPB`对象，都是常驻在内存中的。  
而每列数据，都会对应一个`CFile`，也就是说，内存中的打开状态的`CFile`可能会很多。  
所以减少这个“结构”所占用的内存，是有收益的。

目前的特性，还是很少的，用一个`uint32_t`足够表示了。

**随着特性增多，一个`uint32_t`不够了，怎么办**  
可以用`uint32_t`的最后一位来表示，是否需要另外一个“位图”。  
比如：最后一位 值为`1`时，表示还需要额外记载另一个“位图”。  

使用这种方式，就可以进行扩展了。

**特性分为两类（分别对应`incompatible_features`和`compatible_features`）：**  
1. `INCOMPATIBLE`: 不兼容的，这种属性是必须被`Reader`支持的。否则，无法正确的读取文件。  
    如果一个`reader`，看到了一个“不认识”的标记，那么它会拒绝读取文件。  
   举例：对`block`的编码，进行了一个“不兼容”的修改。如果一个`reader`不知道这个修改（即在`reader`中还没有这个“标记”），如果还让它进行读取，就会读取到错误的数据。
2. `COMPATIBLE`: 兼容的特性。 这种属性，如果一个`reader`不认识这种类型的标记，它是可以安全忽略掉的。  
    注意：尽管这个时候，`reader`可以继续，但是它可能无法利用一些“索引上”优点。（也就是说，有时候，会增加一些优化，在`reader`开启这些标记的时候，效率会高）  
   举例：增加了一种`index`，或者“汇总信息”，`reader`可以利用这些信息。但是对于一个老版本的`reader`，因为不识别这些特性，就无法利用。

```
message CFileFooterPB {
  required kudu.DataType data_type = 1;
  required EncodingType encoding = 2;

  // Total number of values in the file.
  required int64 num_values = 3;

  optional BTreeInfoPB posidx_info = 4;
  optional BTreeInfoPB validx_info = 5;

  optional CompressionType compression = 6 [default=NO_COMPRESSION];

  repeated FileMetadataPairPB metadata = 7;

  optional bool is_type_nullable = 8 [default=false];

  // Block pointer for dictionary block if the cfile is dictionary encoded.
  // Only for dictionary encoding.
  optional BlockPointerPB dict_block_ptr = 9;

  optional uint32 incompatible_features = 10;
  optional uint32 compatible_features = 11;
}
```

>> designate: 指派；指定；  

# `message BloomBlockHeaderPB`

```
message BloomBlockHeaderPB {
  required int32 num_hash_functions = 1;
}
```