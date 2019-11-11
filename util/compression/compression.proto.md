[TOC]

文件：`src/kudu/util/compression/compression.proto`

# `enum CompressionType`

压缩类型。

```
enum CompressionType {
  UNKNOWN_COMPRESSION = 999;
  DEFAULT_COMPRESSION = 0;
  NO_COMPRESSION = 1;
  SNAPPY = 2;
  LZ4 = 3;
  ZLIB = 4;
}
```