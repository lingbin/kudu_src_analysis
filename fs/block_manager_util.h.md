[TOC]

文件：`src/kudu/fs/block_manager_util.h`

# `class PathInstanceMetadataFile`

用来“读取”和“写入” 一个`BlockManager`的“`path instance`元数据”文件。

这个类**不是“线程安全”的**，并发的访问这个类，必须要在外部进行显式的“并发控制”。

对应的`protobuf`类是`PathInstanceMetadataPB`。

## 成员

### `env_`
封装“底层文件系统”的所有接口。

### `block_manager_type_`
用来表示`BlockManager`的类型。

注意是“字符串类型”（`std::string`），只有两个取值：`"file"`或`"log"`。

默认值是`FLAGS_block_manager`。 定义参见`src/kudu/fs/fs_manager.cc`.

1. 对于`linux`系统，默认值是`log`;
2. 其它系统，默认值是`file`；

>> 问题：为什么类型是“字符串”，而不是一个“枚举”类型？

### `filename_`
“元数据”文件的文件名。（包含文件的“完整绝对路径”）。

说明：实际上，在所有“数据根目录”中，这个文件的“文件名”都是相同的。如果只是记录用来记录“文件名”，那么就不需要单独使用一个变量了。   
这里使用一个“单独的变量”，就是用来记录“完整路径”的。

### `metadata_`
所对应的`PathInstanceMetadataPB`对象。

### `lock_`
是一个`FileLock`，一个文件保护锁。用来防止并发读取一个文件。

**注意1：检查一个文件是否已经被加锁的方法。**

该变量的类型为`std::unique_ptr<FileLock>`，在加上“文件锁”以后，该属性会被赋值为对应的“`FileLock`”对象。  
如果当前文件处于“未加锁”的状态，那么它的值为`nullptr`。  

也就是说，如果需要检查“文件是否已经上锁”，那么只需要检查`lock_`的值即可。  
如果`(lock_ == nullptr)`，那么就是没有加锁。

**注意2：因为是“文件锁”，所以当一个线程对某个文件进行加锁后 ，另外的所有线程的加锁操作都会失败**  
说明：如果一个文件处于加锁状态，另一个线程尝试加锁，会直接返回“加锁失败”，而不是像“普通`mutex`”那样，进行阻塞等待。

### `health_status_`
当前“元数据”文件的“健康状态”。

因为在进程启动时，都是根据 用户配置的多个“数据根目录”，来进行初始化。用户指定了多少个“数据根目录”，就会生成多少个`PathInstanceMtadata`对象。  

如果其中的部分“数据根目录”有问题，比如因为磁盘损坏无法访问，那么就会在`health_status_`中进行标注为“不健康”的。

```
  Env* env_;
  const std::string block_manager_type_;
  const std::string filename_;
  std::unique_ptr<PathInstanceMetadataPB> metadata_;
  std::unique_ptr<FileLock> lock_;
  Status health_status_;
```

## 接口方法

### `Create()`

创建、写入数据、刷盘、关闭 一个“元数据文件”。

这里持久化的内容，就是一个`PathInstanceMetadataPB`对象。

在该方法中，先是生成一个`PathInstanceMetadataPB`对象，然后将这个`protobuf`对象序列化，最后写入到磁盘。

**参数含义：**  
1. 参数`uuid`：是这个`instance`（即当前根目录）所对应的`uuid`。
2. 参数`all_uuids`：在当前进程配置的所有“数据根目录”对应的`UUID`列表;

注意：参数`all_uuids`中，一定是包含参数`uuid`的。

```
Status PathInstanceMetadataFile::Create(const string& uuid, const vector<string>& all_uuids) {
  DCHECK(!lock_) <<
      "Creating a metadata file that's already locked would release the lock";
  DCHECK(ContainsKey(set<string>(all_uuids.begin(), all_uuids.end()), uuid));

  // Create a temporary file with which to fetch the filesystem's block size.
  //
  // This is a safer bet than using the parent directory as some filesystems
  // advertise different block sizes for directories than for files. On top of
  // that, the value may inform intra-file layout decisions made by Kudu, so
  // it's more correct to derive it from a file in any case.
  string created_filename;
  string tmp_template = JoinPathSegments(
      DirName(filename_), Substitute("getblocksize$0.XXXXXX", kTmpInfix));
  unique_ptr<WritableFile> tmp_file;
  RETURN_NOT_OK(env_->NewTempWritableFile(WritableFileOptions(),
                                          tmp_template,
                                          &created_filename, &tmp_file));
  SCOPED_CLEANUP({
    WARN_NOT_OK(env_->DeleteFile(created_filename),
                "could not delete temporary file");
  });
  uint64_t block_size;
  RETURN_NOT_OK(env_->GetBlockSize(created_filename, &block_size));

  PathInstanceMetadataPB new_instance;

  // Set up the path set.
  PathSetPB* new_path_set = new_instance.mutable_path_set();
  new_path_set->set_uuid(uuid);
  new_path_set->mutable_all_uuids()->Reserve(all_uuids.size());
  for (const string& u : all_uuids) {
    new_path_set->add_all_uuids(u);
  }

  // And the rest of the metadata.
  new_instance.set_block_manager_type(block_manager_type_);
  new_instance.set_filesystem_block_size_bytes(block_size);

  return pb_util::WritePBContainerToPath(
      env_, filename_, new_instance,
      pb_util::NO_OVERWRITE,
      FLAGS_enable_data_block_fsync ? pb_util::SYNC : pb_util::NO_SYNC);
}
```

说明1：  
调用该方法时，这个文件一定**不能**已经被加了`文件锁`。

因为这里最终会向该文件写入内容，调用的是`pb_util::WritePBContainerToPath()`，在这个方法中，在写入内容以后，会关闭文件。 而“关闭文件”出触发释放在该文件上的“文件锁”。

这样，如果有其它线程自认为“已经加了‘文件锁’”，而进行一些操作的过程中。这个“文件锁”却因为在这里被释放了，也就是说，在之前那个“加锁的”线程 感知不到 的情况下，清理掉了锁，这可能会导致“错误的结果”。

>> a safer bet than ... : 比...更安全  
>> intra-file: 文件内部的  

说明2：  
在获取文件系统中，一个文件的“块大小”时，这里采用的方式是：先创建一个临时文件，然后从该文件获取它的“文件块大小”。

> 备注：这里的“文件块大小”，说的是“文件系统的块大小”，就是`系统调用stat()`函数返回的`blksize_t`。

另外一种可能的方式是：从该文件的“父目录”中，向“文件系统”获取“文件块大小”。

但是：采用“新建临时文件，然后从‘临时文件’来获取这个信息”，是更安全的。

原因是：  
1. 一些文件系统会建议“目录”和“文件”使用不同的“文件块大小”；
2. 这个“块大小”的值，可能会影响`kudu`将数据在“文件内部”的 排列方式。

> 备注：这里的第2个理由，在`log`格式的`BlockManager`在组织文件结构的时候，会使用到“文件系统块大小”。

说明3：  
这里新建的临时文件，文件名为`getblocksize.kudutmp.XXXXXX`.

说明4：  
这里使用了`SCOPED_CLEANUP宏`，来做到在函数结束后，自动删除文件。

说明5：  
如果在刚刚创建了这个临时文件后，系统宕机，即`SCOPED_CLEANUP宏`来不及起作用，进程就没有了。  
那么在重启进程后，会有人来负责删除这些“临时文件”。  
在`DataDirManager::Open()`中，即在进程启动过程中，会专门清理这些临时文件。

说明6：  
一个细节问题：`protobuf`结构生成的方法的说明。  

在将“函数参数`all_uuids`中的所有`uuid`”都添加到“`new_path_set`中的`all_uuids`成员”中时，调用的方法是`add_all_uuids()`。

在`protobuf`中，如果一个成员(加入名字为`names`)是`repeated`类型，那么在生成的`.cpp`文件中，其中“向其中添加一个成员的方法”，名字就是`add_names(xxx)`。

而这里，在`protobuf`结构(`PathInstanceMetadataPB`)的成员变量名字就是`all_uuids`，所以声明的方法就是`add_ + all_uuids + ()`，即`add_all_uuids()`。  

也就是说，这个方法中的`all`并不是表示“所有的”的意思，即这里的`add_all_uuids()`并不是说要“要添加所有的`uuids`”, 而是说“要将一个`uuid`添加到`all_uuids`属性中”。

### `LoadFromDisk()`

打开文件、读取、验证、关闭 “元数据”文件。

在该函数返回成功，可能会有两种情况之一：
1. 当前类的`metadata_`成员会被“文件中的保存的`PathInstanceMetadataPB`结构”覆盖掉；
2. 遇到了磁盘故障。这个情况，该方法会返回`Status::OK`，但是`health_status_`成员会被设置为“`non-OK`”.

```
Status PathInstanceMetadataFile::LoadFromDisk() {
  DCHECK(!lock_) <<
      "Opening a metadata file that's already locked would release the lock";

  unique_ptr<PathInstanceMetadataPB> pb(new PathInstanceMetadataPB());
  RETURN_NOT_OK_FAIL_INSTANCE_PREPEND(pb_util::ReadPBContainerFromPath(env_, filename_, pb.get()),
      Substitute("Failed to read metadata file from $0", filename_));

  if (pb->block_manager_type() != block_manager_type_) {
    return Status::IOError(Substitute(
      "existing data was written using the '$0' block manager; cannot restart "
      "with a different block manager '$1' without reformatting",
      pb->block_manager_type(), block_manager_type_));
  }

  uint64_t block_size;
  RETURN_NOT_OK_FAIL_INSTANCE_PREPEND(env_->GetBlockSize(filename_, &block_size),
      Substitute("Failed to load metadata file. Could not get block size of $0", filename_));
  if (pb->filesystem_block_size_bytes() != block_size) {
    return Status::IOError("Wrong filesystem block size", Substitute(
        "Expected $0 but was $1", pb->filesystem_block_size_bytes(), block_size));
  }

  metadata_.swap(pb);
  return Status::OK();
}
```

说明1：调用该方法时，这个文件一定*不能*已经被加了`文件锁`。

原因和`Create()`类似。其中从文件读取`protobuf`结构的时候，会有“关闭文件”的操作，从而会触发释放“文件锁”的操作。

说明2：在读取以后，进行的校验有：
1. 磁盘上保存的`block_manager_type` 要等于 “当前对象的`block_manager_type`”；  
    根本原因是：两种类型的`BlockManager`不是互相兼容的。
2. 会从“元数据”文件本身，向“文件系统”获取它的“文件系统块大小”，然后和磁盘上读取出来的`protobuf`结构中的“`filesystem_block_size_bytes`”相比较，两者必须相等。 

说明3：去获取“数据根目录”的“文件系统块大小”时，最准确的方法就是`从文件本身`来获取。（参见`Create()`方法，在那个方法中，是通过创建临时文件来获取“文件系统块大小”的）而在这里，因为已经有了`instance file`这个文件，所以直接通过这个文件来获取就行了。

>> 问题1：如果`block_size`发生了变化，会有什么问题？

**注意：这里会检查`block_size`是否发生了变化，那么就会加载失败。**
参见`src/kudu/fs/fs.proto`中的`PathInstanceMetadataPB`。    
如果手动将“数据文件”从一个目录拷贝到另一个目录（那么就要求在这两个目录中，对应的"文件系统块大小"必须是相同的）。否则会无法成功加载。

### `Lock()`

对“元数据文件”进行加“文件锁”。

注意：该文件必须存在。

如果该文件已经被加锁，那么返回失败。

释放该文件的“文件锁”，只有在`Unlock()`方法；  
那么会去调用`Unlock()`的场景有：
1. 当前`PathInstanceMetadataFile`对象被析构；
2. 进程退出；

注意：在“元数据文件”的`fd`被关闭的时候，也会释放这个“文件锁”。

也就是说，不可以在一个“已经加锁的”文件上，去调用`Create()`和`LoadFromDisk()`(因为这两个方法中，有从文件读取`protobuf`结构，和 将`protobuf`结构写入到文件 的操作，其中会有“关闭文件”的操作，即会释放“文件锁”)。

```
Status PathInstanceMetadataFile::Lock() {
  DCHECK(!lock_);

  FileLock* lock;
  RETURN_NOT_OK_FAIL_INSTANCE_PREPEND(env_->LockFile(filename_, &lock),
                                      "Could not lock block_manager_instance file. Make sure that "
                                      "Kudu is not already running and you are not trying to run "
                                      "Kudu with a different user than before");
  lock_.reset(lock);
  return Status::OK();
}
```

### `Unlock()`

```
Status PathInstanceMetadataFile::Unlock() {
  DCHECK(lock_);

  RETURN_NOT_OK_FAIL_INSTANCE_PREPEND(env_->UnlockFile(lock_.release()),
                                      Substitute("Could not unlock $0", filename_));
  return Status::OK();
}
```

### `SetInstanceFailed()`
将当前对象的“将康状态”设置为`non-ok`。

比如：在遇到了“磁盘故障”的时候。

注意：如果有错误的话，是不能保证该对象的`metadata_`成员是有效的。甚至有可能是一个未设置的值。

所以，在“不健康”的状态下，不能访问`metadata_`成员。

### `healthy()`
返回当前的是否健康。返回值是`bool`类型，如果健康，就返回`true`。

比如说：如果一个`path instance`文件，所处的磁盘是坏的，那么在该`path instance`上调用`healthy()`方法，结果应该返回`false`。

### `health_status()`
直接返回`health_status_`。

```
 void SetInstanceFailed(const Status& s = Status::IOError("Path instance failed")) {
    health_status_ = s;
  }

  bool healthy() const {
    return health_status_.ok();
  }

  const Status& health_status() const {
    return health_status_;
  }
```

### `static CheckIntegrity()`
检查一组`PathInstanceMetadataFile`对象的“完整性”。

注意：这个方法是用来检查“完整性”，并不是检查“是否健康”。 （“是否健康”由`healthy()`来标识）  
  在检查过程中，“不健康”的`PathInstanceMetadataFile`对象会被忽略掉。

注意1：虽然在这个方法的实现中，都是根据`uuid`和`PathInstanceMetadataFile`来进行校验的，但是如果需要返回错误信息，都是会转化为`目录`（即在错误信息中，包含的指示性文字，都是“目录”）。

这是因为：`uuid`和`instance files`都是内部实现细节，用户并不需要关心。

**注意2：这个函数的调用场景**  
目前只有一处会调用该函数：在`DataDirManager::Open()`中。也就是说，在`DataDirManager`被首次打开的时候，也就是“进程刚启动，去加载‘所有的数据根目录’的时候”。

这里的`instances`参数，包含的是：所有已经加载的“数据目录”对应的`PathInstanceMetadataFile`对象。

```
Status PathInstanceMetadataFile::CheckIntegrity(
    const vector<unique_ptr<PathInstanceMetadataFile>>& instances) {
  CHECK(!instances.empty());

  int first_healthy = -1;
  for (int i = 0; i < instances.size(); i++) {
    if (instances[i]->healthy()) {
      first_healthy = i;
      break;
    }
  }
  if (first_healthy == -1) {
    return Status::NotFound("no healthy data directories found");
  }

  // Map of instance UUID to path instance structure. Tracks duplicate UUIDs.
  unordered_map<string, PathInstanceMetadataFile*> uuids;

  // Set of UUIDs specified in the path set of the first healthy instance. All
  // instances will be compared against it to make sure all path sets match.
  set<string> all_uuids(instances[first_healthy]->metadata()->path_set().all_uuids().begin(),
                        instances[first_healthy]->metadata()->path_set().all_uuids().end());

  if (all_uuids.size() != instances.size()) {
    return Status::IOError(
        Substitute("$0 data directories provided, but expected $1",
                   instances.size(), all_uuids.size()));
  }

  for (const auto& instance : instances) {
    // If the instance has failed (e.g. due to disk failure), there's no
    // telling what its metadata looks like. Ignore it, and continue checking
    // integrity across the healthy instances.
    if (!instance->healthy()) {
      continue;
    }
    const PathSetPB& path_set = instance->metadata()->path_set();

    // Check that the instance's UUID has not been claimed by another instance.
    PathInstanceMetadataFile** other = InsertOrReturnExisting(
        &uuids, path_set.uuid(), instance.get());
    if (other) {
      return Status::IOError(
          Substitute("Data directories $0 and $1 have duplicate instance metadata UUIDs",
                     (*other)->dir(), instance->dir()),
          path_set.uuid());
    }

    // Check that the instance's UUID is a member of all_uuids.
    if (!ContainsKey(all_uuids, path_set.uuid())) {
      return Status::IOError(
          Substitute("Data directory $0 instance metadata contains unexpected UUID",
                     instance->dir()),
          path_set.uuid());
    }

    // Check that the instance's UUID set does not contain duplicates.
    set<string> deduplicated_uuids(path_set.all_uuids().begin(),
                                   path_set.all_uuids().end());
    string all_uuids_str = JoinStrings(path_set.all_uuids(), ",");
    if (deduplicated_uuids.size() != path_set.all_uuids_size()) {
      return Status::IOError(
          Substitute("Data directory $0 instance metadata path set contains duplicate UUIDs",
                     instance->dir()),
          JoinStrings(path_set.all_uuids(), ","));
    }

    // Check that the instance's UUID set matches the expected set.
    if (deduplicated_uuids != all_uuids) {
      return Status::IOError(
          Substitute("Data directories $0 and $1 have different instance metadata UUID sets",
                     instances[0]->dir(), instance->dir()),
          Substitute("$0 vs $1", JoinStrings(all_uuids, ","), all_uuids_str));
    }
  }

  return Status::OK();
}
```

这里检查完整性，检查的内容有：
1. 是否存在“健康的”的“数据根目录”。（允许部分“数据目录”是“不健康的”，但是不能 所有目录 都不健康）
2. 检查配置的“数据根目录”的数量，要和在“健康的”数据根目录中读取到的`all_uuids`是相同的。
3. 检查是否所有的“数据根目录”，都有不同的`uuid`。（即，不同的“数据根目录”，不能使用重复的`uuid`）;
4. 检查任何一个目录的`uuid`，都要在`PathSet::all_uuids`中。
5. 检查任何一个目录中保存的`PathSet::all_uuids`，其中都不能有重复的`uuid`。
6. 所有“数据根目录”下的`all_uuids`，都要是完全相同的。   
    检查方法：先从一个“健康的”数据根目录中，获得`all_uuids`。然后遍历所有健康的“数据根目录”，所遍历的“数据根目录”的`all_uuids`(转化为`std::set`类型，记为`deduplicated_uuids`)，然后要满足`all_uuids == deduplicated_uuids`。  
    也就是说，如果所有“数据根目录”中记录的`all_uuids`都和“健康的数据目录的`all_uuids`”相同，那么所有“数据根目录”的`all_uuids`就是相同的。  

这个函数中，会忽略掉“不健康”的目录；而且，没有检查`all_uuids`的数量，必须等于当前处于“健康状态”的所有`uuid`的数量。

**场景1：其中的某个磁盘损坏（即对应的`path instance`会处于“不健康”状态），是可以通过检查**  
比如：原来有5个目录，每个目录都是“完全一致”的状态。但是这时，其中一个目录的磁盘发生错误，也就是说，其中一个“数据根目录”无法再进行任何读写操作。 这种状态，也是可以通过检查的。

**场景2：其余都不变化，只是添加了1个新的空目录（`path instance`会处于“不健康”状态），无法通过检查**  
上面检查内容的“第`2`条”。 在重新启动时，发现当前的“数据根目录”的数量，大于在“当前”

在所有的健康的“数据根目录”中，记录的是`A`，`B`，`C`3个；因为添加了一个，所以目前配置的“数据根目录”有`4`个（`A`, `B`, `C`, `D`），那么不满足第`2`条，所以检查不通过。

**场景3：其余都不变化，去掉了一个“旧目录”，然后同时添加了1个新的空目录（`path instance`处于“不健康”状态），可以通过检查**   
在所有的‘健康的’“数据根目录”中，记录的是`A`，`B`，`C`3个；
因为删除了一个，加入删除了`C`；同时添加了一个，记为`D`，所以目前配置的“数据根目录”有`3`个（`A`, `B`, `D`）。  
所以可以通过上面“第`2`条”的检查。  
因为“不健康”的“数据根目录”，在检查时会被忽略，所以也可以通过其它检查。

>> 问题: 上面的“场景3”，是否应该被通过？

# `RETURN_NOT_OK_FAIL_INSTANCE_PREPEND`宏

在`.cpp`文件中定义，所以只能本文件使用。

**宏参数的说明：**  
1. `status_expr`参数：表示一个 执行结果为`Status`类型的语句。
2. `msg`参数：是一个字符串，在需要打印错误日志时，打印的错误信息。

**执行流程：**  

评估传入进来的`Status`变量：
1. 如果是`Status::OK`，那么什么都不做。（因为是个宏，所以会继续执行原函数）。
2. 如果`non-ok`状态：
    + 如果是 1) `not found`；或者 2) `disk failure` ，那么会  
      - 打印一条错误日志；
      - 并且将当前对象的`health_status_`状态修改为“不健康”，
      - 会调用`return Status::OK()`，**原函数会结束运行**  
       （注意这个时候返回的状态是`OK`，不是错误）  
    + 如果是其它错误，那么直接调用`returan _status`， **原函数会结束运行**  

注意1：这里有两种特殊的`Status`（即`_s.IsNotFound()`和`_s.IsDiskFailure()`），这两种状态下，调用当前“宏”的函数会返回`Status::OK`，而不是失败。

注意2：在遇到这两种错误的时候，“原函数”会结束运行，不会继续执行。  
也就是说，如果“原函数”中在该宏的后面，还有一些其它逻辑，实际上是得不到处理的。  
当然，因为实际上是遇到的错误的状态（比如文件不存在），也不需要再继续处理。  

**注意3：也正是因为这两种`Status`的特殊处理，所以在当前的实现中（“调用该宏的多个函数”，“以及外部调用这些函数的地方”，以及一些会去使用`PathInstanceMetadataFile`对象的地方），即使返回的值是`Status::OK`，还需要进一步的检查（因为这两种状态下，会设置`health_status_`，所以需要检查的时候，检查的内容就是`health_status_`）。**   

也就是说，即使构造`PathInstanceMetadataFile`的地方，成功返回`Status::OK`，但实际上这个`PathInstanceMetadataFile`也是不能使用的。

比如1：在`static PathInstanceMetadataFile::CheckIntegrity()`中，会先去检查“是否所有的数据目录都是‘不健康’的”（参见其中的`first_healthy`标志）  

比如2：在`DataDirManager`中，有很多地方，在使用`PathInstanceMetadataFile`的时候，都是要先通过`intance.healthy()`来判断一下是否为“健康的”，只有是“健康的”状态下，才会进一步的处理。

>> thwart: 挫败；阻碍；  
>> blatant: 喧嚣的；

**返回值的说明：**  
如果有“磁盘故障”，那么在从“文件系统”中读取文件时会失败，最终导致读取“元数据文件”时“操作系统”可能会返回`Status::NotFound`（也不仅在“磁盘正常、但文件不存在”的情况下会返回`Status::NotFound`，在有“磁盘故障”的时候，也可能会返回`Status::NotFound`）。 
因此，对于这种“磁盘故障”，处理逻辑和 “不存在对应的‘元数据文件’”是一样的。

处理的方式，参见上面的“执行流程”部分。

```
#define RETURN_NOT_OK_FAIL_INSTANCE_PREPEND(status_expr, msg) do { \
  const Status& _s = (status_expr); \
  if (PREDICT_FALSE(!_s.ok())) { \
    const Status _s_prepended = _s.CloneAndPrepend(msg); \
    if (_s.IsNotFound() || _s.IsDiskFailure()) { \
      health_status_ = _s_prepended; \
      LOG(INFO) << "Instance is unhealthy: " << _s_prepended.ToString(); \
      return Status::OK(); \
    } \
    return _s_prepended; \
  } \
} while (0)
```

说明：这里`status_expr`是一个返回`Status`对象的表达式。

注意：这里的`_s`的定义，它是直接定义了一个“引用”。也就是说，这个“引用变量”它所引用的对象是一个“表达式返回的临时`Status`对象”。 

不过，只要这个“作用域”还有效，那么这个“临时对象”就不会被析构，所以是没问题的。

但是，如果是遇到了错误，并需要返回一个`Status`时，那么**就需要拷贝一下**该`Status`对象（见上面的`CloneAndPrepend()`）。






