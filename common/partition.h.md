[TOC]

文件：`src/kudu/common/partition.h`

# `class Partition`

`Partition`类，用来描述一个`Tablet`所应该负责保存的`Row`信息。

每个`TabletMetadata`都有一个`Partition`属性(即每个`tablet`对应一个`Partition`对象)。

说明：对应的`protobuf`结构是`PartitionPB`，参见`src/kudu/common/common.proto`.

>> primarily: 主要地；首要地；  

`Partition`类中，主要包含的成员是：当前分区的`start key` 和`end key`。

>> 名字说明：  
>> `partition key`: 指的是`Partition`的上下边界。参见下面的`成员`部分的描述。  
>> 给定一个`Row`，是可以生成对应的`partition key`，然后就可以判断该`Row`应该是属于哪个`Partition`。

## 成员

### `hash_buckets_`

其中包含的元素数，等于当前table中 `hash 分区`的层级数。

>> 每个元素值，表示在“当前hash 分区层级”的什么含义？

在`hash_buckets_`中的每个元素值，是当前`partition`在对应层级（hash分区）中对应的`hash bucket`(“hash桶号”)。

### `partition_key_start_`

### `partition_key_end_`

每个`partition key`包含两个部分：
1. 0个或多个`hash bucket`;
2. 最后是一个`range`分区

对于`range partition key`的部分，其中是按照`range partition`的分区列，逐个进行编码。

```
// 每个`partition key`的编码方法

      hash1     hash2    hash3           range 
    |--------+--------+--------+---------------------|
    
```

```
  std::vector<int32_t> hash_buckets_;

  std::string partition_key_start_;
  std::string partition_key_end_;
```

## 接口列表

该对象就是一个`POD`，提供了`getter`和`setter`方法。

然后提供了和`PartitionPB`对象，相互转化的方法： `ToPB()`和`FromPB()`。

### `range_key_start()`
从`partition_key_start_`中抽取出`range分区`的部分；

如果这部分为空（长度为0）,那么表示 没有“下边界”，即“负无穷”。

### `range_key_end()`
从`partition_key_end_`中抽取出`range分区`的部分；

如果这部分为空（长度为0）,那么表示 没有“上边界”，即“正无穷”。

#### `private range_key()` 

工具函数，从`partition_key`中，解析出来`range key`的部分。(因为一个`partition key`中，前半部分是根据`hash partition`的层级，对应的`hash bucket`数值)。

```
Slice Partition::range_key(const string& partition_key) const {
  size_t hash_size = kEncodedBucketSize * hash_buckets().size();
  if (partition_key.size() > hash_size) {
    Slice s = Slice(partition_key);
    s.remove_prefix(hash_size);
    return s;
  } else {
    return Slice();
  }
}
```

## `class PartitionSchema`

描述一个表的数据行，应该如何分布。

注意：每个`PartitionSchema`对象，描述的是一个表的整体分区方法，并不是“一个分区自身”的属性。 **也就是说，每个`table`对应一个`PartitionSchema`对象**。

对应的`protobuf`结构是`PartitionSchemaPB`，参见`src/kudu/common/common.proto`。

概括的说，一个`PatitionSchema`对象（每个`table`都对应一个该对象），就是负责针对 该表中的每行数据，将它的“主键”转化为一个“`partition key`”。使用这个`partition key`，可以决定该行数据所属的`tablet`。

**组成内容：**    
在每个`PartitionSchema`对象中，可能包含“零个”或“多个”的`hash`分区，最后也可能再包含“一个”`range分区`。 （注意：`hash分区`在`range分区`之前，即如果两者同时存在，那么是先`hash分区`，然后再`range分区`）。

每个`hash分区` 包含“一个或多个列（主键的子集）”。但是要注意的是：如果有多层`hash分区`，那么一个列，只能在其中一个`hash`分区中。

类似的，“range分区列”也必须是主键的子集。在如果一个列在 “range分区列”中，那么也应该只存在一次。即在`create table`语句中，声明“range分区”时，不能使用类似`PARTITION BY RANGE (date, id, id)`等多次使用同一个列。

>> 注意：对于一个属于“复合主键”的列，是可以 同时在“hash分区”和“range分区”中。

```
// “hash分区列”和“range分区列”都是`id`
create table million_rows_one_range (id string primary key, s string)
  partition by hash(id) partitions 50,
  range (id) (partition 'a' <= values < '{')
  stored as kudu;
```

**给定一行数据，如何计算它所属的`hash bucket`**  
对于给定的数据行，首先将“hash分区列”的值，编码成二进制（是可以按照“字典序”比较的），然后计算出`hash值`(`uint64_t`类型)，最后进行“取余计算”得出对应的桶(`int32_t`类型）。

**给定一行数据，如何得到`hash partition key`**  

`hash partition key`的内容，是一个编码结果（每个“hash分区”的层级对应一个`int32_t`）。编码对象是 当前`hash分区`的`hash bucket`值。

给定一行数据，先根据它的“主键”计算出来的`hash bucket`，然后按照`hash分区`的层级，逐层地编码进它的`partition key`（编码是保序的）。

>> 说明：也就是说，给定一行数据，为了得到它的`partition key`，必然要先计算出来它所属的`hash bucket`。

**给定一行数据，如何得到`range partition key`**  
在`range分区`中，可能包含“全部主键列”，也可能只包含“部分主键列”。 在编码时，对于“声明表时所指定的‘range分区列’”，按照顺序，逐个编码。

>> tricky: 棘手的; 狡猾的；  

>> 说明：上面说的是“一行数据”和“该行数据 所对应的`partition key`”之间的转化方法。但是在用`partition key`来表示一个`Partition`中数据范围的时候，是有些特殊的。

**每个`tablet`对应的`patition key`范围**  
根本原因：一个`tablet`的边界，并不是一定要 对齐到行数据。（参见`CreatePartitions()`中的 “填充空洞”的部分）

比如：在`kudu`目前的实现中，
+ 对于一个表，它的`absolute-start key`和`absolute-end key`都是一个`empty key`（使用“empty key”来表示“边界没有限制”，即正负无穷。在`lower`位置表示“负无穷”，在`upper`位置表示“正无穷”）。 这个`empty key`，并不对应任何的数据行；（这个例子是为了说明：目前`tablet`的边界，不一定是一条真实存在的数据）。
+ 每个`Partition`也是类似的。

在建表时，会创建`partition`集合，这时会将`absolute-start`和`absolute-end`同时放入到“下一层级”中。

**注意：在kudu中，`Partition`信息和`Partition Schema`的信息被认为是“元数据”（“元数据”不会被认为是敏感信息），所以在打印`debug`信息时，不进行`redact`。**  

但是，“每行数据”中的`partition key`会被认为是“敏感信息”，所以需要进行`redact`。

也就是说，对于那些格式化多个`Partition`或`Partition schema`的方法中，不会进行`redact`；而对于仅格式化一行数据的`partition key`的方法中，会进行`redact`。

根本原因是：“用户的数据”被认为是“敏感”的；“元数据”(包括`Partition`和`PartitionSchema`)被认为是“不敏感”的。

## `struct PartitionSchema::RangeSchema`
描述`range`分区方式。

```
  struct RangeSchema {
    std::vector<ColumnId> column_ids;
  };
```

注意1：这里只描述的“参与`range分区`的column列表”（因为是一个列表，所以也隐含了 顺序），但是并没有描述 每个“range分区”的边界。

说明：从用途上讲，`PartitionSchema`只是指明了具体的分区结构（用哪些列进行“hash分区”或者“range分区”），而不包含详细的分区信息（每个分区所能表示的`key`范围）。

而每个“Range分区”的边界，是在`Partition`对象中指定的。在用户创建表后，每个`Tablet`对象中就会维护它所对应的`Partition`对象。

## `struct PartitionSchema::HashBucketSchema`
描述一个表的`hash`分区方式。

注意：一个`HashBucketSchema`对象，仅代表一层“hash分区”。

```
  struct HashBucketSchema {
    std::vector<ColumnId> column_ids;
    int32_t num_buckets;
    uint32_t seed;
  };
```

## 成员

```
  std::vector<HashBucketSchema> hash_bucket_schemas_;
  RangeSchema range_schema_;
```

说明：在确定了`hash_bucket_schemas_`和`range_schema_`以后，该表对应的“总tablet”数就确定了。 

```
假如一个表有3层hash分区（对应的 bucket 数分别为 x, y, z），1 层 range分区(对应的区间有 `m` 个)。

那么：
    总tablet数目 = x * y * z * m
```

## 接口列表

### `CreatePartitions()`

**用途：**  
处理建表请求时，根据用户提供的“`range分区`方式”，创建出对应`Partition`对象。

**注意：在当前类（`PartitionSchema`）中，并没有直接维护`Partition`对象。**  
一个表的所有`tablet`（每个`tablet`会对应一个`Partition`）信息，实际上在`master`中维护的（其中的`TabletInfo`类）。

#### **说明：当前函数中，`split_rows`和`range_bounds`的含义。**

参考： https://kudu.apache.org/2016/08/23/new-range-partitioning-features.html  

在很早以前的版本中，“range分区”是通过指定`split points`来进行划分的。通过这些`split points`，将整个值区间划分为多个 “连续的”、“不相交的”区间。

这个方法的问题是：对于“时序数据”，无法很好的为将来时间的数据提前划分好`区间`，造成最后一个分区会成为热点。

为了解决上述问题，在`kudu 0.10`之后，不再采用这种方式。而是通过显式指定“每个range分区范围”的方式。在这种方式下，只有用户指定的区间才会生成“range分区”，如果用户没有指定的区间，是不会自动生成`range分区`的。（这样，对于将来时间的数据，可以通过手动动态增加“分区”的方式，来避免最后一个分区成为热点）

>> 扩展：其实标准的解决方法是：是让“range分区”支持分裂，但是分裂的实现代价比较高，所以`kudu`暂时没有采用。不过，这个新的方法，并不会阻碍kudu将来支持分裂。

但是在代码中，仍然是支持通过指定`split points`的。即目前用户可以有3种方式来指定区间划分：
1. 只指定`split points`;
2. 只指定`bounds`（即显式指定每个分区的范围）
3. 同时使用上面两种方式。

在当前函数中，
1. `split_rows`：对应上面的`split points`方式；
2. `range_bounds`：对应上面的“显式指定各个区间边界(`range bounds`)”；

#### 建表语句举例

参见如下示例：在建表时，可以同时使用`hash分区`和`range分区`。对于`range区分`，即可以指定一个范围，也可以指定某一个固定的值。

注意：如下方式，是用“显式指定区间边界”的方式。

即使是`value = '0'`，也**不是**在指定`split points`，而是指定区间边界（`range bounds`）。对应的区间是`['0', '00')`(左闭右开区间)。（对于这种`等号`，会对`upper`按照类型去获取一个比指定的值 大一个单位 的值）。

```
create table million_rows_three_ranges (id string primary key, s string)
  partition by hash(id) partitions 10,
  range (
    partition 'a' <= values < '{', 
    partition 'A' <= values < '[',
    partition value = '0') 
  stored as kudu;
  
这里，在每个hash分区下的`range分区` 区间分布如下：

    |----==------==============--------===============-------|
         ^             ^                    ^
         |             |                    |
    ['0', '00')    ['a', '{')          ['A', '[')
```

```
Status PartitionSchema::CreatePartitions(const vector<KuduPartialRow>& split_rows,
                                         const vector<pair<KuduPartialRow,
                                                           KuduPartialRow>>& range_bounds,
                                         const Schema& schema,
                                         vector<Partition>* partitions) const {
  const KeyEncoder<string>& hash_encoder = GetKeyEncoder<string>(GetTypeInfo(UINT32));

  // Create a partition per hash bucket combination.
  *partitions = vector<Partition>(1);
  for (const HashBucketSchema& bucket_schema : hash_bucket_schemas_) {
    vector<Partition> new_partitions;
    // For each of the partitions created so far, replicate it
    // by the number of buckets in the next hash bucketing component
    for (const Partition& base_partition : *partitions) {
      for (int32_t bucket = 0; bucket < bucket_schema.num_buckets; bucket++) {
        Partition partition = base_partition;
        partition.hash_buckets_.push_back(bucket);
        hash_encoder.Encode(&bucket, &partition.partition_key_start_);
        hash_encoder.Encode(&bucket, &partition.partition_key_end_);
        new_partitions.push_back(partition);
      }
    }
    partitions->swap(new_partitions);
  }

  std::unordered_set<int> range_column_idxs;
  for (const ColumnId& column_id : range_schema_.column_ids) {
    int column_idx = schema.find_column_by_id(column_id);
    if (column_idx == Schema::kColumnNotFound) {
      return Status::InvalidArgument(Substitute("range partition column ID $0 "
                                                "not found in table schema.", column_id));
    }
    if (!InsertIfNotPresent(&range_column_idxs, column_idx)) {
      return Status::InvalidArgument("duplicate column in range partition",
                                     schema.column(column_idx).name());
    }
  }

  vector<pair<string, string>> bounds;
  vector<string> splits;
  RETURN_NOT_OK(EncodeRangeBounds(range_bounds, schema, &bounds));
  RETURN_NOT_OK(EncodeRangeSplits(split_rows, schema, &splits));
  RETURN_NOT_OK(SplitRangeBounds(schema, std::move(splits), &bounds));

  // Create a partition per range bound and hash bucket combination.
  vector<Partition> new_partitions;
  for (const Partition& base_partition : *partitions) {
    for (const auto& bound : bounds) {
      Partition partition = base_partition;
      partition.partition_key_start_.append(bound.first);
      partition.partition_key_end_.append(bound.second);
      new_partitions.push_back(partition);
    }
  }
  partitions->swap(new_partitions);

  // Note: the following discussion and logic only takes effect when the table's
  // partition schema includes at least one hash bucket component, and the
  // absolute upper and/or absolute lower range bound is unbounded.
  //
  // At this point, we have the full set of partitions built up, but each
  // partition only covers a finite slice of the partition key-space. Some
  // operations involving partitions are easier (pruning, client meta cache) if
  // it can be assumed that the partition keyspace does not have holes.
  //
  // In order to 'fill in' the partition key space, the absolute first and last
  // partitions are extended to cover the rest of the lower and upper partition
  // range by clearing the start and end partition key, respectively.
  //
  // When the table has two or more hash components, there will be gaps in
  // between partitions at the boundaries of the component ranges. Similar to
  // the absolute start and end case, these holes are filled by clearing the
  // partition key beginning at the hash component. For a concrete example,
  // see PartitionTest::TestCreatePartitions.
  for (Partition& partition : *partitions) {
    if (partition.range_key_start().empty()) {
      for (int i = static_cast<int>(partition.hash_buckets().size()) - 1; i >= 0; i--) {
        if (partition.hash_buckets()[i] != 0) {
          break;
        }
        partition.partition_key_start_.erase(kEncodedBucketSize * i);
      }
    }
    if (partition.range_key_end().empty()) {
      for (int i = static_cast<int>(partition.hash_buckets().size()) - 1; i >= 0; i--) {
        partition.partition_key_end_.erase(kEncodedBucketSize * i);
        int32_t hash_bucket = partition.hash_buckets()[i] + 1;
        if (hash_bucket != hash_bucket_schemas_[i].num_buckets) {
          hash_encoder.Encode(&hash_bucket, &partition.partition_key_end_);
          break;
        }
      }
    }
  }

  return Status::OK();
}
```

步骤1：首先计算出多个层级的“hash分区”对应的`Partition`对象；    

多层“hash分区”，最终生成的`Partition`的数量，是各个层级的“hash分区”的桶数 乘积。

在计算下一轮的时候，对于上一轮的结果中的每个`Partition`，会生成当前层级的“hash分区”的`hash桶数` 个 新的`Partition`对象。

这里的每个`Partition`对象，都会将每个层级的`hash bucket`同时添加到`partition_key_start_`和`partition_key_end_`成员的末尾。

步骤2：然后检查提供的“range分区列”：1) 是否都存在; 2) 是否有重复。  
这些分区列，在`Schema`中必须是存在的。 而且，同一个列，不能指定“range分区列”的时候使用两次。

>> 扩展说明： 同一个主键列，单独在“hash分区”和“range分区”中的，都不能重复使用。但是可以 同时在“hash分区”和“range分区”中。

步骤3：根据用户指定的`split points`和`range bounds`组建 “range区间”，并且根据需要进行相应的拆分。  

对应如下几个函数：  
```
  vector<pair<string, string>> bounds;
  vector<string> splits;
  RETURN_NOT_OK(EncodeRangeBounds(range_bounds, schema, &bounds));
  RETURN_NOT_OK(EncodeRangeSplits(split_rows, schema, &splits));
  RETURN_NOT_OK(SplitRangeBounds(schema, std::move(splits), &bounds));
```

步骤4：将上面“hash分区”创建出来的`Partition`列表，与这里的“range分区”连接起来。  
这个时候，会将“range bounds”添加到`Partition`对象中的`partition_key_start_`和`partition_key_end_`成员中。

如果上面“hash分区”创建出来 `x`个`Partition`对象， 这里的`range bounds`中有 `y` 个区间，那么最终声明的`Partition`对象 数量为 `x * y` 个。 

步骤5：将hash分区之间的“洞”填充上。

需要进行这个“填充洞”操作的前提是： 
1. 当前表中有“至少一层 hash分区”；
2. 并且它的range分区的上下边界，至少有一侧是 “无边界的”，即可为“正负无穷”。

经过前面的步骤，目前得到的`Partition`列表中，每个`Partition`对象都会覆盖一段数据范围，但是整体上看，可能是有空洞的(`range分区`的“上下边界”为“正负无穷”时)。

**填充这些“洞”的原因：**  
而在查询过程中的一些操作，比如“分区裁剪”、“在client端缓存元信息”，如果`Partition`之间没有空洞，那么会更容易。 所以

**填充方法：**  
如果仅有一层“hash”分区，仅看`range partition key`部分，对于“第一个”和“最后一个”分区（即“range 区间”的`lower`或`upper`为“无穷”的分区），将这个边界值扩大，让两个相邻的`Partition`之间没有“洞”。

如果有“两层”或“两层以上”的hash分区，那么每个层级之间也会有“洞”，这个时候也要进行填充。

说明：如何判断一个`Partition`是否为 某个hash分区层级上的“第一个”或“最后一个”分区？
1. `hash_buckets`（“hash的桶号”）为`0`的“hash分区”，就是该层的“hash分区”的 首个分区。 
2. `hash_buckets`（“hash的桶号”）的值，等于该层“hash分区”的总桶数 - 1，即`hash_bucket_schemas_[i].num_buckets - 1`，那么就是最后一个层级。

处理方法:  
1. 如果某个`Partition`的`lower`边界的`range partition key`为空，那么“从右向左” 检查`hash_buckets`的值，如果桶号为`0`(表示该层级的第一个分区)，那么就把`partition_key_start_`中删除 从“该位置”到“末尾”的内容；
2. 如果某个`Partition`的`upper`边界的`range partition key`为空，那么“从右向左”逐个“hash分区”的层级进行处理。
  + 首先清空从“当前`hash bucket`位置”，到“末尾”的所有数据；
  + 其次，检查当前`hash bucket`是否为 当前hash分区 的最后一个分区；如果不是最后一个，那么将当前`hash_bucket + 1`编码到 `partition_key_end_`的末尾。

>> 说明： `std::string::erase(n)`方法，是删除 从指定下标位置（`n`） 到 字符串的末尾。（注意不是：删除头部的`n`个元素）。

```
// 表结构如下：
  CREATE TABLE t (a VARCHAR, b VARCHAR, c VARCHAR, PRIMARY KEY (a, b, c))
  PARITITION BY [HASH BUCKET (a), HASH BUCKET (b), RANGE (a, b, c)];

  其中，两层hash分区的桶数，都是2

// 指定的`split points`如下：
  { a: "a1", b: "b1", c: "c1" }
  { b: "a2", b: "b2" }
// 或者指定的`range bounds`如下：
  ["",                          (a: "a1", b: "b1", c: "c1"))
  [(a: "a1", b: "b1", c: "c1"), (b: "a2", b: "b2"))
  [(b: "a2", b: "b2"),          "")

// 注意：对于“现有显式指定的列”，这里会用“最小值”进行填充。  
// 这里没有显式`c`列，它是字符串类型，最小值为`""`(空字符串)。

// 对于上述range分区，会产生3个区间，如下：
  [ (      ""),   ("a1b1c1") )
  [ ("a1b1c1"),   (  "a2b2") )
  [ (  "a2b2"),   (      "") )
  
// 注意：即使`c`列在`upper bound`的位置，因为用户没有提供， 
// 在编码的时候，仍然使用最小值来进行填充。
// 即使用空字符串(`""`)进行填充。

// 上述分区方式（2层hash分区，每个hash分区的桶数为2, 最后一层range分区有3个区间），
// 所以总的`Partition`数量为: 2 * 2 * 3 = 12 个

// 将“hash分区”和“range分区”结合起来，直接构造出来的所有`Partition`列表为：
  1.  [ (0, 0,       ""),   (0, 0, "a1b1c1") )
  2.  [ (0, 0, "a1b1c1"),   (0, 0,   "a2b2") )
  3.  [ (0, 0,   "a2b2"),   (0, 0,       "") )
      
  4.  [ (0, 1,       ""),   (0, 1, "a1b1c1") )
  5.  [ (0, 1, "a1b1c1"),   (0, 1,   "a2b2") )
  6.  [ (0, 1,   "a2b2"),   (0, 1,       "") )
      
  7.  [ (1, 0,       ""),   (1, 0, "a1b1c1") )
  8.  [ (1, 0, "a1b1c1"),   (1, 0,   "a2b2") )
  9.  [ (1, 0,   "a2b2"),   (1, 0,       "") )
      
 10.  [ (1, 1,       ""),   (1, 1, "a1b1c1") )
 11.  [ (1, 1, "a1b1c1"),   (1, 1,   "a2b2") )
 12.  [ (1, 1,   "a2b2"),   (1, 1,       "") )

// 可见，上面的空洞位置在：
//   1. 在 p1  的"lower"之前
//   2. 在 p3  的"upper" 和 p4  的"lower"之间
//   3. 在 p6  的"upper" 和 p7  的"lower"之间
//   4. 在 p9  的"upper" 和 p10 的"lower"之间
//   5. 在 p12 的"lower"之后

// “填充洞”之后，最终的结果为：
  1.  [ (_, _,        _),   (0, 0, "a1b1c1") )
  2.  [ (0, 0, "a1b1c1"),   (0, 0,   "a2b2") )
  3.  [ (0, 0,   "a2b2"),   (0, 1,        _) )
     
  4.  [ (0, 1,        _),   (0, 1, "a1b1c1") )
  5.  [ (0, 1, "a1b1c1"),   (0, 1,   "a2b2") )
  6.  [ (0, 1,   "a2b2"),   (1, _,        _) )
     
  7.  [ (1, _,        _),   (1, 0, "a1b1c1") )
  8.  [ (1, 0, "a1b1c1"),   (1, 0,   "a2b2") )
  9.  [ (1, 0,   "a2b2"),   (1, 1,        _) )
     
 10.  [ (1, 1,        _),   (1, 1, "a1b1c1") )
 11.  [ (1, 1, "a1b1c1"),   (1, 1,   "a2b2") )
 12.  [ (1, 1,   "a2b2"),   (_, _,        _) )
 
  _ signifies that the value is omitted from the encoded partition key.
```

#### `private SplitRangeBounds()`
在“range分区”中同时制定了`split points`和`range bounds`，那么这个方法用`split points`来将`range bounds`进行切分成多个区间。

**说明1：传入的“`splits`参数”和“`bounds`参数”都是有序的。**  

说明2：如果`bounds`为空，那么说明当前“range分区”只使用的`split points`，即最终的划分结果，会覆盖整个值域。

**注意：如果同时使用了`splits`和`bounds`（即两者都不为空），那么要求：所有的`splits`，必须落在`bounds`上。**

```
          bounds_1               bounds_2
<--------|++++++++|------------|++++++++++|---------------->
                      ^              ^
                      |              |
                    splits_1       splits_2
                    
如上图，bounds中指定了两个区间。
1. splits_1 是无效的，因为没有落在任何一个`bounds`上；
2. splits_2 是有效的，因为落在`bounds_2`上；
```

```
Status PartitionSchema::SplitRangeBounds(const Schema& schema,
                                         vector<string> splits,
                                         vector<pair<string, string>>* bounds) const {
  int expected_bounds = std::max(1UL, bounds->size()) + splits.size();

  vector<pair<string, string>> new_bounds;
  new_bounds.reserve(expected_bounds);

  // Iterate through the sorted bounds and sorted splits, splitting the bounds
  // as appropriate and adding them to the result list ('new_bounds').

  auto split = splits.begin();
  for (auto& bound : *bounds) {
    string& lower = bound.first;
    const string& upper = bound.second;

    for (; split != splits.end() && (upper.empty() || *split <= upper); split++) {
      if (!lower.empty() && *split < lower) {
        return Status::InvalidArgument("split out of bounds", RangeKeyDebugString(*split, schema));
      }
      if (lower == *split || upper == *split) {
        return Status::InvalidArgument("split matches lower or upper bound",
                                       RangeKeyDebugString(*split, schema));
      }
      // Split the current bound. Add the lower section to the result list,
      // and continue iterating on the upper section.
      new_bounds.emplace_back(std::move(lower), *split);
      lower = std::move(*split);
    }

    new_bounds.emplace_back(std::move(lower), upper);
  }

  if (split != splits.end()) {
    return Status::InvalidArgument("split out of bounds", RangeKeyDebugString(*split, schema));
  }

  bounds->swap(new_bounds);
  CHECK_EQ(expected_bounds, bounds->size());
  return Status::OK();
}
```

**说明1：最后结果中的区间数量，是可以计算出来的**  
因为每个`split points`默认会将原有的区间数量 增大1个。  
对应上面的`int expected_bounds = std::max(1UL, bounds->size()) + splits.size();`  

这里，如果`bounds`为空，实际上就相当于`splits`切分的是`[-oo, +oo]`，即潜在的是有一个区间的。  
所以： `切分前的区间数量 = std::max(1UL, bounds->size())`。

在函数的最后，还会进行检查，“实际生成的区间数量”是否等于“预期的数量”。  
对应 `CHECK_EQ(expected_bounds, bounds->size());`

**说明2: 因为`splits`和`bounds`都是有序的，所以实现的逻辑比较简单。**  
就是：从前向后迭代两个`bounds`和`splits`，对于每个`bound`，如果`splits`的元素满足`lower < split < upper`，那么当前`bound`就可以切分为`[lower, split), [split, upper)`两个区间。

### 调整边界的方法

最终的`Partition`中，对于“range分区”，一定是“左闭右开”区间。

如果用户指定的`range bounds`不是“左闭右开”区间（比如: `xxx << VALUES << YYY` 或者`VALUES == xxx`），那么需要进行转化。

有两个方法：
1. 将`lower`从 “不包含” 调整为 “包含”；
2. 将`upper`从 “包含” 调整为 “不包含”；

注意：这两个方法中，传入的`KuduPartialRow`对象，其实都是用户在创建表时所传入的对象，即属于表的“元信息”（不是具体的数据），所以在打印日志时，不会进行redact`。

#### `MakeLowerBoundRangePartitionKeyInclusive()`

用户在指定“range分区”时，没有包含“左边界”，需要转化为“包含左边界”。

说明1：为了把`lower`从 “不包含”转化为“包含”，那么一定需要 增加`lower`的现值。

```
Status PartitionSchema::MakeLowerBoundRangePartitionKeyInclusive(KuduPartialRow* row) const {
  bool increment;
  RETURN_NOT_OK(IncrementRangePartitionKey(row, &increment));

  if (!increment) {
    vector<string> components;
    AppendRangeDebugStringComponentsOrMin(*row, &components);
    return Status::InvalidArgument("Exclusive lower bound range partition key must not "
                                   "have maximum values for all components",
                                   RangeKeyDebugString(*row));
  }

  return Status::OK();
}
```

注意：如果无法增大当前的“range分区列”的值，说明所有的“range分区列”一定都已经被赋值，而且所赋的值是对应类型的最大值。这个时候，说明用户指定的区间有问题（区间内无法插入 任何值），所以向用户返回错误。

#### `MakeUpperBoundRangePartitionKeyExclusive()`

用户在指定“range分区”时，包含了“右边界”，需要转化为“不包含右边界”。

和`MakeLowerBoundRangePartitionKeyInclusive()`类似，都需要增大当前的“range分区列”，但是有两点不同：

1. 如果所有的“range分区列”都没有被赋值，那么就不需要进行任何转化（因为，在`upper`中，一个列如果没有“被赋值”，那么表示“它的值”是正无穷，即是一个不可能被取到的，这个时候 “包含”或“不包含”这个值，都是一样的）；
2. 如果当前的“range分区列”已经是最大值，那么将所有的“range分区列”都`Unset()`掉，即转化为 所有的“range分区列”都没有被赋值。（因为在`upper`中，如果一个列没有赋值，表示它的值为正无穷）。

```
Status PartitionSchema::MakeUpperBoundRangePartitionKeyExclusive(KuduPartialRow* row) const {
  bool all_unset = true;
  for (const ColumnId& column_id : range_schema_.column_ids) {
    int32_t idx = row->schema()->find_column_by_id(column_id);
    if (idx == Schema::kColumnNotFound) {
      return Status::InvalidArgument(Substitute("range partition column ID $0 "
                                                "not found in range partition key schema.",
                                                column_id));
    }
    all_unset = !row->IsColumnSet(idx);
    if (!all_unset) break;
  }

  if (all_unset) return Status::OK();

  bool increment;
  RETURN_NOT_OK(IncrementRangePartitionKey(row, &increment));
  if (!increment) {
    for (const ColumnId& column_id : range_schema_.column_ids) {
      int32_t idx = row->schema()->find_column_by_id(column_id);
      RETURN_NOT_OK(row->Unset(idx));
    }
  }

  return Status::OK();
}
```

处理逻辑：
1. 先检查是否所有的“range分区列”都已经被赋值。如果所有列都没有被赋值，那么直接返回。
2. 尝试递增 range分区列 的值：
  + 2.1 如果递增成功，那么直接返回；
  + 2.2 如果递增失败，那么就将所有的“range分区列”都`Unset()`掉。

##### `private IncrementRangePartitionKey()`

工具函数。将当前的“range分区列”作为一个整体，增大1个单位。

如果当前的值已经是最大值（这时，所有的分区列的值，都已经是最大值），那么这时是 无法再增大该“分区列”整体的值，通过参数`*increment == false`来通知上层。

在上层，有两个场景：
1. 正试图增大的是“lower bound”，那么会报错：因为用户所指定的、不被包含的`lower bound`已经是最大值，那么也就是说没有有效的值 能够在它的值域，所以报错。
2. 正试图增大的是`upper bound`，那么上层会通过将所有“range分区列”都`Unset()`掉，从而表示一个“正无穷”。

```
Status PartitionSchema::IncrementRangePartitionKey(KuduPartialRow* row, bool* increment) const {
  vector<int32_t> unset_idxs;
  *increment = false;
  for (auto itr = range_schema_.column_ids.rbegin();
       itr != range_schema_.column_ids.rend(); ++itr) {
    int32_t idx = row->schema()->find_column_by_id(*itr);
    if (idx == Schema::kColumnNotFound) {
      return Status::InvalidArgument(Substitute("range partition column ID $0 "
                                                "not found in range partition key schema.",
                                                *itr));
    }

    if (row->IsColumnSet(idx)) {
      RETURN_NOT_OK(IncrementColumn(row, idx, increment));
      if (*increment) break;
    } else {
      RETURN_NOT_OK(IncrementUnsetColumn(row, idx));
      *increment = true;
      break;
    }
    unset_idxs.push_back(idx);
  }

  if (*increment) {
    for (int32_t idx : unset_idxs) {
      RETURN_NOT_OK(row->Unset(idx));
    }
  }

  return Status::OK();
}
```

说明1：如果一列在`KuduPartialRow`中没有被赋值，那么就相当于该列的值为“相应类型的最小值”，所以它一定是可以被增加的。

说明2：在操作过程中，如果一列已经是“最大值”，那么说明该列无法再被增加，那么就只能继续再去尝试增大“前一列”的值。 而对于这个值为“最大值”的列，要进行`Unsert()`掉，这样后面的处理会把这个列看做是“最小值”。

**操作逻辑：**  
1. 对于所有的`range分区列`，按照“从右向左”的顺序，依次检查。
2. 如果1个列可以被增加，那么对其增加后，就可以结束了；
3. 如果1个列“不可以”被增加，说明当前列一定被赋过值了，并且赋的值是相应类型的最大值。那么就**取消该列的赋值(通过`KuduPartialRow::Unset(xx)`)**，然后继续检查它前面的列。

注意1：在一个`kudu`表中，“`range分区列`”的顺序，和“复合主键”中`列`的顺序，可能是不同的。这里是按照“range分区列”的顺序，从右向左操作，而不是按照“复合主键”的顺序。

**使用参数`bool increment`来标识 是否成功递增**  
如果所有“range分区列”都没有被增加（说明，所有的“range分区列”都被赋值了，并且赋的值都是相应类型的 最大值），那么`*increment = false`。否则，`*increment = true`.

**注意2：这里的`*increment`是传入参数，并不是函数的返回值，返回值类型是`Status`**。    
只有在“range分区列”中的某个列，是无效的列（在`Schema`中找不到），才会范围`Status::InvalidArgument()`。 

否则，都是返回`Status::OK()`，即使`*increment = false`。

注意3：只有在`*increment == true`的时候，才会真正进行`Unset()`操作。 即如果没有成功进行递增，是不会修改传入的`KuduPartialRow`对象的。

**示例：**    

```
  // (1, 2, 3)       -> (1, 2, 4)
  // (1, 2, 127)     -> (1, 3, _)
  // (1, 127, 3)     -> (1, 127, 4)
  // (1, _, 3)       -> (1, _, 4)
  // (_, _, _)       -> (_, _, 1)
  // (1, 127, 127)   -> (2, _, _)
  // (127, 127, 127) -> fail!
```

### `private Validate()`
检查当前的“分区方式”是否有效。

对`hash partition`中的列，检查的内容有：
1. “分桶数”要大于等于2。 因为只有一个分桶时，那么该“hash分区”没有意义。
2. 必须指定“分区列”(即“分区列”的数量必须大于1)；
3. 分区列必须是`key`列（不能是`value`列）
4. 分区列不能是重复（即，1个key列，最多只能出现在一个`hash`分区中）。
5. 所指定的分区列，必须是存在的。

对于`range partition`中的列，检查的内容有：
1. 必须是存在的列；
2. 必须是`key`列；

```
Status PartitionSchema::Validate(const Schema& schema) const {
  set<ColumnId> hash_columns;
  for (const PartitionSchema::HashBucketSchema& hash_schema : hash_bucket_schemas_) {
    if (hash_schema.num_buckets < 2) {
      return Status::InvalidArgument("must have at least two hash buckets");
    }

    if (hash_schema.column_ids.empty()) {
      return Status::InvalidArgument("must have at least one hash column");
    }

    for (const ColumnId& hash_column : hash_schema.column_ids) {
      if (!hash_columns.insert(hash_column).second) {
        return Status::InvalidArgument("hash bucket schema components must not "
                                       "contain columns in common");
      }
      int32_t column_idx = schema.find_column_by_id(hash_column);
      if (column_idx == Schema::kColumnNotFound) {
        return Status::InvalidArgument("must specify existing columns for hash "
                                       "bucket partition components");
      } else if (column_idx >= schema.num_key_columns()) {
        return Status::InvalidArgument("must specify only primary key columns for "
                                       "hash bucket partition components");
      }
    }
  }

  for (const ColumnId& column_id : range_schema_.column_ids) {
    int32_t column_idx = schema.find_column_by_id(column_id);
    if (column_idx == Schema::kColumnNotFound) {
      return Status::InvalidArgument("must specify existing columns for range "
                                     "partition component");
    } else if (column_idx >= schema.num_key_columns()) {
      return Status::InvalidArgument("must specify only primary key columns for "
                                     "range partition component");
    }
  }

  return Status::OK();
}
```

### 局部函数

这里`局部函数`是自己起的名字，表示该函数只在`.cpp`中定义，即只能在当前`cpp`文件内部可用，外部文件无法使用。

#### `ExtractColumnIds()` 
工具函数，从一个`PartitionSchemaPB_ColumnIdentifierPB`的列表中，抽取出来对应的`column_id`。

注意：在结构`PartitionSchemaPB_ColumnIdentifierPB`中，有两种可能：1) 使用`column_id`；2) 使用`column_name`。

针对这两种方式，会先通过不同的方式获取到`column id`，然后进行填充。

```
namespace {
// Extracts the column IDs from a protobuf repeated field of column identifiers.
Status ExtractColumnIds(const RepeatedPtrField<PartitionSchemaPB_ColumnIdentifierPB>& identifiers,
                        const Schema& schema,
                        vector<ColumnId>* column_ids) {
    column_ids->reserve(identifiers.size());
    for (const auto& identifier : identifiers) {
      switch (identifier.identifier_case()) {
        case PartitionSchemaPB_ColumnIdentifierPB::kId: {
          ColumnId column_id(identifier.id());
          if (schema.find_column_by_id(column_id) == Schema::kColumnNotFound) {
            return Status::InvalidArgument("unknown column id", SecureDebugString(identifier));
          }
          column_ids->push_back(column_id);
          continue;
        }
        case PartitionSchemaPB_ColumnIdentifierPB::kName: {
          int32_t column_idx = schema.find_column(identifier.name());
          if (column_idx == Schema::kColumnNotFound) {
            return Status::InvalidArgument("unknown column", SecureDebugString(identifier));
          }
          column_ids->push_back(schema.column_id(column_idx));
          continue;
        }
        default: return Status::InvalidArgument("unknown column", SecureDebugString(identifier));
      }
    }
    return Status::OK();
}
}
```

#### `SetColumnIdentifiers()`

将一个`column_id`列表，转化为一个`PartitionSchemaPB_ColumnIdentifierPB`结构的列表。

```
namespace {
// Sets a repeated field of column identifiers to the provided column IDs.
void SetColumnIdentifiers(const vector<ColumnId>& column_ids,
                          RepeatedPtrField<PartitionSchemaPB_ColumnIdentifierPB>* identifiers) {
    identifiers->Reserve(column_ids.size());
    for (const ColumnId& column_id : column_ids) {
      identifiers->Add()->set_id(column_id);
    }
}
} // namespace
```

#### `ColumnIdsToColumnNames()`

将一组`column_ids`转化为`column name`的列表。

```
namespace {
string ColumnIdsToColumnNames(const Schema& schema,
                              const vector<ColumnId>& column_ids) {
  vector<string> names;
  for (const ColumnId& column_id : column_ids) {
    names.push_back(schema.column(schema.find_column_by_id(column_id)).name());
  }

  return JoinStrings(names, ", ");
}
} // namespace
```

#### `IncrementUnsetColumn()`

因为该列在`KuduPartialRow`中**没有被赋值**，那么对这个值增加1个单位，其实就是对该类型的最小值加`1`.

```
  Status IncrementUnsetColumn(KuduPartialRow* row, int32_t idx) {
    DCHECK(!row->IsColumnSet(idx));
    const ColumnSchema& col = row->schema()->column(idx);
    switch (col.type_info()->type()) {
      case INT8:
        RETURN_NOT_OK(row->SetInt8(idx, INT8_MIN + 1));
        break;
      case INT16:
        RETURN_NOT_OK(row->SetInt16(idx, INT16_MIN + 1));
        break;
      case INT32:
        RETURN_NOT_OK(row->SetInt32(idx, INT32_MIN + 1));
        break;
      case INT64:
      case UNIXTIME_MICROS:
        RETURN_NOT_OK(row->SetInt64(idx, INT64_MIN + 1));
        break;
      case STRING:
        RETURN_NOT_OK(row->SetStringCopy(idx, Slice("\0", 1)));
        break;
      case BINARY:
        RETURN_NOT_OK(row->SetBinaryCopy(idx, Slice("\0", 1)));
        break;
      case DECIMAL32:
      case DECIMAL64:
      case DECIMAL128:
        RETURN_NOT_OK(row->SetUnscaledDecimal(idx,
            MinUnscaledDecimal(col.type_attributes().precision) + 1));
        break;
      default:
        return Status::InvalidArgument("Invalid column type in range partition",
                                       row->schema()->column(idx).ToString());
    }
    return Status::OK();
  }
```

说明：
1. 对于整型，就是相应类型的最小值 + 1。
2. 对于`STRING`和`BINARY`类型：最小值就是空字符串(`""`)，“次小值”为`"\0"`；
3. 对于`DECIMAL`类型：最小值由它的`precision`决定，就是负的 `precision`个`9`组成的十进制数。   
    （比如：`DECIMAL(6, 3)`中`precision`的值为6，所以它的最小值为`-999999`，所以“次小值”为 `-999999 + 1 = -999998`）。

注意： 这里不处理`FLOAT`和`DOUBLE`类型，它们不能作为`key`列。

说明：对于`DECIMAL`类型，既然能表示整数，也能表示小数，那么为什么“次小值”是拿“最小值”加`1`,而不是加`0.1`或`0.001`等。

这个是由该`DECIMAL`类型的`precision`和`scale`决定的，其实一旦`precision`和`scale`确定以后，这个`DECIMAL`类型所能表示的值域，就确定了（是可枚举的）。

比如: 上面`DECIMAL(6, 3)`的例子，如果是加上`0.001`，即等到的`-999999.001`，这个数的“有效数字”位数已经超过了`precision`，即用`DECIMAL(6, 3)`是无法表示它的。

而`DECIMAL(6, 3)`能表示的数中，比`-999999`大的 下一个数就是`-999998`，即加上“整型数值`1`”。

#### `IncrementColumn()`

因为该列在`KuduPartialRow`中已经被赋值，那么对这个值增加`1`个单位，就是对所赋的值 增加`1`个单位。

注意：有可能“现有值”已经是该类型的最大值，这种情况下，不进行增加，返回`*success = false`。交由上层进行处理。

```
namespace {

  Status IncrementColumn(KuduPartialRow* row, int32_t idx, bool* success) {
    DCHECK(row->IsColumnSet(idx));
    const ColumnSchema& col = row->schema()->column(idx);
    *success = true;
    switch (col.type_info()->type()) {
      case INT8: {
        int8_t value;
        RETURN_NOT_OK(row->GetInt8(idx, &value));
        if (value < INT8_MAX) {
          RETURN_NOT_OK(row->SetInt8(idx, value + 1));
        } else {
          *success = false;
        }
        break;
      }
      case INT16: {
        int16_t value;
        RETURN_NOT_OK(row->GetInt16(idx, &value));
        if (value < INT16_MAX) {
          RETURN_NOT_OK(row->SetInt16(idx, value + 1));
        } else {
          *success = false;
        }
        break;
      }
      case INT32: {
        int32_t value;
        RETURN_NOT_OK(row->GetInt32(idx, &value));
        if (value < INT32_MAX) {
          RETURN_NOT_OK(row->SetInt32(idx, value + 1));
        } else {
          *success = false;
        }
        break;
      }
      case INT64: {
         int64_t value;
         RETURN_NOT_OK(row->GetInt64(idx, &value));
         if (value < INT64_MAX) {
           RETURN_NOT_OK(row->SetInt64(idx, value + 1));
         } else {
           *success = false;
         }
         break;
       }
      case UNIXTIME_MICROS: {
        int64_t value;
        RETURN_NOT_OK(row->GetUnixTimeMicros(idx, &value));
        if (value < INT64_MAX) {
          RETURN_NOT_OK(row->SetUnixTimeMicros(idx, value + 1));
        } else {
          *success = false;
        }
        break;
      }
      case DECIMAL32:
      case DECIMAL64:
      case DECIMAL128: {
        int128_t value;
        RETURN_NOT_OK(row->GetUnscaledDecimal(idx, &value));
        if (value < MaxUnscaledDecimal(col.type_attributes().precision)) {
          RETURN_NOT_OK(row->SetUnscaledDecimal(idx, value + 1));
        } else {
          *success = false;
        }
        break;
      }
      case BINARY: {
        Slice value;
        RETURN_NOT_OK(row->GetBinary(idx, &value));
        string incremented = value.ToString();
        incremented.push_back('\0');
        RETURN_NOT_OK(row->SetBinaryCopy(idx, incremented));
        break;
      }
      case STRING: {
        Slice value;
        RETURN_NOT_OK(row->GetString(idx, &value));
        string incremented = value.ToString();
        incremented.push_back('\0');
        RETURN_NOT_OK(row->SetStringCopy(idx, incremented));
        break;
      }
      default:
        return Status::InvalidArgument("Invalid column type in range partition",
                                       row->schema()->column(idx).ToString());
    }
    return Status::OK();
  }
} // anonymous namespace

```

说明：
1. 对于整型：直接加就是相应类型的最小值 + 1。
2. 对于`STRING`和`BINARY`类型：在当前值的后面添加`\0`。
   注意从`KuduPartialRow`中获取出来的值是`Slice`类型，是不能直接修改其中所间接引用的地址的。  
   所以需要先临时创建一个`std::string`对象，然后修改之后，在通过`SetXxxCopy()`方法，去设置成新的值（这个方法会进行数据的拷贝，所以在调用此方法后，可以安全的析构掉刚刚临时创建的`std::string`对象）
3. 对于`DECIMAL`类型：直接将“现有值”加`1`。

注意： 这里不处理`FLOAT`和`DOUBLE`类型，它们不能作为`key`列。

### 与`PartitionSchemaPB`结构的相互转化

#### `static FromPB()`
根据传入的`PartitionSchemaPB`对象，构造出一个`PartitionSchema`对象。

注意：因为当前函数是静态方法(`static`方法)，所以在使用的时候，不是要先构造出一个空的`PartitionSchema`对象（记为`obj`），然后`obj->FromPB(xxx)`的方式，而应该是`PartitionSchema::FromPB(xxx, &obj)`的方式。

```
Status PartitionSchema::FromPB(const PartitionSchemaPB& pb,
                               const Schema& schema,
                               PartitionSchema* partition_schema) {
  partition_schema->Clear();

  for (const PartitionSchemaPB_HashBucketSchemaPB& hash_bucket_pb : pb.hash_bucket_schemas()) {
    HashBucketSchema hash_bucket;
    RETURN_NOT_OK(ExtractColumnIds(hash_bucket_pb.columns(), schema, &hash_bucket.column_ids));

    // Hashing is column-order dependent, so sort the column_ids to ensure that
    // hash components with the same columns hash consistently. This is
    // important when deserializing a user-supplied partition schema during
    // table creation; after that the columns should remain in sorted order.
    std::sort(hash_bucket.column_ids.begin(), hash_bucket.column_ids.end());

    hash_bucket.seed = hash_bucket_pb.seed();
    hash_bucket.num_buckets = hash_bucket_pb.num_buckets();
    partition_schema->hash_bucket_schemas_.push_back(hash_bucket);
  }

  if (pb.has_range_schema()) {
    const PartitionSchemaPB_RangeSchemaPB& range_pb = pb.range_schema();
    RETURN_NOT_OK(ExtractColumnIds(range_pb.columns(), schema,
                                   &partition_schema->range_schema_.column_ids));
  } else {
    // Fill in the default range partition (PK columns).
    // like the sorting above, this should only happen during table creation
    // while deserializing the user-provided partition schema.
    for (int32_t column_idx = 0; column_idx < schema.num_key_columns(); column_idx++) {
      partition_schema->range_schema_.column_ids.push_back(schema.column_id(column_idx));
    }
  }

  return partition_schema->Validate(schema);
}

```

**注意1：描述“分区列”时，要注意顺序。**  
+ 对于`hash分区`： 在`HashBucketSchema`对象中的`column_id`列表中，是按照`column_id`的升序排列的。（即用户指定的顺序，可能不是按照`column_id`的升序排列的，这时内部会自动按照升序进行排列。计算hash的时候，在将这些`hash column`转化为二进制的时候，也是按照这个升序的顺序）这样做的作用是：保证无论用户怎么指定，在各个地方都会按照相同的方法，来计算出`hash`的桶。    
    所以，在上述实现中，**在获取到`column_id`列表后，会对这个列表进行排序**。
+ 对于`range分区`：完全按照用户指定的`column`排序（即不会按照`column_id`的升序）


**注意2：对于没有`range分区`时，如何指定`range分区的‘分区列’`**  
如果当前`table`中没有`range分区`，那么在指定`range_schema_`中的“分区列”时，使用当前的“所有主键列”。

也就是说，在构建出来`PartitionSchema`对象以后，`hash_bucket_schemas_`和`range_schema_`成员都是有内容的（都不会是空的）。

#### `ToPB()`
将当前`PartitionSchema`对象，转化为一个`PartitionSchemaPB`对象。

```
void PartitionSchema::ToPB(PartitionSchemaPB* pb) const {
  pb->Clear();
  pb->mutable_hash_bucket_schemas()->Reserve(hash_bucket_schemas_.size());
  for (const HashBucketSchema& hash_bucket : hash_bucket_schemas_) {
    PartitionSchemaPB_HashBucketSchemaPB* hash_bucket_pb = pb->add_hash_bucket_schemas();
    SetColumnIdentifiers(hash_bucket.column_ids, hash_bucket_pb->mutable_columns());
    hash_bucket_pb->set_num_buckets(hash_bucket.num_buckets);
    hash_bucket_pb->set_seed(hash_bucket.seed);
  }

  SetColumnIdentifiers(range_schema_.column_ids, pb->mutable_range_schema()->mutable_columns());
}
```

### 编码相关的方法

#### `EncodeKey()`

给定一行数据，将它所对应的`partition key`编码到指定的`buff`中。

注意：如果该方法失败了，在`buff`中可能会有部分数据。

根据传入的`Row`类型（`KuduPartialRow`或`ConstContiguourRow`），有两个重载函数。

>> 说明：`hash bucket`就是在该层的“hash分区”中，所对应的“hash桶的序号”。

编码方法：
1. 首先编码`hash分区`：逐层地编码，对于每个`hash分区`层级，计算对应的`hash bucket`，然后编码到`buff`中；
2. 然后编码`range分区`：按照指定的`range分区列`的顺序，将这些“range分区列”的值，编码到`buff`中。

```
Status PartitionSchema::EncodeKey(const KuduPartialRow& row, string* buf) const {
  const KeyEncoder<string>& hash_encoder = GetKeyEncoder<string>(GetTypeInfo(UINT32));

  for (const HashBucketSchema& hash_bucket_schema : hash_bucket_schemas_) {
    int32_t bucket;
    RETURN_NOT_OK(BucketForRow(row, hash_bucket_schema, &bucket));
    hash_encoder.Encode(&bucket, buf);
  }

  return EncodeColumns(row, range_schema_.column_ids, buf);
}

Status PartitionSchema::EncodeKey(const ConstContiguousRow& row, string* buf) const {
  const KeyEncoder<string>& hash_encoder = GetKeyEncoder<string>(GetTypeInfo(UINT32));

  for (const HashBucketSchema& hash_bucket_schema : hash_bucket_schemas_) {
    int32_t bucket;
    RETURN_NOT_OK(BucketForRow(row, hash_bucket_schema, &bucket));
    hash_encoder.Encode(&bucket, buf);
  }

  return EncodeColumns(row, range_schema_.column_ids, buf);
}
```

>> 问题：这两个方法是一样的，实际上应该是可以通过利用模板函数，将两者合并为1个。

#### `private static EncodeColumns()`

是静态方法(`static`方法)，根据`RowType`的类型（`KuduPartialRow`和`ConstContiguousRow`）,提供了两个重载方法。

针对所提供的`Row`，以及对应的`column_id`列表，将该`Row`中这些列的值编码到对应的`buff`中。

**注意1：**  
这里`column_ids`中的所有列 都是`key`列

**注意2：为什么要提供`column_ids`参数**  
这里既然已经给定的`Row`，其实对应的`Schema`就可以知道了，然后也就知道了该`Schema`中的所有“主键列”。

之所以要提供`column_ids`，是因为这里并不是要将“全部的key列”都进行编码，可能只是编码一个“主键列”的子集。

说明：  
因为编码内容是“指定列的值”，所以这个方法，就是用来编码“range分区”的分区列，即用来编码“range partition key”的。（也就是说，其它方法`EncodeKey()`、`EncodeRangeKey()`等，都会调用该方法）

```
Status PartitionSchema::EncodeColumns(const ConstContiguousRow& row,
                                      const vector<ColumnId>& column_ids,
                                      string* buf) {
  for (int i = 0; i < column_ids.size(); i++) {
    ColumnId column_id = column_ids[i];
    int32_t column_idx = row.schema()->find_column_by_id(column_id);
    const TypeInfo* type = row.schema()->column(column_idx).type_info();
    GetKeyEncoder<string>(type).Encode(row.cell_ptr(column_idx), i + 1 == column_ids.size(), buf);
  }
  return Status::OK();
}

Status PartitionSchema::EncodeColumns(const KuduPartialRow& row,
                                      const vector<ColumnId>& column_ids,
                                      string* buf) {
  for (int i = 0; i < column_ids.size(); i++) {
    int32_t column_idx = row.schema()->find_column_by_id(column_ids[i]);
    CHECK(column_idx != Schema::kColumnNotFound);
    const TypeInfo* type_info = row.schema()->column(column_idx).type_info();
    const KeyEncoder<string>& encoder = GetKeyEncoder<string>(type_info);

    if (PREDICT_FALSE(!row.IsColumnSet(column_idx))) {
      uint8_t min_value[kLargestTypeSize];
      type_info->CopyMinValue(min_value);
      encoder.Encode(min_value, i + 1 == column_ids.size(), buf);
    } else {
      ContiguousRow cont_row(row.schema(), row.row_data_);
      encoder.Encode(cont_row.cell_ptr(column_idx), i + 1 == column_ids.size(), buf);
    }
  }
  return Status::OK();
}
```

说明1： 因为这是在编码`key`列，而`key`列是不可以为`null`的。所以在实现的时候，对于从一个`Row`中获取对应“单元格”的数据，使用的是`cell_ptr()`，而不是`nullable_cell_ptr()`。

说明2： 对于`ConstContiguousRow`来说，它的所有列都是被赋值的（注意其中部分列的值是可能为`null`的），所以这里在实现时，对于`ConstContiguourRow`类型，**不需要再检查某个列的值是否存在**（实际上，`ConstContiguourRow`也没有提供相应的方法。对于`KuduPartialRow`类型，提供了`IsColumnSet()`方法来检查）

注意1：对于`KuduPartialRow`类型，在编码的时候，如果某个`主键列`在`KuduPartialRow`中并没有被赋值，那么就将该类型的“最小值”编码进去。

>> 问题：对于`KuduPartialRow`类型，如果某个列的值没有被赋值，为什么赋予最小的值？ 另外，这里的实现，先构造的`min_value`数组长度是`kLargstTypeSize`，并且该数组并没有被清零。那么如果当前类型的实际长度是小于`kLargstTypeSize`，那么在编码的时候，实际上会编码进去多少？

#### `private EncodeRangeKey()`

**作用：**  
给定一个`Row`,将它的`range分区`的“分区列”的值，编码为`range partition key`的格式。

**使用场景：**  
用来对`splits`和`bounds`(都是`KuduPartialRow`类型)进行编码，即作为`EncodeRangeSplits()`和`EncodeRangeBounds()`方法中会调用该方法。

注意1：该方法要求，在所给的`Row`数据中，除了`range分区`列以外，其余的列都不能被赋值。

原因是由该方法的使用场景决定的。该方法在`创建表`时使用，用户提供了一组“range分区 列表”（其中每个条目都包含一个分区的上下边界），将这些“上下边界”编码成相应的`range partition key`。

用户提供的这个列表，作用只有一个，就是指定每个“range分区”的边界，所以必然只能有`range分区列`，不能有“其它列”的值。

注意2：对于一个“range分区列”，如果在所给的`Row`中没有被赋值，那么在编码中会用“该列的类型对应的最小值”进行填充。

>> 问题：为什么要用最小值来填充？ 比如`range分区列`是`A, B, C`3列，如果没有提供`B`列的值，那么如果填充了最小值，是不是构成的区间就不对了？（对于一行数据，如果`B`列的值不是最小值，那么会找不到对应的`tablet`）。

注意3：如果所给的`Row`数据中，任何列都没有被赋值，那么返回的编码结果是一个空字符串。

**说明：在该方法中，只是进行“一些检查”，真正的编码工作是通过`EncodeColumns()`来完成的。**  

检查的内容有：
1. 是否所有的"range分区列”都已经被赋值。如果都没有被赋值，那么编码结果是一个“空字符串”；
2. 是否所有的“非range分区列”都没有被赋值。 如果有被赋值，那么就返回失败。


```
Status PartitionSchema::EncodeRangeKey(const KuduPartialRow& row,
                                       const Schema& schema,
                                       string* key) const {
  DCHECK(key->empty());
  bool contains_no_columns = true;
  for (int column_idx = 0; column_idx < schema.num_columns(); column_idx++) {
    const ColumnSchema& column = schema.column(column_idx);
    if (row.IsColumnSet(column_idx)) {
      if (std::find(range_schema_.column_ids.begin(),
                    range_schema_.column_ids.end(),
                    schema.column_id(column_idx)) != range_schema_.column_ids.end()) {
        contains_no_columns = false;
      } else {
        return Status::InvalidArgument(
            "split rows may only contain values for range partitioned columns", column.name());
      }
    }
  }

  if (contains_no_columns) {
    return Status::OK();
  }
  return EncodeColumns(row, range_schema_.column_ids, key);
}
```

#### `private EncodeRangeSplits()`

将用户指定的`split points`（到这里会转化为`KuduPartialRow`），转化为相应的`range partition key`的格式，并将**按照升序**插入到结果中。

```
Status PartitionSchema::EncodeRangeSplits(const vector<KuduPartialRow>& split_rows,
                                          const Schema& schema,
                                          vector<string>* splits) const {
  DCHECK(splits->empty());
  for (const KuduPartialRow& row : split_rows) {
    string split;
    RETURN_NOT_OK(EncodeRangeKey(row, schema, &split));
    if (split.empty()) {
        return Status::InvalidArgument(
            "split rows must contain a value for at least one range partition column");
    }
    splits->emplace_back(std::move(split));
  }

  std::sort(splits->begin(), splits->end());
  auto unique_end = std::unique(splits->begin(), splits->end());
  if (unique_end != splits->end()) {
    return Status::InvalidArgument("duplicate split row", RangeKeyDebugString(*unique_end, schema));
  }
  return Status::OK();
}
```

**注意1：对于`split points`中，至少要在其中指定一个`range分区列`的值。**  
即上面在实现中，会检查`split.empty()`。

**注意2：编码之后的结果，是经过“排序”的**  
对应上面的实现中的`std::sort(...)`。

**注意3：编码之后的结果，是不允许有重复的值**  
在上面实现中，在排序之后，通过`std::unique(...)`来帮助检查是否有“重复”。

#### `private EncodeRangeBounds()`

提供一组`range bounds`，将它们编码为`partition key`的形式。然后**按照“升序”**插入到输出参数`encoded_range_bounds`中。

```

Status PartitionSchema::EncodeRangeBounds(const vector<pair<KuduPartialRow,
                                                            KuduPartialRow>>& range_bounds,
                                          const Schema& schema,
                                          vector<pair<string, string>>* range_partitions) const {
  DCHECK(range_partitions->empty());
  if (range_bounds.empty()) {
    range_partitions->emplace_back("", "");
    return Status::OK();
  }

  for (const auto& bound : range_bounds) {
    string lower;
    string upper;
    RETURN_NOT_OK(EncodeRangeKey(bound.first, schema, &lower));
    RETURN_NOT_OK(EncodeRangeKey(bound.second, schema, &upper));

    if (!lower.empty() && !upper.empty() && lower >= upper) {
      return Status::InvalidArgument(
          "range partition lower bound must be less than the upper bound",
          RangePartitionDebugString(bound.first, bound.second));
    }
    range_partitions->emplace_back(std::move(lower), std::move(upper));
  }

  // Check that the range bounds are non-overlapping
  std::sort(range_partitions->begin(), range_partitions->end());
  for (int i = 0; i < range_partitions->size() - 1; i++) {
    const string& first_upper = range_partitions->at(i).second;
    const string& second_lower = range_partitions->at(i + 1).first;

    if (first_upper.empty() || second_lower.empty() || first_upper > second_lower) {
      return Status::InvalidArgument(
          "overlapping range partitions",
          strings::Substitute("first range partition: $0, second range partition: $1",
                              RangePartitionDebugString(range_partitions->at(i).first,
                                                        range_partitions->at(i).second,
                                                        schema),
                              RangePartitionDebugString(range_partitions->at(i + 1).first,
                                                        range_partitions->at(i + 1).second,
                                                        schema)));
    }
  }

  return Status::OK();
}
```

**注意1： 在每组`range bound`，如果显式指定的`lower`和`upper`，那么一定要满足`lower < upper`，否则报错。**  
如果没有显式指定的`lower`或`upper`，那边表示对应的边界没限制，即“负无穷”或“正无穷”。

**注意2：返回结果是有序的。**  
对应上面实现中的`std::sort(...)`

**注意3：在多组`range bound`之间，不能有重叠。**  
上面的实现中，在排序以后，逐个 检查 “当前range的上界”与“下一个range的上界”进行比较，来判断是否有重合。  

**注意4：在判断多个`range bound`之间是否有重叠时，要注意“边界值”有可能是空**  
对应上述实现中，在每次比较的时，如果发现`first_upper.empty() || second_lower.empty()`，那么一定是有重复了。  

因为边界为`empty()`，实际上表示的就是“正负无穷”的含义。   
比如`first_upper.empty()`，说明前一个区间的“最大范围”已经到了“正无穷”，所以无论右边的区间的`lower`为多少，都是有重复的。

### 解码相关方法

#### `DecodeRangeKey()`

给定一个已经对“range分区列”进行“编过码”的结果，进行解码，放入到`KuduPartialRow`中。

```
Status PartitionSchema::DecodeRangeKey(Slice* encoded_key,
                                       KuduPartialRow* row,
                                       Arena* arena) const {
  ContiguousRow cont_row(row->schema(), row->row_data_);
  for (int i = 0; i < range_schema_.column_ids.size(); i++) {

    if (encoded_key->empty()) {
      // This can happen when decoding partition start and end keys, since they
      // are truncated to simulate absolute upper and lower bounds.
      continue;
    }

    int32_t column_idx = row->schema()->find_column_by_id(range_schema_.column_ids[i]);
    const ColumnSchema& column = row->schema()->column(column_idx);
    const KeyEncoder<faststring>& key_encoder = GetKeyEncoder<faststring>(column.type_info());
    bool is_last = i == (range_schema_.column_ids.size() - 1);

    // Decode the column.
    RETURN_NOT_OK_PREPEND(key_encoder.Decode(encoded_key,
                                             is_last,
                                             arena,
                                             cont_row.mutable_cell_ptr(column_idx)),
                          Substitute("Error decoding partition key range component '$0'",
                                     column.name()));
    // Mark the column as set.
    BitmapSet(row->isset_bitmap_, column_idx);
  }
  if (!encoded_key->empty()) {
    return Status::InvalidArgument("unable to fully decode range key",
                                   KUDU_REDACT(encoded_key->ToDebugString()));
  }
  return Status::OK();
}
```

注意：如果传入的`encoded_key`是给定`Partition`对象的`partition_start_key_`或`partition_end_key_`，因为这其中的部分列，在进行“填充洞”的时候，可能被`truncate`掉了，所以在解码的过程中，可能不会把所有的“range分区列”都赋值。

### 计算`hash bucket`
#### `private static BucketForRow()`

工具函数。

给定一个`Row`和对应的`HashBucketSchema`，计算出该行数据对应的`hash bucket`。

```
template<typename Row>
Status PartitionSchema::BucketForRow(const Row& row,
                                     const HashBucketSchema& hash_bucket_schema,
                                     int32_t* bucket) {
  string buf;
  RETURN_NOT_OK(EncodeColumns(row, hash_bucket_schema.column_ids, &buf));
  uint64_t hash = HashUtil::MurmurHash2_64(buf.data(), buf.length(), hash_bucket_schema.seed);
  *bucket = hash % static_cast<uint64_t>(hash_bucket_schema.num_buckets);
  return Status::OK();
}

//------------------------------------------------------------
// Template instantiations: We instantiate all possible templates to avoid linker issues.
// see: https://isocpp.org/wiki/faq/templates#separate-template-fn-defn-from-decl
//------------------------------------------------------------
template
Status PartitionSchema::BucketForRow(const KuduPartialRow& row,
                                     const HashBucketSchema& hash_bucket_schema,
                                     int32_t* bucket);
template
Status PartitionSchema::BucketForRow(const ConstContiguousRow& row,
                                     const HashBucketSchema& hash_bucket_schema,
                                     int32_t* bucket);
```

#### `private static BucketForEncodedColumns()`

给定一个已经编码过的`值`，以及对应的`HashBucketSchema`，计算出“该行数据”对应的`hash bucket`。

与`BucketForRow()`相比，区别在于是否需要计算下`hash bucket`，在`BucketForRow()`方法中，是需要计算一次的。

```
int32_t PartitionSchema::BucketForEncodedColumns(const string& encoded_key,
                                                 const HashBucketSchema& hash_bucket_schema) {
  uint64_t hash = HashUtil::MurmurHash2_64(encoded_key.data(),
                                           encoded_key.length(),
                                           hash_bucket_schema.seed);
  return hash % static_cast<uint64_t>(hash_bucket_schema.num_buckets);
}
```

### 打印调试信息

#### `PartitionDebugString()`

给定一个`Partition`对象，打印出相应的`debug`信息。

一个打印结果的示例如下：
```
HASH (a) PARTITION 0, HASH (b) PARTITION 1, RANGE (a, b, c) PARTITION ("a1", "b1", "c1") <= VALUES < ("a2", "b2", "")
```

```
string PartitionSchema::PartitionDebugString(const Partition& partition,
                                             const Schema& schema) const {
  // Partitions are considered metadata, so don't redact them.
  ScopedDisableRedaction no_redaction;

  vector<string> components;
  if (partition.hash_buckets_.size() != hash_bucket_schemas_.size()) {
    return "<hash-partition-error>";
  }

  for (int i = 0; i < hash_bucket_schemas_.size(); i++) {
    string s = Substitute("HASH ($0) PARTITION $1",
                          ColumnIdsToColumnNames(schema, hash_bucket_schemas_[i].column_ids),
                          partition.hash_buckets_[i]);
    components.emplace_back(std::move(s));
  }

  if (!range_schema_.column_ids.empty()) {
    string s = Substitute("RANGE ($0) PARTITION $1",
                          ColumnIdsToColumnNames(schema, range_schema_.column_ids),
                          RangePartitionDebugString(partition.range_key_start(),
                                                    partition.range_key_end(),
                                                    schema));
    components.emplace_back(std::move(s));
  }

  return JoinStrings(components, ", ");
}
```

说明：`partition key`信息被认为是“元数据”，所以在打印的时候，不进行`redact`。（因为只有“用户的数据”，才需要进行`redact`）。

#### `PartitionKeyDebugString()`

给定一个`Row`对象（或者 “复合主键”的编码结果），返回一个`partition key`的字符串形式的`debug`信息。

根据参数的传入类型，提供3个重载方法。

一个输出示例为：
```
HASH (a, b): 0, HASH (c): 20, RANGE (a, b, c): (0, "", "")
```

```
string PartitionSchema::PartitionKeyDebugString(const ConstContiguousRow& row) const {
  return PartitionKeyDebugStringImpl(row);
}

string PartitionSchema::PartitionKeyDebugString(const KuduPartialRow& row) const {
  return PartitionKeyDebugStringImpl(row);
}

string PartitionSchema::PartitionKeyDebugString(Slice key, const Schema& schema) const {
  vector<string> components;

  size_t hash_components_size = kEncodedBucketSize * hash_bucket_schemas_.size();
  if (key.size() < hash_components_size) {
    return "<hash-decode-error>";
  }

  for (const auto& hash_schema : hash_bucket_schemas_) {
    uint32_t big_endian;
    memcpy(&big_endian, key.data(), sizeof(uint32_t));
    key.remove_prefix(sizeof(uint32_t));
    components.emplace_back(
        Substitute("HASH ($0): $1",
                    ColumnIdsToColumnNames(schema, hash_schema.column_ids),
                    BigEndian::ToHost32(big_endian)));
  }

  if (!range_schema_.column_ids.empty()) {
      components.emplace_back(
          Substitute("RANGE ($0): $1",
                     ColumnIdsToColumnNames(schema, range_schema_.column_ids),
                     RangeKeyDebugString(key, schema)));
  }

  return JoinStrings(components, ", ");
}

```

说明：因为每行数据，在编码时，首部都是对应的`hash分区`的编码结果，每个层级会编码进一个`hash bucket`(是`int32_t`类型)，所以“复合主键”的编码结果中个，长度一定是要大于"`sizeof(int32_t) * hash分区的层数`"。

##### `private PartitionKeyDebugStringImpl()`

```
template<typename Row>
string PartitionSchema::PartitionKeyDebugStringImpl(const Row& row) const {
  vector<string> components;

  for (const HashBucketSchema& hash_bucket_schema : hash_bucket_schemas_) {
    int32_t bucket;
    Status s = BucketForRow(row, hash_bucket_schema, &bucket);
    if (s.ok()) {
      components.emplace_back(
          Substitute("HASH ($0): $1",
                     ColumnIdsToColumnNames(*row.schema(), hash_bucket_schema.column_ids),
                     bucket));
    } else {
      components.emplace_back(Substitute("<hash-error: $0>", s.ToString()));
    }
  }

  if (!range_schema_.column_ids.empty()) {
      components.emplace_back(
          Substitute("RANGE ($0): $1",
                     ColumnIdsToColumnNames(*row.schema(), range_schema_.column_ids),
                     RangeKeyDebugString(row)));
  }

  return JoinStrings(components, ", ");
}

template
string PartitionSchema::PartitionKeyDebugStringImpl(const KuduPartialRow& row) const;
template
string PartitionSchema::PartitionKeyDebugStringImpl(const ConstContiguousRow& row) const;

```

#### `RangePartitionDebugString()`

给定两个`Row`对象（分别表示 一个`Partition`的“上限”和“下限”），返回相应的`字符串`形式，用于`debug`。

指定`Row`数据有两种方式（`KuduPartialRow`和`Slice`），提供2个重载方法。

注意1：因为`Partition`信息属于元数据，所以在输出的debug字符串结果中，并不进行`redact`。

注意2：在用`KuduPartialRow`来指定`lower_bound`和`upper_bound`时，可能是无效的，即在这个`Row`中，`range分区`的“分区列”没有被赋值。

注意3：如果`lower_bound`和`upper_bound`都是有效的，那么组成一个“左闭右开”区间。

```
string PartitionSchema::RangePartitionDebugString(const KuduPartialRow& lower_bound,
                                                  const KuduPartialRow& upper_bound) const {
  // Partitions are considered metadata, so don't redact them.
  ScopedDisableRedaction no_redaction;

  bool lower_unbounded = IsRangePartitionKeyEmpty(lower_bound);
  bool upper_unbounded = IsRangePartitionKeyEmpty(upper_bound);
  if (lower_unbounded && upper_unbounded) {
    return "UNBOUNDED";
  }
  if (lower_unbounded) {
    return Substitute("VALUES < $0", RangeKeyDebugString(upper_bound));
  }
  if (upper_unbounded) {
    return Substitute("VALUES >= $0", RangeKeyDebugString(lower_bound));
  }
  // TODO(dan): recognize when a simplified 'VALUE =' form can be used (see
  // org.apache.kudu.client.Partition#formatRangePartition).
  return Substitute("$0 <= VALUES < $1",
                    RangeKeyDebugString(lower_bound),
                    RangeKeyDebugString(upper_bound));
}

string PartitionSchema::RangePartitionDebugString(Slice lower_bound,
                                                  Slice upper_bound,
                                                  const Schema& schema) const {
  // Partitions are considered metadata, so don't redact them.
  ScopedDisableRedaction no_redaction;

  Arena arena(256);
  KuduPartialRow lower(&schema);
  KuduPartialRow upper(&schema);

  Status s = DecodeRangeKey(&lower_bound, &lower, &arena);
  if (!s.ok()) {
    return Substitute("<range-key-decode-error: $0>", s.ToString());
  }
  s = DecodeRangeKey(&upper_bound, &upper, &arena);
  if (!s.ok()) {
    return Substitute("<range-key-decode-error: $0>", s.ToString());
  }

  return RangePartitionDebugString(lower, upper);
}
```

##### `private IsRangePartitionKeyEmpty()`
工具函数: 给定一个`Row`对象，检查`range分区`的“分区列”，是否都已经被赋过值。

检查方法：对于每个`range分区`的“分区列”，通过`KuduPartialRow::IsColumnSet()`方法进行检查。

```
bool PartitionSchema::IsRangePartitionKeyEmpty(const KuduPartialRow& row) const {
  ConstContiguousRow const_row(row.schema(), row.row_data_);
  for (const ColumnId& column_id : range_schema_.column_ids) {
    if (row.IsColumnSet(row.schema()->find_column_by_id(column_id))) return false;
  }
  return true;
}
```

##### `private RangeKeyDebugString()`

给定一个Row，获取一个用于debug的字符串描述

```
string PartitionSchema::RangeKeyDebugString(Slice range_key, const Schema& schema) const {
  Arena arena(256);
  KuduPartialRow row(&schema);

  Status s = DecodeRangeKey(&range_key, &row, &arena);
  if (!s.ok()) {
    return Substitute("<range-key-decode-error: $0>", s.ToString());
  }
  return RangeKeyDebugString(row);
}

string PartitionSchema::RangeKeyDebugString(const KuduPartialRow& key) const {
  vector<string> components;
  AppendRangeDebugStringComponentsOrMin(key, &components);
  if (components.size() == 1) {
    // Omit the parentheses if the range partition has a single column.
    return components.back();
  }
  return Substitute("($0)", JoinStrings(components, ", "));
}

string PartitionSchema::RangeKeyDebugString(const ConstContiguousRow& key) const {
  vector<string> components;

  for (const ColumnId& column_id : range_schema_.column_ids) {
    string column;
    int32_t column_idx = key.schema()->find_column_by_id(column_id);
    if (column_idx == Schema::kColumnNotFound) {
      components.emplace_back("<unknown-column>");
      break;
    }
    key.schema()->column(column_idx).DebugCellAppend(key.cell(column_idx), &column);
    components.push_back(column);
  }

  if (components.size() == 1) {
    // Omit the parentheses if the range partition has a single column.
    return components.back();
  }
  return Substitute("($0)", JoinStrings(components, ", "));
}

```

##### `private AppendRangeDebugStringComponentsOrMin()`
将`range分区`的所有“分区列”的值，转化为“字符串”形式(通过该列类型的`AppendDebugStringForValue()`方法)，添加到`components`参数中

如果一个`range分区`的“分区列”，在`KuduPartialRow`中没有被赋值，那么就填充“该列类型”所对应的最小值。（比如`INT32`的最小值就是`int32_t:MIN`）

>> 问题：为什么要添加“最小的值”？

```
void PartitionSchema::AppendRangeDebugStringComponentsOrMin(const KuduPartialRow& row,
                                                            vector<string>* components) const {
  ConstContiguousRow const_row(row.schema(), row.row_data_);

  for (const ColumnId& column_id : range_schema_.column_ids) {
    int32_t column_idx = row.schema()->find_column_by_id(column_id);
    if (column_idx == Schema::kColumnNotFound) {
      components->emplace_back("<unknown-column>");
      continue;
    }
    const ColumnSchema& column_schema = row.schema()->column(column_idx);

    if (!row.IsColumnSet(column_idx)) {
      uint8_t min_value[kLargestTypeSize];
      column_schema.type_info()->CopyMinValue(&min_value);
      SimpleConstCell cell(&column_schema, &min_value);
      components->emplace_back(column_schema.Stringify(cell.ptr()));
    } else {
      components->emplace_back(column_schema.Stringify(const_row.cell_ptr(column_idx)));
    }
  }
}
```
#### `DebugString()`

将当前`PartitionSchema`中的所有分区信息（包括“hash分区”和“range分区”），打印成一个字符串，每个分区层级之间使用逗号(`','`)连接起来。

这个方法的结果，会被打印到日志中。所以它是将“所有的分区信息”打印在“一行”中（即没有换行符）。

一个示例如下：
```
  HASH (a, b) PARTITIONS 32, HASH (c) PARTITIONS 32 SEED 42, RANGE (a, b, c)
```

```
string PartitionSchema::DebugString(const Schema& schema) const {
  return JoinStrings(DebugStringComponents(schema), ", ");
}
```

#### `DisplayString()`

和`DebugString()`类似的功能。

`DisplayString()` VS `DebugString()`：  
1. `DisplayString()`会将所有的分区信息（包括“hash分区”和“range分区”）,打印成适合在`WEB UI`中显示的字符串。每个分区方式占一行（有换行符）。
2. 如果存在`range`分区，那么`DisplayString()`还会将所有`Partition`的`partition key`的边界也打印出来。

```
string PartitionSchema::DisplayString(const Schema& schema,
                                      const vector<string>& range_partitions) const {
  string display_string = JoinStrings(DebugStringComponents(schema), ",\n");

  if (!range_schema_.column_ids.empty()) {
    display_string.append(" (");
    if (range_partitions.empty()) {
      display_string.append(")");
    } else {
      bool is_first = true;
      for (const string& range_partition : range_partitions) {
        if (is_first) {
          is_first = false;
        } else {
          display_string.push_back(',');
        }
        display_string.append("\n    PARTITION ");
        display_string.append(range_partition);
      }
      display_string.append("\n)");
    }
  }
  return display_string;
}
```

##### `private DebugStringComponents()`

```
vector<string> PartitionSchema::DebugStringComponents(const Schema& schema) const {
  vector<string> components;

  for (const auto& hash_bucket_schema : hash_bucket_schemas_) {
    string s;
    SubstituteAndAppend(&s, "HASH ($0) PARTITIONS $1",
                        ColumnIdsToColumnNames(schema, hash_bucket_schema.column_ids),
                        hash_bucket_schema.num_buckets);
    if (hash_bucket_schema.seed != 0) {
      SubstituteAndAppend(&s, " SEED $0", hash_bucket_schema.seed);
    }
    components.emplace_back(std::move(s));
  }

  if (!range_schema_.column_ids.empty()) {
    string s = Substitute("RANGE ($0)", ColumnIdsToColumnNames(schema, range_schema_.column_ids));
    components.emplace_back(std::move(s));
  }

  return components;
}
```

### `PartitionContainsRow()` -- 判断一行数据是否在给定的`partition`中

给定一个`Partition`和一个`Row`对象，检查这个`Row`是否属于所给的`Partition`。

根据`Row`的类型，分为两个重载函数。

```
Status PartitionSchema::PartitionContainsRow(const Partition& partition,
                                             const KuduPartialRow& row,
                                             bool* contains) const {
  return PartitionContainsRowImpl(partition, row, contains);
}

Status PartitionSchema::PartitionContainsRow(const Partition& partition,
                                             const ConstContiguousRow& row,
                                             bool* contains) const {
  return PartitionContainsRowImpl(partition, row, contains);
}
```

#### `PartitionContainsRowImpl()`

具体的实现方法。

检查方法是：
1. 针对该`Row`，计算在所有层级的`hash bucket`，要和`Partition`对象中的所有`hash bucket`数值 都相等。
2. 对于`range`分区，要求该行数据的`range partition key`部分，要同时满足如下两个条件：
    + 大于或等于 `partition.range_key_start()`;
    + 小于 `partition.range_key_end()`，或者 `partition.range_key_end()`为空。

注意1：一个`Partition`的`range partition key`部分，是不包含相应的`hash bucket`的（只包含`range 分区`对应的部分）。

说明：`partition.range_key_end()`表示没有“上边界”，即取值范围是“正无穷”。

```
template<typename Row>
Status PartitionSchema::PartitionContainsRowImpl(const Partition& partition,
                                                 const Row& row,
                                                 bool* contains) const {
  CHECK_EQ(partition.hash_buckets().size(), hash_bucket_schemas_.size());
  for (int i = 0; i < hash_bucket_schemas_.size(); i++) {
    const HashBucketSchema& hash_bucket_schema = hash_bucket_schemas_[i];
    int32_t bucket;
    RETURN_NOT_OK(BucketForRow(row, hash_bucket_schema, &bucket));

    if (bucket != partition.hash_buckets()[i]) {
      *contains = false;
      return Status::OK();
    }
  }

  string range_partition_key;
  RETURN_NOT_OK(EncodeColumns(row, range_schema_.column_ids, &range_partition_key));

  // If all of the hash buckets match, then the row is contained in the
  // partition if the row is gte the lower bound; and if there is no upper
  // bound, or the row is lt the upper bound.
  *contains = (Slice(range_partition_key).compare(partition.range_key_start()) >= 0)
           && (partition.range_key_end().empty()
                || Slice(range_partition_key).compare(partition.range_key_end()) < 0);

  return Status::OK();
}
```










