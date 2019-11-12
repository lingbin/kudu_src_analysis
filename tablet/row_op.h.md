[TOC]

文件： `src/kudu/tablet/row_op.h`

说明：名词解释：  
“修改操作”有三种：`insert`/`update`/`delete`

有时也分为两种: `insert`/`mutate`。  
其中`mutate`包含两种：`update`和`delete`。

## `struct RowOp`概述

在一个`WriteTransaction`中，用来跟踪 “一行数据”的操作。

注意：在一个`WriteTransaction`中，是可能要修改多行数据的。其中每行数据，对应一个`RowOp`对象。

**注意：**  
如果只看该类的名字(`RowOp`)，可能会认为该类中只包含 对一行数据的“操作内容”（对应成员`decoded_op`）。但是实际上，该类的成员属性中，还包含一些相关“上下文”属性，比如对应的“行锁”、“所属的`RowSet`”等。

## 成员

**`decoded_op`成员**  
`DecodedRowOperation`类型(参见`src/kudu/common/row_operations.h`文件)，从客户端请求(`request`)中，解析出来的“行操作”。

**`orig_result_from_log`成员**  

如果操作是在从log中回放，那么指向 原来的结果（即leader上执行的结果，在log中获取到）；

**`key_probe`成员**  
会有多种格式（比如：`key-encoded`和`ContiguousRow`格式，`bloom probe structure`等）。

这个成员在`Prepare()`阶段时被设置。

说明： `RowSetKeyProbe`类型，用来缓存一个 编码过、并且hash过的 key，用来 探测 `Rowset`。

**`row_lock`成员**  
当前`Row`所对应的“行锁”。这个锁已经被持有。在`Prepare()`阶段进行设置。

**`valid`成员**  
用来标识该请求是否已经被检查过，是否为一个有效的操作。

**`checked_present`成员**  
标识`present_in_rowset`属性是否已经被填充。

+ 如果该值为`false`: 那么`present_in_rowset`一定为`nullptr`；
+ 如果该值为`true`: 
  - 如果`present_in_rowset`也为`nullptr`: 那么表示 当前操作要针对的key，是不存在的。
  - 如果`present_in_rowset`非空，那么表示当前key是已存在的。

**`present_in_rowset`成员**

是一个`RowSet*`指针。表示当前key所属的`RowSet`。

**`result`成员**  
该“行操作”的结果。

```
  DecodedRowOperation decoded_op;

  // If this operation is being replayed from the log, set to the original
  // result. Otherwise nullptr.
  const OperationResultPB* orig_result_from_log = nullptr;

  // The key probe structure contains the row key in both key-encoded and
  // ContiguousRow formats, bloom probe structure, etc. This is set during
  // the "prepare" phase.
  gscoped_ptr<RowSetKeyProbe> key_probe;

  // The row lock which has been acquired for this row. Set during the "prepare"
  // phase.
  ScopedRowLock row_lock;

  // Flag whether this op has already been validated as valid.
  bool valid = false;

  // Flag whether this op has already had 'present_in_rowset' filled in.
  // If false, 'present_in_rowset' must be nullptr. If true, and
  // 'present_in_rowset' is nullptr, then this indicates that the key
  // for this op does not exist in any RowSet.
  bool checked_present = false;

  // The RowSet in which this op's key has been found present and alive.
  // This will be null if 'checked_present' is false, or if it has been
  // checked and found not to be alive in any RowSet.
  RowSet* present_in_rowset = nullptr;

  // The result of the operation.
  gscoped_ptr<OperationResultPB> result;
```

## 设置结果的方法

既然本类是表示一个“行操作”，那么就应该能够表示 该行操作的结果。

提供了几个设置结果的方法（注意：如下的方法 **最多只能被调用其中一个，而且是最多一次**）。

**`SetFailed()`方法**  
标记失败。 通过参数来传递失败原因。

**`SetInsertSucceeded()`方法**
表示插入成功。在参数中，会传递 写入的mem rowset的id。

>> 备注： “mrs” 是 "mem rowset"的缩写

**`SetMutateSucceeded()`方法**
表示 update/delete成功。 参数中传递 修改的结果。

**`SetSkippedResult()`方法**
表示一个被忽略的操作，所对应的结果。

```
  // Only one of the following four functions must be called, at most once.
  void SetFailed(const Status& s);
  void SetInsertSucceeded(int mrs_id);
  void SetMutateSucceeded(gscoped_ptr<OperationResultPB> result);
  void SetSkippedResult(const OperationResultPB& result);
```

## `set_original_result_from_log()`方法

如果当前操作是在进程启动时，从WAL进行回放时产生的，那么可能需要查看之前的操作结果（在 raft的 COMMIT message中存储），从而知道 真正要去apply的 RowSet。

注意：参数中的指针，它的生命周期，和当前`RowOp`对象一样。






