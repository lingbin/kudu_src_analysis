[TOC]

# `faststring`类

文件：`./src/kudu/util/faststring.h`

**说明1：本类的存在的意义：**  
就是用来替换掉`std::string`。避免了`std::string`在使用上一些低效的地方。

**说明2：一句话说明它的重要性：**  
数据库就是用来存放和读取数据的，所以对于“字符串”或者“二进制”的相关操作是非常多的。  
字符串相关的“拷贝”、“动态扩展”等操作，如果能够提高效率，那么会很大程度上提高系统的整体效率。

在`std::string`中，有一些默认的操作，是比较费时的。参加下面的说明。

**说明3：`faststring` VS `Slice`:**   
实际上也是`std::string`和`Slice`的区别。

两者区别的本质是：`Slice`对象并不真正的`own`内存，而`faststring`中是会`own`内存的。

另外，两者的使用场景也有所不同：
1. `faststring`：用来替换`std::string`的。主要是`std::string`在一些场景下，不够高效。
2. `Slice`：用来“间接引用”数据，因为并不真正的`own`内存，所以可以避免大量“拷贝”的开销。

**说明4：相比较于`std::string`，在`faststring`中，针对一些常用的场景，做的一些优化：**  
1. 默认有一个大小为`32`的`uint8_t array`成员：
  + “栈对象”：因为是`array`，所以将`faststring`作为“栈对象”的时候，如果要求的“容量”小于`32`时，声明对象时并不需要动态的申请内存，直接使用这个数组即可。 不需要使用`new`来动态申请，，所以效率很高。  
    而将`std::string`
  + “堆对象”：而如果去`new`一个`faststring`对象(即将它作为“堆对象”)，如果容量小于`32`，那么只会动态申请一次（即保存`faststring`对象本身的内存）。 但是如果去`new`一个`std::string`，那么需要动态申请两次（一次是`std::string`对象本身，一次是“对象内部所间接引用”的数据）

2. **`resize()`操作，只修改大小，但不进行填充**（即对应的值是“未初始化”的）。但是在`std::string`中，会使用`\0`进行填充。这种填充，对于较大的字符串，是比较费的。

3. 因为是自行封装的类，所以对外的接口，和`std::string`虽然相似，但并不完全相同。 


说明5：一些其它特点：
1. 每次需要动态增长时，都是增长当前容量的 50%。
2. 

## 成员

### `kInitialCapacity`  
当前对象内的“数组成员”的大小。

说明：其实通过“声明一个`enum`”，来达到“声明一个常量”的目的。

### `data_`
本对象所保存的数据。

注意：在当前对象的数据量很小时，`data_`指向的是`initial_data_`。

说明：只有当“数据的容量”超过一定值（`kInitialCapacity`），`data_`就会指向“动态申请”的内存。

注意：本类的“数据”，要么保存在“动态申请的内存”中，要么保存在“数组成员”中。（不会在两者中，各保存一部分）

### `initial_data_`
初始的“字符数组”成员。当数据量很小时，使用这个“数组成员”来保存数据。

### `len_`
数据的长度。

注意：如果要向当前对象添加数据，那么也是从`len_`位置开始进行添加。

### `capacity_`
当前对象能够容纳的最大数据量。

注意：因为在构造该对象的时候，就会使用“内置的数组”，然后按需动态增长，所以一个`faststring`对象的`capacity_`，最小值就是`kInitialCapacity`.

```
public:
  enum {
    kInitialCapacity = 32
  };
  
private:
  uint8_t* data_;
  uint8_t initial_data_[kInitialCapacity];
  size_t len_;
  size_t capacity_;
```

## 接口方法

说明：因为本类就是为了替换`std::string`的，所以接口基本都是和`std::string`一样的。

### 构造函数

提供两个：
1. 默认构造函数：即“容量”是默认大小，实际上就是“数组”成员的大小。
2. 指定“容量”；

**注意：如何区分`data_`中的数据是“动态申请”的，还是指向“数组成员”中的：**  
只需要比较`data_`和`initial_data_`指针即可。

1. 如果`(data_ == initial_data_)`，那么就说明当前对象的“数据”，是在“数组”中的。那么在“析构”当前对象的时候，是不需要单独去通过`delete[]`去释放内存的（因为 数组成员的内存，在对象析构的时候，自动被析构）。
2. 如果两者不相等，那么说明`data_`指向“动态申请的内存”，那么在“析构”的时候，就需要手动通过`delete[] data_`，手动释放内存。

```
faststring() :
    data_(initial_data_),
    len_(0),
    capacity_(kInitialCapacity) {
  }

  explicit faststring(size_t capacity)
    : data_(initial_data_),
      len_(0),
      capacity_(kInitialCapacity) {
    if (capacity > capacity_) {
      data_ = new uint8_t[capacity];
      capacity_ = capacity;
    }
    ASAN_POISON_MEMORY_REGION(data_, capacity_);
  }
```

说明：只有所需要的“容量”大于`kInitialCapacity`，才会进行动态分配内存。

### 析构函数
```
  ~faststring() {
    ASAN_UNPOISON_MEMORY_REGION(initial_data_, arraysize(initial_data_));
    if (data_ != initial_data_) {
      delete[] data_;
    }
  }
```

### 和`std::string`共有的接口

#### `clear()`
删除所有内容。

注意：本函数并不会释放任何内存。当前对象的“容量”保持不变。

```
  void clear() {
    resize(0);
    ASAN_POISON_MEMORY_REGION(data_, capacity_);
  }
```

#### `resize()`
如果指定的“新容量”大于当前的容量，那么将进行“动态扩展”；如果指定的“新容量”小于当前的容量，那么只缩短`len_`，不进行其它操作。

参见：http://www.cplusplus.com/reference/string/string/resize/  
注意：这个函数的实现，是`faststring`类区别于`std::string`的关键之一：
1. 在`std::string::resize()`中，如果需要“动态扩展”，那么新扩展出来的“内存”，会被赋值为`\0`；
2. 而`faststring::resize()`，不会对这些“新扩展”的内存进行任何初始化。

```
  void resize(size_t newsize) {
    if (newsize > capacity_) {
      reserve(newsize);
    }
    len_ = newsize;
    ASAN_POISON_MEMORY_REGION(data_ + len_, capacity_ - len_);
    ASAN_UNPOISON_MEMORY_REGION(data_, len_);
  }
```

#### `reserve()`
设置“容量”。

1. 如果“新容量”大于“当前的容量”，那么进行动态扩展；
2. 如果“新容量”小于等于“当前的容量”，那么不进行任何操作（注意：这时并不会进行“缩容”）；

```
  void reserve(size_t newcapacity) {
    if (PREDICT_TRUE(newcapacity <= capacity_)) return;
    GrowArray(newcapacity);
  }
```

#### `append()`
向当前`faststring`对象中添加数据。

如果“剩余容量”不够，那么就会进行“动态扩展空间”。

```
  // Append the given data to the string, resizing capacity as necessary.
  void append(const void *src_v, size_t count) {
    const uint8_t *src = reinterpret_cast<const uint8_t *>(src_v);
    EnsureRoomForAppend(count);
    ASAN_UNPOISON_MEMORY_REGION(data_ + len_, count);

    // appending short values is common enough that this
    // actually helps, according to benchmarks. In theory
    // memcpy_inlined should already be just as good, but this
    // was ~20% faster for reading a large prefix-coded string file
    // where each string was only a few chars different
    if (count <= 4) {
      uint8_t *p = &data_[len_];
      for (int i = 0; i < count; i++) {
        *p++ = *src++;
      }
    } else {
      strings::memcpy_inlined(&data_[len_], src, count);
    }
    len_ += count;
  }

  void append(const std::string &str) {
    append(str.data(), str.size());
  }
```

**说明：针对“添加`4`个以内字符”的场景进行单独优化。**  

该优化的`commit_id: 446d5d64ae5b53ff1916c8323a9feb06f7fbc42a`  

前提：在代码中，添加“非常短的字符串”的场景是非常多的。

如注释所说：虽然理论上，`memcpy_inlined()`（是一个`memcpy`的`inline`实现）的性能已经是比较好的。

但是在将“添加小于`4`个字符”的分支独立出来以后（直接进行逐个拷贝），性能提升了约`20%`。

#### `push_back()`
添加一个“字符”。

```
  // Append the given character to this string.
  void push_back(const char byte) {
    EnsureRoomForAppend(1);
    ASAN_UNPOISON_MEMORY_REGION(data_ + len_, 1);
    data_[len_] = byte;
    len_++;
  }
```

#### `getter`方法

##### `length()`
```
  size_t length() const {
    return len_;
  }
```

##### `size()`
```
  // Return the valid length of this string (identical to length())
  size_t size() const {
    return len_;
  }
```

##### `capacity()`
```
  size_t capacity() const {
    return capacity_;
  }
```

#### `data()`
```
  // Return a pointer to the data in this string. Note that this pointer
  // may be invalidated by any later non-const operation.
  const uint8_t *data() const {
    return &data_[0];
  }

  // Return a pointer to the data in this string. Note that this pointer
  // may be invalidated by any later non-const operation.
  uint8_t *data() {
    return &data_[0];
  }
```

#### `at()`和`operator[]`
获取指定“下标”的字符值。

注意：这两个方法，都不会检查“是否发生了越界”。

```
  const uint8_t &at(size_t i) const {
    return data_[i];
  }
  
  const uint8_t &operator[](size_t i) const {
    return data_[i];
  }
  
  uint8_t &operator[](size_t i) {
    return data_[i];
  }
```

### 在`std::string`中没有的方法

#### `assign_copy()`
将参数`src`指向的数据，拷贝长度为`len`的数据，到当前`faststring`对象中。

```
  void assign_copy(const uint8_t *src, size_t len) {
    // Reset length so that the first resize doesn't need to copy the current
    // contents of the array.
    len_ = 0;
    resize(len);
    memcpy(data(), src, len);
  }

  void assign_copy(const std::string &str) {
    assign_copy(reinterpret_cast<const uint8_t *>(str.c_str()), str.size());
  }
```

#### `release()`
这个方法，在`std::string`中是没有的。

作用是：将当前的“数据内容”返回，并且清空当前`faststring`内部的状态。  

说明：在本函数返回以后，当前对象中的“表示状态”的变量，都会恢复到初始状态(即一个空的`faststring`的状态。即无论当前状态是否在使用“内置的数组”，在调用该方法以后，都会重新使用"内置数组"来保存数据)。

**返回值的注意事项：**  
1. 如果当前正在使用“动态分配的内存”，那么“返回值”是“动态分配内存地址”；
2. 如果当前正在使用“数组成员”，那么会 1) 新申请一块内存；2）将数据，从“数组”中拷贝到“新申请的内存”中；3) 返回“新申请的内存地址”；

综上，也就是说，这个的返回值：一定是指向一块“动态申请的内存”。 

而且，当前`faststring`也不再`own`这块“动态申请的内存”，所以，“调用者”需要保证，在使用完毕后，要记得通过`delete[]`把内存释放掉。 或者使用`unique_ptr<uint8_t[]>`来封装，从而可以在析构时自动释放内存。

```
  // Releases the underlying array; after this, the buffer is left empty.
  //
  // NOTE: the data pointer returned by release() is not necessarily the pointer
  uint8_t *release() WARN_UNUSED_RESULT {
    uint8_t *ret = data_;
    if (ret == initial_data_) {
      ret = new uint8_t[len_];
      memcpy(ret, data_, len_);
    }
    len_ = 0;
    capacity_ = kInitialCapacity;
    data_ = initial_data_;
    ASAN_POISON_MEMORY_REGION(data_, capacity_);
    return ret;
  }
```

>> 备注：该函数的使用者，只有一个，就是`EncodedKey`的构造函数（参见`src/kudu/common/encoded_key.cc`）。

#### `shrink_to_fit()`

根据当前的“使用情况”，对“当前使用的内存”进行“缩容”。

**为什么会有这个函数：**  
参见`commit_id: dce25916f428659072fd1e140a9081829956468e`  
之所以要有这个函数，针对的场景是：有些`faststring`对象会长期存在（即生命周期很长），而这些对象的“容量”在一般情况下也比较小，但是可能会被“偶然”的提高到一个很大的值（比如偶尔收到了一个很大的请求）。   
可以利用这个函数，来释放它的内存。  
因为如果没有这个函数，那么它所占用的内存，就一直得不到释放。  

注意：在默认情况下，因为“数组”成员的存在，最小的容量为`kInitialCapacity`。

如果是如下两种情况之一，那么会直接返回：
1. 如果当前是在使用“数组成员”。因为“数组”本身是无法进行“缩容”的，所以这时什么都不用做，直接返回；
2. 如果当前没有“多余的空间”，即`capacity_ == len_`，那么也没法进行缩容，直接返回即可。

所以，会执行“缩容”的场景是：
1. 当前正在使用“动态分配的内存”；
2. 并且有多余的空间，即`capacity > len_`;

```
// 场景1：
    array :  |xxxxxxxxxxxxxxx|
    data_ :  |abcdefghijklmnopqrstuvwxyz______|    --说明： 尾部的`_`表示 “剩余的可用空间”
    
    这时，`data_`指向“动态分配的空间”（这时，并不关心`array`中的内容），而且存在“剩余可用空间”。
    
    所以如果在该`faststring`对象上调用`shrink_to_fit()`，会真正进行“缩容”操作。

// 场景2：
    array :  |xxxxxxxx___|
    data_ :  = &array[0]
    
    即，这时`data_`指向“数组”，所以不会进行“缩容”。

// 场景3：    
    array :  |xxxxxxxxxxxxxxx|
    data_ :  |abcdefghijklmnopqrstuvwxyz|    --说明： 尾部没有`_`，表示 没有“剩余的可用空间”
    
    这时，`data_`指向“动态分配的空间”，但 `capacity_ == len_`，所以没有“剩余可用空间”。
    
    所以，也不会进行“缩容”。

// 场景4：    
    array :  |xxxxxxxxxxxxxxx|
    data_ :  |abcdef_________________|    --说明： 尾部没有`_`，表示 没有“剩余的可用空间”    
    
    这时会进行“缩容”，缩容的结果为：
    
    array :  |abcdefxxxxxxxxx|
    data_ :  = &array[0]
    
    即data_ 原来所指定的“动态分配的内存”会被释放，并且重新指向`array`。
```

```
  // Reallocates the internal storage to fit only the current data.
  //
  // This may revert to using internal storage if the current length is shorter than
  // kInitialCapacity. In that case, after this call, capacity() will go down to
  // kInitialCapacity.
  //
  // Any pointers within this instance may be invalidated.
  void shrink_to_fit() {
    if (data_ == initial_data_ || capacity_ == len_) return;
    ShrinkToFitInternal();
  }
```

#### `ToString()`
转化为`std::string`对象。

```
  // Return a copy of this string as a std::string.
  std::string ToString() const {
    return std::string(reinterpret_cast<const char *>(data()), len_);
  }
```

#### `private EnsureRoomForAppend()`
```
  // If necessary, expand the buffer to fit at least 'count' more bytes.
  // If the array has to be grown, it is grown by at least 50%.
  void EnsureRoomForAppend(size_t count) {
    if (PREDICT_TRUE(len_ + count <= capacity_)) {
      return;
    }

    // Call the non-inline slow path - this reduces the number of instructions
    // on the hot path.
    GrowToAtLeast(len_ + count);
  }
```

#### `private GrowToAtLeast()`
```
void faststring::GrowToAtLeast(size_t newcapacity) {
  // Not enough space, need to reserve more.
  // Don't reserve exactly enough space for the new string -- that makes it
  // too easy to write perf bugs where you get O(n^2) append.
  // Instead, alwayhs expand by at least 50%.

  if (newcapacity < capacity_ * 3 / 2) {
    newcapacity = capacity_ *  3 / 2;
  }
  GrowArray(newcapacity);
}
```

说明：这里每次去“提升容量”的时候，都会至少提升“当前容量的一半”。

>> 备注：这种“提升容量”的方式，是在`commit_id:e03df2c45f94b91f1b8999e0aad1343659355ab0`中加入的。  

>> 问题：注释中的`write perf bugs`是指什么？

#### `private GrowArray()`
```
void faststring::GrowArray(size_t newcapacity) {
  DCHECK_GE(newcapacity, capacity_);
  std::unique_ptr<uint8_t[]> newdata(new uint8_t[newcapacity]);
  if (len_ > 0) {
    memcpy(&newdata[0], &data_[0], len_);
  }
  capacity_ = newcapacity;
  if (data_ != initial_data_) {
    delete[] data_;
  } else {
    ASAN_POISON_MEMORY_REGION(initial_data_, arraysize(initial_data_));
  }

  data_ = newdata.release();
  ASAN_POISON_MEMORY_REGION(data_ + len_, capacity_ - len_);
}
```

说明1：这是一个工具函数。在调用该函数之前，已经经过了判断，即在该函数内，一定满足`newcapacity >= capacity_`。

说明2：为了保证在出现问题的时候，自动析构“申请出来的`uint8_t`数组”，这里使用`std::unique_ptr<uint8_t>`来封装了“`new`返回的结果”。

最后，通过调用`std::unique_ptr<>::release()`来释放掉，即在超出作用域后，不会在自动析构。

#### `private ShrinkToFitInternal()`

```
void faststring::ShrinkToFitInternal() {
  DCHECK_NE(data_, initial_data_);
  if (len_ <= kInitialCapacity) {
    ASAN_UNPOISON_MEMORY_REGION(initial_data_, len_);
    memcpy(initial_data_, &data_[0], len_);
    delete[] data_;
    data_ = initial_data_;
    capacity_ = kInitialCapacity;
  } else {
    std::unique_ptr<uint8_t[]> newdata(new uint8_t[len_]);
    memcpy(&newdata[0], &data_[0], len_);
    delete[] data_;
    data_ = newdata.release();
    capacity_ = len_;
  }
}
```

## 该类的一些说明

### 对“不常用的路径”不使用`inline`

参见`commit_id: 0a70458e48e35739067ecc778090afd58d9d781f`的`commit msg`。

在该提交之前，所有的函数都是`inline`的。

但是，在该类的使用场景中，通常在进行`append()`或者`push_back()`之前，都已经预先`resize()`了足够的内存。

也就是说，在`append()`和`push_back()`方法内，通常是不需要进行“动态扩展内存”的。

基于这个现实，在这个提交中，将“动态扩展容量”的逻辑，都从`inline`函数（指`append()`、`reserve()`）中移了出来，用`非inline`的方式实现（即在`.cpp`文件中实现）。

作者指出，这样做的好处是：这样会减少在“关键路径”上需要被`inline`的代码大小，能提高效率。

>> 问题：减少了“内敛的”代码大小，为什么可以提高效率？是因为：指令的cpu缓存命中率提高了？

### 添加`ASAN`

>> 问题：参见`commit_id: 264b6b6df261beacb3c64363b334f153ed8af49e`
>> 添加`ASAN`的原理是？ 
