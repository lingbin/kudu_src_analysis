[TOC]

文件：`src/kudu/cfile/block_pointer.h`

# `class BlockPointer`
表示一个`CFile Block`在文件（`fs::Block`）中的“位置信息”。

在一个文件中的“位置信息”，就有两个部分：
1. 偏移量 -- 当前`cfile block`距离当前`cfile`的起始位置的偏移量，对应`offset_`成员；
2. 该`cfile block`的长度 -- `size_`成员；

**该类对应的`protobuf`结构是`BlockPointerPB`**（参见文件`src/kudu/cfile/cfile.proto`）

## 成员

```
  uint64_t offset_;
  uint32_t size_;
```

注意1：这里两者虽然都是“整型”，但是类型却不是相同的：  
`offset_`是`uint64_t`类型；`size_`是`uint32_t`类型。

注意2：因为`size_`是`uint32_t`类型，所以一个`CFile Block`最大为`INT32_MAX`。

## 接口列表

### `构造函数`

```
  BlockPointer() {}
  BlockPointer(const BlockPointer &from) :
    offset_(from.offset_),
    size_(from.size_) {}

  explicit BlockPointer(const BlockPointerPB &from) :
    offset_(from.offset()),
    size_(from.size()) {
  }

  BlockPointer(uint64_t offset, uint64_t size) :
    offset_(offset),
    size_(size) {}
```

注意1：它定义了一个参数为`protobuf`结构（`BlockPointerPB`）的构造函数。  

注意2：本类没有提供，类似`DecodeFrom(const BlockPointerPB& pb)`的函数（从一个`protobuf`结构，来解析参数，并给当前的对象赋值），而是提供了一个利用`protobuf`结构来“直接构造”的方式。  
> 备注：下面的`DecodeFrom()`是从“一段二进制流”中，解析出当前对象。

对比：如果使用`DecodeFrom(const BlockPointerPB& pb)`，一般有两种方法：
1. 先构造一个空的`BlockPointer`对象，然后调用`obj.DecodeFrom(pb)`来初始化；  
    因为本类就两个整型成员，所以直接赋值更方便。  
2. 将`DecodeFrom()`设计为静态方法：
  + 2.1 通过“参数”来将 构造出来的`BlockPointer`对象返回；
  + 2.2 通过“返回值”来将 构造出来的`BlockPointer`对象返回；

即，使用`DecodeFrom()`方法：
1. 要么需要先构造出一个没用的“空对象”，然后再对它进行赋值；
2. 要么就要需要“拷贝对象”；

也就是说，都没有直接进行构造，更直接。

```
// 2.1 通过“参数”
static Status DecodeFrom(const BlockPointerPB& pb, BlockPointer* obj) {
    ...
    return Status::OK;
}

// 使用处的代码
BlockPointer obj;
EXIT_NOT_OK(DecodeFrom(pb, &obj));

// 分析：
先构造一个“空对象”，然后再给对象的各个成员赋值。  
还不如构造该对象的时候，直接就向它赋值。即使用`BlockPointerPB`作为“构造函数”的参数。

// 2.2 通过“返回值”
static BlockPointer DecodeFrom(const BlockPointerPB& pb) {
    BlocPointer obj(pb.xx, pb.yy);
    return obj;
}

// 使用处的代码
BlockPointer obj = DecodeFrom();
obj = DecodeFrom(pb);

// 分析：
这需要拷贝对象；
```

因为本类就两个整型成员，所以“直接构造”更方便。  

>> 扩展：会使用`DecodeFrom()`的形式，一般是以下几种场景之一：
>> 1. 需要进行一些安全检查的；
>> 2. 需要进行一些转化的。比如：`PathSetPB`和`DataDirGroup`之间的转化(参见`src/kudu/fs/data_dirs.h`);
>> 3. 对象的构造较复杂，尤其是需要涉及到内存申请等操作（因为在构造函数中不应该抛出异常），所以这时应该使用专门的方法来构造。
>> 4. 希望重用当前对象。即可能是一条处理流水线，然后在不同时间，会用一个“普通对象”来保存多个不同的`protobuf`对象；

### `ToString()`
将当前的“位置信息”打印成“字符串”。

```
  std::string ToString() const {
    return strings::Substitute("offset=$0 size=$1", offset_, size_);
  }
```

### `template EncodeTo()`
将当前的“位置信息”，编码到一个`StrType`中。

说明：模板参数`StrType`，表示一个“类`std::string`结构”。在`kudu`中有两种：`faststring`和`std::string`。  
不过目前的代码实现中，传递给这个函数的都是`faststring`类型。

```
  template<class StrType>
  void EncodeTo(StrType *s) const {
    PutVarint64(s, offset_);
    InlinePutVarint32(s, size_);
  }
```

**注意：这里的编码是“变长编码”的。**

>> 问题：为什么一个使用“普通函数”，一个使用`inline`函数？

### `DecodeFrom()`
从一段二进制中，解析出“位置信息”，并赋予当前对象的相应属性中。

```
  Status DecodeFrom(const uint8_t *data, const uint8_t *limit) {
    data = GetVarint64Ptr(data, limit, &offset_);
    if (!data) {
      return Status::Corruption("bad block pointer");
    }

    data = GetVarint32Ptr(data, limit, &size_);
    if (!data) {
      return Status::Corruption("bad block pointer");
    }

    return Status::OK();
  }
```

说明：这里的`limit`参数，只是为了安全才传递的。 即在解析的过程中，指针不能超过`limit`的值。

### `CopyToPB()`
将当前对象转化为`BlockPointerPB`类型。

```
  void CopyToPB(BlockPointerPB *pb) const {
    pb->set_offset(offset_);
    pb->set_size(size_);
  }
```

### `getter`方法

#### `offset()`
获取“当前位置”中的“偏移量”；

#### `size()`
获取“当前位置”中的“长度”；

```
  uint64_t offset() const {
    return offset_;
  }

  uint32_t size() const {
    return size_;
  }
```

### `Equals()`
比较两个`BlockPointer`对象。

```
  bool Equals(const BlockPointer& other) const {
    return offset_ == other.offset_ && size_ == other.size_;
  }
```






