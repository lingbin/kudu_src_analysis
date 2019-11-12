[TOC]

文件： `src/tablet/transactions/transaction.h`

## `struct TransactionMetrics`

表示一个事务的相关metric信息，目前只有两类
1. 增删改的行数(`insert`, `upsert`, `update`, `delete`)。
2. `commit_wait`的时间。

```
// All metrics associated with a Transaction.
struct TransactionMetrics {
  TransactionMetrics();
  void Reset();
  int successful_inserts;
  int successful_upserts;
  int successful_updates;
  int successful_deletes;
  uint64_t commit_wait_duration_usec;
};
```

## `enum Transaction::TransactionType`

一共只有两种“事务类型”。

```
enum TransactionType {
    WRITE_TXN,
    ALTER_SCHEMA_TXN,
};
```

>> 问题： “schema change”也是一种事务？

## `enum Transaction::TransactionResult`

事务的结果。任何一个事务，只可能有两种结果：1) 提交; 2)回滚。
```
  enum TransactionResult {
    COMMITTED,
    ABORTED
  };
```

## `enum Transaction::TraceType`

```
  enum TraceType {
    NO_TRACE_TXNS = 0,
    TRACE_TXNS = 1
  };
```

>> 问题：干什么用的？不同的trace_type，行为上有什么不同？

## `class Transaction`

事务的基类，每种“事务类型”可以实现自己的子类。

目前有两种子类：
1. `WRITE`: `WriteTransaction`子类；
2. `ALTER_SCHEMA`：`AlterSchemaTransaction`子类；

### 成员

```
private:
  const consensus::DriverType type_;
  const TransactionType tx_type_;
```

对于`consensus::DriverType`和`TransactionType`的两个成员，因为一旦创建该对象，这两个都是不变的，所以在编程实践上讲，应该把它们设置成`private`成员、然后提供相应的`getter`方法即可。

>> 说明：这里原本还有一个`TransactionState* state_`成员，但是实际上是没有使用的，所以已经向Kudu提交PR，删除掉了（参见：https://gerrit.cloudera.org/#/c/14253/）

注意：虽然基类中不需要`TransactionState* state_`成员，但是每种类型的`Transaction`都是需要对应的`TransactionState`成员的（`TransactionState`也是一个基类，有不同类型的子类。不同类型的事务会实现不同的子类（`WriteTransationState`和`AlterSchemaTransation`），所以各个子类中，都单独定义了相应的`TransactionState`成员。

另外，`Transaction`的各个子类，在重新定义自己的`TransactionState`成员时，可能有不同的方式。
1. 使用`std::unique_ptr<>`来封装。`WriteTransaction`和`AlterSchemaTransaction`都是这种方式。比如 `std::unique_ptr<WriteTransactionState> state_`；
2. 使用`gscoped_ptr<> `来封装。比如，在单测中的`NoOpTransaction`。

当然，因为各个子类中只需要负责维护好对应的`TransactionState`成员即可，具体使用任意方式都是可以的。

### 流程相关的方法

有以下和流程相关的方法：
#### **`Prepare()`**

执行一些准备工作。

注意：
  + 不同类型的`Transaction`，所对应的准备工作是不同的。
  + 但是有一些基本特点：都不会修改任何数据结构，并且都没有任何副作用。

#### **`Start()`**

真正开始一个事务。

在这个阶段，会给该事务赋予一个时间戳。

分为两种：
1. `LEADER replica`: 在leader节点上，在`Prepare()`后，会立即执行该函数。
2. `FOLLOWER/LEARNER replica`: 在follower节点上，只保证在`Apply()`之前执行。  
    因为时间戳是`Leader`赋予的，`FOLLOWR/LEARNER`必须在收到来自`LEADER`的 raft消息后，才能进行`Apply()`）。

`Once Started(Start() return OK), state might have leaked to other replicas/local log and the transaction can't be cancelled without issuing an abort message.`

>> 问题：这段英文注释，是什么意思？

#### **`Apply()`**

不同类型的事务，有不同的工作内容。但是通常会在这个阶段，修改相应的数据结构。

#### **`Finish()`**

执行的操作内容：执行一些清理操作，`TransactionDriver`会在当前执行完`Finish()`以后，将结果返回给客户端。  
  
注意：事务的执行结果只有两种场景：`COMMITTED`或`ABORTED`，在参数中进行指定。 

被调用的两个场景：
1. 事务被`Apply()`以后、并且`commit message`被提交到`log`(注意：有可能还没有被持久化，即可能还没有被刷到磁盘)；
2. 事务被回滚；  

>> 问题：为什么可能还没有被持久化？是指流程本身？还是指没开启`fsync`时（write()已经返回了，但是数据并没有刷盘）？

这个方法的实现，会包含清理操作。

注意：这个方法返回以后，`TransactionDriver`就会向客户端返回结果。

#### **`AbortPrepare()`**

表示在raft复制消息时失败了，调用该方法把`Prepare()`中的一些准备工作给取消掉。

目前只有`WriteTransaction`会用到（所以，该方法有默认实现，是个空的函数）。

### `NewReplicateMsg()`方法

创建当前事务所对应的 `ReplicateMsg`对象。

创建的结果(`consensus::ReplicateMsg`对象)，保存在 参数所指向的内存中。

说明：参数类型是`gscoped_ptr<consensus::ReplicateMsg>* replicate_msg`，即是一个“智能指针的指针”。 该智能指针真正所指向的对象，是在该方法中进行 动态分配的。

比如：`WriteTransaction::NewReplicateMsg()`的实现如下：

```
void WriteTransaction::NewReplicateMsg(gscoped_ptr<ReplicateMsg>* replicate_msg) {
  replicate_msg->reset(new ReplicateMsg);
  (*replicate_msg)->set_op_type(WRITE_OP);
  (*replicate_msg)->mutable_write_request()->CopyFrom(*state()->request());
  if (state()->are_results_tracked()) {
    (*replicate_msg)->mutable_request_id()->CopyFrom(state()->request_id());
  }
}
```

如上，在方法内部，动态实例化了1个`ReplicateMsg`对象，并用来初始化了参数中的智能指针。

## `class TransactionState`

类似于`transaction context`（"事务上下文"）的概念。即包含一个事务执行所需要的相关信息。

是一个基类，每种类型的事务，都有各自对应的子类（`WriteTransactionState`和`AlterTransactionState`）。

### 成员

注意：都是`protected`的，所以子类中可以直接使用；

#### `tx_metrics_`成员

该事务的`metric`信息。

#### `tablet_replica_`成员

该操作，所针对的replica指针。

>> 问题：这里只是简单的保存一个`TabletReplica*`类型的普通指针，不是一个`std::shared_ptr<>`类型。那么怎么防止在事务执行过程中，对应的replica被删除（即`TabletReplcia`可能已经失效的）的问题？

#### `result_tracker_`成员

缓存一个事务的执行结果。

如果有重发的请求，那么可以直接获取数据，而不需要再次执行；

#### `completion_clbk_`成员

回调函数。事务结束后会进行回调。

注意：不是所有的事务，都有回调函数。如果一个事务有回调函数，那么通过`set_completion_callback()`进行指定。

即使有的事务类型，没有对应的回调函数，在实现时也会给它设置一个`TransactionCompletionCallback`成员。（`TransactionCompletionCallback`是一个空实现的基类）这样的好处是：如果不设置一个默认的实现，那么在代码中，在每次使用时都需要先检查是否`null`。而如果有了默认实现，那么就不需要“检查是否为`null`”了。

>> 问题：什么样的事务，没有回调函数。

#### `pool_`成员

对象池。在事务运行过程中，对象都在这个“对象池”中进行分配，事务结束后会自动析构。

具体实现参见`auto_release_pool.h`。

#### `timestamp_`成员

该事务的时间戳。

#### `timestamp_error_`成员

事务运行过程中，与“时钟”有关的错误码。

#### `arena_`成员

内存池。

#### `op_id_`成员

raft消息的`id`。

#### `request_id_`成员
当前请求的`request_id_`。 

注意：只有当前请求是“可跟踪”的时候，才会设置这个成员。否则，这个成员为空。

#### `consensus_round_`成员

当前写请求（写事务）对应的raft结构。

#### `external_consistency_mode_`成员

指定当前的“外部一致性”模式。


#### `txn_state_lock_`

互斥锁。用于当前`TransactionState`对象的一些成员操作的互斥。

注意：子类中的一些成员，也会依赖该锁来保证互斥。

```
  TransactionMetrics tx_metrics_;

  TabletReplica* const tablet_replica_;

  scoped_refptr<rpc::ResultTracker> result_tracker_;

  gscoped_ptr<TransactionCompletionCallback> completion_clbk_;

  AutoReleasePool pool_;

  Timestamp timestamp_;
  uint64_t timestamp_error_;

  Arena arena_;

  // This OpId stores the canonical "anchor" OpId for this transaction.
  consensus::OpId op_id_;

  // The client's id for this transaction, if there is one.
  rpc::RequestIdPB request_id_;

  scoped_refptr<consensus::ConsensusRound> consensus_round_;

  ExternalConsistencyMode external_consistency_mode_;

  mutable simple_spinlock txn_state_lock_;
```

### 构造函数

1. 因为构造函数`protected`，所以外部函数是不可以直接构造该对象的。（目前只有子类的构造函数中，会调用该基类的构造函数）
2. 构造函数中，只穿了一个参数(`TalbetReplica*`)，而该类有很多成员。这些成员的赋值，**都是通过对应的`setter`函数来进行设置的**。

>> 问题:  为什么要通过`setter`方法来赋值，而不是在构造函数中直接赋值。  
>> 答：这些成员并不是每个`TransactionState`都存在的。比如，对于`request_id_`成员，即使在`WriteTransactionState`中，也是有的存在，有的不存在。

所以对于大部分成员，都提供了`has_xxx()`方法，来判断是否存在对应的属性。

比如：该类提供了一个方法(`has_request_id()`)来判断是否存在`request_id_`。即用户程序在调用`request_id()`之前，必须先通过`has_request_id()`来检查下，是否真的存在`request_id_`。

注意：因为都是通过`setter`方法来对成员赋值，所以对于这些成员，要注意访问顺序。在通过`getter`访问前，一定要保证已经通过`setter`进行了赋值。

### 其它函数

因为本类的作用，就是维护一个事务相关的所有相关变量，所以该类的方法，主要是提供一些`getter`和`setter`方法。

## `class TransactionCompletionCallback`

基类，表示一个事务完成时被调用的回调函数。

如果调用者希望在事务结束时得到通知，**那么必须手动设置**（使用`TransactionState::set_completion_callback()`方法）

具体的`Callback`对象，在设置到`TransactionState`中以后，由`TransactionState`负责它的析构（随着`TransactionState`对象的析构而析构，实现的方法是：将该回调函数的对象，在`TransactionState`中，以`gscoped_ptr<>`的方式使用）。

和常常使用的`Closure`一样，有两个成员： `Status`成员和‘错误码’。

**注意：**  
**这个类虽然是一个基类，但是它不是一个抽象类**。只是一个默认什么都不做的基类。所以，在需要的地方，可以直接传入这个类的对象。
使用一个“空类”的好处是：不需要在代码中不断的检查是否为`nullptr`。

具体的回调函数是：`TransactionCompleted()`，默认的实现是一个空函数。

>> 问题：这里的多个方法都很简单，但是没有设置成`inline`的，这里可以提个PR，把这些方法设置成inline的。
>> 说明：虽然没有设置成`inline`，但是对性能是没有影响的，因为各个类型的事务子类，都没有直接使用该类型，而是使用的各个类型对应的子类。

### 成员

这些成员都是`protected`的，所以子类可以直接访问。

```
protected:
  Status status_;
  tserver::TabletServerErrorPB::Code code_;
```

