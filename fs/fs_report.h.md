[TOC]

文件：`src/kudu/fs/fs_report.h`

说明：该文件显示定义了几种错误类型：  
1. 每种类型都会有一个嵌套的内部类(`Entry`)，用来描述错误的详情；
2. 然后成员是`Entry`的列表（表示错误列表）；
3. 并且都提供两个方法：
    + `ToString()`方法：用来输出详情。
    + `MergeFrom()`方法：用来从一个“相同类型的”的错误中，合并所有的“错误条目”。

>> 说明: 有几个错误类型是`LBM`开头的（`LBM`是`LogBlockManager`的缩写），表示`LogBlock`相关的错误。

# `struct MissingBlockCheck`
检查对象：被`Tablet`引用，但是在`BlockManager`中却不存在。

错误类型：`fatal`级别。是不可修复的。

## `struct MissingBlockCheck::Entry`
描述该类型的一个具体错误条目。

## 成员

### `entries`
所有该类型的错误集合。

```
struct MissingBlockCheck {

  // Merges the contents of another check into this one.
  void MergeFrom(const MissingBlockCheck& other);

  // Returns a multi-line string representation of this check.
  std::string ToString() const;

  struct Entry {
    Entry(BlockId b, std::string t);
    BlockId block_id;
    std::string tablet_id;
  };
  std::vector<Entry> entries;
};
```

# `struct OrphanedBlockCheck`

检查对象：在`BlockManager`中存在，但是却不属于任何`Tablet`。

错误类型：不是`fatal`级别，并且是可修复的（删除对应的`Block`即可）。

## `struct OrphanedBlockCheck::Entry`

## 成员

### `entries`
该类型的错误集合。

```
struct OrphanedBlockCheck {

  // Merges the contents of another check into this one.
  void MergeFrom(const OrphanedBlockCheck& other);

  // Returns a multi-line string representation of this check.
  std::string ToString() const;

  struct Entry {
    Entry(BlockId b, int64_t l);
    BlockId block_id;
    int64_t length;
    bool repaired;
  };
  std::vector<Entry> entries;
};
```

# `struct LBMFullContainerSpaceCheck`

`LBM container`是满的，但是却还有剩余的空间。

>> leftover: 剩余物  
>> unpunched: 未打孔，未敲击；  

可能场景是：
1. 在尾部：剩余的分配空间
2. 在中间：未填充的空洞；

错误类型：`non-fatal`，并且是“可修复的”（通过移动数据把“空洞”去除，最后把`container file`尾部的空间释放掉）

## `struct LBMFullContainerSpaceCheck::Entry`

### `container`
对应的“容器”。

### `excess_bytes`
超过的空间大小。

### `repaired`
是否已经进行了修复。

## 成员

### `entries`
该类型的错误集合。

```
struct LBMFullContainerSpaceCheck {

  // Merges the contents of another check into this one.
  void MergeFrom(const LBMFullContainerSpaceCheck& other);

  // Returns a multi-line string representation of this check.
  std::string ToString() const;

  struct Entry {
    Entry(std::string c, int64_t e);
    std::string container;
    int64_t excess_bytes;
    bool repaired;
  };
  std::vector<Entry> entries;
};
```

# `struct LBMIncompleteContainerCheck`

# `struct LBMMalformedRecordCheck`

# `struct LBMMisalignedBlockCheck`

# `struct LBMPartialRecordCheck`

# `strcut FsReport`

表述一次“文件系统级别”的检查报告。

在这个“检查报告”中，既包含 文件系统的一些通用指标信息，也包含一系列的用来描述可能出现不一致的`checks`。

**什么是`check`**  
一个`check`，是指 出现特定的“不一致”的列表。具体类型就是前面定义的各种`XxxCheck`类。  

在每个`check`中，都会包含相应的信息。比如：用来表述缺少一个`Block`的`check`，会在其中包含所缺少的`Block`的`id`。

这些`check`中的内容：
1. 在打印日志的时候，都会转化为字符串。(即提供`ToString()`方法)。
2. 也可以从把另一个 同类型的`check`的内容 合并为自己的内容。（即`MergeFrom()`方法）

注意：在一些`check`的`ToString`方法中，会根据需要进行一些聚合。比如`MissingBlockCheck::ToString()`中，输出结果会按照`Tablet`进行聚合。

在`FsReport`类中，每个`check`都会有一个对应的`boost::optional`类型的成员。这样做的含义是：是否存在某个类型的`check`，取决于相应的上下文。也就是说，不是在所有的场景下，都会用到所有的`check`。

## “不一致”场景的分类

分类标准：
1. 是否为`fatal`级别；
2. 是否“可修复”


按照排列组合，一共有如下4个分类：

1. `fatal`且“不可修复”：即`BlockManager`无法自动进行修复。 这种类型的“不一致”，即使只有一个，也会导致`*FatalErrors()`函数 返回失败。
2. `fatal`但“可修复”：`BlockManager`会自动尝试进行修复。
3. `non-fatal`但“不可修复”：这种“不一致”是比较有趣的，因为虽然不能修复，但是仍然可以正常工作，也可以选择忽略掉。
4. `non-fatal`而且“可修复”：这种类型的“不一致”，可能会被修复，也可以被忽略，或者其它变通的方式“绕过去”。

## `struct FsReport::Stats`

统计信息。

### 成员
#### `live_block_count`
当前存在的`Block`（也就是 尚未被删除的`Block`）的数量。

#### `live_block_bytes`
当前所有`live`状态的`Block`的总大小。

#### `live_block_bytes_aligned`
在针对`BlockManager`计算完“对齐”以后，所有`live`状态的`Block`的总大小。

这个值，对应会 大于等于 `live_block_bytes`。

使用场景是：在计算`LBM`的“外部空间”时非常有用。

>> 问题：干嘛用？

#### `lbm_container_count`
#### `lbm_full_container_count`

### 接口

#### `MergeFrom()`
合并另一个`FsRport::Stats`对象的相应成员。

实际上就是将另一个`FsReport::Stats`对象的成员，与 当前该对象的相应成员 “相加”。

#### `ToString()`
转化为字符串，将各个“指标”输出。

```
  struct Stats {
    int64_t live_block_count = 0;

    int64_t live_block_bytes = 0;

    // Total space usage of all live data blocks after accounting for any block
    // manager alignment requirements. Guaranteed to be >= 'live_block_bytes'.
    // Useful for calculating LBM external fragmentation.
    int64_t live_block_bytes_aligned = 0;

    // Total number of LBM containers.
    int64_t lbm_container_count = 0;

    // Total number of full LBM containers.
    int64_t lbm_full_container_count = 0;
  };
```
## 成员

### `stats`
本次“报告”的一些统计信息。

### `data_dirs`
本次“报告”所对应的“数据目录”。

### `wal_dir`
本次“报告”所对应的“`WAL`的目录”。

### 各种`check`对应的成员

```
  Stats stats;

  // Data directories described by this report.
  std::vector<std::string> data_dirs;

  // WAL directory.
  std::string wal_dir;

  // Metadata directory.
  std::string metadata_dir;

  // General inconsistency checks.
  boost::optional<MissingBlockCheck> missing_block_check;
  boost::optional<OrphanedBlockCheck> orphaned_block_check;

  // LBM-specific inconsistency checks.
  boost::optional<LBMFullContainerSpaceCheck> full_container_space_check;
  boost::optional<LBMIncompleteContainerCheck> incomplete_container_check;
  boost::optional<LBMMalformedRecordCheck> malformed_record_check;
  boost::optional<LBMMisalignedBlockCheck> misaligned_block_check;
  boost::optional<LBMPartialRecordCheck> partial_record_check;
```

## 接口列表

### `MergeFrom()`

将另一个`FsReport`对象的相关内容，合并到 当前`FsReport`中。

注意1：两个`FsReport`对象的`metadata_dir`和`wal_dir`成员必须是相同的。

注意2：不仅要合并各个`check`成员，还要合并`data_dirs`成员。

### `ToString()`

打印本地“报告”的字符串。

注意1： 打印结果是“多行的”，有换行符。

注意2：对于`fatal` 且 “不可修复”的 问题，会打印的比较详细。 其他类型的问题，会为了让输出很明晰，而对结果进行聚合。

### `HasFatalErrors()`

是否存在 `fatal`级别、并且“不可修复”的问题。

目前只有两种类型的`check`是 `fatal`且“不可修复”的:
1. `MissingBlockCheck`;
2. `LBMMalformedRecordCheck`

```
bool FsReport::HasFatalErrors() const {
  return (missing_block_check && !missing_block_check->entries.empty()) ||
         (malformed_record_check && !malformed_record_check->entries.empty());
}
```

### `CheckForFatalErrors()`
和`HasFatalErrors()`检查的内容是相同的。但是“返回值的类型”是不同的。

在存在相应的“不一致”时：
1. `HasFatalErrors()`返回是`bool`类型，返回值为`true`；
2. `CheckForFatalErrors()`的返回值是`Status`类型，返回`Status::Corruption`；

```
Status FsReport::CheckForFatalErrors() const {
  if (HasFatalErrors()) {
    return Status::Corruption(
        "found at least one fatal error in block manager on-disk state. "
        "See block manager consistency report for details");
  }
  return Status::OK();
}
```

### `LogAndCheckForFatalErrors()`
和`CheckForFatalErrors()`类似，只是会先现象日志中打印一条信息。

```
Status FsReport::LogAndCheckForFatalErrors() const {
  LOG(INFO) << ToString();
  return CheckForFatalErrors();
}
```

### `PrintAndCheckForFatalErrors()`
和`LogAndCheckForFatalErrors()`类似，区别是 信息直接输出到`stdout`中，而不是“日志”中。

```
Status FsReport::PrintAndCheckForFatalErrors() const {
  cout << ToString();
  return CheckForFatalErrors();
}
```








