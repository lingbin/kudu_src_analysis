[TOC]

文件：`src/kudu/fs/file_block_manager.h`

# `class FileBlockManager`

一个“基于文件”的`BlockManager`实现，是`BlockManager`的子类。

每个`Block`都会对应自己在磁盘上的文件。

为了防止`Block`所对应的目录太大，将目录分为`3`个层级，具体的划分方式参见`FileBlockLocation`。

**使用`Block_id`进行交互**  
该`FileBlockManager`对象，可以充分利用 “多路径”的优势。  
在将一个`Block`写入到指定路径时，会赋予它一个`ID`（通过解析该`block_id`，能够包含足够的信息（主要是`3`层子目录的信息），从而可以去唯一的识别对应的路径）。

在为了读取而打开一个`Block`的时候，这个`ID`会解析成一个“文件系统路径”。

这个`ID`的结构，限制了一个`BlockManager`最多支持`65535`块磁盘(即“数据根目录”的个数最多有 `2^16`个，即“目录的标识”是一个`int16_t`类型)。

说明：在创建`Block`时，使用`CreateBlockOptions`结构。

>> 备注： 在源代码中，有时会将`FileBlockManager`缩写为`FBM`.

## 成员

### `env_`
基础`Env`对象，提供了对“底层文件系统”交互的所有接口，用来管理文件。

### `dd_manager_`
`DataDirManager`类型的指针。用来进行“目录管理”。

当前`FileBlockManager`对象，会在这些目录中 存放数据。

### `error_manager_`
用来管理磁盘操作出错时的“回调函数”。

### `opts_`
创建当前对象时，所给的`BlockManagerOptions`参数。

参见`src/kudu/fs/block_manager.h`.

### `file_cache_`
文件缓存，用来缓存已经打开的文件句柄。

### `rand_`
用来随机地生成`block_id`。

### `next_block_id_`
下一个`block_id`

>> 问题：为什么不是每次调用“随机函数”来随机生成？

### `lock_`
用来保护对`dirty_dirs_`的“修改”和“访问”。

### `dirty_dirs_`
从一个`Block`所对应的目录 开始创建时，就持续的跟踪该目录，标记它是否为`dirty`的。

**这个属性的作用是**：  
可以帮助我们进行 将“元数据” 刷盘的时候，方便进行一些合并操作。

> 说明：这里的“元数据”，是指一个“文件”或“目录”在文件系统中的元数据。

>> coalescing: 合并  

>> 问题：如何进行合并？

### `metrics_`
当前`BlockManager`的`metric`容器。

注意：这个属性有可能为`nullptr`（表示不采集相关的`metric`信息）。

### `mem_tracker_`
跟踪记录对象的内存消耗。

```
  Env* env_;

  DataDirManager* dd_manager_;

  FsErrorManager* error_manager_;

  const BlockManagerOptions opts_;

  FileCache<RandomAccessFile> file_cache_;

  ThreadSafeRandom rand_;
  AtomicInt<uint64_t> next_block_id_;

  mutable simple_spinlock lock_;
  std::unordered_set<std::string> dirty_dirs_;

  std::unique_ptr<internal::BlockManagerMetrics> metrics_;

  std::shared_ptr<MemTracker> mem_tracker_;

  DISALLOW_COPY_AND_ASSIGN(FileBlockManager);
```

## 接口列表

### 需要实现的虚函数列表

#### `Open()`

```
Status FileBlockManager::Open(FsReport* report) {
  RETURN_NOT_OK(file_cache_.Init());

  // Prepare the filesystem report and either return or log it.
  FsReport local_report;
  set<int> failed_dirs = dd_manager_->GetFailedDataDirs();
  for (const auto& dd : dd_manager_->data_dirs()) {
    // Don't report failed directories.
    // TODO(KUDU-2111): currently the FsReport only reports on containers for
    // the log block manager. Implement some sort of reporting for failed
    // directories as well.
    if (PREDICT_FALSE(!failed_dirs.empty())) {
      int uuid_idx;
      CHECK(dd_manager_->FindUuidIndexByDataDir(dd.get(), &uuid_idx));
      if (ContainsKey(failed_dirs, uuid_idx)) {
        continue;
      }
    }
    // TODO(adar): probably too expensive to fill out the stats/checks.
    local_report.data_dirs.push_back(dd->dir());
  }
  if (report) {
    *report = std::move(local_report);
  } else {
    RETURN_NOT_OK(local_report.LogAndCheckForFatalErrors());
  }
  return Status::OK();
}
```

说明1：该方法，是在初始化`BlockManager`对象的时候，会调用该函数一次。

说明2：对于`FileBlockManager`来说，这个`Open()`方法没做什么实际的事情。 只是在`report`参数不为`nullptr`的时候，将当前的“数据根目录”添加到`FsReport`中。

注意：在这个函数中，对于 访问失败的目录，并不会添加到`FsReport`中。

>> 问题：这些目录，为什么不添加？难道是`FsReport::data_dirs`就只是用来记录“成功访问的路径”？

>> 问题：该方法中，对于`local_report`对象，只是修改了`data_dirs`成员，所以一定是没有`fatal`级别的问题。但是这里最后却还调用了`LogAndCheckForFatalErrors()`，难道只是为了“打印日志”？还是说可以去掉。

#### `CreateBlock()`

一些说明，参见`BlockManager::CreateBlock()`的说明（文件：`src/kudu/fs/block_manager.h`）。

```
Status FileBlockManager::CreateBlock(const CreateBlockOptions& opts,
                                     unique_ptr<WritableBlock>* block) {
  CHECK(!opts_.read_only);

  DataDir* dir;
  RETURN_NOT_OK_EVAL(dd_manager_->GetDirAddIfNecessary(opts, &dir),
      error_manager_->RunErrorNotificationCb(ErrorHandlerType::NO_AVAILABLE_DISKS, opts.tablet_id));
  int uuid_idx;
  CHECK(dd_manager_->FindUuidIndexByDataDir(dir, &uuid_idx));

  string path;
  vector<string> created_dirs;
  Status s;
  internal::FileBlockLocation location;
  shared_ptr<WritableFile> writer;

  int attempt_num = 0;
  // Repeat in case of block id collisions (unlikely).
  do {
    created_dirs.clear();

    // If we failed to generate a unique ID, start trying again from a random
    // part of the key space.
    if (attempt_num++ > 0) {
      next_block_id_.Store(rand_.Next64());
    }

    // Make sure we don't accidentally create a location using the magic
    // invalid ID value.
    BlockId id;
    do {
      id.SetId(next_block_id_.Increment());
    } while (id.IsNull());

    location = internal::FileBlockLocation::FromParts(dir, uuid_idx, id);
    path = location.GetFullPath();
    s = location.CreateBlockDir(env_, &created_dirs);

    // We could create a block in a different directory, but there's currently
    // no point in doing so. On disk failure, the tablet specified by 'opts'
    // will be shut down, so the returned block would not be used.
    RETURN_NOT_OK_HANDLE_DISK_FAILURE(s,
        error_manager_->RunErrorNotificationCb(ErrorHandlerType::DISK_ERROR, dir));
    WritableFileOptions wr_opts;
    wr_opts.mode = Env::CREATE_NON_EXISTING;
    s = env_util::OpenFileForWrite(wr_opts, env_, path, &writer);
  } while (PREDICT_FALSE(s.IsAlreadyPresent()));
  if (s.ok()) {
    VLOG(1) << "Creating new block " << location.block_id().ToString() << " at " << path;
    {
      // Update dirty_dirs_ with those provided as well as the block's
      // directory, which may not have been created but is definitely dirty
      // (because we added a file to it).
      std::lock_guard<simple_spinlock> l(lock_);
      for (const string& created : created_dirs) {
        dirty_dirs_.insert(created);
      }
      dirty_dirs_.insert(DirName(path));
    }
    block->reset(new internal::FileWritableBlock(this, location, writer));
  } else {
    HANDLE_DISK_FAILURE(s,
        error_manager_->RunErrorNotificationCb(ErrorHandlerType::DISK_ERROR, dir));
    return s;
  }
  return Status::OK();
}
```

说明1：首先通过调用`DataDirManager::GetDirAddIfNecessary()`为当前`Block`选一个“数据根目录”。（如果有必要的话，会为它所属的`tablet`添加一个“数据根目录”）。

说明2：第一轮循环时，`BlockId`是通过“递增`next_block_id_`”来生成的，这样的好处是：在没有冲突的情况下，生成的`BlockId`是连续的。

但是如果在有冲突的情况下，会另`next_block_id_`重新取一个随机值。

也就是说，如果遇到“冲突”（即某个`blockId`已经被别人取值过了），那么再生成的`BlockId`（因为会从一个“随机值”重新开始），所以就和之前的“`BlockId`”值不再连续了。

说明3：通过递增`next_block_id_`获取一个`uint64_t`类型数，和`DataDir*`一起组成成`FileBlockLocation`对象。  

**组装成`FileBlockLocation`对象以后，对应的`BlockId`也就确定了。**  

>> 备注：最终在`FileBlockLocation`中，只会保留这个“随机数”的后`6`个字节（前`2`个字节，是这个`DataDir*`的`uuid idx`）。

说明4：当前函数会去 创建该`Block`所对应的 `3`层子目录。

**说明5：创建的`Block`的目录，可能和现有的`Block`是重复的。**

因为随机生成的`uint64_t`中，在最终的`BlockId`中只会被使用“后`6`个字节”。

所以有可能出现如下情况：

1. 两个`BlockId`中，后`6`个字节是完全相同的（即使前2个字节有不同，但是这2个字节不会用来组成`BlockId`）。  
    这时如果选择的“数据根目录”也相同，那么最终生成的`BlockId`就是完全相同的。 那么对应的“存储的绝对路径”也就完全相同，如果遇到这种情况，就重新生成`BlockId`。
2. 两个`BlockId`中，前`5`个字节直接是完全相同的，后3个字节不同。  
    这时如果选择的“数据根目录”也相同，那么最终生成的`BlockId`中，前5个字节也是相同的。  
    也就是说，这两个`Block`对应的“目录层级结构”是相同的，但是最终的“文件名”是不同的。    
    参照下面的举例。

```
// 两个`block_id`，“前5个”字节是相同的。
              | data_dir |    sub_dir      |
                  0     1     2     3     4     5     6     7    
    block_id_1:  aaaa  bbbb  cccc  dddd  eeee  xyzx  yzxy  zxyz
    block_id_2:  aaaa  bbbb  cccc  dddd  eeee  hijk  lmno  pqrs
    
那么生成的 目录结构是：
    |-- data_dir_root
         |-- byte2
              |-- byte3
                   |-- byte4
                        |-- block_id_1
                        |-- block_id_2

即两个`Block`使用相同的“目录层级结构”。

```

说明6： 对于新创建的目录，都会被加入到`dirty_dirs_`中。表示这些目录下的“文件列表”发生了变化。

这些目录有两部分：
1. `FileBlockLocation::CreateBlockDir()`返回的目录；  
    注意：这个方法返回的目录，是“`数据根目录/`”，“`数据根目录/byte2/`”，“`数据根目录/byte2/byte3/`”
2. 当前`Block`的最底层的目录。（即“`数据根目录/byte2/byte3/byte4`”）：对应代码中的`dirty_dirs_.insert(DirName(path));`。

#### `OpenBlock()`

打开一个`Block`。

注意：这里是为了“读取”才打开的`Block`，所以返回的对象是`FileReadableBlock`类型。  
又因为这里是面向接口编程，所以真正返回的类型是`ReadableBlock`类型。

所以，对于上层用户来说，并不感知底层是`ReadableBlock`的具体类型（`FileReadableBlock`或`LogReadableBlock`）。

此函数的逻辑很简单：
1. 从所给的`BlockId`找到 该`Block`的数据文件的完整路径；
2. 打开该文件，并新建对应的`FileReadableBlock`对象。

```
Status FileBlockManager::OpenBlock(const BlockId& block_id,
                                   unique_ptr<ReadableBlock>* block) {
  string path;
  if (!FindBlockPath(block_id, &path)) {
    return Status::NotFound(
        Substitute("Block $0 not found", block_id.ToString()));
  }

  VLOG(1) << "Opening block with id " << block_id.ToString() << " at " << path;

  shared_ptr<RandomAccessFile> reader;
  RETURN_NOT_OK_FBM_DISK_FAILURE(file_cache_.OpenExistingFile(path, &reader));
  block->reset(new internal::FileReadableBlock(this, block_id, reader));
  return Status::OK();
}
```

#### 创建事务的两个方法

##### `NewCreationTransaction()`
将创建一组`FileWritableBlock`的任务，打包起来。

##### `NewDeletionTransaction()`
将删除一组`block`的任务，打包起来。

这两个函数都比较简单，就是新建相应类型的对象即可。

```
unique_ptr<BlockCreationTransaction> FileBlockManager::NewCreationTransaction() {
  CHECK(!opts_.read_only);
  return unique_ptr<internal::FileBlockCreationTransaction>(
      new internal::FileBlockCreationTransaction());
}

shared_ptr<BlockDeletionTransaction> FileBlockManager::NewDeletionTransaction() {
  CHECK(!opts_.read_only);
  return std::make_shared<internal::FileBlockDeletionTransaction>(this);
}
```

#### `GetAllBlockIds()`
获取当前`BlockManager`所管理的所有的`Block`的`id`，包括所有的`ReadableBlock`和`WritableBlock`。

**注意：对于`FileBlockManager`，并不在内存中维护着 所有`Block`的列表。**  

所以，为了获取所有的`BlockId`，需要遍历目录才可以。

因为一个进程中，会配置多个“数据根目录”，为了加快遍历的速度，这里是通过每个“数据根目录”的“线程池”提交task，让它们并行的扫描目录来实现的。

在分别向每个线程池，提交了作业以后，会等待所有的线程池就结束工作。

注意：如果有任何一个“数据根目录”，在扫描过程中，出现了错误，那么这个函数返回错误。

```
Status FileBlockManager::GetAllBlockIds(vector<BlockId>* block_ids) {
  const auto& dds = dd_manager_->data_dirs();
  block_ids->clear();

  // The FBM does not maintain block listings in memory, so off we go to the
  // filesystem. The search is parallelized across data directories.
  vector<vector<BlockId>> block_id_vecs(dds.size());
  vector<Status> statuses(dds.size());
  for (int i = 0; i < dds.size(); i++) {
    dds[i]->ExecClosure(Bind(&GetAllBlockIdsForDataDir,
                             env_,
                             dds[i].get(),
                             &block_id_vecs[i],
                             &statuses[i]));
  }
  for (const auto& dd : dd_manager_->data_dirs()) {
    dd->WaitOnClosures();
  }

  // A failure on any data directory is fatal.
  for (const auto& s : statuses) {
    RETURN_NOT_OK(s);
  }

  // Collect the results into 'blocks'.
  for (const auto& ids : block_id_vecs) {
    block_ids->insert(block_ids->begin(), ids.begin(), ids.end());
  }
  return Status::OK();
}
```
##### `GetAllBlockIdsForDataDirCb()`

```
namespace {
Status GetAllBlockIdsForDataDirCb(DataDir* dd,
                                  vector<BlockId>* block_ids,
                                  Env::FileType file_type,
                                  const string& dirname,
                                  const string& basename) {
  if (file_type != Env::FILE_TYPE) {
    // Skip directories.
    return Status::OK();
  }

  uint64_t numeric_id;
  if (!safe_strtou64(basename, &numeric_id)) {
    // Skip files with non-numerical names.
    return Status::OK();
  }

  // Verify that this block ID look-alike is, in fact, a block ID.
  //
  // We could also verify its contents, but that'd be quite expensive.
  BlockId block_id(numeric_id);
  internal::FileBlockLocation loc(
      internal::FileBlockLocation::FromBlockId(dd, block_id));
  if (loc.GetFullPath() != JoinPathSegments(dirname, basename)) {
    return Status::OK();
  }

  block_ids->push_back(block_id);
  return Status::OK();
}

void GetAllBlockIdsForDataDir(Env* env,
                              DataDir* dd,
                              vector<BlockId>* block_ids,
                              Status* status) {
  *status = env->Walk(dd->dir(), Env::PRE_ORDER,
                      Bind(&GetAllBlockIdsForDataDirCb, dd, block_ids));
}

} // anonymous namespace
```

说明1：在遍历目录的时候，会对所有的目录进行判断。

1. 如果是一个目录，那么它一定不是`Block`的文件，所以直接忽略；
2. 如果是一个文件，那么要求它的必须是一个有效的整型；
3. 如果也是一个有效的整形，那么还要检查是否为一个有效的`BlockId`。 

```
    |-- data_dir_root
         |-- byte2
              |-- byte3
                   |-- byte4
                        |-- block_id_1
                        |-- block_id_2
    
    在遍历的时候，对于“数据根目录”和“3层子目录”，仅通过上面的第1条（忽略目录），就可以全部都忽略掉。
```

说明2：检查一个整型数值，是否为有效的`BlockId`的方法是：  
这里是通过构建`FileBlockLocation`对象，并重新生成目录(`GetFullPath()`)，并查看和“当前要判断的目录”是否相同。如果不同，那么说明不是一个有效的`BlockId`。

注意：这个判断方法，并不是是非常精确的。  
比如，如果手动在相应的目录，按照“3级子目录”的规则，手动创建了一个名字为“整型”的文件，那么也会被该方法认为是一个`Block`。

一个更准确的方法，是通过检查文件的内容，但这个方法比较费，这里就没有采用。

#### `NotifyBlockId()`

在`FileBlockManager`中，因为不在内存中维护`BlockId`，所以这个方法没有任何用途。

是一个空的实现。





### `private`方法

#### `private FindBlockPath()`
给定一个`block_id`，返回它的对应的“目录”。（是个“绝对路径”，‘具体的值’是“数据根目录” + “3级子目录”）

函数的返回值是`bool`类型，表示是否存在该`Block`。

```
bool FileBlockManager::FindBlockPath(const BlockId& block_id,
                                     string* path) const {
  DataDir* dir = dd_manager_->FindDataDirByUuidIndex(
      internal::FileBlockLocation::GetDataDirIdx(block_id));
  if (dir) {
    *path = internal::FileBlockLocation::FromBlockId(
        dir, block_id).GetFullPath();
  }
  return dir != nullptr;
}
```

#### `private DeleteBlocks()`

给定一个`block_id`，删除掉它对应的文件（即，会所占用的文件系统空间会被释放出来）。

注意1：这个修改会立即被持久化到磁盘。即“占用的空间”会立即被释放。

注意2：如果所要删除的`Block`，正在被“读取或者写入”，那么“真正的删除操作”会被延迟到“该`Block`的所有‘读者’和‘写者’”都被关闭以后。即在该`Block`上的所有`FileReadableBlock`和`FileWritableBlock`都被关闭以后，再进行删除文件。

```
Status FileBlockManager::DeleteBlock(const BlockId& block_id) {
  CHECK(!opts_.read_only);

  // Return early if deleting a block in a failed directory.
  set<int> failed_dirs = dd_manager_->GetFailedDataDirs();
  if (PREDICT_FALSE(!failed_dirs.empty())) {
    int uuid_idx = internal::FileBlockLocation::GetDataDirIdx(block_id);
    if (ContainsKey(failed_dirs, uuid_idx)) {
      LOG_EVERY_N(INFO, 10) << Substitute("Block $0 is in a failed directory; not deleting",
                                          block_id.ToString());
      return Status::IOError("Block is in a failed directory");
    }
  }

  string path;
  if (!FindBlockPath(block_id, &path)) {
    return Status::NotFound(
        Substitute("Block $0 not found", block_id.ToString()));
  }
  RETURN_NOT_OK_FBM_DISK_FAILURE(file_cache_.DeleteFile(path));

  // We don't bother fsyncing the parent directory as there's nothing to be
  // gained by ensuring that the deletion is made durable. Even if we did
  // fsync it, we'd need to account for garbage at startup time (in the
  // event that we crashed just before the fsync), and with such accounting
  // fsync-as-you-delete is unnecessary.
  //
  // The block's directory hierarchy is left behind. We could prune it if
  // it's empty, but that's racy and leaving it isn't much overhead.

  return Status::OK();
}
```

说明1：如果要删除的`Block`，所属的“数据根目录”是一个“损坏的”目录，那么直接返回失败。

说明2：在删除一个`Block`的时候，因为`Block`是一个文件，所以也要在`file_cache_`中删除当前`Block`对应的文件缓存。

说明3：真正的删除文件操作，是在`FileCache::DeleteFile()`中进行的。

如果当前`Block`正在被读取或写入，那么文件的删除，那么`FileCache`来保证真正的删除操作，会延迟到所有的“读写操作”都结束以后。

说明4：`Block`文件的“3级子目录”的层级结构，并没有被删除。

>> 问题：这些“子目录”是永远不会被删除？还是说会有专门的线程进行后台清理？或者启动的时候进行检查和清理？

说明5：在本方法中，并没有对该`Block`文件所属的“目录”进行`fsync()`。是因为进行`fsync()`是没有收益的。

对“目录”进行`fsync()`，唯一可能会有的好处是：可能在进程启动的时候，我们的程序不需要手动进行“垃圾清理”了。

但是即使这里进行了`fsync()`，也必须在进程启动的时候，仍然需要进行“垃圾整理”。因为有可能是在`fsync()`之前系统就崩溃了。

也就是说，所以这里对“目录”进行`fsync()`，没有任何好处。

#### `private SyncMetadata()`

对给定的`Block`所属的“目录”，进行`fsync()`操作。

有关这里“元数据”("metadata")的含义，是指`Block`文件所对应的多个“目录”，参见`FileWritableBlock::Close(SyncMode)`的说明。

```
Status FileBlockManager::SyncMetadata(const internal::FileBlockLocation& location) {
  vector<string> parent_dirs;
  location.GetAllParentDirs(&parent_dirs);

  // Figure out what directories to sync.
  vector<string> to_sync;
  {
    std::lock_guard<simple_spinlock> l(lock_);
    for (const string& parent_dir : parent_dirs) {
      if (dirty_dirs_.erase(parent_dir)) {
        to_sync.push_back(parent_dir);
      }
    }
  }

  // Sync them.
  if (FLAGS_enable_data_block_fsync) {
    for (const string& s : to_sync) {
      if (metrics_) metrics_->total_disk_sync->Increment();
      RETURN_NOT_OK_HANDLE_DISK_FAILURE(env_->SyncDir(s),
          error_manager_->RunErrorNotificationCb(ErrorHandlerType::DISK_ERROR,
                                                 location.data_dir()));
    }
  }
  return Status::OK();
}
```

说明1：因为每个`Block`是在某个“数据根目录”下，还有`3`级“子目录”（一共会有`4`个目录）。这里会将 `4`个目录，都分别调用`fsync()`。

说明2：因为目录是有层级的，最好的顺序是“先对‘子目录’进行`fsync()`，再对‘父目录’进行`fsync()`”。  
在`FileBlockLocation::GetAllParentDirs()`中，“数据根目录”和3级“子目录”，是按照“从底向上”的顺序添加的，即“先添加子目录，后添加父目录”的顺序。

所以，在执行`fsync()`的时候，按照“从前向后”的顺序，逐个遍历`parent_dirs`中的元素即可。

说明3：只有在`dirty_dirs_`中的目录，才需要进行`fsync()`。

因为在`dirty_dirs_`中的目录，记录的是之前有过修改的目录。如果一个目录不在该“目录列表”中，那么就就说明该目录没有被修改，所以也就不需要进行对该目录进行`fsync()`。

因为对于每个目录，要检查在`dirty_dirs_`中是否包含，为了防止并发访问，在检查的时候，要进行加锁（`lock_`）

说明4：如果开启了`fsync()`（即`FLAGS_enable_data_block_fsync == true`），才会真正的对这些目录进行`fsync()`操作。

说明5：无论是否开启了`fsync()`，都会将这个`Block`所对应的“`4`个目录”，从`dirty_dirs_`中删除。

# `class FileBlockLocation` -- 内部类

在`.cpp`文件中定义，只有本类可用。

表示一个`Block`对象，在它对应的`file block manager`中的“逻辑位置”。

说明：一个`block ID`就可以唯一的定位该`Block`所对应的文件。每个`ID`都是`uint64_t`类型，并且可以拆分为多个逻辑组成部分。

1. 前`2`个字节：标识 这个`block`的“数据根目录”。参见`src/kudu/fs/fe.proto`  
    说明：每个“数据根目录”会对应一个`uuid`，每个`uuid`在所有`uuid`的列表中（`all_uuids`）有唯一的“下标号”(称为`uuid_idx`)，这`2`个字节中的值，就是这个`uuid_idx`。
2. 后`6`个字节：在一个`data dir`中，唯一的标识一个`block`。  
    因为在一个`data dir`下，随着越来越多的`block`被创建，（`block_id`的）冲突概率会越来越大。  
    如果发生了冲突，那么`BlockManager`会 重试（参见`CreateBlock()`）。

>> 问题：`MSB`和`LSB`各是什么的缩写？

一个`FileBlockLocation`对象，把上述逻辑进行了抽象，所以用户不需要关心这些。

该类的“构造函数”是`private`的，创建该`FileBlockLocation`对象时，使用如下两个`static`方法。
1. 从`FromParts()`来构造；
2. 从`FromBlockId()`来构造；

注意： 这个对象是“可拷贝”和“可赋值”的。也就是说，在使用的时候，这个类基本上都是“按值传递”的。

因为在本类中，只有两个成员（一个指针，一个`BlockId`），所以直接进行“值传递”的代价很低。

## 关于本类的总结

实际上，只需要给定一个`BlockId`（本质上是一个`uint64_t`类型的整型），理论上就可以得到该`Block`说对应的文件：  
1. 前`2`个字节是`uuid_idx`，从`DataDirManager`中，就可以知道对应的“数据根目录”地址，也就能得到`DataDir*`；
2. 后`6`个字节，是当前`Block`的自身的信息。从其中的“前`3`个字节”就可以知道 当前`Block`的数据，相对于 它所属的“数据根目录” 的相对路径。

将两者结合起来，就可以知道这个`Block`所对应的文件，在当前机器上的“绝对路径”。

也就是说，**理论上，给定一个`BlockId`，就可以知道它在当前机器上的“绝对路径”。**  

### 那为什么需要封装出来一个`FileBlockLocation`对象，而不是直接使用`BlockId`。

一句话解释：就是为了方便使用。

如果仅仅有`uuid_idx`(从`BlockId`中可以计算出来)，那么如果要知道对应的`DataDir*`，那么还需要查询`DataDirManager`对象。也就是对于使用者，必须要能够访问到`DataDirManager`对象，才能计算出最终的“绝对路径”。

也就是说，对于一个使用者，如果希望从`block_id`找到它对应的“绝对路径”，那么需要传递两个参数（`DataDirManager`和`BlockId`）。

而对于每个“数据根目录”，对应的`uuid`是唯一的、不变的。

所以为了能够方便地使用，就封装出来一个`FileBlockLocation`类，然后在其中缓存了`DataDir*`，这样在使用的时候，就不需要去查询`DataDirManager`对象。  
对于使用者，也就只需要`FileBlockLocation`这一个参数就行了。

### 给定一个block_id，那么它的完整路径是：

参见`GetFullPath()`函数。

完整路径是： /`$data_root_dir`/`$byte2`/`$byte3`/`$byte4`/`$block_id_str`

其中`block_id_str`，就是`BlockId::ToString()`的返回值。 实际上就是将一个`uint64`按照`%16`来打印。

即打印的结果字符串长度最小为`16`位，如果不足就补充“前导`0`”。

说明：因为`uint64_t`的取值范围是`[0 : 18446744073709551615]`，即最长`20`位。

如果`block_id`的数值超过了16位，那么就原值输出。

## 成员

### `data_dir_`
对应的“目录”。

### `block_id_`
一个`Block`所对应的`block_id`.

```
  DataDir* data_dir_;
  BlockId block_id_;
```
## 接口方法

### `private 构造函数`
```
  FileBlockLocation(DataDir* data_dir, BlockId block_id)
      : data_dir_(data_dir), block_id_(block_id) {}
```

### `data_dir()`和`block_id()` -- `getter`方法 

```
  DataDir* data_dir() const { return data_dir_; }
  const BlockId& block_id() const { return block_id_; }
```

### `static FromParts()`  -- 静态方法

根据`data_dir_idx`和`BlockId`，构建`FileBlockLoacation`对象。

构建方法：实际上就是取`data_dir_idx`的“后`16`位”, 以及传入的参数`block_id`的“后`48`位”，共同组成一个新的`uint64_t`的数值（会放入到一个`BlockId`结构中）。

```
FileBlockLocation FileBlockLocation::FromParts(DataDir* data_dir,
                                               int data_dir_idx,
                                               const BlockId& block_id) {
  DCHECK_LT(data_dir_idx, kuint16max);

  // The combined ID consists of 'data_dir_idx' (top 2 bytes) and 'block_id'
  // (bottom 6 bytes). The top 2 bytes of 'block_id' are dropped.
  uint64_t combined_id = static_cast<uint64_t>(data_dir_idx) << 48;
  combined_id |= block_id.id() & ((1ULL << 48) - 1);
  return FileBlockLocation(data_dir, BlockId(combined_id));
}
```
说明：`data_dir_idx`的数值，最大不超过`2^16`。

说明2：为了防止在移位时发生溢出，在计算`mask`(二进制表示时，是一段`1`)的时候，使用的是`1ULL`。  

`ULL`的后缀，表明这个数值`1`是 `uint64_t`类型。

### `static FromBlockId()`  -- 静态方法

```
FileBlockLocation FileBlockLocation::FromBlockId(DataDir* data_dir,
                                                 const BlockId& block_id) {
  return FileBlockLocation(data_dir, block_id);
}
```

### `static GetDataDirIdx()`  -- 静态方法

从一个“完整的”`BlockId`中，计算出它的`data_dir_idx`。

其实就是从`block_id`中，取出它的“前`16`位”。

```
  static int GetDataDirIdx(const BlockId& block_id) {
    return block_id.id() >> 48;
  }
```

注意：参数中的`BlockId`，是一个完整的`BlockId`。

### `GetFullPath()`

获取当前`FileBlockLoaction`的“完整路径”。

```
string FileBlockLocation::GetFullPath() const {
  string p = data_dir_->dir();
  p = JoinPathSegments(p, byte2());
  p = JoinPathSegments(p, byte3());
  p = JoinPathSegments(p, byte4());
  p = JoinPathSegments(p, block_id_.ToString());
  return p;
}
```

返回的路径为：
```
   /one-data-dir-root/byte2/byte3/byte4/block_id
```

### `CreateBlockDir()`

创建当前`BlockLocation`对应的所有“子目录”。

因为对于每个`Block`，在保存它的数据时，都是在“数据根目录”之下，还有3层“子目录”。  
所以，在创建一个`Block`的时候（见`FileBlockManager::CreateBlock()`），需要把这些“子目录”都创建出来。

**注意：不同的`Block`，计算出来的“绝对路径”，是可能相同的。**  
因为对于每个`Block`，它的“子目录”只是简单的通过`uint64_t`类型的`block_id`中的“前`5`个”字节来组织的。（前`2`个字节决定“数据根目录”，第`3~5`个字节决定“3层子目录”）  
尽管对于不同的`Block`来说，它们的`block_id`一定是不同的（`uint64_t`类型，一共`8`个字节），但是它们的“前`5`个字节”可能是相同的。

```
                | data_dir |    sub_dir      |
                  0     1     2     3     4     5     6     7    
    block_id_1:  aaaa  bbbb  cccc  dddd  eeee  xyzx  yzxy  zxyz
    block_id_2:  aaaa  bbbb  cccc  dddd  eeee  hijk  lmno  pqrs
    
    上面两个`block_id`，整体是不同的，但是“前5个”字节是相同的。
```

所以，在一个目录下，是可能有多个`block`的文件。  因为每个`Block`对应的“文件名”就是`BlockId`，所以虽然它们在同一个“目录”下，但是“文件名”是不同的。

说明1：在创建成功以后，在传入的参数`created_dirs`中返回 本次成功创建的目录。

如果对应的目录已经存在，那么就不需要再创建了，也不会在`create_dirs`中返回。

说明2：因为目录是有层级的，这里向`created_dirs`中添加多个目录的顺序，是“先添加‘子目录’，再添加‘根目录’”。   
这样做的原因是：上层调用该函数时，“自动清理”时，就可以按照顺序进行删除，先删除“子目录”，后删除“父目录”。

**注意：本方法参数`created_dirs`中要添加的目录，并不是在当前函数中“创建出来的目录”，而是每个“创建目录的‘父目录’”。**  

比如说：

```
对于`block_id_1`，创建的子目录层级结构如下：

    |-- data_dir_root
         |-- byte2
              |-- byte3
                   |-- byte4
                        |
即一共创建了 3 个目录：
1. /data_dir_root/byte2
2. /data_dir_root/byte2/byte3/
3. /data_dir_root/byte2/byte3/byte4

但是返回的3个目录是：
1. /data_dir_root/
2. /data_dir_root/byte2
3. /data_dir_root/byte2/byte3

两者的对应关系如下：（因为创建了“左边的目录”，所以需要在`create_dirs`中添加“右边的目录”）

创建的目录                                      返回的目录
1. /data_dir_root/byte2               ---      /data_dir_root/ 
2. /data_dir_root/byte2/byte3/        ---      /data_dir_root/byte2   
3. /data_dir_root/byte2/byte3/byte4   ---      /data_dir_root/byte2/byte3

```

原因是：在`create_dirs`中返回的目录，是调用者(`FileBlockManager::CreateBlock()`)中，会被标记为需要进行“对目录进行`fsync()`操作”的。 
而如果在一个“目录”（记为`parent_dir`）下创建了“子目录”(记为`sub_dir`)，那么需要对`parent_dir`进行`fsync()`.
也就是说，如果返回了“目录A”，那么一定是因为在“目录A”下面新建了“文件或目录”（在本函数中，就是新建了“子目录”）。

```
假如一个`BlcokId`所对应的目录层级结构中， 假如`/data_dir_root/byte2`已经存在。
那么

创建的目录：                                 返回的目录
1. /data_dir_root/byte2/byte3/        ---   /data_dir_root/byte2
2. /data_dir_root/byte2/byte3/byte4   ---   /data_dir_root/byte2/byte3
```

```
Status FileBlockLocation::CreateBlockDir(Env* env,
                                         vector<string>* created_dirs) {
  DCHECK(env->FileExists(data_dir_->dir()));

  bool path0_created;
  string path0 = JoinPathSegments(data_dir_->dir(), byte2());
  RETURN_NOT_OK(env_util::CreateDirIfMissing(env, path0, &path0_created));

  bool path1_created;
  string path1 = JoinPathSegments(path0, byte3());
  RETURN_NOT_OK(env_util::CreateDirIfMissing(env, path1, &path1_created));

  bool path2_created;
  string path2 = JoinPathSegments(path1, byte4());
  RETURN_NOT_OK(env_util::CreateDirIfMissing(env, path2, &path2_created));

  if (path2_created) {
    created_dirs->push_back(path1);
  }
  if (path1_created) {
    created_dirs->push_back(path0);
  }
  if (path0_created) {
    created_dirs->push_back(data_dir_->dir());
  }
  return Status::OK();
}
```

### `GetAllParentDirs()`

获取当前`BlockLocation`所对应的“父目录”。

一共有4个元素: “基础目录”和“3层子目录”。

说明：目录是有层级的，向参数`parent_dirs`中添加的时候，是有顺序的：**先添加‘子目录’，后添加‘父目录’**。

```
void FileBlockLocation::GetAllParentDirs(vector<string>* parent_dirs) const {
  string path0 = JoinPathSegments(data_dir_->dir(), byte2());
  string path1 = JoinPathSegments(path0, byte3());
  string path2 = JoinPathSegments(path1, byte4());

  // This is the order in which the parent directories should be
  // synchronized to disk.
  parent_dirs->push_back(path2);
  parent_dirs->push_back(path1);
  parent_dirs->push_back(path0);
  parent_dirs->push_back(data_dir_->dir());
}
```

说明：这里向`parent_dirs`中添加“目录”的顺序。  

按照“先添加‘子目录’，然后再添加‘父目录’”的顺序。

这样做的原因：这些目录，上层在写完数据时，会进行`fsync()`操作。 而`fsync()`时，比较安全的做法就是：**先`fsync()`子目录，然后再`fsync`父目录**。

>> 猜测，待验证：  
>> 1. 如果先`fsync()`父目录，那么假如在将所有子目录`fsync()`之前，系统宕机了，那么重启后，可能出现，磁盘上只有“父目录”，而没有“子目录”。 即这是一个不完整的目录结构。  
>> 2. 而如果先`fsync()`子目录，那么在进行一半的时候宕机，那么在磁盘上，就会“要么所有的目录层级都存在”，要么“都不存在”。（因为目录是“从上到下”进行的，如果“父目录”中没有被修改，那么就遍历不到这个子目录。）

### 获取当前`block`相应的“子目录”

#### `private byte2()`
获取`block_id_`中的第`2`个字节，按照“16进制整数”的方式 转化为字符串。  
对应“3层目录”中的第`1`层目录。


#### `private byte3()`
获取`block_id_`中的第`3`个字节，按照“16进制整数”的方式 转化为字符串。  
对应“3层目录”中的第`2`层目录。

#### `private byte4()`
获取`block_id_`中的第`4`个字节，按照“16进制整数”的方式 转化为字符串。  
对应“3层目录”中的第`3`层目录。

```
  string byte2() const {
    return StringPrintf("%02llx",
                        (block_id_.id() & 0x0000FF0000000000ULL) >> 40);
  }
  string byte3() const {
    return StringPrintf("%02llx",
                        (block_id_.id() & 0x000000FF00000000ULL) >> 32);
  }
  string byte4() const {
    return StringPrintf("%02llx",
                        (block_id_.id() & 0x00000000FF000000ULL) >> 24);
  }
```

说明1: 这里取出“指定字节”的值的基本方法是：先按位相与，然后再移位。

说明2： 在打印的时候，格式化字符串是`%02llx`，即打印出来`2`个字符宽度、并且按照“16进制数值”格式的字符串。

比如：如果在移位之后是`16`，那么打印的字符串为`"0F"`。

# `class FileWritableBlock`  -- 内部类

在`.cpp`文件中定义，只有本类可用。

说明1：内部会有两个成员：分别是当前`Block`所属的`FileBlockManager`，以及自己的位置（`FileBlockLocation`对象），这样，在`Close()`的时候，就可以通过调用`BlockManager::SyncMetadata()`方法来将“元数据”信息刷盘。

说明2：将`FileBlockLocation`作为成员，而不是简单的使用`BlockId`，虽然会占用更多的内存，但是因为某一个时刻，需要进行写出数据`Block`是很少的（也就是`FileWritableBlock`对象的数量 是比较少的），所以问题不大。

## `enum FileWritableBlock::SyncMode`

表示在关闭当前`FileWritableBlock`的时候，是否要将其中的数据 持久化到磁盘上。

这个类型的含义：表示的是“是否要刷盘”，而不是“如何去刷盘”。

> 说明：这个名字起的不好。因为这个名字中的"mode"，容易让人误解为“模式”，即即可能会误解为有两种模式。  
> 一个更好的方式是，用一个`bool need_sync`来表示。

如果按照“模式”来理解，一般情况下，在运行的过程中，只能选择一种“模式”。 而这里，是两种“模式”都存在的：

+ 如果是正常的关闭，即调用了`FileWritableBlock::Close()`，那么会将“数据”通过调用`fsync()`，从而保证会被持久化到磁盘。
+ 如果是出错了，调用`FileWritableBlock::Abort()`，或者“没有显式的调用`Closet()`就要析构当前对象时”，那么并不会调用`fsync()`。

```
  enum SyncMode {
    SYNC,
    NO_SYNC
  };
```

## 成员

### `block_manager_`
类型为`FileBlockManager*`，表示当前`block`所属的`BlockManager`对象指针。

注意：因为是个指针，所以要保证该`BlockManager`的生命周期，要长于 当前`FileWritableBlock`对象。

### `location_`
类型为`FileBlockLocation`，当前`block`的位置。

### `writer_`
类型为`std::sharde_ptr<WritableFile>`，当前`block`所对应的、要去写数据的 底层文件。

### `state_`
类型是`WritableBlock::State`，参见`src/kudu/fs/block_manager.h`。

### `bytes_appended_`
已经写入的数据大小。

```
  FileBlockManager* block_manager_;
  const FileBlockLocation location_;
  shared_ptr<WritableFile> writer_;
  State state_;
  size_t bytes_appended_;
```

## 接口列表

说明：当前类的方法接口，大部分 和父类`WritableBlock`完全一样。

只新增了`2`个新的接口: `HandlerError()`接`FlushDataAsync()`

### 构造函数 和 析构函数

#### 构造函数

注意：从代码实现可以看出来，本类是不负责`Block`文件的“创建”和“删除”的。

因为传入的参数中，是一个构造好的`WritableFile`的。  
而且这个文件是已经被创建好的，在本类中，没有任何“创建文件，打开文件”等相关的逻辑，而是直接向其中写数据。

#### 析构函数

会检查当前`Block`是否曾经被显式`Close()`或`Abort()`过，如果没有，那么会调用`Abort()`将它关闭。

注意1：`Abort()`操作，并不会调用`fsync()`把已经写入的数据刷盘。

注意2：检查的方式是：检查`state_`成员的值。

### 需要实现的父类的纯虚函数接口

#### `Close()`
关闭当前`Block`对象。

注意：会调用`private Close(SYNC)`方法。其中的参数`SYNC`表示要把当前`Block`的“数据”和“元数据”都持久化到磁盘上。

```
Status FileWritableBlock::Close() {
  return Close(SYNC);
}
```

#### `Abort()`

丢弃当前`Block`对象。

注意1：调用的是`Close(NO_SYNC)`，其中的`NO_SYNC`就表示，只关闭当前`Block`，但是不把它的“数据”和“元数据”持久化到磁盘。

注意2：还会从`BlockManager`中把当前`block_id`删除（对应代码中的`BlockManager::DeleteBlock()`）。 也就是调用过这个方法之后，当前`Block`就相当于是“不再存在”了。

```
Status FileWritableBlock::Abort() {
  RETURN_NOT_OK(Close(NO_SYNC));
  return block_manager_->DeleteBlock(id());
}
```

#### `getter`方法

##### `block_manager()`

```
BlockManager* FileWritableBlock::block_manager() const {
  return block_manager_;
}
```

##### `id()`

这个方法是“最顶级父类”`Block`类中的接口

```
const BlockId& FileWritableBlock::id() const {
  return location_.block_id();
}
```

##### `BytesAppended()`

```
size_t FileWritableBlock::BytesAppended() const {
  return bytes_appended_;
}
```

##### `state()`
```
WritableBlock::State FileWritableBlock::state() const {
  return state_;
}
```

#### 追加数据的函数 

##### `Append()`

通过调用`AppendV()`来实现。

```
Status FileWritableBlock::Append(const Slice& data) {
  return AppendV(ArrayView<const Slice>(&data, 1));
}
```

说明：这里会构造出 只含有一个元素的`ArrayView`对象，然后去调用`AppendV()`。

注意1：这里构造的`ArrayView`对象，其中的模板参数是`const Slice`，即其中的元素(`Slice`)是不能被修改的。

注意2：这里构造的`ArrayView`对象，在调用`AppendV()`函数时，是**按值传递**的。原因是`ArrayView`对象很小（只有一个指针和一个`size`成员），直接拷贝的代价并不高，参见`src/kudu/util/arrayview.h`。

##### `AppendV()`

```
Status FileWritableBlock::AppendV(ArrayView<const Slice> data) {
  DCHECK(state_ == CLEAN || state_ == DIRTY) << "Invalid state: " << state_;
  RETURN_NOT_OK_HANDLE_ERROR(writer_->AppendV(data));
  RETURN_NOT_OK_HANDLE_ERROR(location_.data_dir()->RefreshAvailableSpace(
      DataDir::RefreshMode::ALWAYS));
  state_ = DIRTY;

  // Calculate the amount of data written
  size_t bytes_written = accumulate(data.begin(), data.end(), static_cast<size_t>(0),
                                    [&](int sum, const Slice& curr) {
                                      return sum + curr.size();
                                    });
  bytes_appended_ += bytes_written;
  return Status::OK();
}
```

说明1：从这个函数实现可以看出，只要向一个`Block`中添加了内容，那么它的状态就会变成`DIRTY`。

说明2：这里用到了`std::accumulate()`工具函数。  
参见：http://www.cplusplus.com/reference/numeric/accumulate/ 

就是参数：两个迭代器，1个初始值，还有一个累加函数。  
返回的结果是：将两个迭代器之间的所有数字，通过累加函数进行相加，得到最终的结果。  

注意：这里`bytes_appended_`是本类的属性，无论是否开启了`metric`信息，每次追加的时候，都会进行累加。

但是如果没有开启`metric`信息统计，在`Close()`的时候，不会将 这个累加值 添加到对应的`Metric`信息中。参见`Close()`函数。

#### `Finalize()`

调用这个方法，表示当前`Block`不可以再进行任何写入操作。

注意：当前的`State`有4个状态：`CLEAN`/`DIRTY`/`FINALIZED`/`CLOSED`。调用这个方法时，状态只能是`CLOSED`状态以外的3个状态。

```
Status FileWritableBlock::Finalize() {
  DCHECK(state_ == CLEAN || state_ == DIRTY || state_ == FINALIZED)
      << "Invalid state: " << state_;

  if (state_ == FINALIZED) {
    return Status::OK();
  }
  VLOG(3) << "Finalizing block " << id();
  if (state_ == DIRTY &&
      FLAGS_block_manager_preflush_control == "finalize") {
    FlushDataAsync();
  }
  state_ = FINALIZED;
  return Status::OK();
}
```

说明1：如果当前`Block`已经被`Finalize()`过，那么直接返回`Status::OK()`。

说明2：如果当前的状态为`DIRTY`，说明在当前`Block`中，之前有写入了新数据。 如果`FLAGS_block_manager_preflush_control == "finalize"`，那么就会触发“异步的flush”（通过调用`FlushDataAsync()`）。

注意1：在`C++`中，使用`==`来比较两个字符串，会去比较字符串的值（而不是仅仅比较指针）。  
对应代码中的`FLAGS_block_manager_preflush_control == "finalize"`，这里会比较“字符串”的内容。

### 新添加的接口

#### `FlushDataAsync()`
当前类中新增加的接口，不是父类中继承的。

实际上就是用“异步模式”（`FLUSH_ASYNC`）去调用`WritableFile::Flush()`。

该函数的触发，取决于`FLAGS_block_manager_preflush_control`的值：
1. 如果值为`"finalize"`，那么在调用`FileWritableBlock::Finalize()`时会触发。
2. 如果值为`"close"`，那么在调用`FileBlockCreationTransaction::CommitCreatedBlocks()`时会触发。

```
Status FileWritableBlock::FlushDataAsync() {
  VLOG(3) << "Flushing block " << id();
  RETURN_NOT_OK_HANDLE_ERROR(writer_->Flush(WritableFile::FLUSH_ASYNC));
  return Status::OK();
}
```

说明：调用该方法，并不会阻塞等待所有的数据被`fsync()`完成以后才返回，而是直接就会返回（后台会触发异步`fsync()`的逻辑）。

也就是说，该方法返回后，并**不保证**数据已经被`fsync`到磁盘了。

#### `HandleError()`

```
void FileWritableBlock::HandleError(const Status& s) const {
  HANDLE_DISK_FAILURE(
      s, block_manager_->error_manager()->RunErrorNotificationCb(
          ErrorHandlerType::DISK_ERROR, location_.data_dir()));
}
```

### 私有函数

#### `private Close(SyncMode)`

```
Status FileWritableBlock::Close(SyncMode mode) {
  if (state_ == CLOSED) {
    return Status::OK();
  }

  Status sync;
  if (mode == SYNC &&
      (state_ == CLEAN || state_ == DIRTY || state_ == FINALIZED)) {
    // Safer to synchronize data first, then metadata.
    VLOG(3) << "Syncing block " << id();
    if (FLAGS_enable_data_block_fsync) {
      if (block_manager_->metrics_) block_manager_->metrics_->total_disk_sync->Increment();
      sync = writer_->Sync();
    }
    if (sync.ok()) {
      sync = block_manager_->SyncMetadata(location_);
    }
    WARN_NOT_OK(sync, Substitute("Failed to sync when closing block $0",
                                 id().ToString()));
  }
  Status close = writer_->Close();

  state_ = CLOSED;
  writer_.reset();
  if (block_manager_->metrics_) {
    block_manager_->metrics_->blocks_open_writing->Decrement();
    block_manager_->metrics_->total_bytes_written->IncrementBy(BytesAppended());
    block_manager_->metrics_->total_blocks_created->Increment();
  }

  // Either Close() or Sync() could have run into an error.
  HandleError(close);
  HandleError(sync);

  // Prefer the result of Close() to that of Sync().
  return close.ok() ? close : sync;
}
```

说明1： 如果是重复关闭（即状态已经是`CLOSED`），那么直接返回`Status::OK`。

说明2：参数类型为`SyncMode`，不是说的“刷盘的模式”，而是说“是否要刷盘。”

>> 备注：这个名字中的"mode"，容易让人误解。

说明3：触发“刷盘”的条件
1. 参数`SyncMode mode == SYNC`： 这个是前提.   
    比如`Abort()`中调用该函数，传递的`mode == NO_SYNC`，即不进行任何数据刷盘，而是直接丢弃掉当前`Block`。
2. 对于“数据”部分，需要查看用户是否开启了`fsync`（查看`FLAGS_enable_data_block_fsync`配置项）。只有用户开启了`fsync == true`，才会进行刷盘。
3. 对于“元数据”部分，是一定要刷盘的。

注意1：对于一个文件，在关闭时需要“先调用`fsync(fd)`，然后再`close(fd)`”。
参见`linux man close`

> A successful close does not guarantee that the data has been successfully saved to disk, as the kernel defers writes.  It is not common for a file system to flush the buffers when the stream is closed.  If you need to be sure that the data is physically stored use fsync(2).  (It will depend on the disk  hardware at this point.)

因为要关闭一个文件，`close()`是必须要调用的。

但如果只调用`close(fd)`，它返回成功时，并不保证数据完全被持久化硬盘上。

如果要保证数据被持久化到磁盘，必须要调用`fsync()`。

当然，一旦一个`fd`被关闭，就不能再使用它了，所以也就不能在调用`fsync(fd)`了。所以两者的顺序一定是：先调用`fsync(fd)`，然后再调用`close(fd)`。

**注意2：`fsync()`和`fdatasync()`的区别**  

参见：
1. `linux man fsync`
2. https://blog.csdn.net/cywosp/article/details/8767327  linux 同步IO: sync、fsync与fdatasync
3. https://www.cnblogs.com/kercker/p/8610988.html  Linux系统中fflush，sync，syncfs，fdatasync，fsync的比较  
4. https://wenchao.ren/2019/03/确保数据落盘/   确保数据落盘

总结就是：  

向文件写入数据，会经历两层缓冲区：“用户空间缓冲区”和“内核缓冲区”。
+ 用户空间缓冲区：主要是“库函数的缓冲区”，比如`c语言标准库`中的`stdio`中，就维护着一个缓冲区。在调用库函数`fflush()`的时候，会把数据从“库函数的缓冲区”刷入到“内核缓冲区”（注意：`fflush()`成功返回，并不代表这数据被成功持久化到磁盘，而只能保证数据被刷入到了“内核缓冲区”中）。
+ 内核缓冲区：系统调用`write()`函数等，都是将数据写入到“内核缓冲区”（`linux page cache`）。  

linux提供的“内核缓冲区”的机制，被称为“延迟写”（delayed write）。

数据被写入到“内核缓冲区”以后，也不一定会立即被刷入磁盘。一般操作系统中会有一个`update`的守护进程，周期性的将其中的数据刷入磁盘。或者“内存缓冲区”被写满的时候，会主动触发写入磁盘。

而我们的应用程序，经常的需求是 必须在数据成功写入磁盘以后，才能返回。这时就需要调用`fsync()`或`fdatasync()`，这两个函数成功返回以后，才能保证数据被成功持久化到磁盘。

这两个区别是：
1. `fsync()`：不仅会写入“数据”，还会保证“该文件的元数据”（即`inode`的内容，包括文件大小，访问时间`st_atime`和`st_mtime`等）。
2. `fdatasync()`：也会写入“数据”，但是只在必要的时候才会更新“文件的元数据”。（比如：文件大小发生变化时，那么`fdatasync()`也会去更新“文件的元数据”；但是如果只是文件的访问时间发生变化，那么`fdatasync()`就不会去修改文件的元数据）

也正是因为`fsync()`一定会修改“文件的元数据”，所以它对性能的影响比较大。

正如`linux man fsync`中所说：`fsync()`一定会有两次磁盘写操作：一个是“写数据”；一个是“写元数据”；

> Unfortunately, fsync() will always initiate two write operations: one for the newly written data and another one in order to update the modification time stored in the inode.  If the modification time is not a part of the transaction concept fdatasync() can be used to avoid unnecessary inode disk write operations.

因为文件的“数据”和“元数据”通常都是在磁盘上不同的位置，所以每次`fsync()`就需要至少两次“磁盘查找”（普通磁盘上，一次寻道时间为`10ms`）。

所以说`fsync()`的性能开销很大。

而对于一般的应用程序，不需要关心“访问时间、修改时间”等“文件的元信息”，所以使用`fdatasync()`即可。

`fdatasync()`只会在必要的时候，即如果本次不更新元数据。
比如：文件的尺寸（`st_size`）如果变化，是需要立即同步的，否则OS一旦崩溃，即使文件的数据部分已同步，由于`metadata`没有同步，依然读不到修改的内容。  
而“最后访问时间”(`atime`)/“最后修改时间”(`mtime`)是不需要每次都同步的，只要应用程序对这两个时间戳没有苛刻的要求，基本无伤大雅。

**综上：在目前所有的数据库系统中，基本都是`fdatasync()`来保证数据被持久化到磁盘的。**  

**注意3：在“目录”下新建文件时，需要在“目录”上也调用`fsync()`。**  

也参考上面的链接。

`fsync(fd)`和`fdatasync(fd)`只保证传入的参数`fd`对应的文件被持久化，但是不保证它的“父目录”也被持久化。

但是对于文件系统来说，目录中也保存着文件信息。在一个“目录”(`dir_fd`)下创建一个文件(`file_fd`)，需要在它的“父目录”中添加一条记录，即也需要修改它的“父目录”。 

而`fsync(file_fd)`，并不保证`dir_fd`的内容被持久化到磁盘。 即如果此时发生掉电，这个文件就无法被找到了。  

所以对于“新建文件”来说，即需要在文件上调用`fsync()`，也需要在父目录上调用`fsync()`。（即，既需要`fsync(file_fd)`，也需要`fsync(dir_fd))`）

对它的“父目录”调用`fsync()`，在本函数(`Close()`)的实现中，称为是“持久化`Block`的‘元数据’”(代码中对应：`block_manager_->SyncMetadata(xx)`)。

因为当前类就是`FileWritableBlock`，即一个“用来写数据的`Block`”，创建该类的对象的目的，就是要新建一个文件，来保存一个`Block`的数据。所以在`FileWritableBlock::Close()`中，即需要对`block`文件进行`fsync()`，也需要对“目录”进行`fsync()`。

**注意4： 在需要`fsync()`的时候，比较安全的方法是： **先写“数据”，然后再写“元数据”**。**  

这里代码中所说的“数据”和“元数据”，都是指的文件（要注意区分，和平时所说的不是一个含义）: 
+ “数据”：是指该`Block`所对应的文件；
+ “元数据”：是指该`Block`“所属的目录”。

>> 备注：单看字面意思，对“元数据”，容易有如下两种误解：
>> 
>> 1. 错误的理解为 “在文件系统”中，该`block`文件的“inode”信息。   
>>     实际上，这个`inode`的刷盘，在对“block文件”调用`fdatasync()`时，就已经保证了。
>> 2. 错误的理解为 `Block`的元数据（`Block`自身的元数据，`block_id`、大小之类的）。
>>     对于`Block`的元数据信息，不是由当前`FileBlockManager`来决定，而是在上层`FsManager`中来持久化的。

猜测推荐这个顺序的原因：
1. 如果先`fsync`文件，然后再`fsync`目录：如果在中途宕机，即还没有“`fsync`目录”的时候。这个时候，在重启以后，因为文件查找是按照层级的，因为“目录文件”中还没有该条目，所以就当这个文件不存在即可。 而且，因为中途宕机了，这里也没有给客户端返回成功。
2. 如果先`fsync`目录，然后再`fsync`文件：如果在中途宕机，因为目录中已经有该文件的条目了，但是该文件却找不到，这样“文件系统”就必须解决这个问题才能继续了。

注意5：如果用`mode = NO_SYNC`调用本函数，只是不会“文件”和"父目录"调用`fsync()`，但是其实“`block`对应的数据文件”可能已经被创建过了，甚至里面已经通过`Append()`或`AppendV()`向里面添加过数据了。

也就是说，虽然在`Close()`中不进行显式的`fsync()`，只要系统不崩溃，这些文件中对应的数据早晚会被 持久化到磁盘上。 

所以，如果要删除某个`Block`的数据，使用者仅仅通过调用`Close(NO_SYNC)`是不够的。还需要进行一些清理操作的，才能将对应的文件和目录，从磁盘上清理掉。

参见`Abort()`的实现，其中会调用`block_manager_->DeleteBlock(id())`，其中将该`Block`的文件删除掉。

所以：如下两种方式会将`Block`的文件删除掉：
1. 显式调用`FileWritableBlock::Abort()`；
2. 直接 析构掉当前`FileWritableBlock`对象（没显式调用`Close()`），这样在“析构函数”中，会去调用`Abort()`。 

# `class FileReadableBlock` -- 内部类

在`.cpp`文件中定义，只有本类可用。

`ReadableBlock`的子类。

**注意1：必须要控制该对象的大小**

原因是：在系统中，该类的实例，可能会有 数百万个。所以，必须要严格的控制该类的大小，从而减少内存占用。

因为每个`tablet`会有很多个`Block`，而每次读取一个`Block`都会创建一个`FileReadableBlock`对象。  
而且同一个物理的`Block`可能会被多人并发读取时，也会创建多个`FileReadableBlock`对象。

也是因为这个原因，这里和`FileWritableBlock`不同，**没有将`FileBlockLocation`作为类成员，而只是简单的使用了`BlockId`**。

所以，本类的成员，越少越好，越小越好。 如果要向本类中添加成员，一定要慎重。

**说明1：通过`BlockId`可以获得对应的`DataDir`**  

```
const DataDir* dir = block_manager_->dd_manager_->FindDataDirByUuidIndex(
    internal::FileBlockLocation::GetDataDirIdx(block_id_));
```

说明2：有了`DaraDir`，如果需要的话，就可以和`BlockId`一起构造出`FileBlockLocation`对象。

## 成员

### `block_manager_`
当前`Block`对象所属的`FileBlockManager`对象。

### `block_id_`
当前`Block`的`id`。

### `reader_`
当前`Block`所对应的“读取文件”。

### `closed_`
原子变量，用来表示当前`Block`是否已经被关闭。

**对比： 在`FileWriableBlock`中，是使用一个枚举类型`WritableBlock::State`来标识“是否已经被关闭”。**

注意：该变量要是一个“原子变量”的原因是，要将`Close()`函数做成是“线程安全”的。

```
  FileBlockManager* block_manager_;
  const BlockId block_id_;
  shared_ptr<RandomAccessFile> reader_;
  AtomicBool closed_;
```

>> 问题：改进点：如果将这里的`shared_ptr`，修改为`rep_ref`，也能降低该类的大小。

## 接口列表

### 需要实现的父类的纯虚函数接口

#### `Close()`

注意：`Close()`是“线程安全”的。
```
Status FileReadableBlock::Close() {
  if (closed_.CompareAndSet(false, true)) {
    reader_.reset();
    if (block_manager_->metrics_) {
      block_manager_->metrics_->blocks_open_reading->Decrement();
    }
  }

  return Status::OK();
}
```

说明：通过“原子变量”`closed_`来标识是否已经被关闭。

注意：如果有并发多个线程调用同一个`Block`的该方法，最终只有一个人会去调用`RandomAccessFile::reset()`，将当前对象中的`智能指针`重置为`nullptr`。

注意：当前`reader_`（实际上是一个`std::shared_ptr<RandomAccessFile>`对象），它在`FileCache`中也会保存该指针，所以这里将它`reset()`以后，对应的文件，并不是会立即被关闭。  只有当`FileCache`中也没有再引用该文件（比如，该文件已经从`cache`中被淘汰掉），才会真正的关闭该文件。

#### `getter`方法

##### `block_manager()`
##### `id()`
##### `Size()`

```
BlockManager* FileReadableBlock::block_manager() const {
  return block_manager_;
}

const BlockId& FileReadableBlock::id() const {
  return block_id_;
}

Status FileReadableBlock::Size(uint64_t* sz) const {
  DCHECK(!closed_.Load());

  RETURN_NOT_OK_HANDLE_ERROR(reader_->Size(sz));
  return Status::OK();
}
```

说明：这里调用`Size()`的时候，要求 当前`Block`尚未被关闭。

#### 读取数据的接口
##### `Read()`

```
Status FileReadableBlock::Read(uint64_t offset, Slice result) const {
  return ReadV(offset, ArrayView<Slice>(&result, 1));
}
```

和`FileWritableBlock::Write()`相似，这里也是先转化为`ArrayView<Slice>`，然后调用`ReadV()`来实现。

同样，调用`ReadV()`时，也是“按值传递”构造出来的`ArrayView<>`对象。

注意1： 和`FileWritableBlock::Write()`不同的是：模板参数不是`const Slice`的，因为这里`Slice`对象的内容需要被修改（读取的内容，会放到这个`Slice`中）。

注意2：对于`Slice`对象，经常在使用过程中，都不会修改它内部的值。  
**但是实际上，是可以修改的，这里的使用就是一个例子。**  

> 借鉴：我们经常需要修改数据时，都是传递一个`std::string*`的指针。  在写入完毕以后，可能还会再次拷贝数据的内容。  
>  其实，可以借鉴这里的使用方式。先申请一段内容，然后构建一个`Slice`对象（指向刚才申请的内存），如果需要修改时，就传递这个`Slice`对象。

注意3： 这里的`Slice`对象是“按值传递”的，并不是按照“指针”或者“引用”。   
因为`Slice`对象是很小的，所以按值传递，是没有性能问题的。  

而传递“指针”或者“引用”，因为有“间接跳转”的过程，所以性能不见得比“按值传递”高。

#### `ReadV()`

```
Status FileReadableBlock::ReadV(uint64_t offset, ArrayView<Slice> results) const {
  DCHECK(!closed_.Load());

  RETURN_NOT_OK_HANDLE_ERROR(reader_->ReadV(offset, results));

  if (block_manager_->metrics_) {
    // Calculate the read amount of data
    size_t bytes_read = accumulate(results.begin(), results.end(), static_cast<size_t>(0),
                                   [&](int sum, const Slice& curr) {
                                     return sum + curr.size();
                                   });
    block_manager_->metrics_->total_bytes_read->IncrementBy(bytes_read);
  }

  return Status::OK();
}
```

注意：要求当前`Block`的状态，一定要处于“打开状态”的。


#### `memory_footprint()`

```
size_t FileReadableBlock::memory_footprint() const {
  DCHECK(reader_);
  return kudu_malloc_usable_size(this) + reader_->memory_footprint();
}
```

### 新添加的接口

和`FileWritableBlock`类似，这个类也增加了一个`HandleError()`方法;

#### `HandleError()`

```
void FileReadableBlock::HandleError(const Status& s) const {
  const DataDir* dir = block_manager_->dd_manager_->FindDataDirByUuidIndex(
      internal::FileBlockLocation::GetDataDirIdx(block_id_));
  HANDLE_DISK_FAILURE(s, block_manager_->error_manager()->RunErrorNotificationCb(
      ErrorHandlerType::DISK_ERROR, dir));
}
```

>> 问题：`FsErrorManager`类的作用？

# `class FileBlockCreationTransaction`

在`.cpp`文件中定义，只有本类可用。

继承自`BlockCreationTransaction`。

```
class FileBlockCreationTransaction : public BlockCreationTransaction {
  ...
}
```
## 成员

### `created_blocks_`

本次要批量创建的`Block`列表。

```
  std::vector<std::unique_ptr<FileWritableBlock>> created_blocks_;
```

## 接口列表

都是需要实现的“虚函数接口”。

### `AddCreatedBlock()`

添加一个`Block`到“待创建列表”。

```
void FileBlockCreationTransaction::AddCreatedBlock(
    std::unique_ptr<WritableBlock> block) {
  FileWritableBlock* fwb =
      down_cast<FileWritableBlock*>(block.release());
  created_blocks_.emplace_back(unique_ptr<FileWritableBlock>(fwb));
}
```

注意：从该函数的实现看，当前函数并不是“线程安全”的，因为对`create_blocks_`的访问并没有进行“锁保护”。（在父类`BlockCreationTransaction`的说明中也强调了）。

注意2：因为参数中是`WritableBlock`类型，但是在`create_blocks_`中是`FileWritableBlock`类型（是`WritableBlock`的子类），所以这里在添加之前，进行了“强制转化”（对应代码中的`down_cast<>()`）。

### `CommitCreatedBlocks()`

创建属于当前事务的 所有的`Block`。

注意1：如果`FLAGS_block_manager_preflush_control == "close"`，那么需要触发这些`Block`的“异步刷盘”。

>> coalesce: 合并；结合；联合  

这个的好处是：给“操作系统”机会，如果要写的数据在磁盘上的连续`page`上，那么“操作系统”有机会进行“IO合并”。

注意2：该函数中，最后会将所有的`Block`都关闭(即会调用`Close()`方法)。

```
Status FileBlockCreationTransaction::CommitCreatedBlocks() {
  if (created_blocks_.empty()) {
    return Status::OK();
  }

  VLOG(3) << "Closing " << created_blocks_.size() << " blocks";
  if (FLAGS_block_manager_preflush_control == "close") {
    // Ask the kernel to begin writing out each block's dirty data. This is
    // done up-front to give the kernel opportunities to coalesce contiguous
    // dirty pages.
    for (const auto& block : created_blocks_) {
      RETURN_NOT_OK(block->FlushDataAsync());
    }
  }

  // Now close each block, waiting for each to become durable.
  for (const auto& block : created_blocks_) {
    RETURN_NOT_OK(block->Close());
  }
  created_blocks_.clear();
  return Status::OK();
}
```

# `class FileBlockDeletionTransaction`  -- 内部类

在`.cpp`文件中定义，只有本类可用。

继承自`BlockDeletionTransaction`。

```
class FileBlockDeletionTransaction : public BlockDeletionTransaction {
  ...
}
```

## 成员

### `fbm_`
对应的`FileBlockManager`的指针。

注意：该`FileBlockManager`对象的生命周期，一定要大于当前的`FileBlockDeletionTransaction`对象；

### `deleted_blocks_`
用来保存所有“待删除”的`Block`列表。

注意：和`FileBlockCreationTransaction`不同，在`deleted_blocks_`中保存的不是`Block`对象，这里保存的是`BLockId`。

```
  FileBlockManager* fbm_;
  std::vector<BlockId> deleted_blocks_;
```

## 接口列表
都是需要实现的“虚函数接口”。

### `AddDeletedBlock()`

```
void FileBlockDeletionTransaction::AddDeletedBlock(BlockId block) {
  deleted_blocks_.emplace_back(block);
}
```
注意：该方法并不是“线程安全”的。

### `CommitDeletedBlocks()`

该方法的一些说明，参见父类`BlockDeletionTransaction::CommitDeletedBlocks()`。

说明：所有被删除的`Block`列表，都会在参数`deleted`中返回。

```
Status FileBlockDeletionTransaction::CommitDeletedBlocks(std::vector<BlockId>* deleted) {
  deleted->clear();
  Status first_failure;
  for (BlockId block : deleted_blocks_) {
    Status s = fbm_->DeleteBlock(block);
    // If we get NotFound, then the block was already deleted.
    if (!s.ok() && !s.IsNotFound()) {
      if (first_failure.ok()) first_failure = s;
    } else {
      deleted->emplace_back(block);
      if (s.ok() && fbm_->metrics_) {
        fbm_->metrics_->total_blocks_deleted->Increment();
      }
    }
  }

  if (!first_failure.ok()) {
    first_failure = first_failure.CloneAndPrepend(strings::Substitute("only deleted $0 blocks, "
                                                                      "first failure",
                                                                      deleted->size()));
  }
  deleted_blocks_.clear();
  return first_failure;
}
```

说明1：参见`FileBlockManager::DeleteBlock()`。如果对应的`Block`没有正在被读取，那么它所对应的`Block`文件会被立即删除掉。否则，会延迟到 当它的“读者”和“写者”都被关闭以后，再删除文件。

说明2：如果`FileBlockManager::DeleteBlock()`返回`Status::NotFound()`，那么不认为是失败，而会被认为是删除成功。因为该函数就是要删除`Block`，返回`NotFound`就表示该`Block`是不存在的，结果是相同的，所以就认为是成功删除了。

当然，如果是`NotFound`，那么说明没有执行“删除文件”的操作，所以不会更新`metric`指标`total_blocks_deleted`。


