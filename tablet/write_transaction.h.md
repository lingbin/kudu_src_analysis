[TOC]

文件: `src/kudu/transactions/write_transaction.h`

## `enum WritePrivilegeType`

“写权限”的类型。

```
// Privileges required for write operations.
enum WritePrivilegeType {
  INSERT,
  UPDATE,
  DELETE,
};

static constexpr size_t kWritePrivilegeMax = WritePrivilegeType::DELETE + 1;
```

## `class WriteTransactionState`

继承自`TransactionState`。

“写事务”（多个 ‘插入’和‘修改’操作）的上下文。

这个类几乎包含“写事务”相关的所有信息，包括
1. 对于每一行，都有一个`RowOp`结构体。一个`RowOp`结构包括：
    + 经过“解码的”和“投影的(`projected`)”数据；
    + 行锁 的引用；
    + 写操作（`insert/mutate`）的执行结果；
2. 与`Replicate`和`Commit`相关的PB结构。

**所有和事务相关的指针，都是在该类中进行 owned。**  有两种析构方式：
1. `Reset()`方法；
2. 析构函数；

**注意**：  
所有被获取到的“行锁”，直到下面3种情况之一，才会被释放：
1. 析构函数；
2. 调用`Reset()`；
3. 调用`release_locks()`时；

在向`WAL`中写入数据时，使用 该类 来记录 写操作（inserts/updates）的 修改位置，并且把这个位置信息 写入到 `commit message`中（会被写入到`WAL`中）。

>> 问题： 在Kudu中，raft wal log 和 存储引擎的wal是否“合二为一”了？

注意：`WriteTransactionState`不是线程安全的。

### 成员

#### **`owned_repsponsed_`成员**  
仅仅针对 `follower txns`。   

对于`leader txns`的`response`是有RPC框架来创建和销毁的。但是`follower txn`的response是需要自己进行维护的。

>> 问题： 对于`follower txns`， 为什么kudu仍然走了请求处理的流程？ 感觉实际上可能是没有必要的，这里等理清楚代码后进行说明。 如果不需要，尝试给kudu提交PR。

#### **`request_`成员和`response_`成员**  
这两个成员的生命周期，都是有RPC框架进行维护的（即在本类中，不负责这两个对象的创建和析构）。

`request_`总是在 构造函数中进行初始化。

`response_`，如果是`leader txn`，那么在构造函数中进行初始化。如果是`follwer txn`，那么指向 `owned_responsed_`成员。

也就是说，**这两个指针成员，一定不会为 `nullptr`**。

#### **`row_ops_`成员**  
从`request_`中解析出来的多个“行操作”的具体内容。

由父类的`txn_state_lock_`提供 互斥保护。  

#### **`stats_array_`成员**  

是一个`ProbeStats`类型的数组。

在当前事务的内存池（`arena`）中进行分配的（所以会随着事务的析构而析构）。

`ProbeStats`类，用来记录每个 行操作 过程中的统计信息("statistics")，详见`rowset.h`

**`mvcc_tx_`成员**

对应的MVCC transaction对象，在`Prepare()`阶段创建。

**`tablet_components_`成员**  
当前tablet的`TabletComponents`结构。

说明：  
`TabletComponents`结构，用来记录当前tablet的组成信息，即`memrowset`集合和`rowset`集合。

这个结构是不可修改的。所以可以在一个事务中，引用这个对象，从而保证在事务过程中，该对象不会被修改。

**`schema_lock_`成员**  

用来对schema加锁。 方式在写入过程中，其它人都schema进行并发修改。

在设置`mvcc_tx_`的时候，会获取这个锁（即进行加锁操作）。

**`schema_at_decode_time_`成员**  

`Schema*`类型，指向第一次解析该事务的时候，该tablet的schema。

在`Apply()`之前，还会检查下schema，从而保证没有并发修改的`schema change`操作。

由父类的`txn_state_lock_`提供 互斥保护。  

>> 问题：因为这个是一个指针类型，那么在事务执行过程中，schema发生了变化，那么怎么保证这个`schema`指针依然是有效的？ 即当前`schema_`指针所指向的对象，依然是有效的？

```
  tserver::WriteResponsePB owned_response_;
  
  const tserver::WriteRequestPB* request_;
  tserver::WriteResponsePB* response_;

  // Encapsulates state required to authorize a write request. If 'none', then
  // no authorization is required.
  boost::optional<WriteAuthorizationContext> authz_context_;

  std::vector<RowOp*> row_ops_;

  // Array of ProbeStats for each of the operations in 'row_ops_'.
  // Allocated from this transaction's arena during SetRowOps().
  ProbeStats* stats_array_ = nullptr;

  // The MVCC transaction, set up during PREPARE phase
  gscoped_ptr<ScopedTransaction> mvcc_tx_;

  // The tablet components, acquired at the same time as mvcc_tx_ is set.
  scoped_refptr<const TabletComponents> tablet_components_;

  shared_lock<rw_semaphore> schema_lock_;

  const Schema* schema_at_decode_time_;
```

### 一些需要注意的`getter`和`setter`

大部分`getter`和`setter`方法，如果没有特别的注意事项，那么这里就忽略掉，不单独说了。

#### `SetMvccTx()`方法  

绑定当前“写事务”所对应的 "mvcc txn"（`ScopedTransaction`对象）。

这个方法应该在 事务时间戳 获取之后，被调用 有且仅有1次。

它会从 "mvcc txn"中，将时间戳 拷贝到 `WriteTransactionState`对象。


#### `set_tablet_components()`方法

>> 问题：在kudu中，有些方法是“大写字母”开头，有些方法是“小写”（使用下划线连接），这里的规则到底是？

在`Apply()`开始以后、并且在修改内存结构之前，正好调用1次。

### 和事务流程相关的函数

#### `StartApplying()`方法
通知"mvcc txn", 即将要将修改 apply 到内存结构。

在该方法调用之后，当前事务 必须 在一段时间后调用`Commit()`方法。

>> 问题： 如何保证必须调用？  如果宕机了，是不是会有影响？

#### `CommitOrAbort()`方法
将"mvcc txn"结束（commit或者 abort，通过调用`ReleaseMvccTxn()`），然后释放“行锁”、“schema 锁”等。

如果是`COMMITTED`，那么在这个方法之后，所做的修改，对其它用户就是可见的了。

注意： 
+ 只能被调用 1次。
+ `startApplying()`必须已经被调用过。
+ 在此方法之后，`request_`和`response_`都会被置为`nullptr`，所以不能再访问这两个成员。

#### `ReleaseMvccTxn()`方法

将"mvcc txn"结束（根据参数传递的结果，分别调用"mvcc txn"的 `Commit()`或者`Abort()`方法）。

最后将 `mvcc_tx_`指针 置为 `nullptr`;


### 其它方法

#### `ReleaseTxResultPB()`方法

>> 问题：这个方法是用来干啥的？ 为什么要release result?

注意：这个方法只能被调用一次。如果调用两次，进程会crash。


## `class WriteTransaction`

继承自 `Transaction`，对应一个写请求。

主要是实现了基类的几个事务流程方法（即`Prepare()`, `AbortPrepare()`, `Start()`, `Apply()`, `Finish()`）

相比较于父类，该子类特有的成员只有一个：`WriteTransactionState state_`（与“写事务”相关的所有属性，基本都在这个结构中）。

### `Apply()`方法





```
```













