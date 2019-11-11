[TOC]

文件： `src/kudu/common/scan_spec.h`

## `class ScanSpec`

用来表述一个“扫描”操作所对应的信息。

## 成员

### `predicates_`
查询的过滤条件。

是`std::unordered_map<std::string, ColumnPredicate>`类型：
+ key 是 column_name;
+ value 是 对应的过滤条件;

### `lower_bound_key_`
### `exclusive_upper_bound_key_`
### `lower_bound_partition_key_`
### `exclusive_upper_bound_partition_key_`
### `cache_blocks_`
### `limit_`

```
  std::unordered_map<std::string, ColumnPredicate> predicates_;
  const EncodedKey* lower_bound_key_;
  const EncodedKey* exclusive_upper_bound_key_;
  std::string lower_bound_partition_key_;
  std::string exclusive_upper_bound_partition_key_;
  bool cache_blocks_;
  boost::optional<int64_t> limit_;
```



















