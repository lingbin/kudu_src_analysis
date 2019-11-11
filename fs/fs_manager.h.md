[TOC]

文件：`src/kudu/fs/fs_manager.h`

# `struct FsManagerOpts`

## 成员

### `metric_entity`
所有的`metric`项目都会被分组。

如果该成员为`nullptr`，那么当前`FsManager`中就不会产生`metrics`信息。

默认值是`nullptr`。

### `parent_mem_tracker`

当前`FsManager`内部创建`MemTracker`对象的时候，都是将这个`parent_mem_tracker`作为 “父`MemTracker`”。

如果该成员为`nullptr`，那么新创建的`MemTracker`对象，都会将“父亲”设置为`root tracker`。

默认值是`nullptr`。

### 路径相关配置

注意：参见`FsManager::init()`的实现。

这些目录的指定，有一些要求：
1. 必须是“绝对路径”，即第一个字符必须是 `/`；
2. 每个目录的两端，不能有“空白符”。

#### `wal_root`
`WAL`日志的路径(注意：是个`dir`，不是`file`)。

注意：这个值必须被赋值（不能为空）。

#### `data_roots`
数据目录。（注意：每个元素都是`dir`）

说明：**是一个列表**。如果这些目录是在不同的磁盘上，那么就相当于使用了“多盘”。

注意：如果这个属性没有设置，那么会使用`wal_root`。

#### `metadata_root`
存储元数据的目录。  

注意：如果这个属性没有设置，那么这个属性的值有 两种情况：
1. 如果元数据还不存在，那么和`data_roots`一样，会使用`wal_root`；
2. 如果元数据已经存在了，那么使用`data_roots[0]`；

说明：在`kudu 1.6`之前，如果没有设置，那么就使用`data_roots`的第一个路径（即`data_roots[0]`）。

所以现在关于`metadata_root`的确定方法是：  
检查 “`data_roots[0]`下是否有文件，
1. 如果有，那么就使用这个元素；
2. 如果没有，就使用`wal_root`;

### `block_manager_type`
`BlockManager`的类型。

取值有两种：1) `file`； 2) `log`；

默认取值是`FLAGS_block_manager`，该`flag`的默认值与“操作系统”有关，对于`linux`,默认值是`log`。

### `read_only`

当前`FsManager`是否只支持“读操作”。

默认值是`false`，即默认支持“读写”。

### `consistency_check`
检查`data_roots`中各个目录中的“元数据文件”中记录的内容，和当前配置的“data_roots”是否一致。

```
struct FsManagerOpts {
  FsManagerOpts();

  explicit FsManagerOpts(const std::string& root);

  scoped_refptr<MetricEntity> metric_entity;

  std::shared_ptr<MemTracker> parent_mem_tracker;

  std::string wal_root;

  std::vector<std::string> data_roots;

  std::string metadata_root;

  std::string block_manager_type;
  
  bool read_only;

  fs::ConsistencyCheckBehavior consistency_check;
```

##  接口列表

### 构造函数
有两个构造函数。

大部分属性，都是使用“默认值”进行构造。

有参数的‘构造函数’，是指定了`wal_root`属性（这个构造函数，只在单测中使用）。

```
FsManagerOpts::FsManagerOpts()
  : wal_root(FLAGS_fs_wal_dir),
    metadata_root(FLAGS_fs_metadata_dir),
    block_manager_type(FLAGS_block_manager),
    read_only(false),
    consistency_check(ConsistencyCheckBehavior::ENFORCE_CONSISTENCY) {
  data_roots = strings::Split(FLAGS_fs_data_dirs, ",", strings::SkipEmpty());
}

FsManagerOpts::FsManagerOpts(const string& root)
  : wal_root(root),
    data_roots({ root }),
    block_manager_type(FLAGS_block_manager),
    read_only(false),
    consistency_check(ConsistencyCheckBehavior::ENFORCE_CONSISTENCY) {}
```

# `class FsManager`

作用：
1. 统一管理对 “数据文件”和“元数据”的读取；
2. 负责管理“文件目录结构”。

上层用户的接口是这个类

在上层用户中，不应该知道文件的具体位置。而只是调用该类的相应接口：
+ open the block xyz
+ write a new schema metadata file for table kwz

目前的目录组织结构为：
```
//    <kudu.root.dir>/data/
//    <kudu.root.dir>/data/<prefix-0>/<prefix-2>/<prefix-4>/<name>
```

>> 问题：这里的`prefix-x`是什么意思？

## 成员

### `env_`
封装底层文件系统的所有接口；

### `opts_`
创建当前`FsManager`的选项；

### 路径相关的属性

这几个“和路径相关的属性”都是在`Init()`中进行初始化的。

对于类型是“列表”的属性，其中的“目录”是经过去重的。

#### `canonicalized`是对目录进行的转化内容

>> canonicalized: 规范化  

>> **就是将一个路径转化为“绝对路径”**。

参见：`src/kudu/util/env.h`中的`Env::Canonicalize()`方法。

具体的规则是：
1. 将一个“相对路径”转化为“绝对路径”；
2. 将其中的`"."`和`".."`都进行转化，从而消除掉；
3. 解析出“软链”。

#### `canonicalized_wal_fs_root_`
经过规范化的、保存`wal`的目录；

#### `canonicalized_metadata_fs_root_`
经过规范化的、保存`metadata`的目录；

#### `canonicalized_data_fs_roots_`
经过规范化的、保存“数据”的目录；

说明：这是一个列表。

注意：第一个“数据目录”会被用来保存“元数据”。

#### `canonicalized_all_fs_roots_`
经过规范化的、上述三者的目录合集；

说明：这是一个列表。

### `metadata_`
类型为`InstanceMetadataPB`，当前节点的“元数据”。

注意区分：不是`PathInstanceMetadataPB`对象。

### `error_manager_`
当前节点的`FsErrorManager`。

### `dd_manager_`
当前节点的`DataDirManager`。

### `block_manager_`
当前节点的`BlockManager`。

### `oid_generator_`
用来生成`32`位的`UUID`。

### `initted_`
标识当前对象是否已经经过“初始化”。

```
  Env* env_;

  const FsManagerOpts opts_;

  CanonicalizedRootAndStatus canonicalized_wal_fs_root_;
  CanonicalizedRootAndStatus canonicalized_metadata_fs_root_;
  CanonicalizedRootsList canonicalized_data_fs_roots_;
  CanonicalizedRootsList canonicalized_all_fs_roots_;

  std::unique_ptr<InstanceMetadataPB> metadata_;

  std::unique_ptr<fs::FsErrorManager> error_manager_;
  std::unique_ptr<fs::DataDirManager> dd_manager_;
  std::unique_ptr<fs::BlockManager> block_manager_;

  ObjectIdGenerator oid_generator_;

  bool initted_;
```

## 定义的一些“路径”相关的常量

```
const char *FsManager::kWalDirName = "wals";
const char *FsManager::kWalFileNamePrefix = "wal";
const char *FsManager::kWalsRecoveryDirSuffix = ".recovery";
const char *FsManager::kTabletMetadataDirName = "tablet-meta";
const char *FsManager::kDataDirName = "data";
const char *FsManager::kCorruptedSuffix = ".corrupted";
const char *FsManager::kInstanceMetadataFileName = "instance";
const char *FsManager::kConsensusMetadataDirName = "consensus-meta";
```

说明：从变量名字中可以进行区分，所对应的是一个“目录”，还是一个“文件”。

## 接口列表

### `private`方法

#### `Init()`

>> sanitize: 使…无害；给…消毒  
>> canonicalizes: 规范化转换。 这里指的就是转化为“绝对路径”。  

初始化、规范化 所配置的各种目录。并为“特定tablet的元数据”最终确定相应的目录。

说明1：必须显示的指定`wal_root`，否则报错。
只要指定了`wal_root`，没有指定其它的目录（`data_roots`和`metadata_root`），那么会根据`wal_root`来自动指定。

说明2：`StripWhiteSpace()`（参见：`src/kudu/gutil/strings/strip.cc`）的作用，是清理一个字符串两端的“空白符”。

说明3：在调用`Env::Canonicalize()`的时候，传入的参数是，当前指定的目录的“父目录”。   
原因是：有可能用户指定的目录是不存在的，但是“父目录”必须是存在的。

如果“父目录”也不存在，那么就直接使用“用户指定的目录”，但是会将该目录标识为“失败的”（`CanonicalizedRootAndStatus`对象中有两个成员，一个是`path`, 一个是`status`。这里表示失败，就是将这个`status`成员标识为失败。）。

说明4：在将所有目录都“规范化”以后，会赋值给对应的各个属性变量。

说明5：如果`canonicalized_wal_fs_root_`和`canonicalized_metadata_fs_root_`进行“规范化”失败了，那么当前进程就无法成功启动。

也就是说，只有存放“数据”的`canonicalized_data_fs_roots_`中，允许有部分目录是不存在的。

说明6：如果没有指定“数据目录”，那么使用`canonicalized_wal_fs_root_`作为数据目录；

说明7：如果没有指定“metadata目录”，那么分为两种情况：
1. 如果“第一个数据目录”中存在`"tablet-meta"`子目录，那么`metadata_fs_root_`就使用“第一个数据目录”；
2. 如果不存在，那么使用`wal_fs_root_`。

#### `InitBlockManager()`

创建`BlockManager`对象。

注意：这个方法，只是初始化当前对象中的`block_manager_`对象属性，不会进行任何磁盘的操作。

### `public`方法

#### `PartialOpen()`

#### `Open()`

“初始化” 和 “加载”基本的文件系统的元信息，并检查是否一致。

在非`read-only`模式下，如果发现了不一致，会尝试进行修复。

步骤1： 会先遍历所有的目录，加载`InstanceMetadataPB`。 如果不存在该文件的目录，会记录下来（放到`missing_roots`中）；

步骤2：要保证所有“辅助的目录”都已经存在。

>> ancillary: 辅助的；  

“辅助的目录”包括：
1. `wal root dir`: `wal_root/wals`;
2. `tablet metadata dir`: `metadata_root/tablet-meta`;
3. `consensus metadata dir`: `metadata_root/consensus-meta`;

注意：这3个都是“目录”，不是文件。而且这3个“目录”必须是已经存在的。

步骤3：如果`opts_.consistency_check == ConsistencyCheckBehavior::UPDATE_ON_DISK`，那么对于没有`InstanceMetadataPB`的目录，在磁盘上进行创建。

注意：这些没有`/root/instance`文件的目录，只能是两种情况之一：
1. 是“不存在的”目录；
2. 如果已经存在，那么必须是“空目录”。

步骤4：读取磁盘，初始化`DataDirManger`对象

步骤5：如果不是`read_only`模式，那么清理磁盘上的“临时文件”，并检查对所有“根目录”的权限。

步骤6：向“错误管理器”(`error_manager_`)中，针对`DISK_ERROR`，注册一个错误处理函数。

步骤7：初始化`BlockManager`。

步骤8：如果开启了`fsync`，将创建的“目录”和“文件”，都进行`fsync`。

步骤9：打印一些INFO信息。 已经打开的目录；新创建的目录；当前的`instance metadata`；

#### `CreateInitialFileSystemLayout()`

步骤1：调用`Init()`进行一些基本的初始化；

步骤2：在所有的“根目录”中，创建`instance metadata`文件。

步骤3：创建“辅助目录”；

“辅助的目录”包括：
1. `wal root dir`: `wal_root/wals`;
2. `tablet metadata dir`: `metadata_root/tablet-meta`;
3. `consensus metadata dir`: `metadata_root/consensus-meta`;

步骤4：创建`DataDirManager`对象；

步骤5：如果开启了`fsync`，将创建的“目录”和“文件”，都进行`fsync`。

#### 获取“路径”的方法

```
vector<string> FsManager::GetDataRootDirs() const {
  // Get the data subdirectory for each data root.
  return dd_manager_->GetDataDirs();
}
  
  std::string GetWalsRootDir() const {
    DCHECK(initted_);
    return JoinPathSegments(canonicalized_wal_fs_root_.path, kWalDirName);
  }

  std::string GetTabletWalDir(const std::string& tablet_id) const {
    return JoinPathSegments(GetWalsRootDir(), tablet_id);
  }
  
string FsManager::GetTabletWalRecoveryDir(const string& tablet_id) const {
  string path = JoinPathSegments(GetWalsRootDir(), tablet_id);
  StrAppend(&path, kWalsRecoveryDirSuffix);
  return path;
}

string FsManager::GetWalSegmentFileName(const string& tablet_id,
                                        uint64_t sequence_number) const {
  return JoinPathSegments(GetTabletWalDir(tablet_id),
                          strings::Substitute("$0-$1",
                                              kWalFileNamePrefix,
                                              StringPrintf("%09" PRIu64, sequence_number)));
}

string FsManager::GetTabletMetadataDir() const {
  DCHECK(initted_);
  return JoinPathSegments(canonicalized_metadata_fs_root_.path, kTabletMetadataDirName);
}

// Return the path for a specific tablet's superblock.
string FsManager::GetTabletMetadataPath(const string& tablet_id) const {
  return JoinPathSegments(GetTabletMetadataDir(), tablet_id);
}

// List the tablet IDs in the metadata directory.
Status FsManager::ListTabletIds(vector<string>* tablet_ids) {
  string dir = GetTabletMetadataDir();
  vector<string> children;
  RETURN_NOT_OK_PREPEND(ListDir(dir, &children),
                        Substitute("Couldn't list tablets in metadata directory $0", dir));

  vector<string> tablets;
  for (const string& child : children) {
    if (!IsValidTabletId(child)) {
      continue;
    }
    tablet_ids->push_back(child);
  }
  return Status::OK();
}

// Return the path where InstanceMetadataPB is stored.
string FsManager::GetInstanceMetadataPath(const string& root) const {
  return JoinPathSegments(root, kInstanceMetadataFileName);
}

  // Return the directory where the consensus metadata is stored.
  std::string GetConsensusMetadataDir() const {
    DCHECK(initted_);
    return JoinPathSegments(canonicalized_metadata_fs_root_.path, kConsensusMetadataDirName);
  }

  // Return the path where ConsensusMetadataPB is stored.
  std::string GetConsensusMetadataPath(const std::string& tablet_id) const {
    return JoinPathSegments(GetConsensusMetadataDir(), tablet_id);
  }
```

```
路径

说明：在末尾有'/'的表示目录；没有'/'的表示一个文件。

### wal相关
    $wal_root/
        |--"instance"
        |--"wals"/
            |--$tablet_id_1/
                |--"wal-"$(seq_num)
            |--$(tablet_id_1)".recovery"
            |--$tablet_id_2/
                |--"wal-"$(seq_num)
            |--$(tablet_id_2)".recovery"
         
    说明：`$(seq_num)`是一个 9位的整型，如果位数不够，会填充前导 0。 
    
     
### data_roots相关
    $data_root_1/
        |--"instance"
        |--"data"/
            |--"block_manager_instance"


### metadata相关
    $metadata_root/
        |--"instance"
        |--"tablet-meta"/
            |--$tablet_id_1
            |--$tablet_id_2
        |--"consensus-meta"/
            |--$tablet_id_1
            |--$tablet_id_2
```


















# 问题：

1. 如果指定的`wal_root`, `data_root`, `metadata_root`不存在，会是什么结果？是自动创建，还是启动失败？

