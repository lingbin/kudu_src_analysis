[TOC]

## `class TransactionDriver`

事务的驱动类。 负责协调事务的执行过程。

每个事务，都会对应一个`TransactionDriver`对象(注意：在一个事务中，可能会修改多行数据)。

### 通用流程

在不同的`Raft`角色上（`leader`或`follower`)，事件的驱动方式是不同的。

通用的流程是：
1. `Init()`: 在新创建一个`Driver`对象上，会调用`Init()`方法。
    + 1.1. 如果是`REPLICA`上创建的，说明当前操作已经处于`REPLICATING`状态了（因为一定是从raft leader 复制过来的，所以也就不需要再去触发复制操作）。在这个场景，事务已经被`leader`序列化过了，所以我们在这里也会调用`transaction_->Start()`。

>> 问题：为什么在`leader`上序列化过了，那么就需要在这里也调用 `txn->Start()`？

2. `ExecuteAsync()`：向`prepare_pool_`提交`PrepareTask()`，并且立即返回（因为是“异步执行”）。
3. `PrepareTask()`在事务上调用`Prepare()`：  
    一旦prepare成功，如果尚未复制日志（当前节点为`LEADER`），赋予一个时间戳，并且调用`transaction_->Start()`，最后调用`consensus->Replicate()`，然后把`replication state`修改为`REPLICATING`。  
    另一方面，如果已经被赋值到大多数（比如，当前节点是follower节点，并且`ReplicationFinished()`已经被成功执行），那么可以直接执行`ApplyAsync()`
4. `RaftConsensus`去调用`ReplicationFinished()`  
    由raft协议来触发（根据当前节点的commit index大于对应log的id）。
    + 4.1  在`follower`上，可能会在`Prepare()`结束之前就发生，所以必须检查是否已经执行过上面的第3步。
    + 4.2  在`leader`上，只有在`Prepare()`之后才会进行“raft协议流程”，所以上面的检查（第3步是否已经执行）是一定成立的。
    
    如果`Prepare()`已经结束，那么触发`ApplyAsync()`。
5. `ApplyAsync()`向`apply_pool_`提交`ApplyTask()`，然后`ApplyTask()`会调用`transaction_->Apply()`。
    当`Apply()`被调用时，就会去修改内存结构。这些修改对用户还是不可见的。 当`Apply()`结束，会向`WAL`的队列中中追加一条`CommitMsg`，用来保存操作的结果信息。  
    当`CommitMsg`被追加到log的队列中以后，`Driver`会执行`Finalize()`，然后这个事务的修改就对 其它事务 可见了。 然后，`Drivre`会向 客户端 发送回复，事务也到此结束。  
    该事务所修改的内存数据结构，都是已经持久化的了。


### “获取行锁” 和 “‘写事务’的开始” 

**在`leader`上，必须先“获取所有‘行锁’”(`AcquireLockForOp()`)，然后再开启事务开启(调用`tablet->StartTransaction()`方法)。**

>> 说明：在`tablet->StartTransaction()`中，会赋予这个事务一个时间戳。

这样才能保证一点：在每一行上，时间戳是单调递增的。

如果先“开启事务”，然后再“获取行锁”，那么可能出现如下的场景：

```
//
//   Thread 1         |  Thread 2
//   ----------------------
//   Start tx 1       |
//                    |  Start tx 2
//                    |  Obtain row lock
//                    |  Update row
//                    |  Commit tx 2
//   Obtain row lock  |
//   Delete row       |
//   Commit tx 1
//
// This would cause the mutation list to look like: @t1: DELETE, @t2: UPDATE which is invalid,
// since we expect to be able to replay mutations in increasing timestamp order on a given row.
//
// This requirement is basically two-phase-locking: the order in which row locks are acquired for
// transactions determines their serialization order. If/when we support multi-node serializable
// transactions, we'll have to acquire _all_ row locks (across all nodes) before obtaining a
// timestamp.
//
// Note that on non-leader replicas this requirement is no longer relevant. The leader assigned
// the timestamps and serialized the transactions properly, so calling tablet_->StartTransaction()
// before lock acquisition on non-leader replicas is inconsequential.
```

这种操作序列的结果会变为：`@t1: DELETE, @t2: UPDATE `，这是不合理的（因为`@t2`的事务时间戳大，但是因为它先获取到行锁，所以`@t2`先执行，它在现有版本的数据上进行更新。但是实际上因为`@1`的执行，会删掉了现有版本。`@t2`是一个`update`操作，本应该因为没有对应的数据而返回失败）。

这个需求，实际上就是“两阶段锁(`tow-phase-locking`)”：事务获取行锁的顺序，决定了它们的序列化顺序。 

如果将来Kudu支持了 “多tablet事务”，那么必须（在所有相关节点上）获取所有的行锁，然后才能去获取一个时间戳。

>> 这个思路：说明Kudu目前计划的“跨tablet事务”，是基于“两阶段锁”，是悲观的。

在“非leader”节点上，不要求这个顺序。因为时间戳是`leader`分配的，只要在`leader`上是可序列化的，在`follower`上，这两个操作的 顺序不重要了。

### 为了保证“执行且只执行一次”，需要跟踪记录事务

事务的“仅执行一次”语义，要求记录下 已经执行过的事务，并且能够向客户端回放：**也就是如果收到一个重复的请求，能够向客户端回复相同的内容**。

对于单个服务器的操作，比如某个命令，这个功能可以封装在`RPC层`。

但是对于需要在多个服务器复制的操作，比如写事务，因为一个`RPC`的多个拷贝，可能来自不断的地方，所以需要特别注意。

比如说：一个client发起一个`RPC`，它先向当前的`leader`发送请求，但是因为某种原因没有收到回复，然后这个client向`follower`发起该操作的重试，但其实这个角色为`follower`的`tablet server`可能已经从之前的`leader`收到了相同的操作。

“事务的prepare阶段”是在单线程中做的，所以这是一个比较理想的位置，去向`ResultTracker`去注册一下当前`follower transaction`，从而去除重复的请求。 但是，因为来自客户端的`RPC`是在它们第一次到达的时候进行注册的，是在 transaction prepare阶段之前进行的，这时来自另一个replica的请求可能已经到达（比如当前节点成为leader），所以需要记录下所有这些可能的交叉。

相关的约束：
1. 在一个replica成为leader之前（在接收来自client的请求之前），它已经把当前所有的请求都放到队列中了。
2. 如果一个replica不是leader，那么它在prepare阶段，拒绝client的请求。
3. 被复制的事务，也就是那些 作为follower接收到的事务，在prepare阶段会注册到`ResultTraker`中。这约束了因为来自client端的请求在prepare阶段产生的交叉，因为它1)如果能看到之前的请求而中止，2)没观察到，那么就表示该请求是第一次执行；

对于一个新选出来的leader，可能会出现如下的场景: 

```
// CR - Client Request
// RR - Replica Request
//
//
//             a) ---------                                  b) ---------
//                    |                                             |
//       RR prepares->1                                 CR arrives->1
//                    |                                             |
//                    2<-CR arrives                                 2<-RR prepares
//                    |                                             |
//                    3<-CR attaches                                3<-CR prepares/aborts
//                    |                                             |
//      RR completes->4                                 CR retries->4
//                    |                                             |
// CR replies cached->5                 CR attaches/replies cached->5
//                    |                                             |
//                ---------                                     ---------
//
```

场景a) 比较简单。在原来的leader已经prepare(1)以后，客户端重试一个请求(2)。当它尝试向`ResultTracker`注册(2)时，这个来自客户端的请求会“绑定到(3)”之前的执行流程中。 最终，当之前的执行流程(`RR`)结束，这个来自客户端的请求(`CR`)也会获得相同的结果。

场景b) 略微复杂。当`CR`到来(1)时，之前的leader 还没有进行prepare，也就是说当前`CR`会被标记为一个“新请求”，并进行相应的处理。与此同时，如果有`RR`进行prepare(2)。 因为`RR` 必须优先于`CR`，所以`RR`必须是哪个得到真正执行的，也就是说，它必须将`CR`强制中止。  使用如下两种方式：  
1. 它立即调用`ResultTracker::TrackRpcOrChangeDriver()`，这会保证它会被注册，并且会保证它的结果才是会被保存的。
2. 然后，当`CR`尝试去prepare(3), 它会观察到它已经不是该事务的`driver`（因为它的`attemp_no`已经被`RR`的`attempt_no`所替换），然后太会调用`ResultTracker::FailAndRespond()`，最终导致client端进行重试(4, 5)。

当一个client收到一个因为上面步骤(2)产生的错误信息，它会重试当前操作(4)，这次重试的请求，会被绑定到相应的`RR`的执行，并且会获取到和`RR`相同的结果(5)。

一些注意事项：

上面的图，并没有区分`RR`的`arrive`和`prepare`，因为这两步是原子的（向`ResultTracker`注册 和 进行prepare）；

即使，在`RR`prepare之前就进行了`CR`的处理，也是可以的。因为它会因为该replica尚未成为leader而被中止(上面的约束 1/2)



