[TOC]

文件： `src/kudu/fs/block_id.h`

# `class BlockId`
对应的`protobuf`结构为`BlockIdPB`

和`ColumnId`(参见`src/kudu/common/schema.h`)一样，这里其实就是将`uint64_t`封装起来。

## 成员

对象的成员只有一个，就是`id_`。

额外定义了一个 常量的静态成员： `kInvalidId`，值为`0`。

```
  static const uint64_t kInvalidId;
  uint64_t id_;
```
## 接口列表

定义了基本的`getter`和`setter`方法，以及和`BlockIdPB`相互转化的函数。

### `ToString()`

```
  std::string ToString() const {
    return StringPrintf("%016" PRIu64, id_);
  }
```

注意：这里会将`uint64_t`的整型，打印成一个长度为`16`的 字符串。如果位数不到，就填充前导`0`.

# `struct BlockIdHash`  --函数对象
是一个“函数对象”，重载了`operator()`方法，用来对`BlockId`进行hash计算。

```
struct BlockIdHash {
  size_t operator()(const BlockId& block_id) const {
    return block_id.id();
  }
};
```

# `struct BlockIdCompare` --函数对象
是一个“函数对象”，重载了`operator()`方法，用来对`BlockId`进行大小比较。

```
struct BlockIdCompare {
  bool operator()(const BlockId& first, const BlockId& second) const {
    return first < second;
  }
};
```

# `struct BlockIdEqual` -- 函数对象
是一个“函数对象”，重载了`operator()`方法，用来对`BlockId`比较是否相等。

```
struct BlockIdEqual {
  bool operator()(const BlockId& first, const BlockId& second) const {
    return first == second;
  }
};
```

# `BlockIdSet`  -- `std::unordered_set<BlockId>` 的别名

实际上是一个`std::unordered_set<>`，只不过定制了它的“模板参数”。

```
typedef std::unordered_set<BlockId, BlockIdHash, BlockIdEqual> BlockIdSet;

```
