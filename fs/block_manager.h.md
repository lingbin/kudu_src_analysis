[TOC]

文件： `src/kudu/fs/block_manager.h`


# `class Block`

`Block`是`Kudu`与本地文件系统交互时，最小的单位。

是一个“基类”，对于“读写”操作，分别有不同的子类：`WritableBlock`和`ReadableBlock`。

## 设计理念：

`Block`的接口设计，反应了`Kudu`中磁盘存储 的设计理念：
1. `Block`是`append only`的；
2. `Block`一旦被写入，就不可以再修改；
3. 对于“读取”操作是线程安全的，即可能会有“多个读者”并发的读取它。
4. 对于“写入”不是线程安全的。即“多个写者”不可以并发修改一个`Block`;

## 接口

### `id()`

返回当前`Block`的`id`;

基类只提供了这一个接口，是个纯虚函数。 

```
class Block {
 public:
  virtual ~Block() {}

  virtual const BlockId& id() const = 0;
};
```

# `class WritableBlock`

描述一个`可供写入的块`，**这个`Block`是已经被打开的**。

**注意1：这个类也是一个“抽象类”**   
该类没有任何“成员属性”，只提供了一些列的“纯虚函数接口”（和“写操作”相关的功能）。

根据功能的不同，有两种子类：

1. `FileWritableBlock`： 用来写数据的；
2. `LogWritableBlock`： 用来写`WAL`的；

**注意2：只能有一个线程对它进行写入，并且只能进行“追加写入”。**  

**注意3：`Close()`操作是一个非常昂贵的操作。**  
因为它要将 “脏块”和“元数据”都刷到磁盘上。

额外提供了两个接口，来提高`Close()`的性能：
1. 在调用`Close()`之前，先调用`Finalize()`：  
    在`FLAGS_block_manager_preflush_control`被设置成`finalize`时，如果在调用两个函数中间有“足够多的工作”，那么这种方式，会让在`Close()`的时候，有比较少的写入IO。
2. 在一组`Blocks`上调用`CloseBlocks()`：  
    这个方法能够保证：  
    + `dirty block`的刷盘操作会尽可能地被“批量处理”，从而减少IO。
    + 在等待写入函数返回时，是“并发地”的进行等待。

>> 问题：什么样的块是“脏块”？

>> 问题：`CloseBlocks()`是谁的方法？

注意4：如果一个`WritableBlock`没有被显式的`Close()`，那么它最终会被删除（也称为`aborted`），即并不会被刷盘，而是直接删除。

## `enum WritableBlock::State`

### `CLEAN`
表示在当前`Block`中，没有“脏数据”；

### `DIRTY`
表示在当前`Block`中，有“脏数据”；

### `FINALIZED`
表示数据已经写完，后续不可以 再写入数据。

注意：这个状态，不保证数据被持久化到磁盘。（但是实际上确实会有数据已经落盘，具体落盘多少，由操作系统根据当时的IO负载，执行调度）

### `CLOSED`
当前`Block`已经被关闭。 不可以再进行“任何操作”。

注意：这个状态下，保证`Block`的数据已经被落盘。

```
  enum State {
    CLEAN,
    DIRTY,
    FINALIZED,
    CLOSED
  };
```

## 接口列表

这些接口都是“纯虚函数”，在子类中进行实现。

### 析构函数

注意：这里会检查 当前`Block`是否已经被显式的被`Close()`掉，如果没有，那么会调用`Abort()`（会把当前`Block`删掉）。

说明：检查是否已经被`Close()`的方法：在每个`WritableBlock`子类中，都有一个`WritableBlock::State`类型的成员属性，如果该属性的值为`CLOSED`，那么就是已经被`Close()`过。

### `Close()`

把该`Block`在内存中的数据写入到磁盘，并且删除原有的内存结构。

如果成功返回，就保证这个`Block`的数据已经被成功持久化到磁盘上。

### `Abort()`

和`Close()`类似，但是**只会删除在内存中数据结构，并不把数据刷盘**。

也就是说，如果该函数返回成功，那么当前`Block`对象就相当于“不存在的”。

### `block_manager()`
返回当前`Block`所属的`BlockManager`对象。

### `Append()`
把数据添加到`Block`中。

注意1：该方法返回后，并不保证数据被持久化到磁盘上。 如果要保证被持久化，必须调用`Close()`。

注意2：传入的数据是`Slice`类型。  
也就是说，在实现的时候，在该方法内部，可能需要进行数据拷贝。

### `AppendV()`
将“多个数据”添加到`Block`中。

注意：和`Append()`类似，该方法返回后，并不保证数据被持久化到磁盘上。 如果要保证被持久化，必须调用`Close()`。

### `Finalize()`

表示当前`Block`不会再进行任何写入。

注意：该方法，同样不保证数据被持久化。

注意2：当`FLAGS_block_manager_preflush_control`的值为`finalize`，那么在该方法中，会进行触发异步刷新“dirty blocks”到磁盘。

如果在“最后一次调用`Append()`” 和 “将来调用`Close()`”  之间，有一些其它的工作需要去做，那么通过先调用`Finalize()`，会减少 “将来调用`Close()`时” 等待IO完成的时间。（因为在真正调用`Close()`之前，操作系统可能已经异步刷盘了很多数据）

这和 “预读”（`readahead`或`prefetching`）的特点是类似的。

>> 问题：什么意思？

### `BytesAppended()`
返回已经写入的数据大小，即前面通过调用`Append()`和`AppendV()`传入的数据大小。

### `state()`
返回当前`Block`的状态。

```
class WritableBlock : public Block {
  virtual ~WritableBlock() {}

  virtual Status Close() = 0;

  virtual Status Abort() = 0;

  virtual BlockManager* block_manager() const = 0;

  virtual Status Append(const Slice& data) = 0;

  virtual Status AppendV(ArrayView<const Slice> data) = 0;

  virtual Status Finalize() = 0;

  virtual size_t BytesAppended() const = 0;

  virtual State state() const = 0;
}
```

# `class ReadableBlock`

描述一个“可供读取”的块，**这个`Block`是已经被打开的**。

**注意1： 和`WritableBlock`一样，这个类也是“抽象类”。**

和`WritableBlock`一样，该类没有任何“成员属性”，只提供了一些列的“纯虚函数接口”（和“读取”相关的功能）。

根据功能的不同，有两个子类：
1. `FileReadableBlock`: 用来读取数据文件；
2. `LogReadableBlock`：用来读取`WAL`；

**注意2：对于同一个物理块，可能会创建多个`ReadableBlock`对象。**  

**注意3：即使是同一个`ReadableBlock`对象，也有可能会同时被 多个线程 同时使用，并发读取。**

## 接口列表

### 析构函数

### `Close()`
清理该`Block`的内存数据；（和`WritableBlock::Close()`不同，这里是读取操作，所以不会有“将数据刷盘”的逻辑）

### `block_manager()`
返回当前`Block`所属的`BlockManager`对象。

### `Size()`
返回当前`Block`在磁盘上占用的空间大小。

注意：该方法的返回值，并不是“已经读取了多少数据”，而是该`Block`的大小。

### `Read()`
在当前`Block`中，从`offset`位置，读取`result.size()`大小的数据。

说明：传入的参数`result`是一个`Slice`类型。也就是说，该`Slice`对象所间接引用的内存，会被修改（`Block`是不可修改的，所以该`Block`的内容是不会被修改的）。

注意：因为该方法并没有参数返回“实际读取了多少数据”，所以要求在返回`Status::OK`的时候，必然是正常读取了`result.size()`大小的数据，即是一个“定长的”读取结果。

也就是说：如果当前`Block`中剩余的内容，小于`result.size()`，那么会返回“错误”。

### `ReadV()`
和`Read()`类似，不过这里是将读取的数据 放入到多个`Slice`中。

### `memory_footprint()`
返回当前`Block`对象所占用的内存（包括当前对象本身，以及内部的成员）。

```
class ReadableBlock : public Block {
 public:
  virtual ~ReadableBlock() {}

  virtual Status Close() = 0;

  virtual BlockManager* block_manager() const = 0;

  virtual Status Size(uint64_t* sz) const = 0;

  virtual Status Read(uint64_t offset, Slice result) const = 0;

  virtual Status ReadV(uint64_t offset, ArrayView<Slice> results) const = 0;

  // Returns the memory usage of this object including the object itself.
  virtual size_t memory_footprint() const = 0;
};
```

# `struct CreateBlockOptions`

创建`Block`的选项。

用来查找存放当前`Block`的`DataDirGroups`。

说明：在将来，这个结构，可能会被用来 根据“不同`Block`的类型”来指定“对应的目录”。  
比如：对于`bloomfilter Block`，优先放在`SSD`中。

```
struct CreateBlockOptions {
  const std::string tablet_id;
};

```

# `struct BlockManagerOptions`

创建`BlockManager`对象的选项。

## 成员

>> 扩展：这里`metric_entity`和`parent_mem_tracker`属性，在`FsManagerOpts`对象中也存在，作用也是类似的。

### `metric_entity`

`BlockManager`所对应的`Metric`信息。

如果该成员为`nullptr`，那么当前`BlockManager`中就不会产生`metrics`信息。

默认是`nullptr`。

### `parent_mem_tracker`

当前`BlockManager`内部创建`MemTracker`对象的时候，都是将这个`parent_mem_tracker`作为 “父`MemTracker`”。

如果该成员为`nullptr`，那么新创建的`MemTracker`对象，都会将“父亲”设置为`root tracker`。

默认值是`nullptr`。

### `read_only`

当前`BlockManager`是否为“只读”的。

默认值是`false`，即不是“只读的”。

```
struct BlockManagerOptions {
  BlockManagerOptions();

  scoped_refptr<MetricEntity> metric_entity;
  std::shared_ptr<MemTracker> parent_mem_tracker;
  bool read_only;
};
```

# `class BlockManager`

工具类，用来管理所有`Block`的生命周期。

**注意1：该类的所有方法都是“线程安全”的。**

**注意2： 这个类是一个“抽象类”。**  

该类没有任何“成员属性”，只提供了一些列的“纯虚函数接口”（和“管理`Block`”相关的功能）。

根据功能的不同，有两种子类：

1. `FileBlockManager`： 用来写数据的；
2. `LogBlockManager`： 用来写`WAL`的；

## 接口

### `static block_manager_types()`  -- 静态方法
返回在当前系统上，`BlockManager`所支持的类型。

在`linux`系统上，支持两种：
1. `file`;
2. `log`;

在其它系统上，只支持一种（`file`）。

>> 问题：这两种实现，有什么区别？

```
  static std::vector<std::string> block_manager_types() {
#if defined(__linux__)
    return { "file", "log" };
#else
    return { "file" };
#endif
  }
```

### `Open()`

一个`BlockManager`会将“它自身的一些数据”持久化到磁盘。这个`Open()`方法，就是根据“已经保存在磁盘上”的该`BlockManager`自身的数据，构建出`BlockManager`对象，并进行一些“一致性检查”。 

如果该`BlockManager`不是在“只读”模式下进行构建的，那么在发现“存在不一致”的时候，会尝试进行修复。

在读取并创建`BlockManager`的过程中，会有一些统计信息（包括检查结果，修复结果等）：
+ 如果`FsReport* report`参数不是`nullptr`，那么，保存在`report`参数中；
+ 如果`report`参数为`nullptr`，那么会将这些信息，打印到日志中。 如果存在`fatal`级别的“不一致”（即无法自动修复的“不一致”），那么会返回错误。

>> 问题：如果`report != nullptr`，那么遇到`fatal`级别的“不一致”时，是否会返回错误？

注意：如果在磁盘上，并没有相应的`BlockManager`数据，或者无法打开，那么会返回错误。

### `CreateBlock()`

根据提供的创建选项(`CreateBlockOptions`对象)，创建一个新的`Block`对象，并打开它。

在创建过程中，会生成该`Block`对象的`BlockId`。

注意：这个操作，并不保证它会被刷盘。 如果要确认刷盘，必须要调用`Block::Close()`函数。

注意：如果因为某个原因，导致无法创建，那么不要修改 “传入的`block`参数”。  
也就是说，只有“成功创建”的时候，才可以修改`block`参数。

### `OpenBlock()`

打开一个`Block`，用来进行读取，即返回一个`ReadableBlock`对象。

**注意：尽管 删除一个已经打开的`Block`（即已经成功调用过了`OpenBlock()`）是安全的，但是如果“删除`Block`操作”和`OpenBlock()`并发，那么会有问题**。

因为，在某些`BlockManager`的子类中，这种并发会导致“错误的行为”。  

比如：`OpenBlock()`可能会成功，但是后续的`ReadableBlock`操作可能会失败。 

>> 说明：这个行为是“错误的”，意思是：既然一个`Block`被成功打开了，那么它就应该可以成功的进行读取。

注意2：和`CreateBlock()`类似，在出现错误的时候，不要修改“传入的`block`参数”。

### `NewCreationTransaction()`

创建一个事务（专用用来创建`Block`的事务，即`block creation transaction`），从而可以整个创建流程中的操作都打包起来：  
1. 把新`block`创建出来 的操作；
2. 把“已注册的”的`Block`都`close`掉 的操作； 

### `NewDeletionTransaction()`

创建一个事务（专用用来删除`Block`的事务，即`block deletion transaction`），从而可以整个删除流程中的操作都打包起来。

和`BlockManager::DeleteBlock()`相似，只有在该`Block`的“最后一个`Reader`或`Writer`”都被`Close`以后，才会进行真正的把该`Block`删除掉。

注意：`NewCreationTransaction()`方法返回的是`std::unique<>`; 但是`NewDeletionTransaction()`返回的是`std::shared_ptr<>`；

>> 问题：为什么返回两种类型的智能指针？

### `GetAllBlockIds()`

获取当前`BlockManager`所管理的所有的`Block`的`id`，包括所有的`ReadableBlock`和`WritableBlock`。

说明：结果是通过“传入的`block_ids`参数”中返回的。

注意1：在返回结果中：
1. 不保证任何的顺序；
2. 多次调用该函数，结果中的顺序也不是确定的（即使结果中包含完全相同的`block_id`列表）。

注意2：因为是并发场景，所以在某个调用获取到一个`block_id`列表以后，其中的某个`Block`可能已经不存在了。（因为被并发删除了）。

### `NotifyBlockId()`

通知`BlockManager`，某个`block_id`已经存在了。  

**这个方法的作用：**  
允许`BlockManager`尽量 “顺序的”使用`block_id`，并且能够避免那些已经被 “外部引用的”、之前尚未发现的`id`”。 

比如：某个目录坏掉了，这个目录下的所有`Block`就读不到了。

>> 问题：举个使用场景的例子。

### `error_manager()`
获取相应的`FsErrorManager`对象指针。用来处理错误信息。

```
class BlockManager {
 public:
  virtual ~BlockManager() {}

  virtual Status Open(FsReport* report) = 0;

  virtual Status CreateBlock(const CreateBlockOptions& opts,
                             std::unique_ptr<WritableBlock>* block) = 0;

  virtual Status OpenBlock(const BlockId& block_id,
                           std::unique_ptr<ReadableBlock>* block) = 0;

  virtual std::unique_ptr<BlockCreationTransaction> NewCreationTransaction() = 0;
  virtual std::shared_ptr<BlockDeletionTransaction> NewDeletionTransaction() = 0;

  virtual Status GetAllBlockIds(std::vector<BlockId>* block_ids) = 0;

  virtual void NotifyBlockId(BlockId block_id) = 0;

  virtual FsErrorManager* error_manager() = 0;
};
```

# `class BlockCreationTransaction`

将“创建`Block`”所需要的多个操作打包起来。

有两个原因：
1. 打包起来以后，对应的`BlockManager`可以针对多个`Block`的同步，进行尽可能的优化，从而提高性能；
2. 可以在一个“逻辑操作”中，跟踪记录“`Block`的所有相关创建操作”；

这个类**不是线程安全**的。

如果不是必需的情况下，不建议 在多个线程中共享同一个该对象。

如果一定要在多个线程中共享，那么应该 显式加上 额外的互斥控制 逻辑，从而做到“线程安全”。

**注意：和本`.h`文件中的其它几个类是一样的，这个类也是一个“抽象类”**。  

该类没有任何“成员属性”，只提供了`2`个的“纯虚函数接口”（和“创建`Block`”相关的功能）。

根据功能的不同，有两种子类：
1. `FileBlockCreationTransaction`： 用来写数据的；
2. `LogBlockCreationTransaction`： 用来写`WAL`的；

**使用场景**
1. 根据不同的场景，创建对应类型的`block create transaction`;
2. 向每个block中添加数据；
3. 最后提交这个事务（其中会将这些`Block`都关闭掉）

**注意：这里虽然叫“事务”，但实际上只是“将多个操作打包在一起”，并不具备`ACID`特性**  
也就是说：这里不保证所有的“创建操作”都“原子成功”。  

所以，`CommitXxx()`的返回值是`Status`类型。如果返回了失败，是可能只有部分`Block`被创建的。

如果事务中只有部分`Block`被创建（返回了失败，而这部分成功创建的`Block`是可能已经被刷到磁盘上的），对于这些残留的`Block`，会有异步机制来清理它们。

**注意：因为本类是要创建`Block`，即可写的`Block`，所以其中的对象，一定是`WritableBlock`类型。**  

## 接口

### `AddCreatedBlock()`

向事务中添加一个`Block`。

### `CommitCreatedBlocks()`

提交当前事务（“创建`Block`的事务”），并且将所有的`Block`都关闭。

注意：在该函数成功返回的时候，所有的`Block`都保证已经持久化了。

```
class BlockCreationTransaction {
 public:
  virtual ~BlockCreationTransaction() = default;

  virtual void AddCreatedBlock(std::unique_ptr<WritableBlock> block) = 0;

  virtual Status CommitCreatedBlocks() = 0;
};
```

# `class BlockDeletionTransaction`

该类用来打包 一组删除`Block`的操作。

该类的很多方面，都和`class BlockCreationTransaction`类似。

该类也有两个动机：
1. 打包起来以后，对应的`BlockManager`可以针对多个`Block`进行批量删除，进行尽可能的优化，从而提高性能；
2. 可以在一个“逻辑操作”中，跟踪记录“Block的所有 删除 操作”；

这个类**不是线程安全**的。

如果不是必需的情况下，不建议 在多个线程中共享同一个该对象。

如果一定要在多个线程中共享，那么应该 **显式加上 额外的互斥控制** 逻辑，从而做到“线程安全”。

**注意：和本`.h`文件中的其它几个类是一样的，这个类也是一个“抽象类”。**

根据功能的不同，有两种子类：

1. `FileBlockDeletionTransaction`： 用来写数据的；
2. `LogBlockDeletionTransaction`： 用来写`WAL`的；

该类没有任何“成员属性”，只提供了`2`个的“纯虚函数接口”（和“删除`Block`”相关的功能）。

**注意：和`BlockCreationTransaction`是类似的，虽然叫“事务”，但实际上只是“将多个操作打包在一起”，并不具备`ACID`特性**  
也就是说：这里不保证所有的“删除操作”都“原子成功”。  

所以，`CommitXxx()`的返回值是`Status`类型。如果返回了失败，是可能只有部分`Block`被删除（记录在传入参数`deleted`中）。

## 接口

### `AddDeletedBlock()`

向“事务”中添加一个`待删除`的`Block`。

### `CommitDeletedBlocks()`

提交该“删除`Block`的事务”。

注意1：对于一个`Block`，只有当它的最后一个“Reader”或“Writter”都已经被关闭了，（即对应的`ReadableBlock`和`WritableBlock`对象都已经调用了`Close()`方法），这个`Block`才会真正被删除。

注意2：真正要删除的`Block`集合，是通过上面的`AddDeletedBlock()`函数逐个添加的。  并不是传入的`deleted`参数。`deleted`参数，是用来保存，在执行过程中，那些被删除掉的`Block`列表。

注意3：无论当前函数是返回成功，还是失败，那些被删除的`Block`列表，都会被放入到“传入的参数`deleted`参数”中。也就是说，比如说，函数返回失败，其中只有部分`Block`被删除，那么在`deleted`参数中，那些被成功删除的`Block`就会被放进去。

注意4：如果执行过程中，如果遇到某个错误，**并不是**立即返回失败，而是会继续执行。但是如果返回失败的话，那么返回的`Status`中所描述的错误，是首个遇到的错误。 参见`FileBlockDeletionTransaction::CommitDeletedBlocks()`的实现（文件：`src/kudu/fs/file_block_manager.h`） 

```
class BlockDeletionTransaction {
 public:
  virtual ~BlockDeletionTransaction() = default;

  virtual void AddDeletedBlock(BlockId block) = 0;

  virtual Status CommitDeletedBlocks(std::vector<BlockId>* deleted) = 0;
};
```


# `GetFileCacheCapacityForBlockManager()`  -- 全局函数

计算在一个`BlockManager`内，“可打开的文件句柄”的上限值。

相关的`FLAGS_block_manager_max_open_files`(默认值为`-1`，表示不限制)。

```
// Compute an upper bound for a file cache embedded within a block manager
// using resource limits obtained from the system.
int64_t GetFileCacheCapacityForBlockManager(Env* env);

int64_t GetFileCacheCapacityForBlockManager(Env* env) {
  // Maximize this process' open file limit first, if possible.
  static std::once_flag once;
  std::call_once(once, [&]() {
    env->IncreaseResourceLimit(Env::ResourceLimitType::OPEN_FILES_PER_PROCESS);
  });

  uint64_t rlimit =
      env->GetResourceLimit(Env::ResourceLimitType::OPEN_FILES_PER_PROCESS);
  // See block_manager_max_open_files.
  if (FLAGS_block_manager_max_open_files == -1) {
    // Use file-max as a possible upper bound.
    faststring buf;
    uint64_t buf_val;
    if (ReadFileToString(env, "/proc/sys/fs/file-max", &buf).ok() &&
        safe_strtou64(buf.ToString(), &buf_val)) {
      rlimit = std::min(rlimit, buf_val);
    }

    // Callers of this function expect a signed 64-bit integer, so we need to
    // cap rlimit just in case it's too large.
    return std::min((2 * rlimit) / 5, static_cast<uint64_t>(kint64max));
  }
  LOG_IF(FATAL, FLAGS_block_manager_max_open_files > rlimit) <<
      Substitute(
          "Configured open file limit (block_manager_max_open_files) $0 "
          "exceeds process open file limit (ulimit) $1",
          FLAGS_block_manager_max_open_files, rlimit);
  return FLAGS_block_manager_max_open_files;
}
```

说明1：这个方法在实现时，利用到了`std::call_once()`函数。

该方法能够保证：在任何并发情况下，该函数 最多只会被执行一次。

要配置`std::once_flag`一起使用。 并且要为了让多个线程的多次调用该函数时，在`std::call_once()`中的`std::once_flag`对象是同一个，这里的`std::once_flag`是一个`static`的变量。

参见：http://www.cplusplus.com/reference/mutex/call_once/  

说明2： 如果`FLAGS_block_manager_max_open_files == -1`，那么会去读取系统的“文件句柄限制”（通过内存文件`/proc/sys/fs/file-max`），这个文件中的值 会作为一个“上限”。

注意1：在`FLAGS_block_manager_max_open_files == -1`时，那么实际的限制值，就是计算出来的。  
这里的计算方法就是取 **总的限制值的 `40%`**。  

# FLAGS变量

## `FLAGS_block_manager_max_open_files`

所有`Block`所能够占用的“最大文件句柄数”，默认值是`-1`，表示不限制。

注意：如果设置，必须设置为一个“正数”。 （赋值为`0`是不允许的）。


## `FLAGS_block_manager_preflush_control`

用来控制`Block`的数据刷盘策略，表示何时将数据 `pre-flush` 到磁盘。

可能的取值有3个：
1. `finalize`: 当不再进行“写”操作的时候，就开始进行`pre-flush`，即调用`WritableBlock::Finalize()`方法以后。
2. `close`: 在“事务提交”的时候，才开始进行`pre-flush`。
3. `never`: 不进行任何的`pre-flush`，只有在调用`WritableBlock::Close()`时，才进行刷盘。

**默认值为`finalize`的考虑**  
是为了**在多盘场景下，优化“吞吐”**的目标。 方法是：在调用`fsync()`之前，通过异步地 将`Block`的数据刷盘。

1. 好处是：会增大吞吐；
2. 坏处是： 如果磁盘数量很少，并且“`WAL`”和“数据”在相同的磁盘上，那么可能会造成“较大的延迟”。

使用这个默认值的考虑因素（即，基于如下几种方式，延迟不会是很大的问题）：
+ `Raft`协议 已经将 “延迟”分散到了整个集群中；
+ “对延迟敏感”的应用，可以将`WAL`放在单独的磁盘；
+ “对延迟非常敏感”的应用，应该使用`SSD`来保存`WAL`；
+ 用户可以随时将该配置项的值修改为`never`，即吞吐会减小，并且延迟也会变好（即延迟会变低）。


