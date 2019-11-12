[TOC]

文件: `src/kudu/tablet/rowset_metadata.h`

# `class RowSetMetadata`

用来跟踪记录 一个`RowSet`的 `data blocks`。  
（也就是说，因为一个`tablet`中会有多个`RowSet`，所以一个`tablet`中也会有多个`RowSetMetadata`对象）

在一个tablet中，当`MemRowSet`进行`flush`时，
1. 会创建一个`RowSetMetadata`对象;
2. `DiskRowSetWriter`会“创建”并“写入数据”到 “不可修改”的`blocks`：用来保存“列数据”、`bloom filter`和`adhoc-index`。

说明： `adhoc-index`： 就是指“复合主键”的索引。

在`flush`作业结束以后（所有的`block`数据也已经填充完毕），那么会将`RowSetMetadata`也`flush`下来。  
此时，只会有一个`block`会包含所有的`tablet metadata`，所以“flush RowSetMetadata” 就会触发对整个`TabletMetadata`进行flush。

>> 问题：见注释：这句话什么意思？

注意：  
“`metadata`的回写”是 `lazy` 的，通常的应用方法是：

1. 在磁盘上创建一个文件；
2. 修改内存状态，指向新文件；
3. 在内存中的`RowSetMetadata`对象中进行修改；
4. 触发“异步地”flush；

回调函数的行为（在`metadata`被写入完毕以后）：
1. 从磁盘删除旧的数据文件；
2. 删除 和旧的内存数据相关的 `log anchors`;

## 子类型

```
  typedef boost::container::flat_map<ColumnId, BlockId> ColumnIdToBlockIdMap;
```

注意：这是一个`flat_map`类型。 

关于`flat_map`类型, 参见： 
1. https://www.boost.org/doc/libs/1_65_1/doc/html/boost/container/flat_map.html
2. https://medium.com/@gaussnoise/why-standardizing-flat-map-is-a-bad-idea-7efb59fe6cea  

## 成员


## 接口列表

### `static CreateNew()`  -- 工厂方法




# ``













