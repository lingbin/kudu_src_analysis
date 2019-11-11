[TOC]

文件： `src/kudu/util/env.h`

# `struct SpaceInfo`
描述“剩余的磁盘空间”。 是`Env::GetSpaceInfo()`的返回值。

```
// Returned by Env::GetSpaceInfo().
struct SpaceInfo {
  int64_t capacity_bytes; // Capacity of a filesystem, in bytes.
  int64_t free_bytes;     // Bytes available to non-privileged processes.
};
```

# `class Env`

这个类是一个“抽象类”，提供了所有和“文件系统”交互的接口(都是“纯虚函数”)，在不同的系统中，有不同的实现。

> 备注：在kudu现在的实现中，只针对`linux(posix)`提供了实现。见文件`src/kudu/util/env_posix.cc`

## `enum Env::CreateMode`

表示一个文件应该如何被创建。

其实就是决定调用`open()`函数时，传递的“选项值”。具体的参见`DoOpen()`（见文件`src/kudu/util/env_posix.cc`）

```
  // Governs if/how the file is created.
  //
  // enum value                      | file exists       | file does not exist
  // --------------------------------+-------------------+--------------------
  // CREATE_IF_NON_EXISTING_TRUNCATE | opens + truncates | creates
  // CREATE_NON_EXISTING             | fails             | creates
  // OPEN_EXISTING                   | opens             | fails
  
  enum CreateMode {
    CREATE_IF_NON_EXISTING_TRUNCATE,
    CREATE_NON_EXISTING,
    OPEN_EXISTING
  };
```

具体的设置方式如下(见`DoOpen()`的实现)：

```
  int flags = O_RDWR;
  switch (mode) {
    case Env::CREATE_IF_NON_EXISTING_TRUNCATE:
      flags |= O_CREAT | O_TRUNC;
      break;
    case Env::CREATE_NON_EXISTING:
      flags |= O_CREAT | O_EXCL;
      break;
    case Env::OPEN_EXISTING:
      break;
    default:
      return Status::NotSupported(Substitute("Unknown create mode $0", mode));
  }
  int f;
  RETRY_ON_EINTR(f, open(filename.c_str(), flags, 0666));
```

关于在`open()`中各个选项的说明，参见：
1. `linux man open`  
2. http://joe.is-programmer.com/posts/17463.html  Linux系统调用之open(), close()

**说明：这里定义的`3`种方式，对应的效果如下**


模式                              | 文件已存在             | 文件不存在
----------------------------      |---------------         | -----------
`CREATE_IF_NON_EXISTING_TRUNCATE` | 打开，并删除已有内容   |  创建文件
`CREATE_NON_EXISTING`             | **失败**               |  创建文件
`OPEN_EXISTING`                   | 仅仅打开，不做其它操作 |  **失败**

## `enum Env::FileType`
用来表示文件类型。 在`Env::Walk()`函数中的回调函数中会使用。

注意：如果是“软链”，那么返回的是`FILE_TYPE`（即使是一个“目录”的软链，也是返回`File_TYPE`）。

```
  enum FileType {
    DIRECTORY_TYPE,
    FILE_TYPE,
  };
```

## `enum Env::DirectoryOrder`

迭代目录的时候，使用的顺序。在`Env::Walk()`函数的参数。
```
  enum DirectoryOrder {
    PRE_ORDER,
    POST_ORDER,
  };
```

## `enum Env::ResourceLimitType`
资源限制的类型；

### `OPEN_FILES_PER_PROCESS`
一个进程中，所允许打开的“文件句柄”的最大数量。

说明：在`linux`上，是和`RLIMIT_NOFILE`相关的；

### `RUNNING_THREADS_PER_EUID`
一个进程中，所允许打开的“文件句柄”的最大线程数；

说明：在`linux`上，是和`RLIMIT_NPROC`相关的；

```
  enum class ResourceLimitType {
    OPEN_FILES_PER_PROCESS,
    RUNNING_THREADS_PER_EUID,
  };
```

## 成员

### 静态成员

#### `cosnt static kInjectedFailureStatusMsg`
仅仅在“单测”中使用。

在“扩展”因为文件的时候，用来填充的值。

是一个`const`常量，值为`INJECTED FAILURE`;

```
static const char* const kInjectedFailureStatusMsg;

const char* const Env::kInjectedFailureStatusMsg = "INJECTED FAILURE";
```

## 接口列表

### 静态接口

#### `status Default()`
返回一个“适合当前操作系统”的`Env`对象。

注意：“高级用户”可以自己定制`Env`对象。只需要在这里返回“定制化以后的`Env`对象”即可。

```
// 代码实现见 `src/kudu/util/env_posix.cc`

static pthread_once_t once = PTHREAD_ONCE_INIT;
static Env* default_env;
static void InitDefaultEnv() { default_env = new PosixEnv; }

Env* Env::Default() {
  pthread_once(&once, InitDefaultEnv);
  return default_env;
}
```

说明1：这里定义了一个全局的“静态”变量(`default_env`)，用来保存默认的`Env`对象。

说明2：这里使用了`pthread_once_t`，来保证`default_env`对象只被初始化一次。

### 创建不同用途的文件对象

#### `NewSequentialFile()`
指定一个文件名，创建一个对该文件进行“顺序读”的对象（即`SequentialFile`对象）。

**强调：这个文件是只能“顺序读”的。**  不可以“随机读”，更不支持写入数据。

在成功返回时，对应的文件句柄在参数`result`中；

注意1：因为是要去读取一个文件，所以这个文件一定是要存在的。如果不存在，返回`non-ok`。

**注意2：所返回的对象，是不可以多个线程进行并发读取的，只能由一个线程来进行读取**。  
原因是：因为该对象只能“顺序”读取，所以在实现的时候，在内部有一个变量记录当前的“偏移量”，多个线程进行并发读取的话，这个“偏移量”就乱了。

#### `NewRandomAccessFile()`
指定一个文件名，创建一个对该文件进行“随机读”的对象（即`RandomAccessFile`对象）

**强调：这个文件是只能“顺序读”的。**  不支持写入数据。

注意1：和`NewSequentialFile()`类似，也要求该文件是必须存在的。

注意2：和`NewSequentialFile()`不同，这个对象支持 多个线程并发读取。  
在每次读取时，都会指定它的偏移量，而不是在该对象内部记录着。

说明1：有两个重载函数：一个是用默认的`RandomAccessFileOptions`进行读取，一个是支持指定`RandomAccessFileOptions`来进行读取。

#### `NewWritableFile()`
指定一个文件名，创建一个对该文件进行“写入”的对象（即`WritableFile`对象）。

**注意1：如果文件已经存在，那么会被删除，并创建一个新文件**  

**注意2：所返回的对象，是不可以多个线程进行并发读取的，只能由一个线程来进行读取**。  

说明：和`NewRandomAccessFile()`类似，根据是否可以指定`WritableFileOptions`，有两个重载函数。

#### `NewTempWritableFile()`
指定一个“文件名模板”和"创建选项"(`WritableFileOption`)，创建一个“临时文件”。

注意：这里参数中指定的不是“文件名”，而是“文件名模板”。 真正被创建出来的文件的“文件名”，通过参数`created_filename`返回。

注意：这里的“临时文件”，实际上就是一个`WritableFile`对象。

之所以叫做“临时文件”，是说这个文件，在程序处理过程中是临时的，很快就会被删除。而且它的文件名也基本上会带有类似`tmp`等字样，从而标识这是一个临时文件。

例如：在`PathInstanceMetadataFile::Create()`（参见文件`src/kudu/fs/block_manager_util.cc`）中，为了获取“文件系统的块大小”，就会先创建一个“临时文件”，然后在该文件上去获取“文件系统的块大小”。

#### `NewRWFile()`

指定一个文件名，创建一个对该文件“同时支持 读 写 ”的对象（即`RWFile`）。

说明：根据是否可以指定`RWFileOptions`，有两个重载方法。

#### `NewTempRWFile()`  
和`NewTempWritableFile()`类似，也是创建一个“临时文件”。

说明1： 文件名的规则，和`NewTempWritableFile()`是类似的；

说明2：本质上也是一个`RWFile`对象。

### 获取 文件信息 的接口

#### `FileExists()`
检查一个文件是否存在。

#### `GetChildren()`
指定一个“目录”，获得该目录下的所有“文件”和“目录”。

说明1：获取结果中，是包含`"."`和`".."`的；

说明2：获取结果中，是不会递归向下的。即如果一个孩子也是“目录”，那么在这里并不会递归的获取它的“孩子”。

说明3：获取结果中，都是“字符串”表示的。

说明4：在结果中，只有单纯的名称（即“目录名”或“文件名”），没有路径信息。  
也可以理解为：这里的“结果”，是相对于“传入的`dir`参数”的。

#### `GetFileSize()`
指定一个文件，获取文件的“逻辑大小”。

#### `GetFileSizeOnDisk()`
指定一个文件，获取文件的“物理大小”。

**注意：`GetFileSizeOnDisk` VS `GetFileSize()`**  
1. `GetFileSizeOnDisk()`返回的是 文件实际占用的大小；
2. `GetFileSize()`返回的是，用户视角下的文件大小；

>> 问题：这两者有什么区别？

#### `GetFileSizeOnDiskRecursively()`
指定一个“根目录”，递归地获取所有“文件”和“子目录”上的“物理占用空间的总大小”。

#### `GetFileModifiedTime()`
获取文件的“最后修改时间”，单位是“微秒”。

注意1：这个返回的“时间戳”，是一个本地时间戳。

注意2：多次调用该函数，并不保证返回的结果是“递增的”。

在某些操作系统上，可能最高的差值为`1秒`。

也就是说，用户不应该使用 “这个返回的时间戳”作为 `epoch`来使用。  
因为它自身是可能会变化的，而`epoch`是要求某个时间点是固定的，不变的（因为对于`epoch`，经常有计算和`epoch`之间的差值的需求，而这里因为返回的时间戳会变化，所以这个差值就不固定，会有问题）。

#### `GetBlockSize()`
指定一个文件名，根据它来获取“文件系统的块大小”。

说明：这个的使用场景，就是在`LogBlockManager`中，在存储数据的时候，会将数据尽量向“文件系统块大小”对齐。

#### `GetSpaceInfo()`
指定一个路径，该函数返回该路径所属磁盘的“总容量”，以及对应的“剩余可用空间”。

>> coarse: 粗糙的；  

**注意：这里返回的`free space`，是比较粗糙的，并不是具体到某个字节**   
因为它的计算方式是：`“文件系统块大小” * “块数”`.  

#### `IsDirectory()`
给定一个“路径”，检查是否为一个“目录”。

1. 如果该路径是不存在的，那么返回`non-ok`; 
2. 如果该路径已经存在，返回`ok`。通过参数`is_dir`来表示是否为一个目录。

### 和“当前工作目录”相关的接口

> 说明：目前这两个函数并没有被使用。 不过在“单测”有进行测试。

#### `GetCurrentWorkingDir()`
获取当前的“工作目录”。

结果保存在参数`cwd`中。

#### `ChangeDir()`
将“当前工作目录”修改为 “指定的目录”。

### 文件操作 和 目录操作 的接口

#### `DeleteFile()`
指定一个文件名，删除这个文件。

#### `CreateDir()`
创建一个目录。

>> 问题：如果该目录已经存在，是返回成功，还是失败？

#### `DeleteDir()`
删除一个目录。

>> 问题：如果该“目录”不是空的，那么是否可以删除？

#### `SyncDir()`
指定一个“目录名”，进行`fsync()`操作。

发生的场景是：在某个“父目录”下 增加 和 删除 了内容（“文件”或“子目录”）时，如果要保证这些修改被持久化到磁盘（即宕机重启后仍然是持久化的），那么需要对“父目录”进行`fsync()`操作。

#### `DeleteRecursively()`
递归地删除一个“目录”。

注意：这个操作应该安全的。如果有软链，那么只应该删除“软链自身”，不能删除它“链接的目录”。

#### `RenameFile()`
重命名文件。

#### `Walk()`
迭代一个目录。

对于其中的每个元素（“文件”或“目录”），都会进行回调。

所以用户在使用这个函数时，应该提供一个回调函数。

```
typedef Callback<Status(FileType, const std::string&, const std::string&)> WalkCallback;
```

参数含义：
1. 第`1`个：元素的类型；
2. 第`2`个：当前元素所属的“目录”；
3. 第`3`个：当前元素的“文件名”；是`basename`。

注意：在`Walk()`的过程中，如果遇到了一个错误，那么并不会结束当前执行，而是继续执行；不过最终会返回一个`non-ok`。

#### `Glob()`
指定一个“模式”，返回所有能够匹配该模式的文件。

注意：即使是没有任何文件能够匹配该“模式”，也是会返回`Status::OK`，只不过在参数`paths`中会为空。

#### `Canonicalize()`
给定一个“路径”，对它进行“规范化”。

进行的转化有以下几个：
1. 将一个“相对路径”，转化为“绝对路径”；
2. 消除`"."`和`".."`；
3. 消除所有的“软链”；

注意：所指定的“路径”，必须是存在的。

#### `GetTotalRAMBytes()`
获取当前机器的“总内存大小”。

### 文件锁

#### `LockFile()`

#### `UnlockFile()`

### 资源限制相关的操作

#### `GetResourceLimit()`
指定资源类型，获取对应的限制情况。

#### `IncreaseResourceLimit()`
指定资源类型，增加它的“软限”到最大值（其实，最大就是增大到 等于 “硬限”的值）。

### 判断文件系统的类型

#### `IsOnExtFilesystem()`
给定一个路径，判断是否为`EXT`类型的文件系统。包括`ext2`/`ext3`/`ext4`。

#### `IsOnXfsFilesystem()`
给定一个路径，判断是否为`XFS`类型的文件系统。


### 其它操作

#### `NowMicros()`
获取与某个“固定的时间点”之间的差值，单位是“微秒”。

注意：理论上，这个“固定的时间点”，并不一定是“`1970-01-01 00:00:00`”。

所以，这个函数的使用场景，应该是只用来计算“差值”（即会调用两次，用来衡量经过了多长时间）。

#### `SleepForMicroseconds()`
`sleep`一段时间，单位是“微秒”。

#### `gettid()`
获取当前线程的“thread_id”。

#### `GetExecutablePath()`  
获取当前“可执行文件”的完整路径。

#### `GetKernelRelease()`
获取当前操作系统的`release string`。

#### `EnsureFileModeAdheresToUmask()`

>> 问题：干嘛用的？

#### `IsFileWorldReadable()`

> world readable: 即linux上的所有用户都可读。  
> 对应 `ls -l`命令中查看文件的权限。

>> 附录：在linux上的检查方法：  
>> https://blog.csdn.net/astrotycoon/article/details/8679676  sturct stat 结构体中 st_mode 的含义


























