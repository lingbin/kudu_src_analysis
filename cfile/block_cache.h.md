[TOC]

文件：`src/kudu/cfile/block_cache.h`

# 概述

**强调：这里的`block`，指的是`cfile block`，不是`kudu::fs block`**

本文件中有两个类：
1. `BlockCache`： 就是封装的`Cache`类。表示缓存`cfile block`的整体`Cache`结构（即包含所有的`Cache`条目）；
2. `BlockCacheHandle`: 表示在上面结构中的一个条目。每个被缓存的`cfile block`，会对应一个`BlockCacheHandle`对象。

# `class BlockCache`

本类的目的：封装`kudu::Cache`类，从而“专门用来”缓存`cfile blocks`. 

封装成了一个“单例类”，并且内部使用的`LRU cache`。  
>>说明：在kudu的`Cache`实现中，同时支持`FIFO`和`LRU`，只不过这里只使用`LRU`（即针对`cfile block`的缓存，只会采用`LRU`的‘缓存淘汰逻辑’）。

**强调：这里在cache中保存是`cfile block`，不是`kudu::fs block`**  
从这个角度将，当前类实际上叫做`CFileBlockCache`更清晰。  

## `FileId`别名

```
typedef BlockId FileId;
```
> 备注：`BlockId`的定义，参见文件`src/kudu/fs/block_id.h`。

**说明：为什么要设置这样一个别名**  

一个`BlockId`，是用来唯一的表示一个`kudu::fs block`，也就是说，实际上是一整个`cfile`。

但是在当前类中（当前类叫`BlockCache`，也有`block`的概念），这里的`block`是指一个`cfile block`。 

也就是说，这两个地方的`block`，含义是不同的。 一个是`fs block`， 一个是`cfile block`。

**注意区分："`fs block`" VS "`cfile block`"**  

为了在代码中区分这两种`block`，将`fs block`指定了一个别名`FileId`。（一个 `fs block`就对应一个`cfile`，所以重命名为`FileId`也是比较理解的）。 

> 扩展注意事项：一个`cfile`（也就是一个`fs block`），最终在磁盘上，不一定真的对应一个“文件”。  
>（要看具体的`BlockManager`类型，在`FileBlockManager`中是一个“文件”，但是在`LogBlockManager`中不是。）

## `struct BlockCache::CacheKey`

在`block cache`中的条目的`key`。 （`BlockCache`本质上是一个`hashmap`，所以需要有`key`来标识每个条目）

在`block cache`中的任何一个条目，所包含的值是“该`block`在`cfile`中的‘偏移量’”。

说明1：`CacheKey`对象的内存结构，虽然是成员是两个`uint64_t`类型。**但是在程序中的处理，是当做“二进制字符串”来处理的**。   

所以，两个`CacheKey`变量的相互赋值，是通过`memcpy()`来做的，并不是通过 它默认的“赋值操作符”来实现的。

说明2: 如果底层的`Cache`支持“持久化”，那么这里的`CacheKey`就会被“持久化”。  
这样的好处是：在“重启”和“升级”kudu的时候，就可以利用这个特性，可以直接加载`Cache`，从而避免“冷启动”。

也正是因为这个“持久化”的特性，需要保证`CacheKey`的“确定性”，并且能够兼容将来的修改，是非常重要的。（因为一旦不兼容，那么老版本的`kudu`所保存下来的`cache`，升级后的`kudu`进程就没法使用，从而还是必须“冷启动”）

### 成员

#### `file_id_`
即对应的`kudu::fs BlockId`。

#### `offset_`
当前`cfile block`在`kudu::fs::Block`中的偏移量。

```
  struct CacheKey {
    CacheKey(BlockCache::FileId file_id, uint64_t offset) :
      file_id_(file_id.id()),
      offset_(offset)
    {}

    uint64_t file_id_;
    uint64_t offset_;
  } PACKED
```

注意：这里的`PACKED`，参见`src/kudu/gutil/port.h`，实际上是`__attribute__ ((packed))`.   
关于`__attribute__`的说明，可以参见：
1. https://blog.shengbin.me/posts/gcc-attribute-aligned-and-packed  GCC中的aligned和packed属性
2. http://blog.sina.com.cn/s/blog_7e719f0501012tkt.html  __attribute__((packed))详解

简单的说，`__attribute__ ((packed))`的作用就是，这个“结构体”内的成员，在编译过程中不会进行“字节对齐优化”，而是按照各个字段实际占用的字节，进行字节排列（实际上就是按照`1字节`进行对齐，而不是默认的`4`字节）。  

这样的结果是：这个“结构体”的内存是“非常紧凑”的。

也正是因为这个原因，使用`__attribute__ ((packed))`修饰的结构体，访问的时候，都不会直接访问成员。（因为成员的地址，没有经过“内存对齐”，访问的效率可能会差）。

在“拷贝”该对象的时候，是直接通过`memcpy()`来进行复制，而不是通过“逐个成员赋值”的方式。  

## `class BlockCache::PendingEntry`

**本类的目的：用来描述一个正在被插入的“条目”**。实际上就是封装一个`Cache::UniquePendingHandle`对象。

注意：`Cache::UniquePendingHandle`类型，实际上是`std::unique_ptr<>`类型，参见`src/kudu/util/cache.h`

```
  typedef std::unique_ptr<PendingHandle, PendingHandleDeleter> UniquePendingHandle;
```

**注意：为什么要封装一下，而不是直接使用`std::unique_ptr<>`？**   

就是因为从一个`UniquePendingHandle`类型，去获取 它所封装的“数据的指针”是比较冗长的（参见`val_ptr()`函数的实现）。  

通过封装以后，就可以通过  简单的调用`val_ptr()`来获取。

而如果没有这个封装，直接使用`std::unique_ptr<>`，那么如果 使用者希望通过该条目，获取它对应的“数据指针”时，那么必须写一段非常长的 函数调用(`handle_.get_deleter().cache()->MutableValue(&handle_);`)，这比较麻烦。

### 成员

#### `handle_`

类型为`Cache::UniquePendingHandle`。

### 接口列表

#### `构造函数`

```
    PendingEntry()
        : handle_(Cache::UniquePendingHandle(nullptr,
                                             Cache::PendingHandleDeleter(nullptr))) {
    }
    explicit PendingEntry(Cache::UniquePendingHandle handle)
        : handle_(std::move(handle)) {
    }
    PendingEntry(PendingEntry&& other) noexcept : PendingEntry() {
      *this = std::move(other);
    }

    ~PendingEntry() {
      reset();
    }
```

说明：本类就是为了封装一个`Cache::UniquePendingHandle`对象。  

**从构造函数可以看出在使用过程中， 构建`PendingEntry`对象的方法：**  

在使用的时候，就是先通过`Cache::Allocate()`来申请一个`Cache::UniquePendingHandle`对象，然后用来构建`PendingEntry`对象。

#### `operator=()`

```
PendingEntry& operator=(const PendingEntry& other) = delete;


inline BlockCache::PendingEntry& BlockCache::PendingEntry::operator=(
    BlockCache::PendingEntry&& other) noexcept {
  reset();
  handle_ = std::move(other.handle_);
  return *this;
}
```

注意：这里将“赋值构造函数”声明为`delete`，而是提供了一个“移动赋值构造函数”。  

这样的好处是：避免了“默认的‘赋值构造’”，如果代码中误用了，可以及时的发现。

#### `reset()`

```
inline void BlockCache::PendingEntry::reset() {
  handle_.reset();
}
```

说明：因为`handle_`本质上是一个`std::unique_ptr<>`，所以`reset()`就是将该指针赋值为`nullptr`。

#### `valid()`
检查当前`PendingEntry`对象，是否为一个合法的对象。

如果在`cache`中申请不到足够的内存，那么这个对象就是不合法的。通过该方法，就可以检测出来。

```
    bool valid() const {
      return static_cast<bool>(handle_);
    }
```

注意：这里的实现方式：是将`handle_`直接“强制转化”为一个`bool`类型。  

> 说明：其实`handle_`对象的类型是一个`std::unique_ptr<>`(是一个`别名`，参见`src/kudu/util/cache.h`)。  
>     也就是说，只要`handle_`不为`nullptr`，那么就会返回`true`。  

#### `val_ptr()`
返回一个指针，指向要写入数据的地址。

写入到这个地址的数据，就是最终会写入到`cache条目`中的值。

```
    uint8_t* val_ptr() {
      return handle_.get_deleter().cache()->MutableValue(&handle_);
    }
```

**注意这里的技巧：**  
前面已经说过，`handle_`的类型是`std::unique_ptr<PendingHandle, PendingHandleDeleter>`，也就是定义了“删除器”的`std::unique_ptr`。

在定义了“删除器”以后，在每次构造时，都需要传入一个“删除器”对象（这也是代码要实现的“类”）。  
既然是一个“对象”，那么我们就**可以定制这个对象：为这个对象“添加成员”和“添加方法”。**  

这里，
1. 将`Cache*`作为了“删除器”(`PendingHandleDeleter`类型)的成员。  
2. 为`PendingHandleDeleter`类型添加了一个成员函数`cache()`，返回它所属的`Cache*`指针。

因为对于每个`std::unique_ptr<>`来说，它有一个成员`get_deleter()`获取它的“删除器”对象。  
然后在通过调用`PendingHandleDeleter::cache()`获取到`Cache`对象的指针。  
然后就可以调用`Cache`的接口了。  

## 添加一个`cache条目`的写路径

在代码中的`Allocate()`函数的注释中，描述了向`Cache`中添加一个条目的流程（`write path`）:

一个`cfile block`对应的条目，被写入到`Cache`中，分为两个阶段：
1. 首先创建一个`PendingEntry`;
2. 将要“缓存的内容”，直接拷贝到 这个`PendingEntry`条目中，然后再插入到`Cache`中。


```
  //   第1步：
  //   // Allocate space in the cache for a block of 'data_size' bytes.
  //   PendingEntry entry = cache->Allocate(my_cache_key, data_size);
  //   // Check for allocation failure.
  //   if (!entry.valid()) {
  //     // if there is no space left in the cache, handle the error.
  //   }
  // 
  //   第2步：
  //   // Read the actual block into the allocated buffer.
  //   RETURN_NOT_OK(ReadDataFromDiskIntoBuffer(entry.val_ptr()));
  //   // "Commit" the entry to the cache
  //   BlockCacheHandle bch;
  //   cache->Insert(&entry, &bch);
```

也就是说：
```
// 1. 新建一个`PendingEntry`对象            --  Allocate()

// 2.1 将数据拷贝到`PendingEntry`对象中  

// 2.2 然后调用`Cache::Insert()`进行插入    -- Insert()
```

## 成员

### `cache_`
真正用来缓存数据的`Cache`对象的指针。

> 说明：`Cache`是一个基类。所以这里是一个“多态”的场景。

在这里，实际是一个`ShardedCache`类型。

## 接口列表

### 静态方法

#### `static GetConfiguredCacheMemoryTypeOrDie`
解析`GFlags`（是一些用来配置当前`cache`的选项）。

说明：如果解析失败，那么会直接退出进程。

```
Cache::MemoryType BlockCache::GetConfiguredCacheMemoryTypeOrDie() {
    ToUpperCase(FLAGS_block_cache_type, &FLAGS_block_cache_type);
  if (FLAGS_block_cache_type == "NVM") {
    return Cache::MemoryType::NVM;
  }
  if (FLAGS_block_cache_type == "DRAM") {
    return Cache::MemoryType::DRAM;
  }

  LOG(FATAL) << "Unknown block cache type: '" << FLAGS_block_cache_type
             << "' (expected 'DRAM' or 'NVM')";
  __builtin_unreachable();
}
```

说明：这里的`__builtin_unreachable()`的含义，参见：
1. https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html

简单的说：这个是`gcc`的内置函数。  像上面的代码，如果没有写这个函数，会有两个副作用：
1. 可能会产生一个警告：`"warning that control reaches the end of a non-void function."`
2. 在产生汇编代码时，仍然会在“函数的最后”，生成相应的`return语句`的汇编代码（实际上是不需要的）。

#### `static GetSingleton()`
“单例类”中获取对象的接口。

```
  static BlockCache* GetSingleton() {
    return Singleton<BlockCache>::get();
  }
```

### 向`Cache`中写入新条目

#### `Allocate()`

```
BlockCache::PendingEntry BlockCache::Allocate(const CacheKey& key, size_t block_size) {
  Slice key_slice(reinterpret_cast<const uint8_t*>(&key), sizeof(key));
  return PendingEntry(cache_->Allocate(key_slice, block_size));
}
```

说明：参见`PendingEntry`的构造函数部分的说明，这里就是直接使用`PendingEntry`直接去封装一个`Cache::UniquePendingHandle`对象。

#### `Insert()`
```
void BlockCache::Insert(BlockCache::PendingEntry* entry, BlockCacheHandle* inserted) {
  auto h(cache_->Insert(std::move(entry->handle_),
                        /* eviction_callback= */ nullptr));
  inserted->SetHandle(std::move(h));
}
```

说明：`ShardedCache::Insert()`返回的是`Cache::UniqueHandle`类型，即这里`h`的类型。

### 从`Cache`中读取条目

#### `Lookup()`

```
bool BlockCache::Lookup(const CacheKey& key, Cache::CacheBehavior behavior,
                        BlockCacheHandle* handle) {
  auto h(cache_->Lookup(
      Slice(reinterpret_cast<const uint8_t*>(&key), sizeof(key)), behavior));
  if (h) {
    handle->SetHandle(std::move(h));
    return true;
  }
  return false;
}
```

说明：`ShardedCache::Lookup()`返回的是`Cache::UniqueHandle`类型，即这里`h`的类型。

### 其它函数

#### `StartInstrumentation()`

```
void BlockCache::StartInstrumentation(const scoped_refptr<MetricEntity>& metric_entity) {
  std::unique_ptr<BlockCacheMetrics> metrics(new BlockCacheMetrics(metric_entity));
  cache_->SetMetrics(std::move(metrics));
}
```

# `Class BlockCacheHandle`

概述：用来封装一个`Cache::UniqueHandle`对象。**实际上就是“`Cache`中的一个条目”**。  

注意：实际上，一个`Cache::UniqueHandle`，就是一个`std::unique_ptr`的别名(参见文件`src/kudu/util/cache.h`)。

```
typedef std::unique_ptr<Handle, HandleDeleter> UniqueHandle;
```

**强调：自动清理功能是通过在`std::unique_ptr<>`中定制的“删除器”实现的**。   

只不过，当前类中直接将一个对象作为它的成员，在析构的时候，会触发`handle_`成员的析构。  
最终会调用它的“删除器”，而在“删除器”中会去检查“引用计数”，如果引用计数变为`0`，那么就会触发去 释放它所引用的数据。

**为什么要封装出一个类，而不是直接使用`Cache::UniqueHandle`**  
封装出来该类的原因，和封装`PendingEntry`类是相同的。  

都是因为要获取“`cfile block`的数据内容”时（参见`data()`方法），非常不方便。

**说明：被使用的方法：**  
在`CFileReader`类中，如果需要从`BlockCache`中去获取`cfile block`的数据，就会使用该类。

1. 构建一个空的`BlockCacheHandle`对象（记为`bc_handle`）；
2. 将`bc_handle`作为传入参数，去调用`BlockCache::Lookup()`进行查找。  
   在该方法中，会调用底层的`Cache::Lookup()`。  
    + 如果找到，那么会修改`bc_handle`对象（将对应的`UniqueHandle`对象，赋值给`BlockCacheHandle::handle_`；
    + 如果没有找到，那么不进行任何修改。（即`bc_handle`仍然是一个“空对象”）；
3. 在`CFileReader`中，针对两种场景（1) 从`Cache`中读取`cfile block`的数据”; 2) 直接从文件中读取`cfile block`的数据; ）的读取结果，都是封装在一个`BlockHandle`对象中。（也就是说，`CFileReader`中是直接与`BlockHandler`进行交互）。

也就是说，最终：在`Cache::Lookup()`中，查询的结果会被设置到一个`BlockCacheHandle`对象中。通过检查该`BlockCacheHandle`对象的信息：
1. 通过`valid()`来判断是否存在对应的缓存条目；
2. 如果存在，通过`data()`来获取数据内容。

**对比： `PendingEntry` VS `BlockCacheHandle`**  
1. `PendingEntry`: 用于向`Cache`中写入数据；
2. `BlockCacheHandle`: 用于从`Cache`中读取数据；

## 成员

### `handle_`

注意: 这个就是`Cache::UniqueHandle`对象(本质上是`std::unique_ptr<>`是对象，)，不是指针。

在该类析构的时候，这个`std::unique_ptr<>`对象的析构，会触发它的`Deleter`(删除器)就会被调用。   也就是其中缓存的内容，如果引用计数减为0，那么对应的内存就会被释放。  

```
  Cache::UniqueHandle handle_;
```

## 接口列表

### `swap()`
```
  void swap(BlockCacheHandle* dst) {
    std::swap(handle_, dst->handle_);
  }
```

### `data()`

```
  Slice data() const {
    return handle_.get_deleter().cache()->Value(handle_);
  }
```

参见`PendingEntry::val_prt()`的说明。 这里的实现逻辑，和那里是一样的。  

### `valid()`
```
  bool valid() const {
    return static_cast<bool>(handle_);
  }
```

如果`handle_`不为空（`handle_`本质上是一个`std::unique_ptr<>`），只要它的值不是`nullptr`，那么就是“有效的”。

### `private SetHandle()`

```
  void SetHandle(Cache::UniqueHandle handle) {
    handle_ = std::move(handle);
  }
```
在`BlockCache::Lookup()`的时候，对于给定的`CacheKey`，如果找到了对应的条目，那么就会调用该函数。

也就是说，如果该`handle_`被设置，那么当前`BlockCacheHandle`对象就是“有效的”。

# `GFLAGS`变量

## `FLAGS_block_cache_type`

默认值是`DRAM`。

在读取的时候，会先尝试转化为“大写形式”，所以在设置的时候，可以用小写形式。

可以取如下两个值之一：
1. `"DRAM"`：将缓存放在“内存”中。
2. `"NVM"`：将缓存放在“memory-mapped file”中（会使用`memkind lib`库）。


## `FLAGS_block_cache_capacity_mb`

cache的总大小，单位是`MB`。

默认是`512MB`。

## `FLGAS_force_block_cache_capacity`

这个变量的含义是：如果在运行过程中时，如果动态修改了该`gflags`变量的值，是否需要检查所设置的“cache大小”是否合法。

1. 如果该值为`true`，表示不检查；
2. 如果该值为`false`, 那么需要进行检查；

> 备注：每个`gflags`变量，都可以指定一个`validator`。 在修改它的值时，会调用这个`validator`来校验“新值”是否合法。

参见`ValidateBlockCacheCapacity()`, 这里的值为`true`时，带来的结果是，该`gflag`变量对应的`validator`直接返回`true`，就不会进行任何检查。

> 说明：将该变量的名字，改为`FLGAS_need_check_block_cache_capacity`会更好理解一些。

这个值的默认值是为`true`。 

如注释所说，这个配置项是比较奇怪的（因为为`true`的话，表示“不进行检查”，那就没有设置`FLAGS_block_cache_capacity_mb`的必要了）。

实际上，这个变量的目的是：为了根据当前进程的类型，进行区分：
1. 对于除了`master`和`tserver`之外的节点类型，那么不进行验证。
2. 对于`master`和`tserver`这两种节点类型，那么要进行验证。

> 说明：对于除了`master`和`tserver`之外的节点类型，比如是 “某个工具进程”，是不需要进行检查的。 

对于`master`和`tserver`进程，会在启动的时候，手动显式的把该`flags`变量的值设置为`false`。 也就是说，对于这两种节点类型，会进行真正的“容量”检查。  

```
// 在`./src/kudu/master/master_main.cc`和 `./src/kudu/tserver/tablet_server_main.cc`中，都会显示的将该`flag`变量的值设置为`false`:

CHECK_NE("", SetCommandLineOptionWithMode("force_block_cache_capacity",
    "false", gflags::SET_FLAGS_DEFAULT));
```









