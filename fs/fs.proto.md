[TOC]

文件：`src/kudu/fs/fs.proto`

当前文件描述的，是和`local file` 相关的元数据。

>> 名字解释：  
>> `filesystem`: 用户指定的“数据根目录”集合；  
>> `path instance`: 用户会配置多个“数据跟目录”，每个数据根目录，对应一个`PathInstanceMetadataPB`对象，就是一个`path instance`.  

## `message InstanceMetadataPB`

当一个`server`初始化了一个新的文件系统（比如创建了一个新的节点），它都会创建一个`InstanceMetadataPB`结构，并且把它 “持久化”在本地。

说明：在kudu中配置的每个“数据根目录”，对应这里的一个`InstanceMetadataPB`结构。即，在每个“数据根目录”中，都会保存一个内容是该结构的文件。

### `uuid`
在当前机器第一次被使用的时候被赋值，表示一个在当前集群中的“唯一的`UUID`”。

>> 问题：这个`uuid`的生成规则是？

### `format_stamp`
“用户可读的”（human-readable）字符串，表示 当前机器第一次被初始化的“地点”和“时间”。

```
message InstanceMetadataPB {
  required bytes uuid = 1;
  required string format_stamp = 2;

  // TODO: add a "node type" (TS/Master?)
}
```

## `message PathSetPB`

描述一个“文件系统路径”的集合，以及

**注意：为什么使用`uuid idx`来查找对应的`path instance`(实际上，是找`PathInstanceMetadataFile`对象)**  
只所以使用`uuid idx`（整型），而不是使用`uuid`（字符串），是因为`uuid idx`比较短，而`uuid`是一个字符串，相对比较长。  
同时因为需要通过`uuid`到`path instance`的场景非常多，所以这里优化为使用`uuid idx`来找`path instance`，能够提高效率。

>> 备注：代码注释中说的`UUID's position`，就是指的`uuid`在`all_uuids`中的“序号”，即`uuid idx`。

### `uuid`
当前`path instance`的`uuid`，即当前“数据根目录”的`uuid`。

### `all_uuids`
当前`path instance`下的所有`UUID`的集合。

在一个健康的“文件系统”(数据目录)：
1. 对于其中的每个`uuid`，都对应1个`PathInstanceMetadataPB`对象；
2. 所有`PathSetPB`对象（是`PathInstanceMetadataPB`的成员属性，因为每个“数据根目录”会对应一个`PathInstanceMetadataPB`对象，所以也就有一个`PathSetPB`对象）中的`all_uuids`成员是相同的。  
    也就是说，不论有多少个“数据根目录”，他们内部的`PathSetPB`对象中的`all_uuids`是相同的，记录了所有的`数据根目录`列表。

```
// In a healthy filesystem (see below), a path instance can be referred to via
// its UUID's position in all_uuids instead of via the UUID itself. This is
// useful when there are many such references, as the position is much shorter
// than the UUID.
message PathSetPB {
  // Globally unique identifier for this path instance.
  required bytes uuid = 1;

  // All UUIDs in this path instance set. In a healthy filesystem:
  // 1. There exists an on-disk PathInstanceMetadataPB for each listed UUID, and
  // 2. Every PathSetPB contains an identical copy of all_uuids.
  repeated bytes all_uuids = 2;
}
```

## `message PathInstanceMetadataPB`

>> 说明："filesystem instance"： 就是一个“数据根目录”。  

用户可以指定多个“数据根目录”，在每个“数据根目录”中，都会存放一个“元数据文件”（内容就是经过序列化的该结构）。

在创建一个“数据根目录”实例的时候，会创建一个该对象，并且持久化到对应的目录。

```
// A filesystem instance can contain multiple paths. One of these structures
// is persisted in each path when the filesystem instance is created.
message PathInstanceMetadataPB {
  // Describes this path instance as well as all of the other path instances
  // that, taken together, describe a complete set.
  required PathSetPB path_set = 1;

  // Textual representation of the block manager that formatted this path.
  required string block_manager_type = 2;

  // Block size on the filesystem where this instance was created. If the
  // instance (and its data) are ever copied to another location, the block
  // size in that location must be the same.
  required uint64 filesystem_block_size_bytes = 3;
}
```

**注意：如果对“数据文件”进行了拷贝，要求在“目标地址”处的配置的“文件块大小”要和“源地址”是一样的。**  
`filesystem_block_size_bytes`属性，是记录的在 文件系统 上的“块大小”。   因为在组织文件内容的时候，需要用到这个值。  
如果拷贝的“源地址”和“目标地址”中，对应的“文件块大小”是不同的，那么会造成解析出错。

说明：这个`filesystem_block_size_bytes`属性带来的限制，只针对类型为`"log"`的`BlockManager`。对于类型为`file`的`BlockManager`没有这个限制。

## `message BlockIdPB`

标识一个`block`的`id`。

```
message BlockIdPB {
  required fixed64 id = 1;
}
```

注意：这里使用的是`protobuf`的类型是`fixed64`，而不是`uint64`。

参见：https://developers.google.com/protocol-buffers/docs/proto3#scalar  

`protobuf`的文档中说：`fixed64`类型，虽然本质上也是一个`uint64`类型，但是它在编码时永远是`8`个字节。在它的值经常大于`2^56`的时候，性能比普通的`uint64`高（普通的`uint64`采用变长编码）。

## `message BlockRecordType`

```
enum BlockRecordType {
  UNKNOWN = 0;
  CREATE = 1;
  DELETE = 2;
}
```

## `message BlockRecordPB`

每个`block`对应一个`BlockRecordPB`结构。

在该对象中，跟踪记录了对应`block`是否存在（“存在”对应`CREATE`；“不存在”对应`DELETE`）。

注意：在`container metadata file`中，关于`block`的操作内容，是被顺序写入的。

比如：有两条操作记录：第1条是`CREATE foo`，第2条是`DELETE foo`；那么结果就是`block foo`是不存在的。

### `block_id`
`BlockIdPB`类型，该`block`的唯一标识；

### `op_type`
`BlockRecordType`类型，标识当前`block`是否存在；

### `timestamp_us`
该`block`被创建的时间，单位是`微秒`。

### `offset`
该`block`在`container data file`中的偏移量。

如果`op_type == CREATE`，那么这个属性是必需的；  
因为对于一个“存在的”block，必须知道它在文件中的“位置”和“长度”。

### `length`
该`block`在`container data file`中占用的长度。  

如果`op_type == CREATE`，那么这个属性是必需的；  
因为对于一个“存在的”block，必须知道它在文件中的“位置”和“长度”。

```
message BlockRecordPB {
  required BlockIdPB block_id = 1;
  required BlockRecordType op_type = 2;
  required uint64 timestamp_us = 3;

  // The offset of the block in the container data file.
  //
  // Required for CREATE.
  optional int64 offset = 4;

  // The length of the block in the container data file.
  //
  // Required for CREATE.
  optional int64 length = 5;
}
```

## `message DataDirGroupPB`

一个`tablet`的数据，是可能分布在多个“数据根目录”下（每个`DiskRowSet`会有一个独立的目录）。

每个“数据根目录”，都会有一个唯一的`uuid`。

注意1：`uuids`是`repeated`的，即是一个列表。

每个`tablet`都对应一个`DataDirGroupPB`结构，表示在当前机器上，所有属于该`tablet`的所有“数据根目录”。

注意2：这个`uuids`列表，一定不会为空。

注意3：这里的`uuid`是指的每个“数据根目录”的`uuid`，而不是每个`tablet`的“子目录”。  
实际上，每个`tablet`的数据，不是都集中的分布在“一个目录”下的，而是分散的分布在多个“数据根目录”下。 另外，只会为“数据根目录”生成一个`uuid`，其它的“子目录”是不会生成`uuid`的。

```
message DataDirGroupPB {
  repeated bytes uuids = 1;
}
```