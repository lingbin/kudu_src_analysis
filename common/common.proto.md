[TOC]

## `enum DataType`

```
// If you add a new type keep in mind to add it to the end
// or update AddMapping() functions like the one in key_encoder.cc
// that have a vector that maps the protobuf tag with the index.
enum DataType {
  UNKNOWN_DATA = 999;
  UINT8 = 0;
  INT8 = 1;
  UINT16 = 2;
  INT16 = 3;
  UINT32 = 4;
  INT32 = 5;
  UINT64 = 6;
  INT64 = 7;
  STRING = 8;
  BOOL = 9;
  FLOAT = 10;
  DOUBLE = 11;
  BINARY = 12;
  UNIXTIME_MICROS = 13;
  INT128 = 14;
  DECIMAL32 = 15;
  DECIMAL64 = 16;
  DECIMAL128 = 17;
  IS_DELETED = 18; // virtual column; not a real data type
}

```

**如何添加一个新的`DataType`**  

要修改如下两个地方：
1. 添加到（该枚举类型中列表的）最后；
2. 或者 要修改代码中所有的`AddMapping()`函数（比如，在`key_encoder.cc`中就有要修改的`AddMapping()`）。 这个`AddMapping()`的作用是维护一个`vector`（比如各个类型对应的‘编码器’），使用“类型的值” 作为在该`vector`的下标 进行索引。

## `enum EncodingType`

```
enum EncodingType {
  UNKNOWN_ENCODING = 999;
  AUTO_ENCODING = 0;
  PLAIN_ENCODING = 1;
  PREFIX_ENCODING = 2;
  // GROUP_VARINT encoding is deprecated and no longer implemented.
  GROUP_VARINT = 3;
  RLE = 4;
  DICT_ENCODING = 5;
  BIT_SHUFFLE = 6;
}
```

>> 问题：针对不同的类型、不同的数据规律，如何挑选对应的编码类型？

## `message HmsMode`

有关 `Hive Metastore`相关的信息。

说明：在kudu中存在一个插件，用来支持`Hive`的元信息。

```
// Enums that specify the HMS-related configurations for a Kudu mini-cluster.
enum HmsMode {
  // No HMS will be started.
  NONE = 0;

  // The HMS will be started, but will not be configured to use the Kudu
  // plugin.
  DISABLE_HIVE_METASTORE = 3;

  // The HMS will be started and configured to use the Kudu plugin, but the
  // Kudu mini-cluster will not be configured to synchronize with it.
  ENABLE_HIVE_METASTORE = 1;

  // The HMS will be started and configured to use the Kudu plugin, and the
  // Kudu mini-cluster will be configured to synchronize with it.
  ENABLE_METASTORE_INTEGRATION = 2;
};
```

## `message ColumnTypeAttributesPB`

某一些类型的列，是有一些属性的。 

比如：对于Dicimal类型， `Decimal(6, 3)`和`Decimal(20, 10)`虽然都是`Decimal`类型，但是细分起来就不是相同的类型。

```
// Holds detailed attributes for the column. Only certain fields will be set,
// depending on the type of the column.
message ColumnTypeAttributesPB {
  // For decimal columns
  optional int32 precision = 1;
  optional int32 scale = 2;
}
```

注意：不是每种`DataType`都会被设置这个属性。目前只有`Decimal`类型会被设置。

注意2： 在`Kudu`中，字符串类型都是变长的，并且是没有“长度”这个属性的。这点和其它数据库中的`varchar(len)`不同。

## `message ColumnSchemaPB`

```
// TODO: Differentiate between the schema attributes
// that are only relevant to the server (e.g.,
// encoding and compression) and those that also
// matter to the client.
message ColumnSchemaPB {
  optional uint32 id = 1;
  required string name = 2;
  required DataType type = 3;
  optional bool is_key = 4 [default = false];
  optional bool is_nullable = 5 [default = false];

  // Default values.
  // NOTE: as far as clients are concerned, there is only one
  // "default value" of a column. The read/write defaults are used
  // internally and should not be exposed by any public client APIs.
  //
  // When passing schemas to the master for create/alter table,
  // specify the default in 'read_default_value'.
  //
  // Contrary to this, when the client opens a table, it will receive
  // both the read and write defaults, but the *write* default is
  // what should be exposed as the "current" default.
  optional bytes read_default_value = 6;
  optional bytes write_default_value = 7;

  // The following attributes refer to the on-disk storage of the column.
  // They won't always be set, depending on context.
  optional EncodingType encoding = 8 [default=AUTO_ENCODING];
  optional CompressionType compression = 9 [default=DEFAULT_COMPRESSION];
  optional int32 cfile_block_size = 10 [default=0];

  optional ColumnTypeAttributesPB type_attributes = 11;

  // The comment for the column.
  optional string comment = 12;
}
```

注意：
关于“默认值”，在客户端看来，一个列只能有一个默认值。 这里的`read/write default value`都是内部使用的，不能直接暴露出来。

**问题1： 为什么区分“读默认值”和“写默认值”？**  
答：参见注释部分：  
1. 在`create/alter table` 语句对应的请求中，默认值是在`read_default_value`中指定；
2. 在client打开一个table时，客户端会同时收到“读默认值”和“写默认值”。但是“写默认值”，是当前请求中，应该返回给客户端的默认值。


**问题2：为什么要在`ColumnSchemaPB`中指定 `cfile_block_size`? 是不是说同一个表的不同列，可以使用不同的`block_size`?**

## `message ColumnSchemaDeltaPB`

描述一个对`ColumnSchema`的修改。

```
message ColumnSchemaDeltaPB {
  optional string name = 1;
  optional string new_name = 2;

  optional bytes default_value = 4;
  optional bool remove_default = 5;

  optional EncodingType encoding = 6;
  optional CompressionType compression = 7;
  optional int32 block_size = 8;

  optional string new_comment = 9;
}
```

说明：这个`ColumnSchemaDeltaPB`对象是 由客户端组织好以后，再发给server的。在用户请求中，是不感知`column_id`的，所以在这个结构中，只用`name`来标识一个列。 在server端，也是通过`name`去查找指定的列。

## `message SchemaPB`

包含多个column的信息。

`repeated`类型，表示内容是一个 列信息的数组。

>> 扩展：在`kudu`中，不仅这里的`SchemaPB`对象，还有对应的`Schema`对象（非protobuf类），在逻辑上最关键的成员就是这个数组，其中每个元素是一个“列的元信息”。 （只不过，在普通的`Schema`类中，为了方便使用，还有一些其他的成员，包括`name`到`id`的映射等）

```
message SchemaPB {
  repeated ColumnSchemaPB columns = 1;
}
```

## `enum ExternalConsistencyMode`

用来指定达到“外部一致性”的实现方法。

**外部一致性**的含义： 如果client A进行了某个操作后，那么client B就应该能看到这个操作。

它定义了，“在多台机器上的、多个tablet”的 “事务”、或者“操作序列” ，在外部的多个client上的“可见性”。

```
  // Example:
  // 1 - Client A executes operation X in Tablet A
  // 2 - Afterwards, Client A executes operation Y in Tablet B
  //
  //
  // Client B may observe the following operation sequences:
  // {}, {X}, {X Y}
  //
```

注意：在这个例子中，client B 不可能 看到的序列是`{Y, X}`。即先发生了`Y`，然后才发生`X`。

即对于多个事务，需要修改位于多个机器上的多个tablet时，如何对外部 可见。

注意1：“外部一致性”仅仅是限制“一致性”，**对于“原子性”没有任何保证**。也就是说，无论设置“外部一致性”为何种方法，都不会保证 操作序列 是原子性的。（备注：不保证“操作序列”是原子的，也就是说，在一个操作序列上，可能会有部分操作是成功的，部分操作是失败的）

注意2：这里的“外部一致性”，对于在同一个tablet的多个replica上的“一致性”，也没有任何保证。（备注：一个tablet的多个replica上的一致性，是通过raft协议保证的。）

kudu提供两种方法来达到“外部一致性”。

### `CLIENT_PROPAGATED`

这是默认的方法。  
通过用户手动在多个client之间传递“时间戳”，来做到符合“外部一致性”。

对于任何一些“写请求”，都会返回一个时间戳。

从同一个kudu-client发出的任何后续的请求（发生后续请求时，前面的、需要符合“外部一致性”的请求已经返回，并且附带了一个时间戳），都会携带这个时间戳，并且使用这个时间戳去更新“目标机器”的时间戳。

如果需要在多个kudu-client之间传递该时间戳，那么需要 用户 自己来保证。

警告：如果需要在多个kudu-client之间保证“外部一致性”，那么必须手动在多个kudu-client之间传递该时间戳。如果没有传递，那么就无法做到这里的“外部一致性”模式。

### `COMMIT_WAIT`

通过“在提交前等待一段时间”的方式，来达到“外部一致性”。

方法：对于一个事务的执行结果，当前server只有在保证 其它所有的kudu server上的时钟都已经超过 该事务的时间戳（即都认为“该事务的时间戳”已经是“过去的时间”），才会将结果返回（即让该事务的修改内容可见）。

这种方式下，不需要强制用户手动在多个client之间传递 时间戳。

**这种方法的缺点：**  
1. 因为它强依赖 kudu server之间的时钟同步。所以这就需要一定的延迟。
2. 使用这种方式，在下面两种情况，请求会完全失败。
  + kudu server之间，无法进行时钟同步；
  + 机器间进行时钟同步的延迟 超过了某个预先配置的阈值。

>> outright: 完全的，彻底的  

```
enum ExternalConsistencyMode {
  UNKNOWN_EXTERNAL_CONSISTENCY_MODE = 0;

  // This is the default mode.
  CLIENT_PROPAGATED = 1;
  
  COMMIT_WAIT = 2;
};
```

## `enum ReadMode`

client发起读取操作时，可以在`Scan Request`中指定的“读取模式”。

在`server`端，有两个`snapshot`的边界:
1. 最早的`snapshot`: 在一个tablet中，`undo log`的最早记录;
2. 最晚的`snapshot`: 一个时间点，保证后续到达的事务，一定都会大于该时间点。 一般情况下，它的值为`clock->Now()`。但是如果用户手动传递一个值，是可能高于这个值的。

```
----------+-------------------------+-----------------> 时钟
          |                         |
          |                         |
    earliest snapshot             latest snapshot          
                            （“now”  或者 用户传递的时间）
```

### `READ_LATEST`

是“默认的模式”。

这种模式下，server在执行读取的时候，而是直接按照“收到读请求”时的“时间戳”进行读取。

**它不是“可重复读”的**
这种方式，不是“可重复读”的，所以也不会返回一个“快照时间戳”。  

“不可重复读”体现在两个方面：
1. 多次向同一台server发送同一个请求，得到的结果可能不同。
2. 一个server上按照相同的时间戳进行多次读取，得到的结果也可能不同。  
  
原因是：事务的提交是“乱序”的。在两次重试之前，有一些小于该“snapshot时间戳”的事务，可能会被提交了）。

### `READ_AT_SNAPSHOT`

在这种模式下，kudu client在发起请求的时候，会指定一个“快照时间戳”（如果没有指定，那么server会使用当前的时间，作为该“快照时间戳”）。  

**这种模式是“可重复读”的**  
在该模式下，进行多次读取，只要使用该“快照时间戳”，那么读取到的结果是一样的。  

**付出的代价**是：  
server会等待所有小于该“快照时间戳”的事务都结束以后，才会返回。

>> 说明：“外部一致性”主要是对“写入”的方式进行限制。

**和`COMMIT_WAIT`一起使用**  

如果既使用这种“读取模式”，还同时将“外部一致性”设置为`COMMIT_WAIT`：当用户使用前一个返回的`write_timestamp`作为本次读取的“快照读取时间戳”，（因为在`COMMIT_WAIT`模式下，返回“write_timestamp”时，就已经被保证没有后续事务的时间戳会小于`write_timestamp`。）所以，在进行读取时，是不需要任何等待操作的。

**和`CLIENT_PROPAGATED`一起使用**  
如果既使用这种“读取模式”，还同时将“外部一致性”设置为`CLIENT_PROPAGATED`：client必须在“读请求”中传递“时间戳”，这样server才会保证不会再产生 小于该时间戳 的事务。

实现方式是这样的：允许客户端设置1个或2个时间戳。
1. 第1个时间戳：**client必须提供**，是在设置了`CLIENT_PROPAGATED`时，上一个“写请求”返回的时间戳（可能是直接使用，也可能是客户端的一些后台消息传递方式）。server在处理的时候，会检测这个时间戳。
2. 第2个时间戳：可选的。如果设置了，那么是当前“读请求”实际的“快照读取时间戳”。

注意：如果这两个时间戳都同时设置，那么“第2个”一定要 小于或等于 “第1个”。

>> 问题：为什么“第2个”一定要 小于或等于“第1个”？   这样一个场景：如果在一个写操作后（会获取到该写操作的时间戳），经过了很长时间，才发起了读操作。那么这个“读取操作”的时间戳，一定是大于 上面“写操作”的时间戳的。

### `READ_YOUR_WRITE`

>> 说明：
>> 因为模式名字就是“读到你的写”，所以说在这次读取之前，前面应该是已经存在了一次“写操作”（写操作在返回时会返回对应的时间戳），这次读取操作的时候，会传递之前“写操作”的时间戳。

在这种模式下，server在选择时间戳时，根据如下规则：
1. 大于（在消息中传递）的时间戳；
2. 因为等待正在运行的事务结束，会造成延迟，这种模式会尝试将“这种延迟”降低到最小。

具体的说：server选择的时间戳，是在大于“所传递的时间戳”中，选择“最大的时间戳”，从而允许当前“读操作”不会被“正在运行的事务”所阻塞。（但是，如果传递的时间戳 是大于 某些正在运行的事务，那么当前读取也会被阻塞）。

```
------------+--------------+----------------------------------->
            ^              ^                         |
            |              |                         |
    progagated timestamp   | 
                        safe time                   
             
上面“最大的时间戳”的含义是：如果“传递的时间戳”小于 safe time，那么server就直接用“safe time”来进行读取。
```

如果没有在“读请求”中附上“传递的时间戳”，那么server会选择一个“在它之前的事务都已经结束”的时间戳。

本次“读取操作”所最终选择的时间戳，会在查询结果中返回给客户端（作为“快照时间戳”）。kudu客户端可以将该时间戳（“读操作”返回的“快照时间戳”）作为后续读取的“传递的时间戳”，从而避免不必要的等待。

**这种模式 不是 “可重复读”的：**  
两个`READ_YOUR_WRITES`读取，即使提供相同的“传递时间戳”，在执行的时候，也是可能会按照不同的时间戳进行读取，从而产生不同的结果。  
因为`safe time`可能提升了。

但是，这个模式，对于单个cleint，可以做到“read-your-writes”和“read-your-reads”，因为通过传递时间戳，所选择的时间戳，一定是大于最后一次‘读取’或者‘写入’的时间戳。

>> 扩展：从这里的描述来看，即使在“读请求”中指定相同的时间戳，`READ_YOUR_WRITE`的数据，是可能要新于 `READ_AT_SNAPSHOT`的。  
>>   因为`READ_YOUR_WRITE`会不断的推进 读取时所真正使用的时间戳。

```
enum ReadMode {
  UNKNOWN_READ_MODE = 0;

  // This is the default mode.
  READ_LATEST = 1;

  READ_AT_SNAPSHOT = 2;

  READ_YOUR_WRITES = 3;
}
```

## `enum OrderMode`

用来表示当前查询 是否要求 对结果进行排序。

client会在`request`中指定该需求。

注意：如果一个“scan请求”是要求排序的，那么这个请求是“可容忍错误的”(`fault-tolerant`)。即，如果一个`tablet server`在查询过程中宕机了，那么可以向其它server发起重试。  

但是：如果一个“scan请求”是要求排序，那么会有额外的开销。因为`tablet server`需要对来自多个`RowSet`的数据进行排序。

```
enum OrderMode {
  UNKNOWN_ORDER_MODE = 0;
  // This is the default order mode.
  UNORDERED = 1;
  ORDERED = 2;
}
```

## `enum ReplicaSelection`

一个读请求，选择`replica`的方法。

```
// Policy with which to choose among multiple replicas.
enum ReplicaSelection {
  UNKNOWN_REPLICA_SELECTION = 0;
  LEADER_ONLY = 1;
  CLOSEST_REPLICA = 2;
}
```

## `message PartitionSchemaPB`

用来描述一个表(`table`)的分区方法。

注意：对于range分区，不包含每个`range`的 起止点。

但是，对于hash分区的描述比较全面：使用的hash函数、分为几个桶等，都有描述。

在“单层分区”时，可以单独使用`range分区`、或`hash分区`。

但是如果要使用多级分区，最多可以有两层分区：
1. 前面的0个或多级分区是`hash`分区，是必需的；
2. 最后一级分区是`range`分区，是可选的。

也就是说，对于多层分区（大于等于2层），可能有以下情况：
1. 全部层 都是 “hash分区”，没有“range分区”；
2. 前面多层都是“hansh分区”，最后一层是“range分区”；

### `message PartitionSchemaPB::ColumnIdentifierPB`

有两个元素，实际使用时是二选一的(在`ProtoBuf`中使用 `oneof` 关键字)。
1. `id`: column的id;
2. `name`: column的name;

使用场景：
在`client`创建表时，使用`name`。因为`column`的`id`是在`master`上分配的，而在client发起建表需求的时候，`column_id`还没有被分配出来，所以这时只能使用`name`。

其它场景，应该都使用`column_id`。

### `message PartitionSchemaPB::RangeSchemaPB`

描述“range分区”的方法。现在只是指定了“分区列的数组”，即指定了“分区列的集合”。

注意：这里不包括每个range的 起止点。

### `message PartitionSchemaPB::HashBucketSchemaPB`

描述hash分区的方法。

有4个成员：
1. hash列的数组；
2. hash桶数： 注意必须大于等于2；
3. `seed`： hash函数的种子； 
4. 要使用的hash函数；

**`seed`的使用说明：**  
系统管理员，可以为不同的表 指定不同的`seed`，这样可以有利用于数据的打散（即，`seed`的设定是“table级别”的）。

比如：两个表，使用相同的hash列。那么在分别向两个表导入数据时，如果使用“不同的seed”（即同一个主键，在经过hash后的值是不同的，所以更大概率会hash到不同的机器上）。

>> denial: 否认；拒绝；节制；背弃  

问题：`Setting a seed provides some amount of protection against denial of service attacks when the hash bucket columns contain user provided imput`? 这句话是什么意思？ 保护了啥？怎么保护的？

```
message PartitionSchemaPB {

  message ColumnIdentifierPB {
    oneof identifier {
      int32 id = 1;
      string name = 2;
    }
  }

  message RangeSchemaPB {
    repeated ColumnIdentifierPB columns = 1;
  }

  message HashBucketSchemaPB {
    repeated ColumnIdentifierPB columns = 1;

    required int32 num_buckets = 2;

    // Setting a seed provides some amount of protection against denial
    // of service attacks when the hash bucket columns contain user provided
    // input.
    optional uint32 seed = 3;

    optional HashAlgorithm hash_algorithm = 4;
  }

  repeated HashBucketSchemaPB hash_bucket_schemas = 1;
  optional RangeSchemaPB range_schema = 2;
}
```

注意：  
这里`hash_bucket_schemas`是一个`repeated`类型，即 “hash分区”是可以有多个的（但是“range”分区最多只能有一个）。

## `message PartitionPB`

用来表述一个 `partition`。 （一个表会有多个`partition`）

针对`hash partition`和`range partition`，使用不同的成员。

**`hash partition`**  
使用成员`hash_buckets`。 分桶数

**range partition**  
使用成员`partition_key_start`和`partition_key_end`。

注意：这是一个“左闭右开”区间。

```
message PartitionPB {
  // The hash buckets of the partition. The number of hash buckets must match
  // the number of hash bucket components in the partition's schema.
  repeated int32 hash_buckets = 1 [packed = true];
  // The encoded start partition key (inclusive).
  optional bytes partition_key_start = 2;
  // The encoded end partition key (exclusive).
  optional bytes partition_key_end = 3;
}
```

## `message ColumnPredicatePB`

用来表述一个“判断条件”，可以应用到 一个column上，从而进行过滤。

描述下面6种类型 之一（使用了`protobuf`的`oneof`关键字）：
1. 是否属于某个`range`：
2. 是否相等；
3. 是否为空；
4. 是否非空；
5. 是否在一个`list`中；
6. 是否在`bloomfilter`中；

### `message ColumnPredicatePB::BloomFilter`

描述一个`BloomFilter`结构。

### `message ColumnPredicatePB::Range`

描述一个区间。

不同类型使用不同的编码方法。
1. `String/Binary`类型的值：直接使用二进制的字符串格式。
2. 其它类型：直接使用 x86机器 在内存中的格式。 比如：对于`uint32`类型，是个“小端”的存储方式。

注意1:   
是“左闭右开”区间，即`[lower, upper)`。

注意2：  
`Range`类型，不可以用于`null`值。   
因为对于`null`来说，是既不大于任何值，也不小于任何值的。

### `message ColumnPredicatePB::Equality`

描述一个“等值比较”。

编码方式，和`Range`使用相同的方式。

### `message ColumnPredicatePB::InList`

提供一个列表（`repeated`修饰符）。描述column的值“是否在该列表”中。

### `message ColumnPredicatePB::IsNotNull` 和  `message ColumnPredicatePB::IsNull`

就是判断“当前列的值，是否为空”。

因为从名字就能够看出来用途，所以该类型不需要任何属性。

### `message ColumnPredicatePB::InBloomFilter`

判断当前列的值，是否在`BloomFilter`中。

注意：该结构也可以表述：一个列上，同时使用了`BloomFilter`和`Range`条件。

```
// A predicate that can be applied on a Kudu column.
message ColumnPredicatePB {
  optional string column = 1;

  message BloomFilter {
    // The hash times for bloom filter.
    optional int32 nhash = 1;
    // The bloom filter bitmap.
    optional bytes bloom_data = 2 [(kudu.REDACT) = true];
    optional HashAlgorithm hash_algorithm = 3 [default = CITY_HASH];
  }

  message Range {
    // Note that this predicate type should not be used for NULL data --
    // NULL is defined to neither be greater than or less than other values
    // for the comparison operator.
    optional bytes lower = 1 [(kudu.REDACT) = true];
    optional bytes upper = 2 [(kudu.REDACT) = true];
  }

  message Equality {
    optional bytes value = 1 [(kudu.REDACT) = true];
  }

  message InList {
    repeated bytes values = 1 [(kudu.REDACT) = true];
  }

  message IsNotNull {}

  message IsNull {}

  message InBloomFilter {
    // A list of bloom filters for the field.
    repeated BloomFilter bloom_filters = 1;

    optional bytes lower = 2 [(kudu.REDACT) = true];
    optional bytes upper = 3 [(kudu.REDACT) = true];
  }

  oneof predicate {
    Range range = 2;
    Equality equality = 3;
    IsNotNull is_not_null = 4;
    InList in_list = 5;
    IsNull is_null = 6;
    InBloomFilter in_bloom_filter = 7;
  }
}
```

## `message KeyRangePB`

描述在一个tablet上，“复合主键”的range区间。

```
message KeyRangePB {
  optional bytes start_primary_key = 1 [(kudu.REDACT) = true];
  optional bytes stop_primary_key = 2 [(kudu.REDACT) = true];
  required uint64 size_bytes_estimates = 3;
}
```

>> 问题：使用场景是什么？ 为什么`size_bytes_estimates`是`required`字段？

## `message TableExtraConfigPB`

描述 表 的额外配置。

```
message TableExtraConfigPB {
  // Number of seconds to retain history for tablets in this table,
  // including history required to perform diff scans and incremental
  // backups. Reads initiated at a snapshot that is older than this
  // age will be rejected. Equivalent to --tablet_history_max_age_sec.
  optional int32 history_max_age_sec = 1;

  // Priority level of a table for maintenance, it will be clamped into
  // range [-FLAGS_max_priority_range, FLAGS_max_priority_range] when
  // calculate maintenance priority score.
  optional int32 maintenance_priority = 2;
}
```















