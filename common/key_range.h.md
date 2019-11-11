[TOC]

文件：`src/kudu/common/key_range.h`

用来描述一个`key`的区间（即`range分区`的tablet的一个`range`），是“左闭右开”的。

对应的`PB`结构是`KeyRangePB`（参见`src/kudu/common/common.proto`）。

>> 问题1：既然两者的内容完全一样，为什么还要单独定义一个结构，而不是直接使用`KeyRangePB`? 

>> 问题2：为什么要包括一个`size_bytes_`成员。 

## 成员

注意：`start_key_`和`stop_key_`的类型都是`std::string`，而不是`Slice`。

所以构造一个`KeyRange`对象，也就意味着要进行 字符串拷贝。

### `start_key_`

开始点； 

### `stop_key_`

结束点。

### `size_bytes_`

这个`range`的数据大小。