[TOC]

## `struct RowIteratorOptions`

用来封装传递一个“行迭代器”的所有参数。

### 成员

#### `projection`

类型是`Schema*`，当前迭代器所需要读取的列集合。是当前表所有列的一个子集。

>> 注意：这种“子集”关系，称为是一个“投影”。

默认值为`nullptr`。

#### `snap_to_include`

当前读取所对应的“快照”。

对于一条记录，如果该“快照”认为是“未提交”的，那么这条记录就会被忽略。

默认值就是当前时间，包含所有的事务。

#### `snap_to_exclude`

一个快照。对于一条数据，如果该“快照”认为是“已提交”的，那么这条记录会被忽略。

类型是`boost::optional`。 

默认值为`boost::none`。

>> 当然，如果该“快照”认为是“未提交”的，那么也是被忽略。

>> 说明：结合使用`snap_to_exclude`和`snap_to_include`，那么就可以查询在某个时间段内的修改。

```
例如：
---------------+-------------------+--------------->
               ^                   ^
               |                   |
        snap_to_exclude      snap_to_include
        
如上图，如果一个查询，同时指定了`snap_to_exclude`和`snap_to_include`，那么这个迭代器，只会查询这两个快照之间的修改。
```

#### `order`

类型为`OrderMode`（参见`src/kudu/common/common.proto`），表示结果是否需要按照“主键”进行排序。

默认值是`UNORDERED`

注意：`kudu`只会针对“主键”进行排序。 所以只要那些“投影”中包含主键的查询，这里才能要求进行排序（被设置成`ORDERED`）。
#### `io_context`

类型为`IOContext`（参见 `src/kudu/fs/io_context.h`）。

默认值是`nullptr`

#### `include_deleted_rows`

是否在结果中包含那些 被删除的（最后一个修改为删除的）行。

默认值：`false`。即不包含。

```
// Encapsulates all options passed to row-based Iterators.
struct RowIteratorOptions {
  RowIteratorOptions();

  const Schema* projection;

  MvccSnapshot snap_to_include;

  boost::optional<MvccSnapshot> snap_to_exclude;

  OrderMode order;

  const fs::IOContext* io_context;

  bool include_deleted_rows;
};
```

## `struct ProbeStats`

`ProbeStats类`用来记录，每个 行操作 过程中的统计信息("statistics")，主要是‘各种相关的数据结构’被访问的次数。

包括
1. 访问的`bloom-filter`个数；
2. 访问的`cfile`个数；
3. 访问的`delta file`个数；
4. 访问的`MemRowSet`个数；

这些统计信息的使用场景：
1. 这些统计信息，最终会汇总到`tablet`的整体`metric`信息中。
2. 对于开启了`tracing`的RPC，这些信息会在RPC中传递，从而帮助跟踪一个RPC的性能。

```
struct ProbeStats {
  ProbeStats()
    : blooms_consulted(0),
      keys_consulted(0),
      deltas_consulted(0),
      mrs_consulted(0) {
  }

  int blooms_consulted;
  int keys_consulted;
  int deltas_consulted;
  // Incremented for each MemRowSet consulted.
  int mrs_consulted;
};
```

## `class RowSetKeyProbe`

用来缓存一个已经编码过的key，以及对应的hash过的key，用来探测rowset。



## `class RowSet`

这是一个“纯接口类”，全部都是接口，没有属性。 

**注意：几乎全部接口都是返回`Status`，表示是否成功执行**。

**目前有2个子类：`MemRowSet`和`DiskRowSet`**。

### `enum RowSet::DeltaCompactionType`

两种`compaction`类型。 详细介绍可以参见 `tablet.md`。

```
enum DeltaCompactionType {
    MAJOR_DELTA_COMPACTION,
    MINOR_DELTA_COMPACTION
  };
```
### 读写操作接口
#### `CheckRowPresent()`

检查一行数据，在当前`RowSet`中是否存在。

注意：如果一行数据曾经存在，但是已经被删除了(`DELETE`)，那么应该返回“不存在”(即将参数`present`设置为`false`)，就像它从来没有存在过一样。

#### `MutateRow()`

修改（更新/删除）该`RowSet`中的一行数据。

如果当前行在该`RowSet`中不存在，那么返回`Status::NotFound()`。

```
  virtual Status MutateRow(Timestamp timestamp,
                           const RowSetKeyProbe &probe,
                           const RowChangeList &update,
                           const consensus::OpId& op_id,
                           const fs::IOContext* io_context,
                           ProbeStats* stats,
                           OperationResultPB* result) = 0;
```

#### `NewRowIterator()`

返回一个针对当前`RowSet`的迭代器。

注意：该迭代器的各种配置选项“`RowIteratorOptions`对象”，在该迭代器的生命周期内，必须是有效的。

注意：返回的迭代器对象，是没有被“初始化”的。即调用者，仍然需要再调用`Init()`方法。

返回的迭代器类型是`RowwiseIterator`类型，在参数中返回。

```
  virtual Status NewRowIterator(const RowIteratorOptions& opts,
                                std::unique_ptr<RowwiseIterator>* out) const = 0;
```

#### `NewRowIteratorWithBounds()`

和`NewRowIterator()`类似，返回的对象中，是包含了该`RowSet`的上下边界的。

返回的迭代器类型是`IterWithBounds`类型，在参数中返回。

>> 问题： 和`NewRowIterator()`的区别？ 分别在什么场景下使用？

```
  Status NewRowIteratorWithBounds(const RowIteratorOptions& opts,
                                  IterWithBounds* out) const;
```

### 获取信息的接口
#### `CountRows()`
统计当前`RowSet`的行数。

#### `CountLiveRows()`
统计当前`RowSet`中有多少“活着的”数据行数。 即去掉那些“被删除”的数据行。

#### `GetBounds()`
获取当前`RowSet`的上下边界。

结果分别保存在两个参数中：`min_encoded_key`和`max_encoded_key`。

注意：对于那些还处于“可修改”状态的RowSet（比如：`MemRowSet`），这个方法返回`Status::NotImplemented`。

>> 说明：因为`MemRowSet`还可以修改，所以准确的上下边界 还不能确定。

#### `OnDiskSize()`
当前`RowSet`在磁盘上的大小，单位是“字节”。

#### `OnDiskBaseDataSize()`
当前`RowSet`的`base data`部分，占用的磁盘大小。单位是“字节”。

不包括： `bloomfiles`和`ad hoc index`。

#### `OnDiskBaseDataSizeWithRedos()`
当前`RowSet`中，`base data`和`REDO deltas`共占用的磁盘空间。单位是“字节”。

不包括： `bloomfiles`、`ad hoc index`、`UNDO deltas`。

#### `OnDiskBaseDataColumnSize()`
当前`RowSet`中，所有 `column` 共占用的磁盘空间。单位是“字节”。

>> 问题：画个图，说清楚这几个方法的区别？每个方法都包含哪些内容。

#### `metadata()`
获取该`RowSet`的元信息。

#### `DeltaMemStoreSize()`
当前`RowSet`中，所有 `delta memstore`的大小。

#### `DeltaMemStoreEmpty()`

#### `MinUnflushedLogIndex()`
该`RowSet`中，尚未flush的条目，对应的最小的`raft log index`。

### 和`compaction`相关的接口

#### `compact_flush_lock()`

在做`compaction`或`flush`的时候，用来保护某个 tablet的锁。

用来为了防止：在同一个rowset上，同时进行多个`compaction`和`flush`操作。

#### `IsAvailableForCompaction()`
检查一个`RowSet`是否可以用于`compaction`。

注意：当前版本的`kudu`中，`MemRowSet`是不支持`compaction`的，即调用`MemRowSet::IsAvailableForCompaction()`的返回值永远为`false`。

注意：该方法提供了在`RowSet`中提供了实现。

```
  // Return true if this RowSet is available for compaction, based on
  // the current state of the compact_flush_lock. This should only be
  // used under the Tablet's compaction selection lock, or else the
  // lock status may change at any point.
  virtual bool IsAvailableForCompaction() {
    // Try to obtain the lock. If we don't succeed, it means the rowset
    // was already locked for compaction by some other compactor thread,
    // or it is a RowSet type which can't be used as a compaction input.
    //
    // We can be sure that our check here will remain true until after
    // the compaction selection has finished because only one thread
    // makes compaction selection at a time on a given Tablet due to
    // Tablet::compact_select_lock_.
    std::unique_lock<std::mutex> try_lock(*compact_flush_lock(), std::try_to_lock);
    return try_lock.owns_lock() && !has_been_compacted();
  }
```

注意：  
调用这个方法前，一定要已经 拥有"Tablet's compaction selection lock"（参见`Tablet::compact_select_lock_`）。

正是因为多个线程通过`Tablet::compact_select_lock_`锁进行互斥，所以 每次在选择`RowSet`去进行`compaction`的整个“选择过程”，只会有一个线程去检查该`RowSet`。

其实，只需要一个`*compact_flush_lock()`，就可以保证最多只会有一个线程同时去对一个`RowSet`进行`compaction`操作。 

那为什么还要先获取`Tablet::compact_select_lock_`，才能去获取`*compact_flush_lock()`。 实际上是为了防止多个线程之间死锁。 比如说，有两个`RowSet`(rowset_1和rowset2)，两个线程都希望对它们两个进行合并，但是如果没有`Tablet::compact_select_lock_`,那么可能出现死锁，即
+ `thread_1`获取的`rowset_1`的`compaction_flush_lock`，然后等待去获取`rowset_2`的`compaction_flush_lock`；
+ `thread_2`获取的`rowset_2`的`compaction_flush_lock`，然后等待去获取`rowset_1`的`compaction_flush_lock`；

这样两个线程，都无法成功的执行。

但是，如果两个线程在检查之前，先要申请`Tablet::compact_select_lock_`，就可以保证获取到`Tablet::compact_select_lock_`的线程，可以申请到所有需要参与`compaction`的`rowset`的锁。

说明1：返回`false`有两种情况：
1. 加锁失败。说明这时已经有其它线程拥有锁，即其它线程正在对当前`RowSet`进行`compaction`，所以当前的线程就不需要对它再做`compaction`。
2. 该`RowSet`已经做过`compaction`了，所以也不需要对该`RowSet`重复做`compaction`了。

#### `has_been_compacted()`
检查当前`RowSet`是否已经进行过了`compaction`。

在需要对该`RowSet`进行`compaction`的时候，会检查这个值，如果已经进行过，那么就不能再对它重复进行`compaction`。

#### `set_has_been_compacted()`
设置`has_been_compacted_`的值为`true`，表示当前`RowSet`已经进行过了`Compaction`。

一个`RowSet`，一旦经过`compaction`，那么就会从`rowset tree`中移除，后续的`compaction`就不会再考虑当前`RowSet`。

### 后台操作(`compaction`和`flush`)相关的操作
#### `NewCompactionInput()`

为compaction作业创建一个输入(`CompactionInput`对象)，在参数中返回。

参数中的`projection`对象，就是在“输出”中要使用的`Schema`，即所有的输入，都会在它上面进行投影。

```
  virtual Status NewCompactionInput(const Schema* projection,
                                    const MvccSnapshot &snap,
                                    const fs::IOContext* io_context,
                                    gscoped_ptr<CompactionInput>* out) const = 0;
```

### 调试打印相关接口

#### `DebugDump()`
将整个`RowSet`的数据都打印出来，仅用于调试。

#### `ToString()`
打印一个字符串，表示当前的`RowSet`;

当前的实现中，
1. `MemRowSet::ToString()`就简单的打印`"memrowset"`；
2. `DiskRowSet::ToString()`打印的是 `rowset_metadata_->ToString()`;




### `DeltaStoresCompactionPerfImprovementScore()`

计算下，如果进行`minor`或`major delta compaction`,会带来的性能收益。

结果区间是`[0, 1]`（闭区间）

### `FlushDeltas()`

对`delta memstore`进行flush操作。

### `MinorCompactDeltaStores()`

进行`minor compaction`。

### `EstimateBytesInPotentiallyAncientUndoDeltas()`

预计在历史的`UNDO delta store`中数据的大小。

注意：这个值可能是偏大的。

参数`ancient_history_mark`必须是有效的（即不可以是`Timestamp::kInvalidTimestamp`）。

### `InitUndoDeltas()`

初始化`undo delta blocks`。

该方法的结束条件是：
1. 超时：（超过`deadline`）；
2. 所有的 “最大时间戳” 大于 `ancient_history_mark` 的 `undo delta blocks` 都被初始化完成。

调用该方法，也是可以提升`EstimateBytesInPotentiallyAncientUndoDeltas()`的预测结果。

如果返回成功，那么：
1. 在参数`delta_blocks_initialized`中返回实际初始化的block数量。  
2. 在参数`bytes_in_ancient_undos`中返回可以被释放的数据大小。

如果`ancient_history_mark`的值为`Timestamp::kInvalidTimestamp`，那么正在初始化的`block`的“最大时间戳”会被忽略，并且没有任何“基于时间戳的短路”。

如果`deadline`是没有被初始化的，那么久不会强制 deadline。

注意：参数`delta_blocks_initialized` and `bytes_in_ancient_undos`可能会为`nullptr`。

### `DeleteAncientUndoDeltas()`

对于已经初始化的`undo delta blocks`，如果它们的“最大时间戳”是早于`ancient_history_mark`的，那么就删除掉。

>> 说明：如果“最大的时间戳”都是 早于`ancient_history_mark`，那么说明所有的数据上的时间戳都是小于ancient_history_mark`的。  
>> 注意：对于那些在block中，只有部分数据的时间戳是大于`ancient_history_mark`的，是不能删除的。  

注意1：这个方法不会对`rowset metadata`进行`flush`。 如果这个方法返回`OK`，那么调用者需要手动调用`flush`，从而保证这些修改会被持久化到`rowset metadata`中。

注意2: 在`rowset metadata`被flush之前，这些`blocks`是不会被删除的。 因为`RowSet metadata`的flush过程，会调用`tablet metadata`的flush，在其中会 迭代 和 删除 在孤儿列表中的blocks。

如果这个方法返回OK，那么：
1. 在参数`blocks_deleted`返回被删除的block数量。
2. 在参数`bytes_deleted`中返回被删除的数据大小。

注意：`blocks_deleted`和`bytes_deleted`可能会为`nullptr`。




