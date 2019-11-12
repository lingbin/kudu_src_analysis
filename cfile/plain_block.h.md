[TOC]

文件：`src/kudu/cfile/plain_block.h`

# 概述

本文件中的类，就是在描述以`PLAIN_ENCODING`方式，对block的数据进行“编码”和“解码”。

**说明：所针对的“数据类型”，是“定长的数据类型”。  
包括：所有的“整型”、“浮点型”、“`Decimal`”。 不包括`bool`和“变长类型”（`String`和`Binary`）。

# 全局函数

工具函数。将`ptr`所指向的地址，转化为一个`Type`类型的“值”。

```
template<typename Type>
inline Type Decode(const uint8_t *ptr) {
  Type result;
  memcpy(&result, ptr, sizeof(result));
  return result;
}
```

# 常量

## `kPlainBlockHeaderSize`

```
static const size_t kPlainBlockHeaderSize = sizeof(uint32_t) * 2;
```

参见`PlainBlockBuilder::Finish()`方法，在结束当前`CFile Block`时，会编码进一个“元信息”，包括两个值：
1. 当前`Block`中的“值”的个数，即当前`Block`中数据的行数；
2. 当前`Block`中第一行数据，在`RowSet`中的“行号”；

# `template Decode()`  -- 全局模板函数

# `template class PlainBlockBuilder`

继承自`BlockBuilder`类。

**注意：这里的方式是：用一个“模板内”去继承一个“普通类”。**  
因为给定一个模板参数，就会特化层一个具体的类，也就是说，根据传入的模板类型，最终会有很多不同种类的子类。

```
template<DataType Type>
class PlainBlockBuilder final : public BlockBuilder {
  ...
}
```

## 子类型

```
  typedef typename TypeTraits<Type>::cpp_type CppType;
```

这个类型，表示当前“列”的类型，在`C++`中所使用的类型。

注意：见`kCppTypeSize`成员出的说明。根据`Type`的类型，会有很多特化类。每个特化类中的`CppType`是不同的。 
## 成员

### `buffer_`
用来保存“编码的结果”

因为当前类就是要对一个`CFile Block`中的数据，按照`PLAIN_ENCODING`方式进行编码。

注意：在`Finish()`的时候，所返回的`Slice`对象中，所间接引用的数据，就是这个`buffer_`中的数据。

### `const options_`
当前`CFile Block`的写选项。

注意：这是一个`const`指针，当前类并不`own`这个指针（在当前类析构的时候，并不释放它的内容）  
它所引用的对象，是在`CFileWriter`的`options_`成员。

### `count_`
已经添加到当前`CFile Block`的“值”的数量。

### `kCppTypeSize`

说明：这里的根本方式是：**使用一个枚举类型，来定义一个“常量”。**  

注意：因为当前类本身是一个模板类，所以根据不同的模板参数，`TypeTraits<Type>`也会是不同的类。  
这样，形成的结构就是：在不同的`PlainBlockBuilder`的特化类中，`kCppTypeSize`的值是不同的。

```
  faststring buffer_;
  const WriterOptions *options_;
  size_t count_;
  typedef typename TypeTraits<Type>::cpp_type CppType;
  enum {
    kCppTypeSize = TypeTraits<Type>::size
  };
```

说明：

## 接口列表
所有接口都是继承的（即父类中要实现的“纯虚函数”）。

说明：这里的实现：体现了“面向接口编程”的理念。

### 构造函数

```
  explicit PlainBlockBuilder(const WriterOptions *options)
      : options_(options) {
    // Reserve enough space for the block, plus a bit of slop since
    // we often overrun the block by a few values.
    buffer_.reserve(kPlainBlockHeaderSize + options_->storage_attributes.cfile_block_size + 1024);
    Reset();
  }
```

注意：在构造“缓存池”的时候，多申请了一部分空间。（目前是`1024`个字节）

**说明：多留一部分空间的原因：**  
在向当前`CFile Block`中插入数据时，常常会多插入一些“值”（即会超过`cfile_block_size`的值）。

说明2：参见`Reset()`方法的说明：在`Reset()`中，会把`buffer_`的“当前指针”，指向`kPlainBlockHeaderSize`处。  

是因为在添加完数据后，前`kPlainBlockHeaderSize`个字节，会被用来存放一些“元信息”（参见`Finish()`）

也就是说：在向`buffer_`中添加数据时，是从第`kPlainBlockHeaderSize`个字节开始的。 而前`kPlainBlockHeaderSize`个字节，是在`Finish()`的时候，才被填充进去的。

### `Add()`
向当前`CFile Block`中，添加“一组值”。

返回值是：实际被添加进去的“值”的数量。

```
  virtual int Add(const uint8_t *vals_void, size_t count) OVERRIDE {
    int old_size = buffer_.size();
    buffer_.resize(old_size + count * kCppTypeSize);
    memcpy(&buffer_[old_size], vals_void, count * kCppTypeSize);
    count_ += count;
    return count;
  }
```

说明1：这里每次都会调用`faststring::resize()`函数。  

如果当前`faststring`对象有足够的剩余空间，即“`faststring::capacity()`”是超过了“要去`resize()`的目标大小”，那么这个方法，只是修改下`size_`下标，并不会进行内存的“分配”和“删除”。 

而因为在当前对象的“构造函数”中，已经提前分配了足够的空间，所以这里，这里每次调用的开销并不大。

说明2：正是因为每次都会调用`faststring::resize()`函数，因为在该函数中，每次都增加了下标，但是并没有进行任何的填充，所以后面要调用`memcpy()`来将数据填充进去。

注意1：这里“没有处理”在添加数据的过程中，`CFile Block`中途变“满”了的情况。  
也就是说，在实际运行时，当前`CFile Block`的内容，是可能会超过`cfile_block_size`的大小。  

这里不进行处理的原因，是为了实现的方便。  
这也是在构造函数中，要多申请一部分内存的原因（目前是“硬编码”的`1024`个字节）。  
因为预先多申请了内存，那么这里在添加数据的时候，即使超过了，对于`buffer_`来说，也不需要动态的扩展内存。

说明3：在`CFileWriter`中，会使用`IsBlockFull()`方法，来判断当前的`CFile Block`是否已经满了。

+ 如果已经满了，那么就不会在调用`Add()`方法，继续向其中添加数据；
+ 如果未满，而且后续有数据的时候，才会继续调用`Add()`来添加数据；

也就是说，即使当前`CFile Block`的大小超过了`cfile_block_size`的值，超过部分的大小，也最多是当前`Add()`传进来的数据大小。

>> 问题：上层有没有控制单次调用`Add()`时，传入的最大数据量？ 为什么不去判断是否“满”了？

说明4：因为前面每次都会调用`faststring::resize()`方法，所以，即使这里在一次“`Add()`”中要添加的数据量非常大，也不会有问题。因为`resize()`会保证，在`buffer_`中有足够的空间来存储它。

### `IsBlockFull()`

**说明：判断是否为满了的标志：**  
它的“预计的大小”，超过了配置的`WriterOptions::cfile_block_size`。

注意：如果这个`CFile Block`是“满”的，`CFileWriter`会去调用`FinishCurDataBlock()`方法。

```
  virtual bool IsBlockFull() const override {
    return buffer_.size() > options_->storage_attributes.cfile_block_size;
  }
```

### `Finish()`

表示添加数据结束了（即不再添加新的数据）。

函数的返回值：`Slice`对象所“间接引用”的数据，在当前`BlockBuilder`对象的“缓存池”中。

```
  virtual Slice Finish(rowid_t ordinal_pos) OVERRIDE {
    InlineEncodeFixed32(&buffer_[0], count_);
    InlineEncodeFixed32(&buffer_[4], ordinal_pos);
    return Slice(buffer_);
  }
```

### `Reset()`

```
  virtual void Reset() OVERRIDE {
    count_ = 0;
    buffer_.clear();
    buffer_.resize(kPlainBlockHeaderSize);
  }
```

注意：这里会把`buffer_`的“当前指针”，指向`kPlainBlockHeaderSize`处。  

是因为在添加完数据后，前`kPlainBlockHeaderSize`个字节，会被用来存放一些“元信息”（参见`Finish()`）。

### `Count()`
```
  virtual size_t Count() const OVERRIDE {
    return count_;
  }
```

### `GetFirstKey()`
```
  virtual Status GetFirstKey(void *key) const OVERRIDE {
    DCHECK_GT(count_, 0);
    UnalignedStore(key, Decode<CppType>(&buffer_[kPlainBlockHeaderSize]));
    return Status::OK();
  }
```

### `GetLastKey()`
```
  virtual Status GetLastKey(void *key) const OVERRIDE {
    DCHECK_GT(count_, 0);
    size_t idx = kPlainBlockHeaderSize + (count_ - 1) * kCppTypeSize;
    UnalignedStore(key, Decode<CppType>(&buffer_[idx]));
    return Status::OK();
  }
```

# `template class PlainBlockDecoder`