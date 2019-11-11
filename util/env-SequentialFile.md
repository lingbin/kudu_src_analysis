[TOC]

文件: `src/kudu/util/env_posix.cc`

# `class PosixSequentialFile`

继承自`SequentialFile`。

## 成员

### `filename_`
当前的文件名。是一个“绝对路径”。

### `file`
是一个`File*`，即是一个文件句柄。

**注意：这里是用的`c lib`，并不是直接用的系统接口。** 

所以在本类中，对文件进行读取的操作都是`c lib`： 即`fread()`, `fclose()`等。

```
  const string filename_;
  FILE* const file_;

```

## 接口列表

### 构造函数

```
  PosixSequentialFile(string fname, FILE* f)
      : filename_(std::move(fname)), file_(f) {}
```

注意：从这个“构造函数”可以看出，这里的`FILE*`是在外部就已经打开了的。

### 析构函数

注意：在本函数中，会关闭文件。 所以，上层用户不需要再次进行关闭。

```
  virtual ~PosixSequentialFile() {
    int err;
    RETRY_ON_EINTR(err, fclose(file_));
    if (PREDICT_FALSE(err != 0)) {
      PLOG(WARNING) << "Failed to close " << filename_;
    }
  }
```

### `Read()`
从文件中读取一段内容，放入到参数`result`中。

```
  virtual Status Read(Slice* result) OVERRIDE {
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    ThreadRestrictions::AssertIOAllowed();
    size_t r;
    STREAM_RETRY_ON_EINTR(r, file_, fread_unlocked(result->mutable_data(), 1,
                                                   result->size(), file_));
    if (r < result->size()) {
      if (feof(file_)) {
        // We leave status as ok if we hit the end of the file.
        // We need to adjust the slice size.
        result->truncate(r);
      } else {
        // A partial read with an error: return a non-ok status.
        return IOError(filename_, errno);
      }
    }
    return Status::OK();
  }
```

### `Skip()`
跳过指定长度的“文件内容”。

```
  virtual Status Skip(uint64_t n) OVERRIDE {
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    TRACE_EVENT1("io", "PosixSequentialFile::Skip", "path", filename_);
    ThreadRestrictions::AssertIOAllowed();
    if (fseek(file_, n, SEEK_CUR)) {
      return IOError(filename_, errno);
    }
    return Status::OK();
  }
```

### `filename()`

获取文件名。

```
  virtual const string& filename() const OVERRIDE { return filename_; }
};
```

# `class PosixRandomAccessFile`

强调：当前类是基于`pread`的。即每次读取都是要在参数中指定了偏移量。

## 成员

### `filename_`
文件名。

### `fd_`
是系统调用`open()`返回的“文件句柄”。

说明：本类和`PosixSequentialFile`是不同的，不是使用的`c lib`。

```
  const string filename_;
  const int fd_;
```

## 接口列表

### 构造函数

```
PosixRandomAccessFile(string fname, int fd)
      : filename_(std::move(fname)), fd_(fd) {}
```

注意：这里的这个“文件句柄”是外部已经打开过的。  

实际上是在`EnvPosix::NewRandomAccessFile()`中打开的。

### 析构函数

```
  virtual ~PosixRandomAccessFile() {
    int err;
    RETRY_ON_EINTR(err, close(fd_));
    if (PREDICT_FALSE(err != 0)) {
      PLOG(WARNING) << "Failed to close " << filename_;
    }
  }
```
注意：在析构该对象的时候，会关闭该文件。

### `Read()`
```
  virtual Status Read(uint64_t offset, Slice result) const OVERRIDE {
    return DoReadV(fd_, filename_, offset, ArrayView<Slice>(&result, 1));
  }
```

### `ReadV()`
```
  virtual Status ReadV(uint64_t offset, ArrayView<Slice> results) const OVERRIDE {
    return DoReadV(fd_, filename_, offset, results);
  }
```

### `Size()`
获取文件的大小。使用`fstat()`系统调用。

```
  virtual Status Size(uint64_t *size) const OVERRIDE {
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    TRACE_EVENT1("io", "PosixRandomAccessFile::Size", "path", filename_);
    ThreadRestrictions::AssertIOAllowed();
    struct stat st;
    if (fstat(fd_, &st) == -1) {
      return IOError(filename_, errno);
    }
    *size = st.st_size;
    return Status::OK();
  }
```

### `getter`方法
#### `filename()`
```
  virtual const string& filename() const OVERRIDE { return filename_; }
```

### `memory_footprint()`  -- 获取内存占用信息

```
  virtual size_t memory_footprint() const OVERRIDE {
    return kudu_malloc_usable_size(this) + filename_.capacity();
  }
};
```

# `class PosixWritableFile`

非`memory mapped`方式，来向文件中写入数据。

## 成员

### `filename_`
字符串，文件的绝对路径。

### `fd_`
整型，表示“文件句柄”。

### `sync_on_close_`
标识在`close()`该文件之前，是否要进行`fsync()`操作。

### `filesize_`
文件的大小。

在构造该对象的时候，`filesize_`的值，是文件的当前大小（即偏移量为`filesize_`，就是向文件末尾添加数据）。

如果向该文件写入了数据，那么 这个`filesize_`会增加。

也就是说：这个`filesize_`的值，一直是“文件的总大小”。

### `pre_allocated_size_`
预分配的大小。

### `pending_sync_`
是否需要“延迟`fsync`”

### `closed_`
表示当前文件是否“已经关闭”。

```
  const string filename_;
  const int fd_;
  const bool sync_on_close_;

  uint64_t filesize_;
  uint64_t pre_allocated_size_;
  bool pending_sync_;
  bool closed_;
```

## 接口列表

### 构造函数

```
class PosixWritableFile : public WritableFile {
 public:
  PosixWritableFile(string fname, int fd, uint64_t file_size,
                    bool sync_on_close)
      : filename_(std::move(fname)),
        fd_(fd),
        sync_on_close_(sync_on_close),
        filesize_(file_size),
        pre_allocated_size_(0),
        pending_sync_(false),
        closed_(false) {}
```

### 析构函数

注意：析构该对象，会关闭对应的文件。

```
  ~PosixWritableFile() {
    WARN_NOT_OK(Close(), "Failed to close " + filename_);
  }
```

### `Append()`

```
  virtual Status Append(const Slice& data) OVERRIDE {
    return AppendV(ArrayView<const Slice>(&data, 1));
  }
```

### `AppendV()`

```
  virtual Status AppendV(ArrayView<const Slice> data) OVERRIDE {
    ThreadRestrictions::AssertIOAllowed();
    RETURN_NOT_OK(DoWriteV(fd_, filename_, filesize_, data));
    // Calculate the amount of data written
    size_t bytes_written = accumulate(data.begin(), data.end(), static_cast<size_t>(0),
                                      [&](int sum, const Slice& curr) {
                                        return sum + curr.size();
                                      });
    filesize_ += bytes_written;
    pending_sync_ = true;
    return Status::OK();
  }
```

### `PreAllocate()`

见父类`WritableFile::PreAllocate()`中（文件`src/kudu/util/env.h`），对于本函数的说明。

在“已经预分配到的位置”，或者“当前的文件大小”基础上，在“扩展”指定的长度。

**说明1：扩展的初始长度**  
是“已经预分配到的位置”和“当前的文件大小”，两者中的“较大值”。

使用该函数，在底层文件系统中，预分配指定大小的空间。 使用的是系统函数`fallocate()`。

**说明2：使用场景：**  
这种“预分配”的功能，主要在`LogBlockManager`中使用。结合`hole punch`来一起使用。

```
  virtual Status PreAllocate(uint64_t size) OVERRIDE {
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));

    TRACE_EVENT1("io", "PosixWritableFile::PreAllocate", "path", filename_);
    ThreadRestrictions::AssertIOAllowed();
    uint64_t offset = std::max(filesize_, pre_allocated_size_);
    int ret;
    RETRY_ON_EINTR(ret, fallocate(fd_, 0, offset, size));
    if (ret != 0) {
      if (errno == EOPNOTSUPP) {
        KLOG_FIRST_N(WARNING, 1) << "The filesystem does not support fallocate().";
      } else if (errno == ENOSYS) {
        KLOG_FIRST_N(WARNING, 1) << "The kernel does not implement fallocate().";
      } else {
        return IOError(filename_, errno);
      }
    }
    pre_allocated_size_ = offset + size;
    return Status::OK();
  }
```

### `Close()`

```
  virtual Status Close() OVERRIDE {
    if (closed_) {
      return Status::OK();
    }
    TRACE_EVENT1("io", "PosixWritableFile::Close", "path", filename_);
    ThreadRestrictions::AssertIOAllowed();
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    Status s;

    // If we've allocated more space than we used, truncate to the
    // actual size of the file and perform Sync().
    if (filesize_ < pre_allocated_size_) {
      int ret;
      RETRY_ON_EINTR(ret, ftruncate(fd_, filesize_));
      if (ret != 0) {
        s = IOError(filename_, errno);
        pending_sync_ = true;
      }
    }

    if (sync_on_close_) {
      Status sync_status = Sync();
      if (!sync_status.ok()) {
        LOG(ERROR) << "Unable to Sync " << filename_ << ": " << sync_status.ToString();
        if (s.ok()) {
          s = sync_status;
        }
      }
    }

    int ret;
    RETRY_ON_EINTR(ret, close(fd_));
    if (ret < 0) {
      if (s.ok()) {
        s = IOError(filename_, errno);
      }
    }

    closed_ = true;
    return s;
  }
```

说明：如果“预分配”的文件大小，是大于当前的“文件大小”的，那么需要进行`truncate`操作（对应上面的`ftruncate()`）。

说明2：如果在构建该对象的时候，指定了要在`Close`的时候，进行`fsync`(参见`sync_on_close)`属性)，那么这里要进行`fsync()`。  
实际上就是调用 全局工具函数`DoSync()`。


### `Flush()`
针对文件中的“一段数据”（使用“偏移量”和“长度”来表示这段数据），进行`fsync`。

说明：在`linux`系统上，有系统函数`sync_file_range()`来对 文件中的一段数据进行`fsync`。  
在其它系统上，就直接调用`fsync()`

有关`sync_file_range()`的使用，参见：
1. `linux man page`
2. http://blog.yufeng.info/archives/1070  Linux下新系统调用sync_file_range 
3. http://www.voidcn.com/article/p-vwagcekw-bkq.html  Linux下新系统调用sync_file_range提高数据sync的效率 

说明：`sync_file_range()`是绝对不会写 metadata 的，所以用它非常合适的场景是：
1. 每次对文件做了小范围的修改时，立即调用`sync_file_range`，把对应的脏数据刷到磁盘;
2. 然后在结束对文件的修改后，再去调用`fdatasync` (flush dirty data page)或`fsync`(flush dirty data+metadata page)。

如果之前调用过`sync_file_range()`，那么后面调用`fdatasync()`或`fsync()`，都会很快。

说明2：`sync_file_range`的`flag`。  
注意`SYNC_FILE_RANGE_WRITE`是异步的，所以如果你希望达到“数据安全落盘”的目的，那么最好不要使用异步模式。
至少应该在调用`fdatasync`和`fsync`前，使用`SYNC_FILE_RANGE_WAIT_BEFORE` | `SYNC_FILE_RANGE_WRITE` |
`SYNC_FILE_RANGE_WAIT_AFTER`做一次全文件范围的`sync_file_range()`。  
从而保证在调用`fdatasync()`或`fsync()`前，该文件的"dirty page"已经全部刷到磁盘了。

```
  virtual Status Flush(FlushMode mode) OVERRIDE {
    TRACE_EVENT1("io", "PosixWritableFile::Flush", "path", filename_);
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    ThreadRestrictions::AssertIOAllowed();
#if defined(__linux__)
    int flags = SYNC_FILE_RANGE_WRITE;
    if (mode == FLUSH_SYNC) {
      flags |= SYNC_FILE_RANGE_WAIT_BEFORE;
      flags |= SYNC_FILE_RANGE_WAIT_AFTER;
    }
    if (sync_file_range(fd_, 0, 0, flags) < 0) {
      return IOError(filename_, errno);
    }
#else
    if (mode == FLUSH_SYNC && fsync(fd_) < 0) {
      return IOError(filename_, errno);
    }
#endif
    return Status::OK();
  }
```

说明：从man page可以看出来，在调用`sync_file_range()`的时候，如果`nbytes`参数为`0`，那么就相当于对[`offset`, “文件末尾”]这段区间进行 刷盘。 对应上面的`sync_file_range(fd_, 0, 0, flags)`。

### `Sync()`
对当前“文件”整体进行`fsync`(调用全局函数`DoSync()`)

```
  virtual Status Sync() OVERRIDE {
    TRACE_EVENT1("io", "PosixWritableFile::Sync", "path", filename_);
    ThreadRestrictions::AssertIOAllowed();
    LOG_SLOW_EXECUTION(WARNING, 1000, Substitute("sync call for $0", filename_)) {
      if (pending_sync_) {
        pending_sync_ = false;
        RETURN_NOT_OK(DoSync(fd_, filename_));
      }
    }
    return Status::OK();
  }
```

### `getter`方法
#### `Size()`
```
  virtual uint64_t Size() const OVERRIDE {
    return filesize_;
  }
```

#### `filename()`
```
  virtual const string& filename() const OVERRIDE { return filename_; }
```

# `class PosixRWFile`

## 成员

### `filename_`
文件的绝对路径。

### `fd_`
文件句柄

### `sync_on_close_`
在关闭该文件时，是否进行`fsync()`操作。

### `once_`
用来保证`is_on_xfs_`只被初始化一次。

### `is_on_xfs_`
表示当前的文件系统是否为“`XFS`文件系统”。

### `closed_`
表示该文件是否已经关闭。

```
  const string filename_;
  const int fd_;
  const bool sync_on_close_;

  GoogleOnceDynamic once_;
  bool is_on_xfs_;
  bool closed_;
```

## 接口列表

### 构造函数

```
PosixRWFile(string fname, int fd, bool sync_on_close)
      : filename_(std::move(fname)),
        fd_(fd),
        sync_on_close_(sync_on_close),
        is_on_xfs_(false),
        closed_(false) {}
```
### 析构函数

和其它对象一样，在析构函数中，会触发去关闭文件。

```
  ~PosixRWFile() {
    WARN_NOT_OK(Close(), "Failed to close " + filename_);
  }
```

### `Read()`

```
  virtual Status Read(uint64_t offset, Slice result) const OVERRIDE {
    return DoReadV(fd_, filename_, offset, ArrayView<Slice>(&result, 1));
  }
```

### `ReadV()`
```
  virtual Status ReadV(uint64_t offset, ArrayView<Slice> results) const OVERRIDE {
    return DoReadV(fd_, filename_, offset, results);
  }
```

### `Write()`
```
  virtual Status Write(uint64_t offset, const Slice& data) OVERRIDE {
    return WriteV(offset, ArrayView<const Slice>(&data, 1));
  }
```

### `WriteV()`
```
  virtual Status WriteV(uint64_t offset, ArrayView<const Slice> data) OVERRIDE {
    return DoWriteV(fd_, filename_, offset, data);
  }
```

### `PreAllocate()`

```
  virtual Status PreAllocate(uint64_t offset,
                             size_t length,
                             PreAllocateMode mode) OVERRIDE {
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));

    TRACE_EVENT1("io", "PosixRWFile::PreAllocate", "path", filename_);
    ThreadRestrictions::AssertIOAllowed();
    int falloc_mode = 0;
    if (mode == DONT_CHANGE_FILE_SIZE) {
      falloc_mode = FALLOC_FL_KEEP_SIZE;
    }
    int ret;
    RETRY_ON_EINTR(ret, fallocate(fd_, falloc_mode, offset, length));
    if (ret != 0) {
      if (errno == EOPNOTSUPP) {
        KLOG_FIRST_N(WARNING, 1) << "The filesystem does not support fallocate().";
      } else if (errno == ENOSYS) {
        KLOG_FIRST_N(WARNING, 1) << "The kernel does not implement fallocate().";
      } else {
        return IOError(filename_, errno);
      }
    }
    return Status::OK();
  }
```

### `Truncate()`

```
  virtual Status Truncate(uint64_t length) OVERRIDE {
    TRACE_EVENT2("io", "PosixRWFile::Truncate", "path", filename_, "length", length);
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    ThreadRestrictions::AssertIOAllowed();
    int ret;
    RETRY_ON_EINTR(ret, ftruncate(fd_, length));
    if (ret != 0) {
      int err = errno;
      return Status::IOError(Substitute("Unable to truncate file $0", filename_),
                             Substitute("ftruncate() failed: $0", ErrnoToString(err)),
                             err);
    }
    return Status::OK();
  }
```

### `PunchHole()`
```
  virtual Status PunchHole(uint64_t offset, size_t length) OVERRIDE {
#if defined(__linux__)
    TRACE_EVENT1("io", "PosixRWFile::PunchHole", "path", filename_);
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    ThreadRestrictions::AssertIOAllowed();

    // KUDU-2052: xfs in el6 systems induces an fsync in the kernel whenever it
    // performs a hole punch through the fallocate() syscall, even if the file
    // range was already punched out. The older xfs-specific hole punching
    // ioctl doesn't do this, despite eventually executing the same xfs code.
    // To keep the code simple, we'll use this ioctl on any xfs system (not
    // just on el6) and fallocate() on all other filesystems.
    //
    // Note: the cast to void* here (and back to PosixRWFile*, in InitIsOnXFS)
    // is needed to avoid an undefined behavior warning from UBSAN.
    once_.Init(&InitIsOnXFS, reinterpret_cast<void*>(this));
    if (is_on_xfs_ && FLAGS_env_use_ioctl_hole_punch_on_xfs) {
      xfs_flock64_t cmd;
      memset(&cmd, 0, sizeof(cmd));
      cmd.l_start = offset;
      cmd.l_len = length;
      if (ioctl(fd_, XFS_IOC_UNRESVSP64, &cmd) < 0) {
        return IOError(filename_, errno);
      }
    } else {
      int ret;
      RETRY_ON_EINTR(ret, fallocate(
          fd_, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE, offset, length));
      if (ret != 0) {
        return IOError(filename_, errno);
      }
    }
    return Status::OK();
#else
    return Status::NotSupported("Hole punching not supported on this platform");
#endif
  }
```

### `Flush()`
```
  virtual Status Flush(FlushMode mode, uint64_t offset, size_t length) OVERRIDE {
    TRACE_EVENT1("io", "PosixRWFile::Flush", "path", filename_);
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    ThreadRestrictions::AssertIOAllowed();
#if defined(__linux__)
    int flags = SYNC_FILE_RANGE_WRITE;
    if (mode == FLUSH_SYNC) {
      flags |= SYNC_FILE_RANGE_WAIT_AFTER;
    }
    if (sync_file_range(fd_, offset, length, flags) < 0) {
      return IOError(filename_, errno);
    }
#else
    if (mode == FLUSH_SYNC && fsync(fd_) < 0) {
      return IOError(filename_, errno);
    }
#endif
    return Status::OK();
  }
```

### `Sync()`
```
  virtual Status Sync() OVERRIDE {
    TRACE_EVENT1("io", "PosixRWFile::Sync", "path", filename_);
    ThreadRestrictions::AssertIOAllowed();
    LOG_SLOW_EXECUTION(WARNING, 1000, Substitute("sync call for $0", filename())) {
      RETURN_NOT_OK(DoSync(fd_, filename_));
    }
    return Status::OK();
  }
```

### `Close()`
```
  virtual Status Close() OVERRIDE {
    if (closed_) {
      return Status::OK();
    }
    TRACE_EVENT1("io", "PosixRWFile::Close", "path", filename_);
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    ThreadRestrictions::AssertIOAllowed();
    Status s;

    if (sync_on_close_) {
      s = Sync();
      if (!s.ok()) {
        LOG(ERROR) << "Unable to Sync " << filename_ << ": " << s.ToString();
      }
    }

    int ret;
    RETRY_ON_EINTR(ret, close(fd_));
    if (ret < 0) {
      if (s.ok()) {
        s = IOError(filename_, errno);
      }
    }

    closed_ = true;
    return s;
  }
```

### `Size()`
```
  virtual Status Size(uint64_t* size) const OVERRIDE {
    TRACE_EVENT1("io", "PosixRWFile::Size", "path", filename_);
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    ThreadRestrictions::AssertIOAllowed();
    struct stat st;
    if (fstat(fd_, &st) == -1) {
      return IOError(filename_, errno);
    }
    *size = st.st_size;
    return Status::OK();
  }
```

### `GetExtentMap()`

```
  virtual Status GetExtentMap(ExtentMap* out) const OVERRIDE {
#if !defined(__linux__)
    return Status::NotSupported("GetExtentMap not supported on this platform");
#else
    TRACE_EVENT1("io", "PosixRWFile::GetExtentMap", "path", filename_);
    MAYBE_RETURN_EIO(filename_, IOError(Env::kInjectedFailureStatusMsg, EIO));
    ThreadRestrictions::AssertIOAllowed();

    // This allocation size is arbitrary.
    static const int kBufSize = 4096;
    uint8_t buf[kBufSize] = { 0 };

    struct fiemap* fm = reinterpret_cast<struct fiemap*>(buf);
    struct fiemap_extent* fme = &fm->fm_extents[0];
    int avail_extents_in_buffer = (kBufSize - sizeof(*fm)) / sizeof(*fme);
    bool saw_last_extent = false;
    ExtentMap extents;
    do {
      // Fetch another block of extents.
      fm->fm_length = FIEMAP_MAX_OFFSET;
      fm->fm_extent_count = avail_extents_in_buffer;
      if (ioctl(fd_, FS_IOC_FIEMAP, fm) == -1) {
        return IOError(filename_, errno);
      }

      // No extents returned, this file must have no extents.
      if (fm->fm_mapped_extents == 0) {
        break;
      }

      // Parse the extent block.
      uint64_t last_extent_end_offset;
      for (int i = 0; i < fm->fm_mapped_extents; i++) {
        if (fme[i].fe_flags & FIEMAP_EXTENT_LAST) {
          // This should really be the last extent.
          CHECK_EQ(fm->fm_mapped_extents - 1, i);

          saw_last_extent = true;
        }
        InsertOrDie(&extents, fme[i].fe_logical, fme[i].fe_length);
        VLOG(3) << Substitute("File $0 extent $1: o $2, l $3 $4",
                              filename_, i,
                              fme[i].fe_logical, fme[i].fe_length,
                              saw_last_extent ? "(final)" : "");
        last_extent_end_offset = fme[i].fe_logical + fme[i].fe_length;
        if (saw_last_extent) {
          break;
        }
      }

      fm->fm_start = last_extent_end_offset;
    } while (!saw_last_extent);

    out->swap(extents);
    return Status::OK();
#endif
  }
```

>> 问题：具体是什么意思？

### `getter`方法

#### `filename()`
```
  virtual const string& filename() const OVERRIDE {
    return filename_;
  }
```

### `private static InitIsOnXFS()`

```
  static void InitIsOnXFS(void* arg) {
    PosixRWFile* rwf = reinterpret_cast<PosixRWFile*>(arg);
    bool result;
    Status s = DoIsOnXfsFilesystem(rwf->filename_, &result);
    if (s.ok()) {
      rwf->is_on_xfs_ = result;
    } else {
      KLOG_EVERY_N_SECS(WARNING, 1) <<
          Substitute("Could not determine whether file is on xfs, assuming not: $0",
                     s.ToString());
    }
  }
};
```


# 全局工具函数

在`.cpp`文件中定义的.

## `DoSync()`

```
Status DoSync(int fd, const string& filename) {
  MAYBE_RETURN_EIO(filename, IOError(Env::kInjectedFailureStatusMsg, EIO));

  ThreadRestrictions::AssertIOAllowed();
  if (FLAGS_never_fsync) return Status::OK();
  if (FLAGS_env_use_fsync) {
    TRACE_COUNTER_SCOPE_LATENCY_US("fsync_us");
    TRACE_COUNTER_INCREMENT("fsync", 1);
    if (fsync(fd) < 0) {
      return IOError(filename, errno);
    }
  } else {
    TRACE_COUNTER_INCREMENT("fdatasync", 1);
    TRACE_COUNTER_SCOPE_LATENCY_US("fdatasync_us");
    if (fdatasync(fd) < 0) {
      return IOError(filename, errno);
    }
  }
  return Status::OK();
}
```

说明1：调用该方法，并不代表着一定会执行`fsync()`。还受`FLAGS_never_fsync`的控制。

说明2：具体是调用`fsync()`还是`fdatasync()`，是由`FLAGS_env_use_fsync`来决定的。

## `DoOpen()`

```
Status DoOpen(const string& filename, Env::CreateMode mode, int* fd) {
  MAYBE_RETURN_EIO(filename, IOError(Env::kInjectedFailureStatusMsg, EIO));
  ThreadRestrictions::AssertIOAllowed();
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
  if (f < 0) {
    return IOError(filename, errno);
  }
  *fd = f;
  return Status::OK();
}
```

## `DoReadV()`

从文件中读取数据，保存在`results`中。

说明1：如果返回`Status::OK`，表示读取成功。 

1. 如果返回`Status::OK`，说明读取成功，没有遇到错误，也没有遇到“文件末尾”。这时`results`中所有`Slice`对象的长度总和，就是读取的数据的总长度；
2. 如果遇到“文件末尾”，会提前结束，返回`Status::EndOfFile`，这时`results`中的所有`Slice`，可能并没有被完全填充。（所以这时，是不能通过所有`Slice`的长度，来获取到“读取的数据总长度”的）。  
    也就是说，如果遇到了`Status::EndOfFile`，那么是无法知道读取的数据长度。  

说明2: 但是在使用时，如果使用这个`ReadV()`函数，所传入的`Slice`集合，是在外部就已经构建好了长度，能够保证一定不会中途遇到“文件末尾”的。  
所以，如果在上层需要获取“读取的数据长度”，那么就一定是将“所有`Slice`对象”的长度相加即可。

说明3：这里的`V`，表示每次读取的数据，会被放入到多个“目的地”，即参数`results`中的多个`Slice`对象中。  
在实现的时候，是利用`linux`的`preadv()`函数来实现的。

```
Status DoReadV(int fd, const string& filename, uint64_t offset,
               ArrayView<Slice> results) {
  MAYBE_RETURN_EIO(filename, IOError(Env::kInjectedFailureStatusMsg, EIO));
  ThreadRestrictions::AssertIOAllowed();

  // Convert the results into the iovec vector to request
  // and calculate the total bytes requested
  size_t bytes_req = 0;
  size_t iov_size = results.size();
  struct iovec iov[iov_size];
  for (size_t i = 0; i < iov_size; i++) {
    Slice& result = results[i];
    bytes_req += result.size();
    iov[i] = { result.mutable_data(), result.size() };
  }

  uint64_t cur_offset = offset;
  size_t completed_iov = 0;
  size_t rem = bytes_req;
  while (rem > 0) {
    // Never request more than IOV_MAX in one request
    size_t iov_count = std::min(iov_size - completed_iov, static_cast<size_t>(IOV_MAX));
    ssize_t r;
    RETRY_ON_EINTR(r, preadv(fd, iov + completed_iov, iov_count, cur_offset));

    // Fake a short read for testing
    if (PREDICT_FALSE(FLAGS_env_inject_short_read_bytes > 0 && rem == bytes_req)) {
      DCHECK_LT(FLAGS_env_inject_short_read_bytes, r);
      r -= FLAGS_env_inject_short_read_bytes;
    }

    if (PREDICT_FALSE(r < 0)) {
      // An error: return a non-ok status.
      return IOError(filename, errno);
    }
    if (PREDICT_FALSE(r == 0)) {
      // EOF.
      return Status::EndOfFile(
          Substitute("EOF trying to read $0 bytes at offset $1", bytes_req, offset));
    }
    if (PREDICT_TRUE(r == rem)) {
      // All requested bytes were read. This is almost always the case.
      return Status::OK();
    }
    DCHECK_LE(r, rem);
    // Adjust iovec vector based on bytes read for the next request
    ssize_t bytes_rem = r;
    for (size_t i = completed_iov; i < iov_size; i++) {
      if (bytes_rem >= iov[i].iov_len) {
        // The full length of this iovec was read
        completed_iov++;
        bytes_rem -= iov[i].iov_len;
      } else {
        // Partially read this result.
        // Adjust the iov_len and iov_base to request only the missing data.
        iov[i].iov_base = static_cast<uint8_t *>(iov[i].iov_base) + bytes_rem;
        iov[i].iov_len -= bytes_rem;
        break; // Don't need to adjust remaining iovec's
      }
    }
    cur_offset += r;
    rem -= r;
  }
  DCHECK_EQ(0, rem);
  return Status::OK();
}
```

## `DoWriteV()`
向多个`Slice`的数据，依次写入到文件中。

说明：和`DoReadV()`类似，这里的`'V'`，表示要将多组数据（即参数`data`中的多个`Slice`对象），写入到文件中。
在实现的时候，是利用`linux`的`pwritev()`函数来实现的。


**说明：获取“写入数据的总长度的方法”**  
这里和`DoReadV()`不同，不需要考虑会碰到文件末尾的情况。

写入的数据总长度，就是将参数`data`中的多个`Slice`对象的总长度。

```
Status DoWriteV(int fd, const string& filename, uint64_t offset, ArrayView<const Slice> data) {
  MAYBE_RETURN_EIO(filename, IOError(Env::kInjectedFailureStatusMsg, EIO));
  ThreadRestrictions::AssertIOAllowed();

  // Convert the results into the iovec vector to request
  // and calculate the total bytes requested.
  size_t bytes_req = 0;
  size_t iov_size = data.size();
  struct iovec iov[iov_size];
  for (size_t i = 0; i < iov_size; i++) {
    const Slice& result = data[i];
    bytes_req += result.size();
    iov[i] = { const_cast<uint8_t*>(result.data()), result.size() };
  }

  uint64_t cur_offset = offset;
  size_t completed_iov = 0;
  size_t rem = bytes_req;
  while (rem > 0) {
    // Never request more than IOV_MAX in one request.
    size_t iov_count = std::min(iov_size - completed_iov, static_cast<size_t>(IOV_MAX));
    ssize_t w;
    RETRY_ON_EINTR(w, pwritev(fd, iov + completed_iov, iov_count, cur_offset));

    // Fake a short write for testing.
    if (PREDICT_FALSE(FLAGS_env_inject_short_write_bytes > 0 && rem == bytes_req)) {
      DCHECK_LT(FLAGS_env_inject_short_write_bytes, w);
      w -= FLAGS_env_inject_short_read_bytes;
    }

    if (PREDICT_FALSE(w < 0)) {
      // An error: return a non-ok status.
      return IOError(filename, errno);
    }

    DCHECK_LE(w, rem);

    if (PREDICT_TRUE(w == rem)) {
      // All requested bytes were read. This is almost always the case.
      return Status::OK();
    }
    // Adjust iovec vector based on bytes read for the next request.
    ssize_t bytes_rem = w;
    for (size_t i = completed_iov; i < iov_size; i++) {
      if (bytes_rem >= iov[i].iov_len) {
        // The full length of this iovec was written.
        completed_iov++;
        bytes_rem -= iov[i].iov_len;
      } else {
        // Partially wrote this result.
        // Adjust the iov_len and iov_base to write only the missing data.
        iov[i].iov_base = static_cast<uint8_t *>(iov[i].iov_base) + bytes_rem;
        iov[i].iov_len -= bytes_rem;
        break; // Don't need to adjust remaining iovec's.
      }
    }
    cur_offset += w;
    rem -= w;
  }
  DCHECK_EQ(0, rem);
  return Status::OK();
}
```











