[TOC]

文件： `src/kudu/common/row_operations.h`

## `struct DecodedRowOperation`

表示一个从客户端接收的“行操作”。

将一个`Row`编码的方式，参见`src/kudu/wire_protocol.proto`中的`RowOperationsPB`类型。

这里表示，一个经过解码的`Row`操作。

注意：对于不同的“操作类型”，会使用不同的成员。

### `type`
操作类型。

### `row_data`
保存操作的数据内容。

+ 对于`INSERT`和`UPSERT`：包含了整个投影的行；
+ 对于`UPDATE`和`DELETE`：只包含当前行的“主键”；

### `isset_bitmap`
用来标识那些被 client 显式赋值的“单元格”。

对于`INSERT`和`UPDATE`: 当前`bitmap`用来标识哪些“单元格”被显式的赋值，而不是使用“默认值”进行填充。

### `changelist`

对于`UPDATE`和`DELETE`类型，对应的 “变化列表”。

### `split_row`

对于`SPLIT_ROW`类型，用来进行分裂的部分行。

### `result`
每个行操作的结果。是`Status`类型

```
struct DecodedRowOperation {
  RowOperationsPB::Type type;

  // For INSERT or UPSERT, the whole projected row.
  // For UPDATE or DELETE, the row key.
  const uint8_t* row_data;

  // For INSERT or UPDATE, a bitmap indicating which of the cells were
  // explicitly set by the client, versus being filled-in defaults.
  // A set bit indicates that the client explicitly set the cell.
  const uint8_t* isset_bitmap;

  // For UPDATE and DELETE types, the changelist
  RowChangeList changelist;

  // For SPLIT_ROW, the partial row to split on.
  std::shared_ptr<KuduPartialRow> split_row;

  // Per-row result status.
  Status result;

  // Stringifies, including redaction when appropriate.
  std::string ToString(const Schema& schema) const;

  // The 'result' member will only be updated the first time this function is called.
  void SetFailureStatusOnce(Status s);
};
```