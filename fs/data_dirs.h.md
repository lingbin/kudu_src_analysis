[TOC]

文件：`src/kudu/fs/data_dirs.h`

# `struct CanonicalizeRootAndStatus`

>> canonicalization: 规范化，标准化  

>> 说明：在这里，“规范化”的含义：主要就是将路径变为“绝对路径”。

注意：这个类是直接在`kudu namespace`下定义的，不是在`kudu::fs`下。

说明1：目前在很多地方，都是直接传递该对象（或者“该对象的列表”，即`CanonicalizedRootsList`）的。

说明2：将来这个结构，可能会放在`DataDirManager`内部。

**本类的作用**  
正如本列的名字，该类封装了两个成员：1) 经过规范化的“数据根目录”； 2) 该目录的健康状态（是否存在、是否可访问、是否有磁盘错误等）

```
struct CanonicalizedRootAndStatus {
  std::string path;
  Status status;
};
```

# `CanonicalizedRootsList` -- 类型别名

一个`CanonicalizedRootAndStatus`对象的列表（`std::vector<>`）；

```
typedef std::vector<CanonicalizedRootAndStatus> CanonicalizedRootsList;
```

# `UuidByUuidIndexMap`  -- 类型别名
从`uuid index` => `uuid`的映射。

>> 说明：`uuid`是一个字符串，即类型是`std::string`。

```
typedef std::unordered_map<int, std::string> UuidByUuidIndexMap;
```

# `UuidIndexByUuidMap`  -- 类型别名
从`uuid` => `uuid index`的映射。

```
typedef std::unordered_map<std::string, int> UuidIndexByUuidMap;
```

说明：在`DataDirManager`中有很多关于“映射关系”的“类型别名”（每个都表示一种“映射关系”，在`DataDirManager`中对应一个成员，表示一种“映射关系”）。 但是其它的“类型别名”都是在`DataDirManager`中定义的，但是这里将这两个“类型别名”（`UuidByUuidIndexMap`和`UuidIndexByUuidMap`）放在了外面（即不在类`DataDirManager`内部），是因为在其它类（`DataDirGroup`）中，也使用到了这两个类型别名。 

这也是，这里将 这两个“类型别名”的声明，是放在了声明`DataDirGroup`类 的前面的原因。

# 路径常量


## `kDataDirName`

值为`"data"`，在每个“数据根目录”下，都会创建一个“子目录”的名字。

## `kInstanceMetadataFileName`
在上面的`"data"`目录下，会创建一个该文件，也就是`path instance file`，其中保存的内容是`PathInstanceMetadataPB`。

```
const char kInstanceMetadataFileName[] = "block_manager_instance";
const char kDataDirName[] = "data";
```

# `class DataDirGroup`

表示一个由多个“数据根目录”组成的“目录组”(`dir group`)，这些目录是用来存放`Block`的数据。

**对应的`protobuf`结构是`DataDirGroupPB`**。

说明1：每个“数据根目录”在`PathSetPB`中，每个目录会对应一个`UUID`。

说明2：每个`tablet`会对应一个`DataDirGroup`对象，用来表示该`tablet`的数据，所保存在哪些“数据根目录”之下。

**说明3：在“内存结构”和“磁盘（`protobuf`结构）”的标识方式是不同的：**  
1. 在内存中，在每个`DataDirGroup`中，记录所有目录的方式，是通过记录 的是这些`UUID`在`PathSetPB::all_uuids`中的“顺序下标”，即`uuid_idx`。  
2. 在磁盘上，（保存的是`protobuf`类型，即在`DataDirGroupPB`结构中）每个`DataDirGroup`中保存的是 它所包含的所有`UUID`的列表(即直接保存的`uuid`的值)。

>> 备注：  
>> 1. `UUID`实际上是一个“字符串”，即`std::string`类型；
>> 2. `uuid_idx`是一个`int32_t`的类型，表示在“列表”中的“下标号”；

**也就是说，在“从磁盘读取”或“向磁盘写入”一个`DataDirGroup`对象时，需要进行“UUID index”和“UUID”的转化**。  

即，在转化的过程中，需要用到两个结构：
1. 从“内存”到“磁盘”(即从`DataDirGroup`到`DataDirGroupPB`)： `UuidByUuidIndexMap`;
2. 从“磁盘”到“内存”(即从`DataDirGroupPB`到`DataDirGroup`)： `UuidIndexByUuidMap`;

注意：一个“数据根目录”，是可能出现在多个`DataDirGroup`中的（即每个“数据根目录”中会包含多个`tablet`的数据），也就是说，不同`tablet`所对应的`DataDirGroup`对象，其中的`UUID`是可能会重叠的。

说明4：知道了`uuid`或`uuid_idx`，都可以计算出该“数据根目录”的真正物理地址的（需要`DataDirManager`对象）。

## 成员

### `uuid_indices_`

注意：其中的每个元素，是“所表示的目录”在`PathSetPB`中的“下标号”（`uuid_idx`）。

```
  std::vector<int> uuid_indices_;
```

## 接口列表

### `LoadFromPB()`

将一个`DataDirGroupPB`读取对应的数值，初始化当前对象。

这里传入的`uuid_idx_by_uuid`(即`uuid => uuid_idx`)，就是`DataDirManager::idx_by_uuid_`。

```
Status DataDirGroup::LoadFromPB(const UuidIndexByUuidMap& uuid_idx_by_uuid,
                                const DataDirGroupPB& pb) {
  vector<int> uuid_indices;
  for (const auto& uuid : pb.uuids()) {
    int uuid_idx;
    if (!FindCopy(uuid_idx_by_uuid, uuid, &uuid_idx)) {
      return Status::NotFound(Substitute(
          "could not find data dir with uuid $0", uuid));
    }
    uuid_indices.emplace_back(uuid_idx);
  }

  uuid_indices_ = std::move(uuid_indices);
  return Status::OK();
}
```

### `CopyToPB()`

将当前对象转化为`DataDirGroupPB`对象。

这里传入的`uuid_by_uuid_idx`(即`uuid_idx => uuid`)，就是`DataDirManager::uuid_by_idx_`。

```
Status DataDirGroup::CopyToPB(const UuidByUuidIndexMap& uuid_by_uuid_idx,
                              DataDirGroupPB* pb) const {
  DCHECK(pb);
  DataDirGroupPB group;
  for (auto uuid_idx : uuid_indices_) {
    string uuid;
    if (!FindCopy(uuid_by_uuid_idx, uuid_idx, &uuid)) {
      return Status::NotFound(Substitute(
          "could not find data dir with uuid index $0", uuid_idx));
    }
    group.mutable_uuids()->Add(std::move(uuid));
  }

  *pb = std::move(group);
  return Status::OK();
}
```

# `enum DataDirFsType`

描述“文件系统”的类型。

```
// Detected type of filesystem.
enum class DataDirFsType {
  // ext2, ext3, or ext4.
  EXT,

  // SGI xfs.
  XFS,

  // None of the above.
  OTHER
};
```

>> 问题：对于不同的文件系统，处理上有什么区别？

# `enum ConsistencyCheckBehavior`

一个`DataDirManager`在打开的时候，会进行“一致性检查”。详见`DataDirManager::Open()`。

这个结构，用来描述“检查的方式”。

**“是否一致”是指**：  
“内存中记录的`DataDir`”和 “`磁盘上的物理路径`”是否一致。

## `ENFORCE_CONSISTENCY`
最严格，如果有不一致的，那么直接失败。

## `UPDATE_ON_DISK`
如果有不一致，那么修改磁盘上的目录结构，从而达到一致。

注意：`UPDATE_ON_DISK`模式下，`BlockManager`的类型必须要为`log`。    
因为如果`BlockManager`的类型是`"file"`时，是不允许 进行对数据目录进行任何的“添加”和“删除”的。

>> 问题：为什么在`file BlockManager`下，不能够修改文件？

**注意：这时，这个`DataDirManager`对象 一定不是`read_only`的**。  
因为`read_only`模式下，是不可以更新磁盘上的文件的。

## `IGNORE_INCONSISTENCY`
直接忽略掉，不进行任何修改。

```
enum class ConsistencyCheckBehavior {
  // If the data directories don't match the on-disk path sets, fail.
  ENFORCE_CONSISTENCY,

  // If the data directories don't match the on-disk path sets, update the
  // on-disk data to match. The directory manager must not be read-only.
  UPDATE_ON_DISK,

  // If the data directories don't match the on-disk path sets, continue
  // without updating the on-disk data.
  IGNORE_INCONSISTENCY
};
```

>> 问题：为什么 没有一个模式是“当出现不一致的时候，修改内存结构”？


# `struct DataDirMetrics`

每个`DataDir`中，都会包含一个`DataDirMetrics`对象，表示当前“数据根目录”的`metric`指标。

```
struct DataDirMetrics {
  explicit DataDirMetrics(const scoped_refptr<MetricEntity>& entity);

  scoped_refptr<AtomicGauge<uint64_t>> data_dirs_failed;
  scoped_refptr<AtomicGauge<uint64_t>> data_dirs_full;
};
```

# `class DataDir`

在`DataDirManager`中，对于每个“数据根目录”，都会生成一个`DataDir`对象。

**强调：每个`DataDir`对象，表示一个“数据根目录”**（不是“普通的目录”）。

## `enum DataDir::RefreshMode`

每个目录都会被预先定义有大小为`reserved`的容量，参见`FLAGS_fs_data_dirs_reserved_bytes`。

在运行过程中，需要检查一个目录是否为“满”的。这个`enum`类型用来标识“检查模式”的。

**不同“检查模式”的含义：**  
1. 如果模式为`EXPIRED_ONLY`，那么只会针对 在之前的某个时间，已经被认为是“满”了的目录进行检查（即只检查“曾经满过”的目录）。
2. 如果模式为`ALWAYS`，那么所有目录都会检查（无论它是否曾经满过）

**返回值：**  
+ 只有当真正有错误的时候，才会返回一个`bad Status`；
+ 可以用`is_full()`方法来判断一个目录 是否为“满”的；

>> 问题：这里关于`EXPIRED_ONLY`的解释，可能是不对的。从这个变量名字来看，是看不出来要求对应的目录是“曾经满过的”。

```
  enum class RefreshMode {
    EXPIRED_ONLY,
    ALWAYS,
  };
```

## 成员

### `env_`
封装了“底层文件系统”的所有接口。

### `metrics_`
`DataDirMetrics`类型，表示“当前数据根目录”对应的`metric`指标。

### `fs_type_`
当前“数据根目录”对应的“文件系统类型”。

### `dir_`
当前“数据根目录”的完整路径（字符串形式）。

### `metadata_file_`
当前“数据根目录”中，保存元信息的文件（其中保存的是一个`PathInstanceMetadataFile`结构）。

>> 强调：每个“数据根目录” 都会对应一个`PathInstanceMetadataFile`结构。

### `pool_`
当前“数据根目录”的后台工作，所对应的“线程池”。

注意：目前的实现中，每个“数据根目录”，都会有一个独立的“线程池”。

### `is_shutdown_`
标识 当前目录 是否已经关闭。

>> 问题：一个目录被关闭了，代表什么意思？

### `lock_`
互斥锁。

用来保护如下几个变量：`last_space_check_`、`is_full_`和`available_bytes_`。

在程序运行过程中，这些变量会被并发“访问”和“修改”，所以需要进行“锁保护”。

而本类的其它成员，一旦`DataDir`对象被构造出来，其它成员都是不变的，所以也就不需要进行“锁保护”。

### `last_space_check_`
上次检查目录大小的时间。

### `is_full_`
标识当前目录是否已经“满”了。

### `available_bytes_`
当前目录的可用空间。

在`RefreshAvailableSpace()`中进行更新该数值。

```
  Env* env_;
  DataDirMetrics* metrics_;
  const DataDirFsType fs_type_;
  const std::string dir_;
  const std::unique_ptr<PathInstanceMetadataFile> metadata_file_;
  const std::unique_ptr<ThreadPool> pool_;

  bool is_shutdown_;

  mutable simple_spinlock lock_;
  MonoTime last_space_check_;
  bool is_full_;
  int64_t available_bytes_;
```

## 接口列表

### `ExecClosure()`
在当前目录所对应的“线程池”中，执行一个后台任务。

正常情况下，所有`task`都是 异步提交； 但是，如果向“线程池”提交失败了，会在当前提交线程中 立即执行。

```
void DataDir::ExecClosure(const Closure& task) {
  Status s = pool_->SubmitClosure(task);
  if (!s.ok()) {
    WARN_NOT_OK(
        s, "Could not submit task to thread pool, running it synchronously");
    task.Run();
  }
}
```

### `WaitOnClosures()`

等待线程池中的`task`执行结束。
```
void DataDir::WaitOnClosures() {
  pool_->Wait();
}
```

说明：如果之前在`ExecClosure()`的时候，并没有向“线程池”提交成功，那么这里`pool_->Wait()`也不会进行阻塞，而是直接返回。

### `ShutDown()`

等待线程池中的`task`执行结束，并且关闭该线程池。

```
void DataDir::Shutdown() {
  if (is_shutdown_) {
    return;
  }

  WaitOnClosures();
  pool_->Shutdown();
  is_shutdown_ = true;
}
```

注意：从关闭这个`DataDir`就要关闭整个“线程池”来看，每个`DataDir`都会有一个独立的“线程池”。

### `RefreshAvailableSpace()`
更新当前“数据根目录”的“可用剩余空间”。

```
Status DataDir::RefreshAvailableSpace(RefreshMode mode) {
  switch (mode) {
    case RefreshMode::EXPIRED_ONLY: {
      std::lock_guard<simple_spinlock> l(lock_);
      DCHECK(last_space_check_.Initialized());
      MonoTime expiry = last_space_check_ + MonoDelta::FromSeconds(
          FLAGS_fs_data_dirs_available_space_cache_seconds);
      if (MonoTime::Now() < expiry) {
        break;
      }
      FALLTHROUGH_INTENDED; // Root was previously full, check again.
    }
    case RefreshMode::ALWAYS: {
      int64_t available_bytes_new;
      Status s = env_util::VerifySufficientDiskSpace(
          env_, dir_, 0, FLAGS_fs_data_dirs_reserved_bytes, &available_bytes_new);
      bool is_full_new;
      if (PREDICT_FALSE(s.IsIOError() && s.posix_code() == ENOSPC)) {
        LOG(WARNING) << Substitute(
            "Insufficient disk space under path $0: creation of new data "
            "blocks under this path can be retried after $1 seconds: $2",
            dir_, FLAGS_fs_data_dirs_available_space_cache_seconds, s.ToString());
        s = Status::OK();
        is_full_new = true;
      } else {
        is_full_new = false;
      }
      RETURN_NOT_OK_PREPEND(s, "Could not refresh fullness"); // Catch other types of IOErrors, etc.
      {
        std::lock_guard<simple_spinlock> l(lock_);
        if (metrics_ && is_full_ != is_full_new) {
          metrics_->data_dirs_full->IncrementBy(is_full_new ? 1 : -1);
        }
        is_full_ = is_full_new;
        last_space_check_ = MonoTime::Now();
        available_bytes_ = available_bytes_new;
      }
      break;
    }
    default:
      LOG(FATAL) << "Unknown check mode";
  }
  return Status::OK();
}

```

说明1：这里在`mode == RefreshMode::EXPIRED_ONLY`状态下，会要求`last_space_check_`已经被初始化。  
而`last_space_check_`是在`mode== ALWAYS`的时候才会被修改。  
所以，第一次调用该方法时，对应的`mode`一定是`ALWAYS`模式。

> 备注：第一次调用是在`DataDirManager::Open()`中调用的。 即进程启动后，加载所有的“数据根目录”时，会调用一次。

说明2：要注意这里`FALLTHROUGH_INTENDED`宏的使用。

它的定义参见：`src/kudu/gutil/macros.h`, 可以理解为：它什么都不做。  
也就是说，具体的行为是 直接进入下一个`case`分句，而不是`break`掉当前`case`分句；

在这里的含义是：在“检查模式”为`RefreshMode::EXPIRED_ONLY`时，如果“上一次检查目录“可用剩余空间”的时间”已经超过了“设置的更新周期”（即“可用剩余空间”的最长cache时间，即`FLAGS_fs_data_dirs_available_space_cache_seconds`），那么就会直接进入`case RefreshMode::ALWAYS: `部分的执行（即会进行一次检查）。

>> 说明: `linux`的错误码 `ENOSPC`的含义是： No space left on device，即表示设备上没有空间。

说明3：如果是遇到IO错误，并且是错误码是`ENOSPC`的时候，表示磁盘上已经没有剩余空间了。  
但是这个时候，本次更新是可以成功进行的，所以这时会返回`Status::OK`。（这时也会修改`is_full_`和`available_bytes_`/`last_space_check_`）

对于其它的错误，那么认为本次更新是失败的，所以就会返回失败。

说明4：在更新`is_full_`/`avaliable_bytes_`/`last_space_check_`的时候，是要加锁（`lock_`）的。

说明5：在`mode == EXPIRED_ONLY`时，要先读取`last_space_check_`，所以也要加锁（这里使用的`std::lock_guard<>`）。  但是在`FALLTHROUGH_INTENDED`的时候，是先离开`case EXPIRED_ONLY`的作用域，所以这时就会释放锁。   

如果后面需要更新`is_full_`等变量，那就重新再加锁。

也就是说，“向文件系统获取剩余空间”的部分，是在“锁外部”执行的。（在锁内执行的话，性能会差）

# `struct DataDirManagerOptions`

`DataDirManager`的创建选项。

## 成员

### `block_manager_type`
对应的`BlockManager`的类型。

参见`src/kudu/fs/block_manager.h`.

是字符串类型，并且只有两个取值：`"file"`或`"log"`。

默认值是`FLAGS_block_manager`。 定义参见`src/kudu/fs/fs_manager.cc`.

1. 对于`linux`系统，默认值是`log`;
2. 其它系统，默认值是`file`；

### `metric_entity`
`MetricEntity`类型，表示当前`DataDirManager`对象的`metric`指标。默认值为`nullptr`；

### `read_only`
当前`DataDirManager`对象是否为“只读”的。默认值为`false`；

### `consistency_check`
当前`DataDirManager`对象“一致性检查”时，如果 发现不一致时 的处理方法。

默认值为`ENFORCE_CONSISTENCY`（最严格的检查方式）。

```
struct DataDirManagerOptions {
  DataDirManagerOptions();

  std::string block_manager_type;

  scoped_refptr<MetricEntity> metric_entity;

  bool read_only;

  ConsistencyCheckBehavior consistency_check;
};
```

# `class DataDirManager`

说明：在代码中经常将“该类的 变量” 命名为`dd_manager`

**注意：这个进程只有一个`DataDirManager`对象。**

## `enum DataDirManager::LockMode`
加锁的模式。

在当前`DataDirManager::LoadInstances()`时（即当进程启动，初始化`DataDirManager`对象时），就会对这些“元数据文件”（即每个“数据根目录”下的`path instance`文件）进行加锁。

>> mandatory: 强制的；托管的；命令的

### `MANDATORY`
排它锁。一个正常模式启动的进程，会使用这个锁。

### `OPTIONAL`
可选的。 对于`read_only`模式下启动的进程，会加这种锁。

### `NONE`
不加锁。

参见`DataDirManager::LoadInstances()`的说明，了解为什么在`read_only`模式下也要加锁。而且如果有多个`read_only`模式的进程，那么实际上只有第一个进程会真正的加锁(但所有的进程都会成功启动)。

```
  enum class LockMode {
    MANDATORY,
    OPTIONAL,
    NONE,
  };
```

## `enum DataDirManager::DirDistributionMode`
在为一个`tablet`创建`DataDir`的时候，使用该参数来指定 为该`tablet`指定“数据根目录”的方式。

即类型是作为`CreateDataDirGroup()`的参数使用。

该参数的默认值是`USE_FLAG_SPEC`，表示受`FLAGS`变量(`FLAGS_fs_target_data_dirs_per_tablet`)的约束。

### `ACROSS_ALL_DIRS`
表示在所有“数据根目录”中创建。

目前，这个“值”仅仅用来兼容旧版本。

### `USE_FLAG_SPEC`
如字面意思，就是用`FLAGS`变量来表示。

对应的`FLAGS`变量是：`FLAGS_fs_target_data_dirs_per_tablet`。详情参见这个变量的描述。

注意：这个`FLAGS`变量指定的值，只是一个最大值。也就是说，在实际运行过程中，为某个`tablet`找到“数据目录”数量是可能比这个值少的。比如说：指定了每个`tablet`要分布在`3`个目录中，但是磁盘上只有1个目录是“未满的”，那么这个`tablet`就只会放在这`1`个目录中。

```
  enum class DirDistributionMode {
    ACROSS_ALL_DIRS,
    USE_FLAG_SPEC,
  };
```

## 成员

### `env_`
封装了“底层文件系统”的所有接口。

### `opts_`
当前`DataDirManager`对象的创建选项。

### `canonicalized_data_fs_roots_`

>> verbatim: 一字不差地，逐字地

已经“规范化”过的“根目录”列表。

> 说明：进程在启动时，会根据用户配置的数据目录，进行规范化，然后传入到`DataDirManager`的构造函数中，这里就是一个拷贝。

1. 其中“第一个”目录，会被用来存放“元数据”。
2. 在这个列表中的目录，已经被“去重”过了，即其中的元素是没有重复的。

>> 问题：如果指定了两个目录，但这两个目录是“软链”关系，即实际上是同一个目录，Kudu是否能够区分？

### `metrics_`
该`DataDirManager`的`metric`指标。

### `data_dirs_`
所管理的所有目录。

### `uuid_by_root_`
`std::unordered_map<std::string, std::string>`类型。从“根目录的完整路径” => “UUID”的映射。

### `data_dir_by_uuid_idx_`
`std::unordered_map<int, DataDir*>`类型。从`uuid_idx` => `DataDir*`的映射。

### `uuid_idx_by_data_dir_`
`std::unordered_map<DataDir*, int>`类型。从`DataDir*` => `uuid_idx`的映射。

### `group_by_tablet_map_`
`std::unordered_map<std::string, internal::DataDirGroup>`类型。从`tablet_id` => `DataDirGroup`的映射。

### `tablets_by_uuid_idx_map_`
`std::unordered_map<int, std::set<std::string>>`类型。从`uuid_idx` => “`tablet_id`集合”的映射。

注意：每个`uuid_idx`，是可以映射到 “多个`tablet`”的（这的map中的值是`std::set<>`类型）。

### `uuid_by_idx_`
从 `uuid_idx` => `uuid`的映射。

### `idx_by_uuid_`
从 `uuid` => `uuid_idx`的映射。

### `failed_data_dirs_`
`std::set<int>`类型。 

注意：其中的每个成员，是`uuid_idx`（不是“路径的字符串形式”）。

### `dir_group_lock_`
类型是`percpu_rwlock`，是一个“读写锁”。

因为是一个“读写锁”，所以：  
1. 多个线程的读操作，相互不会阻塞，可以并发进行；  
    比如：获取下一个目录，从而进行`Flush()`.
2. 如果有写操作（写锁），会让所有其它线程都阻塞。  
    比如：在创建一个`tablet`的时候（要创建一个`DataDirGroup`）,会加写锁。

**保护的内容：**  
正如它的‘变量名’，这个锁用来保护和 “目录” 相关的结构；

其它的一些结构，包括“其它的映射关系”在内，在启动以后就固定了，在运行过程中是不需要改变的，所以也就不需要加锁。

具体就是用来保护的成员是：`group_by_tablet_map_`和`failed_data_dirs_`。

### `rng_`
`ThreadSafeRandom`类型。

一个随机类型，用来随机的选择目录。

具体的使用场景是：在从一个列表中随机选择一个目录的时候，`kudu`的实现方法是，先将“列表”进行“`shuffle`”一下，然后选择第一个。   

在进行`shuffle`的时候，需要传入一个随机数，就是由这个`rng_`产生的。

```
  Env* env_;
  const DataDirManagerOptions opts_;

  const CanonicalizedRootsList canonicalized_data_fs_roots_;

  std::unique_ptr<DataDirMetrics> metrics_;

  std::vector<std::unique_ptr<DataDir>> data_dirs_;

  typedef std::unordered_map<std::string, std::string> UuidByRootMap;
  UuidByRootMap uuid_by_root_;

  typedef std::unordered_map<int, DataDir*> UuidIndexMap;
  UuidIndexMap data_dir_by_uuid_idx_;

  typedef std::unordered_map<DataDir*, int> ReverseUuidIndexMap;
  ReverseUuidIndexMap uuid_idx_by_data_dir_;

  typedef std::unordered_map<std::string, internal::DataDirGroup> TabletDataDirGroupMap;
  TabletDataDirGroupMap group_by_tablet_map_;

  typedef std::unordered_map<int, std::set<std::string>> TabletsByUuidIndexMap;
  TabletsByUuidIndexMap tablets_by_uuid_idx_map_;

  UuidByUuidIndexMap uuid_by_idx_;
  UuidIndexByUuidMap idx_by_uuid_;

  typedef std::set<int> FailedDataDirSet;
  FailedDataDirSet failed_data_dirs_;

  mutable percpu_rwlock dir_group_lock_;

  mutable ThreadSafeRandom rng_;
```

## 多个映射的关系图

注意：如本类名字，本类是用来管理“`DataDir`”的。  但实际上，**本类的关键，就是维护下面的映射关系**。

```
+------------+             +------------+
|            +------------>+            |
|  root_dir  |             |  uuid      |
|            |             |            |
+------------+             +--+------+--+
                              |      ^
                              |      |
                              v      |
+------------+             +--+------+--+
|            +<------------+            | *
|  DataDir*  |             |  uuid_idx  +-------------------+
|            +------------>+            |                   |
+------------+             +------------+                   |
                              |1                            |
                              |                             |
                              v*                            |1
                           +------------+            +---------------+
                           |            +----------->+               |
                           |  tablet_id |            |  DataDirGroup |
                           |            |            |               |
                           +------------+            +---------------+

```

>> "root_dir"表示一个用户指定的“数据根目录”。

图中的每个箭头，表示有一个映射关系（从箭头的起点对象 => 箭头的终点对象）。

说明1：在`DataDirGroup`对象和`uuid_idx`之间，不是映射关系。 而是`DataDirGroup`内部，就是有`uuid_idx`组成的。（参见`DataDirGroup`类的说明）

说明2：从`uuid_idx` 和`tablet`之间，是“多对多”的关系: 
1. 一个“数据目录”，可以存放多个`tablet`的数据；
2. 一个`tablet`的数据，也可以存放在多个“数据目录”中；

说明3：如果没有特殊标注的，都是“一对一”的关系。

说明4：有些映射关系（除了和`tablet`相关的），都是在`DataDirManager::Open()`中构建的（在`Open()`方法中，只关心“根目录”的具体情况，还没有加载`tablet`的数据，所以没法构建`tablet`相应的结构）.   
参见`DataDirManager::Open()`的说明

## 接口方法

### 静态方法
#### `static GetRootNames()`

给定一组`CanonicalizedRootAndStatus`列表，获取逐个获取其中的“路径字符串”。

```
vector<string> DataDirManager::GetRootNames(const CanonicalizedRootsList& root_list) {
  vector<string> roots;
  std::transform(root_list.begin(), root_list.end(), std::back_inserter(roots),
    [&] (const CanonicalizedRootAndStatus& r) { return r.path; });
  return roots;
}
```

说明：这里使用了`std::transform`。 参见： http://www.cplusplus.com/reference/algorithm/transform/?kw=transform  

#### `static CreateNew()`

创建一个`DataDirManager`对象，并且会在磁盘上创建相应的文件。

如果对应的目录已经存在，会返回失败。

```
Status DataDirManager::CreateNew(Env* env, CanonicalizedRootsList data_fs_roots,
                                 DataDirManagerOptions opts,
                                 unique_ptr<DataDirManager>* dd_manager) {
  unique_ptr<DataDirManager> dm;
  dm.reset(new DataDirManager(env, std::move(opts), std::move(data_fs_roots)));
  RETURN_NOT_OK(dm->Create());
  RETURN_NOT_OK(dm->Open());
  dd_manager->swap(dm);
  return Status::OK();
}
```

#### `static OpenExisting()`

读取磁盘的文件，创建出来一个`DataDirManager`对象。

```
Status DataDirManager::OpenExisting(Env* env, CanonicalizedRootsList data_fs_roots,
                                    DataDirManagerOptions opts,
                                    unique_ptr<DataDirManager>* dd_manager) {
  unique_ptr<DataDirManager> dm;
  dm.reset(new DataDirManager(env, std::move(opts), std::move(data_fs_roots)));
  RETURN_NOT_OK(dm->Open());
  dd_manager->swap(dm);
  return Status::OK();
}
```

### `private` 方法

#### `private Create()`

在磁盘上 初始化 相应的目录。

返回失败的两种场景：
1. 对应的目录已经存在；
2. 该目录所在的磁盘发生了“磁盘错误”；

**注意：该方法只有在首次初始化`DataDirManager`对象时会被调用**  

也就是说：如果是通过读取磁盘上的“持久化的数据”来恢复出来`DataDirManager`对象，是不会调用该方法的。  
参见 `OpenExisting()`方法。

>> 备注：代码中经常提到的`instance file`，实际上就是在每个“数据根目录”下的“元数据文件”，它的内容是`PathInstanceMetadataPB`结构。


```
Status DataDirManager::Create() {
  CHECK(!opts_.read_only);

  // Generate a new UUID for each data directory.
  ObjectIdGenerator gen;
  vector<string> all_uuids;
  vector<pair<string, string>> root_uuid_pairs_to_create;
  for (const auto& r : canonicalized_data_fs_roots_) {
    RETURN_NOT_OK_PREPEND(r.status, "Could not create directory manager with disks failed");
    string uuid = gen.Next();
    all_uuids.emplace_back(uuid);
    root_uuid_pairs_to_create.emplace_back(r.path, std::move(uuid));
  }
  RETURN_NOT_OK_PREPEND(CreateNewDataDirectoriesAndUpdateInstances(
      std::move(root_uuid_pairs_to_create), {}, std::move(all_uuids)),
                        "could not create new data directories");
  return Status::OK();
}
```

注意：在这个方法中，会为每个“根目录”都会生成一个`UUID`。（也就是说，对于每个“数据根目录”，所对应的`uuid`是在这个方法中，才会被初始化的）

注意2：调用`CreateNewDataDirectoriesAndUpdateInstances()`时的第2个参数，这里传递的是“空列表”。  
这个函数的第2个参数，表示的是要更新的`instance file`列表（就是如果这些文件已经存在时，那么需要更新这些“数据根目录”下的这个`path instance`文件（都是`PathInstanceMetadataPB`结构），更新的内容就是这些`instance file`中记录的`all_uuids`）。   

因为每个“数据根目录”中的`path instance`文件中(即`PathInstanceMetadataPB`结构中)，都会保存`all_uuids`，所以如果即使任何一个“数据根目录”发生了变化（比如“增加”或“删除”了一个“数据根目录”），所有的“数据根目录”下的`path instance file`都需要进行更新（因为`all_uuids`发生了变化）。

但是因为“本函数的上下文”是第一次创建各个目录的`instance file`（参见`Create()`函数），也就是说，磁盘上 还没有这些`instance file`，所以也就没有`instance file`需要被更新，所以传递的是一个“空列表”，表示不需要任何更新。

#### `private CreateNewDataDirectoriesAndUpdateInstances()`

初始化所有的“根目录”，并且（如果有指定要更新的`instance file`的话，）会更新 “所指定的数据目录”中的`intance file`。

注意：这个函数会忽略掉所有“不健康”的`intance file`。

```
Status DataDirManager::CreateNewDataDirectoriesAndUpdateInstances(
    vector<pair<string, string>> root_uuid_pairs_to_create,
    vector<unique_ptr<PathInstanceMetadataFile>> instances_to_update,
    vector<string> all_uuids) {
  CHECK(!opts_.read_only);

  vector<string> created_dirs;
  vector<string> created_files;
  auto deleter = MakeScopedCleanup([&]() {
    // Delete files first so that the directories will be empty when deleted.
    for (const auto& f : created_files) {
      WARN_NOT_OK(env_->DeleteFile(f), "Could not delete file " + f);
    }
    // Delete directories in reverse order since parent directories will have
    // been added before child directories.
    for (auto it = created_dirs.rbegin(); it != created_dirs.rend(); it++) {
      WARN_NOT_OK(env_->DeleteDir(*it), "Could not delete dir " + *it);
    }
  });

  // Ensure the data dirs exist and create the instance files.
  for (const auto& p : root_uuid_pairs_to_create) {
    string data_dir = JoinPathSegments(p.first, kDataDirName);
    bool created;
    RETURN_NOT_OK_PREPEND(env_util::CreateDirIfMissing(env_, data_dir, &created),
        Substitute("Could not create directory $0", data_dir));
    if (created) {
      created_dirs.emplace_back(data_dir);
    }

    if (opts_.block_manager_type == "log") {
      RETURN_NOT_OK_PREPEND(CheckHolePunch(env_, data_dir), kHolePunchErrorMsg);
    }

    string instance_filename = JoinPathSegments(data_dir, kInstanceMetadataFileName);
    PathInstanceMetadataFile metadata(env_, opts_.block_manager_type,
                                      instance_filename);
    RETURN_NOT_OK_PREPEND(metadata.Create(p.second, all_uuids), instance_filename);
    created_files.emplace_back(instance_filename);
  }

  // Update existing instances, if any.
  RETURN_NOT_OK_PREPEND(UpdateInstances(
      std::move(instances_to_update), std::move(all_uuids)),
                        "could not update existing data directories");

  // Ensure newly created directories are synchronized to disk.
  if (FLAGS_enable_data_block_fsync) {
    WARN_NOT_OK(env_util::SyncAllParentDirs(env_, created_dirs, created_files),
                "could not sync newly created data directories");
  }

  // Success: don't delete any files.
  deleter.cancel();
  return Status::OK();
}
```

说明1：本函数分为两个阶段：1）`create new data dir`; 2）`upate instance file`。

这两个阶段的操作对象，分别有“第1个”和“第2个”参数来指定；

1. `root_uuid_pairs_to_create`: 要创建`"/data" 子目录`和`path instance file`的“数据根目录”；
2. `instances_to_update`: 要更新的`path instance file`；

说明2：在`create new data dir`阶段所有创建的“文件”和“目录”都会被记录下来，这样在发生失败的时候，会进行清理。

说明3：为了在失败的时候能够“自动地”的清理 中间文件，使用了`MakeScopedCleanup()`函数（会返回一个`ScopedCleanup`对象，参见`src/kudu/util/scoped_cleanup.h`）

说明4：在“自动清理”时，删除的顺序是：先删除所有文件，然后再删除所有目录。  
其实这个顺序比较容易理解，因为“文件”是在“目录”之下的，如果先删除“目录”，其实把文件也一起删除了。但是`Env`的接口是把“目录”和“文件”的操作分开的。（`Env::DeleteFile()`和`Env::DeleteDir()`）

说明5：在“自动清理目录”时，对于要删除的多个目录，这里是“按照逆序”来删除的。  
原因是：
1. “目录”之间是有层级关系的（为了让每次调用`DeleteDir()`都成功，就要求删除的时候，先删除“子目录”，然后再删除“父目录”。 否则的话，如果先删除了“父目录”，那么删除“子目录”时就会因为目录已经不存在而返回失败。）；
2. 这里为了达到按照“先删孩子，再删父亲”的顺序，这里在将“多个目录”添加到`created_dirs`中时，也是按照顺序的，即先添加“父目录”，然后再添加“子目录”。这样，在删除的时候，只需要按照“从后向前”的顺序进行遍历即可。

说明6：在每个“数据根目录”下，都会创建一个名字为`"data"`的子目录（名字就是常量`kDataDirName`的值，参见`src/kudu/fs/data_dirs.h`）。

说明7：在每个“数据根目录”的`"data"`子目录下，都会放一个`instance file`，文件名字是`"block_manager_instance"`(文件名字就是常量`kInstanceMetadataFileName`的值）  

> 注意：这个`instance file`，是在`UpdateInstances()`中才会被真正写入）。

说明8：创建的这些“目录”和“文件”，是否要进行`fsync`，是有`FLAGS_enable_data_block_fsync`的值来决定的。

#### `private UpdateInstances()`
更新磁盘上的`instance file`。

**说明1： 更新什么内容：**  
就是用参数`new_all_uuids`来覆盖掉 在各个`instance file`中的现有值。

**说明2：更新哪些“数据根目录”的`instance file`：**    
由参数`instances_to_update`指定。

+ 对于“不健康”的“instance file”，会被忽略掉。
+ 如果该参数是一个空的列表，那么该函数就什么都不做。

**说明3： 更新的操作步骤**  
因为可能要更新多个文件，如果直接更新，那么可能会出现“部分文件更新成功，部分文件更新失败”。

为了避免这个问题，这里使用的操作步骤是：
1. 先拷贝出来一个副本；
2. 然后更新“原文件”；

如果更新“原文件”的过程发生了失败，那么就会用“副本”把文件替换回来。

这样，就保证了整体的原子性，即
1. 如果成功：也就意味着“所有指定的`instance file`”都被更新了;
2. 如果失败：那么所有的`instance file`的内容都没有任何变化。

```
Status DataDirManager::UpdateInstances(
    vector<unique_ptr<PathInstanceMetadataFile>> instances_to_update,
    vector<string> new_all_uuids) {
  // Prepare a scoped cleanup for managing instance metadata copies.
  unordered_map<string, string> copies_to_restore;
  unordered_set<string> copies_to_delete;
  auto copy_cleanup = MakeScopedCleanup([&]() {
    for (const auto& f : copies_to_delete) {
      WARN_NOT_OK(env_->DeleteFile(f), "Could not delete file " + f);
    }
    for (const auto& f : copies_to_restore) {
      WARN_NOT_OK(env_->RenameFile(f.first, f.second),
                  Substitute("Could not restore file $0 from $1", f.second, f.first));
    }
  });

  // Make a copy of every existing instance metadata file.
  //
  // This is done before performing any updates, so that if there's a failure
  // while copying, there's no metadata to restore.
  WritableFileOptions opts;
  opts.sync_on_close = true;
  for (const auto& instance : instances_to_update) {
    if (!instance->healthy()) {
      continue;
    }
    const string& instance_filename = instance->path();
    string copy_filename = instance_filename + kTmpInfix;
    RETURN_NOT_OK_PREPEND(env_util::CopyFile(
        env_, instance_filename, copy_filename, opts),
                          "unable to backup existing data directory instance metadata");
    InsertOrDie(&copies_to_delete, copy_filename);
  }

  // Update existing instance metadata files with the new value of all_uuids.
  for (const auto& instance : instances_to_update) {
    if (!instance->healthy()) {
      continue;
    }
    const string& instance_filename = instance->path();
    string copy_filename = instance_filename + kTmpInfix;

    // We've made enough progress on this instance that we should restore its
    // copy on failure, even if the update below fails. That's because it's a
    // multi-step process and it's possible for it to return failure despite
    // the update taking place (e.g. synchronization failure).
    CHECK_EQ(1, copies_to_delete.erase(copy_filename));
    InsertOrDie(&copies_to_restore, copy_filename, instance_filename);

    // Perform the update.
    PathInstanceMetadataPB new_pb = *instance->metadata();
    new_pb.mutable_path_set()->mutable_all_uuids()->Clear();
    for (const auto& uuid : new_all_uuids) {
      new_pb.mutable_path_set()->add_all_uuids(uuid);
    }
    RETURN_NOT_OK_PREPEND(pb_util::WritePBContainerToPath(
        env_, instance_filename, new_pb, pb_util::OVERWRITE,
        FLAGS_enable_data_block_fsync ? pb_util::SYNC : pb_util::NO_SYNC),
                          "unable to overwrite existing data directory instance metadata");
  }

  // Success; instance metadata copies will be deleted by 'copy_cleanup'.
  InsertKeysFromMap(copies_to_restore, &copies_to_delete);
  copies_to_restore.clear();
  return Status::OK();
}
```

说明1：所有的“拷贝出来的文件副本”，‘文件名’都是在“原文件名字”的后面，加上`".kudutmp"`作为后缀（即这些文件的全名是`"block_manager_instance.kudutmp"`）。  
（这个后缀是由常量`kTmpInfix`的值，参见`src/kudu/util/path_util.h`）

说明2： 所有的“拷贝出来的文件副本”，都会被记录下来，如果最终因为出现了错误，没有替换掉“原来的`instance file`”，那么这些副本会被删除掉。

说明3：为了自动的删除文件，使用了`MakeScopedCleanup()`函数。（会返回一个`ScopedCleanup`对象，参见`src/kudu/util/scoped_cleanup.h`）

注意：在本函数中，在失败时要删除的内容，都是“文件”，没有目录。

说明4：保存需要“在失败时被删除的文件”的结构有两个：`copies_to_restore`和`copies_to_delete`。

其中要注意下这两个结构的类型：
1. `copies_to_delete`就是一个“`std::unordered_set`”，其中只是简单的保存要删除的”副本文件“的‘文件名’。
2. 而`copies_to_restore`是一个"`std::unordered_map`"，其中是一个映射关系，是从“副本文件名” 到 “原文件名”的映射关系。

>> 备注：这里虽然说是保存的“文件名”，但实际上保存的都是“文件的完整路径”。

在进行“自动清理”（恢复成原状）的时候：  
1. 对于`copies_to_delete`中的文件，会直接删除；
2. 对于`copies_to_restore`中的映射关系，会将 “副本文件名” 重命名为“原文件”。（即将这些文件内容恢复为原来的内容）。

**在整个操作流程中个，这两个变量内容的变化：**  
1. 在第1步：拷贝原有的`instance file`文件，生成新的“副本文件”。 这个时候，所有新创建的“副本文件”，都会添加到`copies_to_delete`结构中。
2. 在第2步：更新原有的`instance file`文件。 在每次更新之前，都先该“副本文件名”从`copies_to_delete`中删除，并且将映射关系添加到`copies_to_restore`中（ “副本文件” -> “原文件”），然后才会去 更新文件内容。  
    + 将“副本文件名”从`copies_to_delete`中删除，是因为这个“副本文件” 所对应的“原文件”马上就要被更新，所以在自动清理（用来恢复成原状）的时候，就不能直接删除这个“副本文件”了，而是要用“重命名”的方式（即将“副本文件名”重命名为“原文件名”）。
    + 先添加“映射关系”，可以找到在“更新失败”的时候，还可以用“拷贝的副本文件” 将 “原文件” 替换回来。  

最后，如果所有文件都成功更新，那么在`copies_to_restore`中，就保存了所有的映射，但是因为所有的`instance file`都成功更新了，所以“这个时候”只需要删除掉所有的“备份文件”即可。  这里的做法是“将`copies_to_restore`的所有`key`都添加到`copies_to_delete`中”，在析构`copy_cleanup`的时候，就会自动进行删除。即会将所有的“备份文件”都重新添加进`copies_to_delete`中。


```
原始状态： 假定有两个文件: `"/path1/data/block_manager_instance"` 和 `"/path2/data/block_manager_instance"`


第1步，创建副本后
    `copies_to_delete` 的内容，有两个元素
        ["/path1/data/block_manager_instance.kudutmp" , "/path2/data/block_manager_instance.kudutmp"]
    
    `copies_to_restore`的内容为 空；
    
第2步，更新原有的`instance file`

    2.1 更新第一个文件之前：
        `copies_to_delete` 的内容，有1个元素
            ["/path2/data/block_manager_instance.kudutmp"]
    
        `copies_to_restore`的内容为 也是1个元素
            {
                 "/path1/data/block_manager_instance.kudutmp" => "/path1/data/block_manager_instance"
            }
        
        ===========
        加入更新该文件失败，那么会触发“恢复”逻辑：
        1) 对于在`copies_to_delete`中的文件，直接删除；
        2) 对于在`copies_to_restore`中的文件，从进行重命名。
        
        因为这时，只有第1个文件(即`/path1/data/block_manager_instance`)文件的内容，可能发生了变化（在更新的过程中，比如遇到IO问题，导致出现了失败），所以这里在重命名之后，原有文件的内容，就恢复原样了。
        ===========

    2.2 如果第一个文件更新成功，继续更新第2个文件之前：
        `copies_to_delete` 的内容，为空的，没有任何元素
            []
    
        `copies_to_restore`的内容为 也是1个元素
            {
                 "/path1/data/block_manager_instance.kudutmp" => "/path1/data/block_manager_instance",
                 "/path2/data/block_manager_instance.kudutmp" => "/path2/data/block_manager_instance"
            }
    
        ===========
        加入更新第2个文件失败，那么会触发“恢复”逻辑：
        1) 对于在`copies_to_delete`中的文件，直接删除；（这时该列表为空，所有不进行任何直接删除）
        2) 对于在`copies_to_restore`中的文件，从进行重命名。
        
        因为这时，原有的2个文件，它们文件的内容，可能都发生了变化。
        这时，将两个文件都重命名后，原有文件的内容，也就恢复原样了。
        ===========
        
最后，如果这2个文件都成功更新了。会将所有的“备份文件”都重新添加进`copies_to_delete`中。
        `copies_to_delete` 的内容，有两个元素
        ["/path1/data/block_manager_instance.kudutmp" , "/path2/data/block_manager_instance.kudutmp"]
    
        `copies_to_restore`的内容为 空；
        
        这时，只需要删除“拷贝的备份文件”即可
```

#### `private Open()`
从磁盘上加载各个`instance file`文件（通过调用`LoadInstances()`方法），并且还会加载各种`index`文件。

说明1：对于不同的`BlockManager`类型，能够容忍的最大的“数据根目录”的数量是不同的：
1. `"file"`: 最多为`2^16 - 1`个，即`65535`个（这个参见`FileBlockLocation`的说明，见文件`src/kudu/fs/file_block_manager.h`）;
2. `"log"`：最多为`2^31 - 1`个，即`INT_MAX`个；

说明2： 如果`opts_.consistency_check == ConsistencyCheckBehavior::UPDATE_ON_DISK`，那么需要进行检查是否处于“一致”状态。

但是，这时要注意的是：`UPDATE_ON_DISK`模式下，`BlockManager`的类型必须要为`log`。  
因为如果`BlockManager`的类型是`"file"`时，是不允许 进行对数据目录进行任何的“添加”和“删除”的。

>> 问题：为什么`file`类型的`BlockManager`，不能进行任何修改？

说明3：在`LoadInstance()`返回的结果中，如果一个`PathInstanceMetadataFile`对象处于“不健康”状态，那么可能有两种情况之一。

如果`opts_.consistency_check == UPDATE_ON_DISK`，针对这两种场景，处理方式为：  
1. 不存在: 这种情况，会进行创建。
2. 遇到了“磁盘故障”(`Statu.IsDiskFailure() == true`)：这种情况会本函数会返回失败。

说明4：如果用户手动增加了某些“数据根目录”，那么在`LoadInstance()`返回的`PathInstanceMetadataFile`列表中，是包含 这些新增的“数据根目录”（即，对应的`PathInstanceMetadataFile`对象）的。  
但是这些新增的“数据根目录”所对应的`PathInstanceMetadataFile`对象，都是“不健康”的。

如果`opts_.consistency_check == UPDATE_ON_DISK`，那么会做两件事情：
1. 对于这些新增的“数据根目录”，都会去创建对应的`path instance`文件（这时会为“新增的数据根目录”创建`UUID`）；
2. 然后更新所有的“数据根目录”下的`path instance`文件（因为`all_uuids`发生了变化）。

> 说明：这里调用`CreateNewDataDirectoriesAndUpdateInstances()`时，第2个参数传递的，仍然是这次加载的结果，即会加载之前成功加载的`path instance file`。而对于“新增的目录”，是刚创建的文件(在创建的时候，会填入内容，是正确的内容，不需要再次被更新)，所以是不需要更新的。 而这些“新增的目录”，在上次的加载结果`loaded_instances`中，状态都是“不健康”的，在`UpdateInstances()`时，会忽略掉这些“不健康”的目录。   
> 也就是说，这些“新增的目录”，其中的`path instance file`是不会被重复更新的。  
> 本更新的`path instance file`，都是在上次加载时，状态为“健康的”的那些目录。

但是要注意的是：如果用户新增了“数据根目录”，这个时候要求“现有的所有‘根目录’”都必须是“健康的”。  
因为只有这样，才能保证所有的“数据根目录”都被更新。

也就是说，比如说某个用户机器上有`10`块盘，然后配置了`5`个数据目录。但是其中一块盘坏了，这时，是不能简单的 只在数据目录列表中增加一个，还必须将那个坏盘 对应的目录 给删除掉。

因为：如果只添加一个，但是不删除坏掉的，那么这里就会有因为无法成功进行更新“所有目录的`path instance file`”而启动失败。

>> 问题：如果用户手动删除了一些“数据根目录”，会造成什么更新？

说明5：在创建了新目录后，会清空`loaded_instance`列表，然后重新加载（即重新调用`LoadInstances()`）.  

这里要注意的是： 因为在`LoadInstances()`中，已经对所有的`instance file`加了“文件锁”。  
而`loaded_instances`的类型是：`vector<unique_ptr<PathInstanceMetadataFile>>`，每个成员都是一个智能指针，将它清空，就会触发对这些智能指针所 引用的`PathInstanceMetadataFile`对象的“析构函数”。而在`PathInstanceMetadataFile`的析构函数中，如果它说表示的`instance file`被加了“文件锁”，那么就会释放这个“文件锁”。 

也就是说，清空`loaded_instance`的行为，会触发释放掉所有`instance file`上的“文件锁”。

当然，后续重新调用`LoadInstances()`的时候，会重新加上“文件锁”。

说明6：正如上面所说，对于新增加的“数据根目录”，在`opts_.consistency_check == UPDATE_ON_DISK`的情况下，会针对这些新增的目录创建相应的`instance file`，并进行更新。 这个时候要求 当前所有的数据目录都是“健康的”。

所以，这时，在重新调用`LoadInstances()`进行重新加载`instance file`以后，会再次检查这次加载上的，所有`PathInstanceMetadataFile`对象的健康状态。 

这时，预期的结果是：所有的`PathInstanceMetadataFile`对象都是健康的。

>> 问题：用户指定的“数据根目录”，是否一定要提前存在？

说明7：如果`opts_.consistency_check != IGNORE_INCONSISTENCY`，那么就检查所有“数据根目录”的一致性（通过调用`PathInstanceMetadataFile::CheckIntegrity()`）。

> 注意1：这里，如果检查“一致性”不通过，那么就会退出。  
> 注意2: 对于处于“不健康的”的“数据根目录”，会被忽略掉。 但是不能所有的目录，都处于“不健康的”状态。

说明8：在检查完“一致性”以后，开始创建“内存结构”，和一些准备工作。
1. 首先会为每个“数据根目录”创建一个“线程池”（每个线程池中只有一个线程）。
2. 然后如果这个“数据根目录”是健康的，检查下每个“数据根目录”所对应的文件系统类型: `EXT`/`XFS`/`OTHER`.
3. 为每个“数据根目录”创建一个`DataDir`对象。(对应的完整路径，就是"`数据根目录/data`")。  
    注意：即使是“不健康”的“数据目录”，也会创建对应的`DataDir`对象。  
4. 对于每个`DataDir`对象，利用它们各自的线程池，启动`task`去删除其中的“临时文件”（以`.kudutmp`结尾的文件）。  
    而且这里会等待删除完毕后，再继续执行）。
5. 在内存中建立所有映射关系。

说明9: 这里建立的各种映射关系内容：
1. `uuid_Idx`, `uuid`, `DataDir`, `root_dir`之间的各种映射关系；
2. 从`tablet`到 `uuid_idx`的映射关系;

> 注意：`root_dir`是“数据根目录”，而`DataDir`是`root_dir`的子目录(`root_dir`下的"data"目录)，就是`$root_dir/data`。

这6个映射关系如下：

```
 +-------------------+            +-------------------+
 |                   |            |                   |
 |    root_dir       +<-----------+    uuid           |
 |                   |            |                   |
 +-------------------+            +-------------------+
                                       |       ^
                                       |       |
                                       v       |
 +-------------------+           +--------------------+         +------------------+
 |                   +---------->+                    |         |                  |
 |    DataDir*       |           |     uuid_idx       +-------->+   tablet         |
 |                   +<----------+                    |         |                  |
 +-------------------+           +--------------------+         +------------------+
```

> 注意：上面的"`uuid_idx` => `tablet`"的映射，类型是`std::unordered_map<int, std::set<std::string>>`。  
> 因为在这里，还不知道具体对应的`tablet`，所以在这里构造时，并没有真正填充对应的tablet，  
> 而是填充的“`uuid_idx` 映射到一个 空的列表”，即这里只是构造了一个“映射的框架”。 具体的`tablet`列表，后面再填充。

>> accordance: 按照，依据；  

说明10：在不同的“一致性检查模式下”，构建“内存映射关系”的方式不同。

说明10.1：如果`opts_.consistency_check != IGNORE_INCONSISTENCY`）  

因为在`非IGNORE_INCONSISTENCY`模式下，对于每个“数据根目录”，前面都已经通过调用`PathInstanceMetadataFile::CheckIntegrity()`来检查过。 所以这个时候，对于处于“健康的”`path instance`，它的内容是最新的。  

所以，我们只要找到任何一个“健康的”`path instance`，它其中包含的`all_uuids`就是“所有健康的”的`uuid`列表。  

> 注意：这个时候，是有可能某个`instance file`中的`UUID`是没有被更新的（因为这个文件是不可被读取的）。   
>    在这种情况下，这些“目录”对应的`path instance`会处于“不健康的”状态。    
>    对于这些目录的处理方法是：这里会把这些目录记录到`failed_data_dirs`中（其中的每个元素，是该“数据根目录”对应的`uuid_idx`，不是“目录字符串”）。 

这时，构建“内存映射关系”的流程为：
1. 遍历`DataDir`列表（就是遍历所有的“数据根目录”）；
2. 如果对应的`path instance`是“不健康的”，那么记录下来；
3. 如果对应的`path instance`是“健康的”，那么 就构建当前“数据根目录”所相关的 “所有映射关系”。

因为上面只针对“健康的”目录构建了“映射关系”，所以“不健康”的`path instance`对应的的映射关系还没有被构建。 这些“不健康的”目录，也是需要构建“映射关系”的。

检查的方法是：
1. 从一个“健康的”的`path instance`中，获取所有的`all_uuids`。
2. 然后遍历所有已经构建的`uuid`列表，找到其中没有构建的`uuid`。
3. 对于每个尚未构建映射关系的`uuid`：
    + 都进行构建映射关系；
    + 更新对应的`metric`指标: `data_dirs_failed`;
    + 将该目录对应的`uuid_idx`插入到`failed_data_dirs`（不健康的“数据根目录”列表）中。
 
> 注意：  
> 在代码实现中，先是按照“从前向后的顺序”遍历`DataDir`列表（即所有的`path instance`列表），从中将“不健康”的目录，逐个添加到`unassigned_dirs`中。  
> 然后在针对“不健康”的目录进行单独处理时，也按照`uuid`的顺序从前向后遍历（这个顺序，和上面遍历`DataDir`的顺序是一样的），所以这里处理“不健康”目录的顺序，和添加到`unassigned_dirs`的顺序是一样的。  
> 所以：在遍历`uuid`列表时，发现的第`n`个“不健康目录”(`n`从`1`开始)，在`unassigned_dirs`中的下标就是`n-1`。（下标从`0`开始）。
> 
> 也是上述的原因，最终`failed_data_dirs`的大小，和`unassigned_dirs`的大小，应该是一样的。

说明10.2：如果`opts_.consistency_check == IGNORE_INCONSISTENCY`  

这个时候，和其它`非IGNORE_INCONSISTENCY`模式最大的区别是：前面并没有进行任何完整性检查（即没有调用过`PathInstanceMetadataFile::CheckIntegrity()`）。  

也就是说，这个时候，其中一个`path instance`记录的`all_uuids`，可能不是全部的`uuid`。

所以，这里实现的方法是：
1. 对于“健康的”目录：添加映射关系；
2. 对于“不健康的”目录：  
    + 添加的`uuid`字符串是一个“伪造的”（`"<unknown uuid $dir>"`）；
    + 添加到`failed_data_dirs`中。

>> 问题：在`!= IGNORE_INCONSISTENCY`的时候，经过了`CheckIntegrity()`以后，到底有什么不同？为什么插入的数据，所有的uuid都存在？
 
说明11：在最后，对于“健康的”目录，要去刷新下他们的目录状态（1) 目录是否已满；2) 目录的剩余空间大小）。

> 备注：这时进程启动后“第一次”去刷新各个目录的“可用剩余空间”（使用的参数是`RefreshMode::ALWAYS`）。

#### `private LoadInstances()`
从各个“数据根目录”中，加载`instance file`文件。

如果加载成功，每个`instance file`文件会转化为一个`PathInstanceMetadataFile`对象，然后所有的`PathInstanceMetadataFile`最终都会保存在参数`loaded_instances`中。

注意1：在参数`loaded_instances`中，也会包含那些 因为1) 不存在、或者2) 磁盘故障 而没有成功加载的文件。 这些文件也会被认为是“成功的加载了”，但是在对应的`PathInstanceMetadataFile`对象内部，会将其记录为处于“不健康”的状态。

注意2：如果所有的`instance file`都处于“不健康”的状态，也会返回失败。

注意3：这里虽然结果是`PathInstanceMetadataFile`对象的列表，但是实际上在`instance file`中保存的内容是`PathInstanceMetadataPB`对象。

>> irreconcileable: 矛盾的；不能协调的；不能和解的；  

如果是某种“不能解决的”问题而导致的加载失败（比如：因为文件已经“被加锁”，导致当前线程无法进行加锁），那么该函数会返回失败。

```
Status DataDirManager::LoadInstances(
    vector<unique_ptr<PathInstanceMetadataFile>>* loaded_instances) {
  LockMode lock_mode;
  if (!FLAGS_fs_lock_data_dirs) {
    lock_mode = LockMode::NONE;
  } else if (opts_.read_only) {
    lock_mode = LockMode::OPTIONAL;
  } else {
    lock_mode = LockMode::MANDATORY;
  }
  vector<string> missing_roots_tmp;
  vector<unique_ptr<PathInstanceMetadataFile>> loaded_instances_tmp;
  for (int i = 0; i < canonicalized_data_fs_roots_.size(); i++) {
    const auto& root = canonicalized_data_fs_roots_[i];
    string data_dir = JoinPathSegments(root.path, kDataDirName);
    string instance_filename = JoinPathSegments(data_dir, kInstanceMetadataFileName);

    unique_ptr<PathInstanceMetadataFile> instance(
        new PathInstanceMetadataFile(env_, opts_.block_manager_type, instance_filename));
    if (PREDICT_FALSE(!root.status.ok())) {
      instance->SetInstanceFailed(root.status);
    } else {
      // This may return OK and mark 'instance' as unhealthy if the file could
      // not be loaded (e.g. not found, disk errors).
      RETURN_NOT_OK_PREPEND(instance->LoadFromDisk(),
                            Substitute("could not load $0", instance_filename));
    }

    // Try locking the instance.
    if (instance->healthy() && lock_mode != LockMode::NONE) {
      // This may return OK and mark 'instance' as unhealthy if the file could
      // not be locked due to non-locking issues (e.g. disk errors).
      Status s = instance->Lock();
      if (!s.ok()) {
        if (lock_mode == LockMode::OPTIONAL) {
          LOG(WARNING) << s.ToString();
          LOG(WARNING) << "Proceeding without lock";
        } else {
          DCHECK(LockMode::MANDATORY == lock_mode);
          return s;
        }
      }
    }

    loaded_instances_tmp.emplace_back(std::move(instance));
  }
```

说明1：首先会检查并设置“加锁模式”（通过`FLAGS_fs_lock_data_dirs`变量）

1. 如果`FLAGS_fs_lock_data_dirs == true`，那么就一定要加锁，而且是“排他锁”(即加锁模式为`LockMode::MANDATORY`)。即只能有一个人来进行使用，不允许多人并发访问数据目录。如果加锁失败，那么当前程序会返回失败（最终进程会退出）。

2. 如果`FLAGS_fs_lock_data_dirs == false`，但是`read_only == true`，也就是出于只读模式下，那么加锁方式会设置为`LockMode::OPTIONAL`。这种模式下，如果加锁失败，那么当前进程并不会退出，而是会继续执行。

> 注意：如果加锁模式是`LockMode::OPTIONAL`，也就是“`可选的`”。  
那么会进行加锁，但是如果加锁失败时，并不报错，而只是打印一条`WARNING`日志，然后就继续执行了。

**为什么只读模式下，仍然要加锁：**  

> 说明：这里的“文件锁”是排他锁，即无论处于什么模式，只要别人拥有锁，后续的人就会加锁失败。

在“只读模式”下，仍然要加锁的原因是：防止有并发的“非只读模式”。  
也就是说，如果在“只读模式”下‘不进行’加锁，那么另一个“不是只读模式”的进程，在尝试加锁时会成功，然后就正常启动，后续也就可以修改“数据的内容”。 那么当前“只读模式”下，就有可能读取到被修改的文件。

而只要在“只读模式”下，也进行加锁，就不会有这个问题。因为其它 并发的“非只读模式”，在启动时，会因为加锁失败而退出，也就不会对数据进行任何修改了。

当前，如果有多个“只读模式”的进程，那么只有第一个“只读模式”的进程会成功加锁，虽然后续的其它“只读模式”在加锁时会失败。但是这个时候，只要发现当前模式是“只读模式”，那么会继续执行，并不返回失败。

说明2：对于每个“数据根目录”下的`instance file`（其中保存的是`PathInstanceMetadataPB`对象，都会转化为一个`PathInstanceMetadataFile`对象）。

说明3：要进行`path instance file`进行加锁（文件锁）的前提：
1. 该`instance file`的状态是“健康的”；
2. 当前的“加锁模式”不是`LockMode::NONE`；

说明4：虽然如果只有部分（非全部）`instance file`是“不健康”的，那么当前函数也是可以成功的。 但是如果所有的`instance file`都是“不健康”的，那么 本次加载会认为是“失败的”。

在`PathInstanceMetadataFile::LoadFromDisk()`中，如果“文件不存在”或者“磁盘故障”，是仍然会返回`Status::OK`的。（参见`src/kudu/fs/block_manager_util.h`中的`RETURN_NOT_OK_FAIL_INSTANCE_PREPEND()`宏）。

**这样做背后的逻辑是：多盘场景下，可以容忍“部分数据盘”出错。**  
如果一个`kudu`进程配置了多个数据目录（通常分布在多个盘上），那么如果其中的某块盘坏了，那么这里因为只是部分数据目录是“不健康”的，所以可以正常加载。但是如果所有的目录都是坏的，即所有的盘都坏了，那么就不能成功启动了。

说明5：在这个函数中加锁后，并没有解锁，要等到用户的逻辑中调用`Unlock()`的时候，才会进行解锁。

1. 在加载失败的场景下：  
    因为这里在加载多个文件的过程中，所有的`PathInstanceMetadataFile`对象，都是放在“临时容器”`loaded_instances_tmp`中，如果加载失败，会去析构这个“临时容器”，最终会去析构其中的`PathInstanceMetadataFile`对象。 而在析构`PathInstanceMetadataFile`的析构函数中，会对这些文件进行解锁。
2. 如果加载是成功的，那么就会一直持有锁，直到析构对应的`PathInstanceMetadataFile`对象（比如这个文件被更新，或者进程结束）。


#### `private GetDirsForGroupUnlocked()`

从当前可用的“数据根目录”中，选择`target_size`个目录。

>> quantified: 量化的；  
>> neglecting: 忽视;  

**说明1：这里的选择策略是：**  
参考论文：<The Power of Two Choices in Randomized Load Balancing>

具体的步骤是：
1. 每次都随机选出来两个“目录”；
2. 选择其中`tablet`数量较少的那个；
3. 如果两者中的`tablet`数量相同，那么就选“可用剩余空间”较多的那个。

最终的结果是：说选择的目录，都是 含有相对较少`tablet`的。

> 注意：这里选择的那个目录，并不是在“所有目录中”对应`tablet`数量最少的，而只是在“随机挑选的两个”中 是较少的。

说明2：参数`group_indices`既会传入内容，也会传出内容。

+ 传入时，当前已经为该`DataDirGroup`选择的“目录”；
+ 传出时：当前函数新选择出来的目录，会添加到末尾。

对于使用者，可以在该函数返回后，通过检查该参数的大小，来判断是否增加了新的“目录”。

在当前函数内，如果一个目录已经在传入参数`group_indices`中，那么这个目录不会再次被考虑。

```
void DataDirManager::GetDirsForGroupUnlocked(int target_size,
                                             vector<int>* group_indices) {
  DCHECK(dir_group_lock_.is_locked());
  vector<int> candidate_indices;
  unordered_set<int> existing_group_indices(group_indices->begin(), group_indices->end());
  for (auto& e : data_dir_by_uuid_idx_) {
    int uuid_idx = e.first;
    DCHECK_LT(uuid_idx, data_dirs_.size());
    if (ContainsKey(existing_group_indices, uuid_idx) ||
        ContainsKey(failed_data_dirs_, uuid_idx)) {
      continue;
    }
    DataDir* dd = e.second;
    Status s = dd->RefreshAvailableSpace(DataDir::RefreshMode::ALWAYS);
    WARN_NOT_OK(s, Substitute("failed to refresh fullness of $0", dd->dir()));
    if (s.ok() && !dd->is_full()) {
      // TODO(awong): If a disk is unhealthy at the time of group creation, the
      // resulting group may be below targeted size. Add functionality to
      // resize groups. See KUDU-2040 for more details.
      candidate_indices.push_back(uuid_idx);
    }
  }
  while (group_indices->size() < target_size && !candidate_indices.empty()) {
    shuffle(candidate_indices.begin(), candidate_indices.end(), default_random_engine(rng_.Next()));
    if (candidate_indices.size() == 1) {
      group_indices->push_back(candidate_indices[0]);
      candidate_indices.erase(candidate_indices.begin());
    } else {
      int tablets_in_first = FindOrDie(tablets_by_uuid_idx_map_, candidate_indices[0]).size();
      int tablets_in_second = FindOrDie(tablets_by_uuid_idx_map_, candidate_indices[1]).size();
      int selected_index = 0;
      if (tablets_in_first == tablets_in_second) {
        int64_t space_in_first = FindOrDie(data_dir_by_uuid_idx_,
                                           candidate_indices[0])->available_bytes();
        int64_t space_in_second = FindOrDie(data_dir_by_uuid_idx_,
                                            candidate_indices[1])->available_bytes();
        selected_index = space_in_first > space_in_second ? 0 : 1;
      } else {
        selected_index = tablets_in_first < tablets_in_second ? 0 : 1;
      }
      group_indices->push_back(candidate_indices[selected_index]);
      candidate_indices.erase(candidate_indices.begin() + selected_index);
    }
  }
}
```

说明1：如函数名，在调用这个方法之前，一定是已经加了`dir_group_lock_`锁的。

但是实际上，在这个函数内，是没有修改`DataDirManager`相关的结构的。

之所以要加锁，是因为在调用该函数的地方，确实都是在锁里的。所以在名字中加上了`"xxxUnlock"`来强调。

说明2：为了执行上述论文的方法，先会遍历一边所有的“目录”，去除不符合条件的，剩余的是“候选集合”。

去除的目录有：
1. 已经被选过的；
2. 损坏的目录；
3. 没有剩余空间的；  
    注意：这里会触发一次“可用剩余空间”的重新统计。

说明3：在执行上述论文的方法，“随机选择两个”的实现方式：
这里的实现是通过：先将整个“候选集”进行 `shuffle`计算，然后取前两个元素的方式 来实现的。

说明4：在进行下一轮的选择前，在上一轮中被选中的那一个“目录”，会被从“候选集”中删除。

说明5：在本函数结构后，不一定 选择的结果中包含的“目录数量”，达到了`target_size`。 因为有可能当前 “有剩余空间的”数据目录确实已经不够了。

#### `private GetDirForBlcok()`
在要创建一个`Block`的时候使用。

给定一个相关的创建选项`CreateBlockOption`，返回对应的“数据根目录”。

说明1：参数`new_target_group_size`  
假如`tablet`当前所对应的“所有数据根目录”的数量为`num_total`，如果当前这些“数据根目录”都是满的，那么会返回`Status::IOError`(`posix`错误码是`ENOSPC`)，并且传入的参数`new_target_group_size`会被设置成 `num_total + 1`。（这样，调用者就可以通过检查这个值，来为它添加“新的数据根目录”）。

**说明1：执行的逻辑：**  
1. 首先获取当前`tablet`现有的“数据根目录”列表；
2. 去除其中“不健康的”目录；
3. 对所有剩余的目录(当前`tablet`所对应的数据根目录中，去除掉‘不健康’的目录)，触发一次“更新目录大小”的操作，然后去除掉其中“已经满了”的目录；  
    （注意：这次是以`EXPIRED_ONLY`的方式去更新目录大小的，所以如果最近已经被更新过，那么本次就不会更新）
4. 如果所有剩余的目录都是“满”的，那么就返回一个`Status::IOError`(其中的`posix code`为`ENOSPC`)。
5. 如果只剩一个目录，那么就选这个；
6. 如果剩余超过`2`个目录以上，那么随机选择一个。  
    这里实现随机的方式是：先将整个列表`shuffle`一下，然后从前两个中，选择一个“可用剩余空间”比较多的。  
说明2：上面的“随机”算法，也可参照`GetDirsForGroupUnlocked()`，两者的“随机”逻辑是类似的。

### `public`方法

#### `CreateDataDirGroup()`
为指定的`tablet`创建一个`DataDirGroup`对象。

说明：每个`tablet`都会有一个最多分布的目录数量（由`FLAGS_fs_target_data_dirs_per_tablet`来指定，默认值是`3`）。

说明2：如果tablet的‘数据分布策略’是“`ACROSS_ALL_DIRS`”，会忽略掉`FLAGS_fs_target_data_dirs_per_tablet`，并且将把数据分布在所有可用的“数据目录”中。  

目前这种‘分布策略’只是为了 向前兼容旧版本，即这样的`tablet`时会使用：它的`superblock`中没有`DataDirGroup`。

说明3：如果遇到如下场景，该函数返回失败：
1. 如果所有磁盘都是“满”的；
2. 所给的`tablet`已经被分配过了`DataDirGroup`对象。

说明4：如果该函数返回了失败，那么本对象(`DataDirManager`)中的各种映射关系，不会有任何改变。

```
Status DataDirManager::CreateDataDirGroup(const string& tablet_id,
                                          DirDistributionMode mode) {
  std::lock_guard<percpu_rwlock> write_lock(dir_group_lock_);
  if (ContainsKey(group_by_tablet_map_, tablet_id)) {
    return Status::AlreadyPresent("Tried to create directory group for tablet but one is already "
                                  "registered", tablet_id);
  }
  // Adjust the disk group size to fit within the total number of data dirs.
  int group_target_size;
  if (FLAGS_fs_target_data_dirs_per_tablet == 0) {
    group_target_size = data_dirs_.size();
  } else {
    group_target_size = std::min(FLAGS_fs_target_data_dirs_per_tablet,
                                 static_cast<int>(data_dirs_.size()));
  }
  vector<int> group_indices;
  if (mode == DirDistributionMode::ACROSS_ALL_DIRS) {
    // If using all dirs, add all regardless of directory state.
    AppendKeysFromMap(data_dir_by_uuid_idx_, &group_indices);
  } else {
    // Randomly select directories, giving preference to those with fewer tablets.
    if (PREDICT_FALSE(!failed_data_dirs_.empty())) {
      group_target_size = std::min(group_target_size,
          static_cast<int>(data_dirs_.size() - failed_data_dirs_.size());

      // A size of 0 would indicate no healthy disks, which should crash the server.
      DCHECK_GE(group_target_size, 0);
      if (group_target_size == 0) {
        return Status::IOError("No healthy data directories available", "", ENODEV);
      }
    }
    GetDirsForGroupUnlocked(group_target_size, &group_indices);
    if (PREDICT_FALSE(group_indices.empty())) {
      return Status::IOError("All healthy data directories are full", "", ENOSPC);
    }
    if (PREDICT_FALSE(group_indices.size() < FLAGS_fs_target_data_dirs_per_tablet)) {
      string msg = Substitute("Could only allocate $0 dirs of requested $1 for tablet "
                              "$2. $3 dirs total", group_indices.size(),
                              FLAGS_fs_target_data_dirs_per_tablet, tablet_id, data_dirs_.size());
      if (metrics_) {
        SubstituteAndAppend(&msg, ", $0 dirs full, $1 dirs failed",
                            metrics_->data_dirs_full->value(), metrics_->data_dirs_failed->value());
      }
      LOG(INFO) << msg;
    }
  }
  InsertOrDie(&group_by_tablet_map_, tablet_id, DataDirGroup(group_indices));
  for (int uuid_idx : group_indices) {
    InsertOrDie(&FindOrDie(tablets_by_uuid_idx_map_, uuid_idx), tablet_id);
  }
  return Status::OK();
}
```

说明1：因为要读取和修改`group_by_tablet_map_`，所以在本函数中，要加`dir_group_lock_`锁。

说明2：如果“数据分布模式”不是`ACROSS_ALL_DIRS`，那么会在所有“数据根目录”中，随机选择目录。
1. 优先选择对应`tablet`较少的“数据根目录”； 
2. 如果两个“数据根目录”对应的`tablet`数相等，那么优先选择“可用剩余空间”比较多的目录。

说明3：如果在选择目录后，发现可用的目录数量 小于 `FLAGS_fs_target_data_dirs_per_tablet`，会打印一条`INFO`级别的日志，说明情况。   

日志内容是：当前可用的数据目录数量，小于期望的目录数量。

但是注意：这个时候，只是打印一条日志，并不停止创建流程。 即创建过程还会继续进行。

说明4：最终，会修改映射关系`group_by_tabelt_map_`和`tablets_by_uuid_idx_map_`。

注意：这个方法，只是修改了内存结构，并没有真正把`tablet`的相关结构内容进行持久化。

>> 问题：持久化是在什么地方进行的？

#### `DeleteDataDirGroup()`
在内存的映射结构中，删除给定`tablet`相关的内容。

```
void DataDirManager::DeleteDataDirGroup(const std::string& tablet_id) {
  std::lock_guard<percpu_rwlock> lock(dir_group_lock_);
  DataDirGroup* group = FindOrNull(group_by_tablet_map_, tablet_id);
  if (group == nullptr) {
    return;
  }
  // Remove the tablet_id from every data dir in its group.
  for (int uuid_idx : group->uuid_indices()) {
    FindOrDie(tablets_by_uuid_idx_map_, uuid_idx).erase(tablet_id);
  }
  group_by_tablet_map_.erase(tablet_id);
}
```

说明：本方法的逻辑很简单，就是维护下内存的映射结构（与`tablet_id`相关的两个映射结构）。

注意：和`CreateDataDirGroup()`一样，这个方法也只是在内存中删除，并不清理磁盘上的持久化的内容。

#### `GetDataDirGroupPB()`
给定一个`tablet_id`，找到对应的`DataDirGroup`对象，然后转化为`DataDirGroupPB`对象。

```
Status DataDirManager::GetDataDirGroupPB(const string& tablet_id,
                                         DataDirGroupPB* pb) const {
  shared_lock<rw_spinlock> lock(dir_group_lock_.get_lock());
  const DataDirGroup* group = FindOrNull(group_by_tablet_map_, tablet_id);
  if (group == nullptr) {
    return Status::NotFound(Substitute(
        "could not find data dir group for tablet $0", tablet_id));
  }
  RETURN_NOT_OK(group->CopyToPB(uuid_by_idx_, pb));
  return Status::OK();
}
```

说明：本方法的逻辑很简单，就是从内存的映射结构中找到相应的结构，然后进行转化即可。

#### `LoadDataDirGroupFromPB()`
从磁盘加载一个`DataDirGroupPB`对象，并添加到 本对象内部的相关映射关系中。

主要就是：和`tablet data dir`相关的映射关系。

```
Status DataDirManager::LoadDataDirGroupFromPB(const std::string& tablet_id,
                                              const DataDirGroupPB& pb) {
  std::lock_guard<percpu_rwlock> lock(dir_group_lock_);
  DataDirGroup group_from_pb;
  RETURN_NOT_OK_PREPEND(group_from_pb.LoadFromPB(idx_by_uuid_, pb), Substitute(
      "could not load data dir group for tablet $0", tablet_id));
  DataDirGroup* other = InsertOrReturnExisting(&group_by_tablet_map_,
                                               tablet_id,
                                               group_from_pb);
  if (other != nullptr) {
    return Status::AlreadyPresent(Substitute(
        "tried to load directory group for tablet $0 but one is already registered",
        tablet_id));
  }
  for (int uuid_idx : group_from_pb.uuid_indices()) {
    InsertOrDie(&FindOrDie(tablets_by_uuid_idx_map_, uuid_idx), tablet_id);
  }
  return Status::OK();
}
```

说明：因为要修改本对象的相关数据接口，所以要加锁（`dir_group_lock_`）;

#### `GetDirAddIfNecessary()`
给定一个`Block`相关的创建选项`CreateBlockOption`，返回对应的“数据根目录”。 
1. 如果不存在对应的目录，那么如果有空闲的“数据目录”，就添加进来。
2. 否则，就返回失败。

说明1：会先通过调用`GetDirForBlock()`来为该`tablet_id`来分配`DataDir`。

说明2：如果所有的目录都是“满”的，在为该`tablet`添加“新目录”之前，还要检查下“别的线程”是否已经为该`tablet`添加过了“数据目录”。   

检查方法是：检查当前`tablet`对应的`DataDirGroup`中数据目录的数量，是否 大于 `GetDirForBlock()`返回的`new_target_group_size`。

注意：这个检查是在`dir_group_lock_`锁保护下的。

因为在`GetDirForBlock()`中，是会先加锁（`dir_group_lock_`）,然后进行的检查，得出来的`new_target_group_size`。 但是在该函数返回后，对应的锁就已经释放了，而运行到这里重新检查（会重新加锁），这段时间内部，可能已经有别的线程为该`tablet`增加了新的目录。

如果该`tablet`已经增加了数据目录，那么就不需要再增加了，直接返回（尾部的最后一个“目录”）即可。

说明3：在调用`GetDirForBlock()`时，如果该`tablet`当前所对应的“所有数据根目录”都是满的，那么`new_target_group_size`会被设置为`“当前对应的目录数量” + 1`。 

所以，在调用`GetDirsForGroupUnlocked()`后，如果增加了“新目录”，那么也就是“只能增加了`1`个”，所以 最后一个就是新增加的目录，即可以通过`group_uuid_indices.back()`获取的，就是新增加的目录。

说明4：会调用`GetDirsForGroupUnlocked()`方法，来为当前`DataGroup`添加新的“目录”。  

说明5：如果添加成功，那么也要更新内存中的映射结构。

**注意：`DataDirManager`并不负责将这些`DataDirGroup`持久化到磁盘**  

在磁盘上，将`DataDirGroup`持久化到磁盘（放在`TabletSuperBlockPB`中），在修改它们的时候，会被重写。

# `FLAGS`变量

## `FLAGS_fs_data_dirs_available_space_cache_seconds`

在`BlockManager`中，计算出一个目录的“可用剩余大小”之后，缓存的时间（单位是“秒”）。

默认值是`10秒`。

## `FLAGS_fs_data_dirs_reserved_bytes`

对于所有的目录，都预留了一部分容量，给“非`kudu`的组件”来使用。(单位是“字节”)。

默认值为`-1`，表示在每块磁盘上，都保留磁盘空间的`1%`。

注意1：如果需要手动设置，必须是显式设置为“具体的字节数”，并且该值必须是“大于`0`的”。

注意2：目前**不支持** 以“百分比”的方式进行设置。

## `FLAGS_fs_lock_data_dirs`
在访问“数据目录”的时候，是否需要加锁来防止并发访问。

默认值是`true`，即要进行加锁。

注意：即使该属性为`true`，也是允许以`read_only`的模式并发的访问数据目录的。 因为在`read_only`模式下，是不会修改任何数据内容的，所以要允许在`read_only`模式下进行并发访问是合理的。

## `FLAGS_fs_target_data_dirs_per_tablet`
每个`tablet`的数据，应该存放在几个“数据根目录”中。

通常情况下：不同的“数据根目录”，都在不同的“磁盘”上。

默认是`3`;

1. 如果配置的值，超过了“可用的‘数据根目录’的数量”，那么数据仅仅会在这些“可用的目录”上；
2. 如果配置的值为`0`,那么数据会分布在所有 健康的数据目录中；

**取舍：**  
1. 每个tablet所使用的“数据目录”较少，也就意味着 如果一块磁盘损坏，受影响的`tablet`会比较少。
2. 如果每个tablet说使用的“数据目录较多”，那么读写请求会分摊到多块盘上，性能会好。

## `FLAGS_fs_data_dirs_consider_available_space`

在为`tablet`或`block`选择要存储的“数据根目录”的时候，是否考虑“剩余空间的大小”。

**为`tablet`选择“数据根目录”：**    
参见`GetDirsForGroupUnlocked()`.  

在随机选择出来两个“数据根目录”以后，并且两者已经保存的`tablet`数量是相同的时候。（备注：如果两者的`tablet`数量不相等，那么就选择那个“已经保存的`tablet`”较少的那个）

这时，如果`FLAGS_fs_data_dirs_consider_available_space == true`，那么就选择那个“可用剩余空间”较多的那个。

**为`Block`选择“数据根目录”：**    
参见`GetDirForBlock()`。  

如果可以候选的“数据根目录”的数量 大于等于 `2`，那么会进行随机算法，随机选择两个后，选择“可用剩余空间”较多的那个。








