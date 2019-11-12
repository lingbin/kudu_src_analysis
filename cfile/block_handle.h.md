[TOC]

文件：`src/kudu/cfile/block_handle.h`

# `class BlockHandle`

说明：在读取一个`cfile block`的时候，这个`cfile block`的位置，有两种情况：
1. 在`BlockCache`中；
2. 不在`BlockCache`中，即在“堆内存”中；

这两种情况，在读取完毕后（不再使用这个`cfile block`）的时候，处理方式不同：

1. 如果是在`BlockCache`中，那么只需要将对应的“引用计数”减`1`即可；
2. 如果不在`BlockCache`中，那么当前`BlockHandle`对象，需要`own`这一段内存。即在析构当前对象时，需要释放对应的“堆内存”。

**这个类的职责：就是封装了上面两种情况的处理，从而让使用者 不需要去关心，对应的`cfile block`的“内存释放问题”。**  

## 成员

### `dblk_data_`
如果当前`cfile block`的数据来自于`BlockCache`，那么这个成员会被赋值，指向在`BlockCache`中的地址。

>> 说明：这个“变量名”的含义： 这里的`'d'`应该是“删除”的意思。 即`"to_delete_blk_data_"`，即一个“待删除的`cfile block`数据”。  
>> 因为本类的作用：就是封装 当不再使用一个`cfile block`的数据时，如何进行资源释放。   
>> 而要释放的内容，就是这个`cfile block`的数据。  

注意：这里的类型是`BlockCacheHandle`对象，而不是“指针”。

如果当前的数据来自于`Cache`，那么这个值被赋过值。所以在当前对象被析构的时候，就会去析构`dblk_data_`对象，然后就会触发`BlockCacheHandle::handle_`成员（实际上是一个`std::unique_ptr<>`对象）的释放，最终释放其中的数据成员。

注意：参见下面的构造函数部分，如果用一个“`BlockCacheHandle`指针”去构造本对象时，会调用`swap()`，从而保证“作为参数的`BlockCacheHandle`对象”，在析构时，不会去释放它所引用的数据内容。

### `data_`
如果当前`cfile block`的数据来自于“堆内存”，那么这个成员会被赋值，指向在“堆内存”中的地址。

### `is_data_owner_`
标识当前`cfile block`的数据，是否来自于“堆内存”。

```
  BlockCacheHandle dblk_data_;
  Slice data_;
  bool is_data_owner_;
```

## 接口列表

### 静态工厂函数

#### `WithOwnedData()`

```
  static BlockHandle WithOwnedData(const Slice& data) {
    return BlockHandle(data);
  }
```

#### `WithDataFromCache()`

```
  static BlockHandle WithDataFromCache(BlockCacheHandle *handle) {
    return BlockHandle(handle);
  }

```

说明：从这两个“静态工厂函数”可以看出来，无论是“堆内存”，还是“`BlockCache`中的条目”，在使用本对象之前，都是要提前获取好的。 

也就是使用流程为：

```
// 1. 获取`cfile block`的数据内容。 可能是“堆内存”，也可能是从`BlockCache`中获取的。

// 2. 调用这两个“静态工厂方法”中的一个，构建`BlockHandle`对象；

// 3. 后续直接使用上面构建的`BlockHandle`对象；

// 4. 使用完毕，直接退出。
//   （“使用者”不用考虑它所引用的`cfile block`数据 是如何释放的，有`BlockHandle`对象自动处理）
```

### 构造函数

```
  BlockHandle()
    : is_data_owner_(false) { }

  // Move constructor and assignment
  BlockHandle(BlockHandle&& other) noexcept {
    TakeState(&other);
  }
  BlockHandle& operator=(BlockHandle&& other) noexcept {
    TakeState(&other);
    return *this;
  }

  ~BlockHandle() {
    Reset();
  }
  
private:
  explicit BlockHandle(Slice data)
      : data_(data),
        is_data_owner_(true) {
  }

  explicit BlockHandle(BlockCacheHandle* dblk_data)
      : is_data_owner_(false) {
    dblk_data_.swap(dblk_data);
  }
  
  DISALLOW_COPY_AND_ASSIGN(BlockHandle);
```

**注意1：这个类是 *不允许* 进行 拷贝和赋值的。但是可以进行“移动构造”和“移动赋值”。**  

注意2：从“默认构造函数”可以看出，一个“空”的`BlockHandle`对象，它的`is_data_owner_`成员为`false`(对应的`dblk_data_`也是没有赋值的)

注意3：在“析构函数”中，当前只调用了`Reset()`（实际上，在`Reset()`中，并没有手动去释放`dblk_data_`成员）

但是既然已经在“析构函数”，那么也就是正在析构“当前对象”。   
所以，作为成员的`dblk_data_`（一个`BlockCacheHandle`对象）就会被析构，它的析构，最终会造成对`BlockCache`中的相应内容的“引用计数”减`1`。  
也就是说，
1. 如果“引用计数”变为`0`，那么就会触发对`BlockCache`中的相应内容进行析构；
2. 如果“引用计数”没有减为`0`，那么就什么都不做。

### `getter`函数

#### `data()`
```
  Slice data() const {
    if (is_data_owner_) {
      return data_;
    } else {
      return dblk_data_.data();
    }
  }
```

### `TakeState()`

工具函数。 获取 另一个`BlockHandle`对象的属性。

在该函数执行完毕后，`other`中已经不再维护任何信息。

```
  void TakeState(BlockHandle* other) {
    Reset();

    is_data_owner_ = other->is_data_owner_;
    if (is_data_owner_) {
      data_ = other->data_;
      other->is_data_owner_ = false;
    } else {
      dblk_data_.swap(&other->dblk_data_);
    }
  }
```

### `Reset()`

清理函数。清空当前对象中的“状态”。
```
  void Reset() {
    if (is_data_owner_) {
      delete [] data_.data();
      is_data_owner_ = false;
    }
    data_ = "";
  }
```

注意：这里在`Reset()`的最后，会执行`data_ = ""`。 它的作用是将`Slice data_`的值设置为空。  

> 说明：一个`size = 0`的`Slice`对象，就会被认为是一个“空”的`Slice`。

>> 问题：这里没有显式的去释放`dblk_data_`的内容。为了防止误用，这里应该是需要显式释放的。