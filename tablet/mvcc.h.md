[TOC]

注意：在Kudu中，使用 事务的“时间戳”作为事务的"txn id"。

## `class MvccSnapshot`  

当前MVCC状态的快照。给定一个 txn id，这个快照 能决定是否可见。

**规则：**
一个事务(它的时间戳为`T`)是被提交的，当且仅当 满足下面两个条件之一：
1. `T < all_committed_before_`;
2. 或者`committed_timestamps_.contains(T)`;

```
    In ASCII form, where 'C' represents a committed transaction,
    and 'U' represents an uncommitted one:
   
                                    /-- none_committed_at_or_after_
                                    |
      CCCCCCCCCCCCCCCCCUUUUUCUUUCUUUxxxxxxxxxxxxx
                       |    \___\___ committed_timestamps_
                       |
                       \- all_committed_before_
                       
    注意：在这个图里，all_committed_before_ 所指向的事务，是“未提交”的。
    但是实际上这个是不一定的。当all_committed_before_还没有及时更新的时候，
    （当一个事务的时间戳 等于 all_committed_before_，
    但是这个时间戳可能是在 committed_timestamps_ 列表中的），
    这个时候，这个事务也是“已提交”的。
```

Kudu在内存中，对于每个tablet，都记录一个`MvccSnapshot`对象，表示该tablet当前最新的MvccSnapshot状态（每一次写事物，都会更新该`MvccSnapshot`对象）。

>> 问题：在一个宕机重启的tablet中，如何重新构造出这个`MvccSnapshot`对象?

>> 问题：`MvccSnapshot`类在读取和写入流程中的作用是？ 给定一个时间戳，判断是否可以被读取到？ 如果是，那么这个时间戳是哪里来的？从数据中获取的？

### 成员

注意`MvccManager`是该类的友元，所以`MvccManager`可以直接 修改 和 访问 下面这些属性。

**`all_committed_before_`成员**  
时间戳 **小于** 该值的事务，都是“已提交”的。

注意： 如果事务的时间戳是 等于 `all_committed_before_`，是不能立即判断是否为“已提交”的（需要再继续检查）。

注意1：该变量的值，一定小于`safe_time_`。 因为只有保证后续到达事务的时间戳都一定大于某个值(记为`t`)，才能保证可以将`all_committed_before_`的值提升到`t`。

注意2：该变量的值，一定小于当前tablet上的`earliest_in_flight_`。因为`earliest_in_flight_`对应的事务，是正在运行中的，所以它一定是大于或等于`all_committed_before_`的。

即如下等式一定是成立的：
`all_committed_before_ = min(earliest_in_flight_, safe_time_)`

**`none_committed_at_or_after_`成员**  
时间戳 **大于或等于** 该值的事务，都是没有提交的。

这个值 = `max(committed_timestamps_) + 1`。 但是因为 `committed_timestamps_`是一个无序的vector，所以在实现时，专门使用该变量进行缓存，这样在判断一个事务状态的时候，可以很大程度的减少遍历`committed_timestamps_`的次数。  
（如果给定一个事务，它的时间戳 大于等于 `none_committed_at_or_after_`，那么不需要遍历`committed_timestamps_`，就可以判断它一定是“未提交的”）

**提升该变量的时间**：  
每次修改`committed_timestamps_`，都需要检查并修改`none_committed_at_or_after_`。 主要有以下两个场景：
1. 提交一个事务（向`committed_timestamps_`添加新的时间戳，记为T）。如果该事务的时间戳，大于或等于 当前的`none_committed_at_or_after_`，那么就要提升。（新的值为`(T + 1)`）;
2. 提升`all_committed_before_`（这时会清理掉`committed_timestamps_`中小于 `all_committed_before_`的新值 的时间戳）。 如果在清理后，`committed_timestamps_`为空，那么就需要将`none_committed_at_or_after_`的值 等于 `all_committed_before_`。

**说明：**  
这里在`committed_timestamps_`为空的时候，需要将`none_committed_at_or_after_`的值 等于 `all_committed_before_`。 是为了 防止`none_committed_at_or_after_`的值 小于`all_committed_before_`。

如果`committed_timestamps_`不为空，那么当前`none_committed_at_or_after_`的值就是等于`max(committed_timestamps_) + 1`，即一定大于当前所有“已提交”的时间戳，这是有插入逻辑保证的（在插入时，会保证`none_committed_at_or_after_`被更新为 当前最大的“已提交”时间戳）。

**`committed_timestamps_`成员**  

是一个vector，对于那些 大于`all_committed_before_` 的事务，都会放在这个vector中。  

> 说明：为什么使用`vector`，而不是使用`unorder_set<>`或`set<>`？  
> 答：在实践中，  
>   1) 这个`vector`的大小通常是很小的;  
>   2) 并且被访问的次数很少（因为大部分请求，都只需要访问  `all_committed_before_`和`none_committed_at_or_after_`即可）。  
>  
> 所以，使用一个 较紧凑的 `vector`（通常只需要 1到2个 cpu cache line），实际的性能会更好一些。

### 构造函数和工厂方法

#### **`MvccSnapshot(const Timestamp& timestamp)`**  
所有小于`timestamp`的事务都是“已提交”的，所有 大于等于 `timestamp`的事务都是“未提交”的。

方法： 将`all_committed_before_`和`none_committed_at_or_after_`都设置成`timestamp`。

#### **`MvccSnapshot()`**  
一个空的`MvccSnapshot`对象，所有对象都会被认为是“未提交”的。

和`CreateSnapshotIncludingNoTransactions()`不同的是，这里传进去的时间戳是`(kMin + 1)`，所以严格的说，如果一个事务的时间戳是kMin，那么这个事务会被当前snapshot认为是 “已提交”的。

#### **`static MvccSnapshot CreateSnapshotIncludingAllTransactions()`**  
创建一个`MvccSnapshot`对象，所有的事务（时间戳）都认为是“已提交”的。

方法： 将`all_committed_before_`和`none_committed_at_or_after_`都设置成`kMax`。

该工厂方法，经常在测试的时候使用。

#### **`static MvccSnapshot CreateSnapshotIncludingNoTransactions()`**  
创建一个`MvccSnapshot`对象，所有的事务（时间戳）都认为是“未提交”的。

方法： 将`all_committed_before_`和`none_committed_at_or_after_`都设置成`kMin`。

### 判断状态的方法

给定一个时间戳，有几种判断场景：
1. 是否“已提交”？
2. 是否“可能已提交”？
3. 是否“可能未提交”？

#### **`IsCommitted()`**

注意：
这里大部分查询都是通过与`all_committed_before_`和`none_committed_at_or_after_`比较就可以判断，所以这部分代码 置为inline的，从而提高效率。  
如果通过上面两个变量判断不了，那么就需要去比较`committed_timestamps_`的所有元素（调用`IsCommittedFallback()`方法），因为会用循环，所以这部分代码 不置为inline。

```
  inline bool IsCommitted(const Timestamp& timestamp) const {
    // Inline the most likely path, in which our watermarks determine
    // whether a transaction is committed.
    if (PREDICT_TRUE(timestamp < all_committed_before_)) {
      return true;
    }
    if (PREDICT_TRUE(timestamp >= none_committed_at_or_after_)) {
      return false;
    }
    // Out-of-line the unlikely case which involves more complex (loopy) code.
    return IsCommittedFallback(timestamp);
  }
```

#### **`MayHaveCommittedTransactionsAtOrAfter()`**  
给定一个时间戳，判断在“大于或等于”该时间戳的事务中，**是否有可能是“已提交”的**。

>> 说明： 方法中的“AtOrAfter” <==> “大于或等于” 参数中的时间戳。

方法：  
如果`timestamp`小于`none_committed_at_or_after_`，那么就返回`true`；否则返回`false`。

用途：  
用来过滤`REDO delta`文件: 如果`MayHaveCommittedTransactionsAtOrAfter(delta_stats.min)`返回`false`，那么说明当前snapshot不需要扫描这个文件（所以不需要加载这个文件）。

```
bool MvccSnapshot::MayHaveCommittedTransactionsAtOrAfter(const Timestamp& timestamp) const {
  return timestamp < none_committed_at_or_after_;
}
```

#### **`MayHaveUncommittedTransactionsAtOrBefore()`**  
给定一个时间戳，判断在‘小于或等于’该时间戳的事务中，**是否有可能是“未提交”的**。

>> 说明： 方法中的“AtOrBefore” <==> “小于或等于” 参数中的时间戳。

方法： 满足如下两个条件之一，就返回`true`:
1. `timestamp > all_committed_before_`;
2. `timestamp == all_committed_before_ && !IsCommittedFallback(timestamp)`

用途：  
用来过滤`UNDO delta`文件： 
+ 如果 `MayHaveUncommittedTransactionsAtOrBefore(delta_stats.max)`返回`false`，说明在当前snapshot的范围内，所有的 `UNDO deltas`文件中的事务都是“已提交”的，所以不需要扫描；
+ 如果返回`true`，那么说明可能会存在一些事务，需要进行undo掉。

```
bool MvccSnapshot::MayHaveUncommittedTransactionsAtOrBefore(const Timestamp& timestamp) const {
  // The snapshot may have uncommitted transactions before 'timestamp' if:
  // - 'all_committed_before_' comes before 'timestamp'
  // - 'all_committed_before_' is precisely 'timestamp' but 'timestamp' isn't in the
  //   committed set.
  return timestamp > all_committed_before_ ||
      (timestamp == all_committed_before_ && !IsCommittedFallback(timestamp));
}
```

#### **`is_clean()`**  

一个`clean`的snapshot是指：由某个‘时间戳’来决定的快照。  
也就是说，这个snapshot把所有小于某个“时间戳”的事务都认为是“已提交”的；把其它的事务（“大于或等于”某个时间戳的事务）都认为是“未提交”的。

方法：判断`committed_timestamps_`是否为空，如果为空，那么说明是`clean`的。

>> 问题：
>> 在一个snapshot只取决于一个时间戳时，那么`committed_timestamps_`一定为空。但是反之是不一定成立的。那为什么这里用这个条件来判断？ 是不是在稍作snapshot（包括构建、添加commit时间戳、删除commit时间戳）的时候，就保证了这个条件。（需要确认下，snapshot的构造方法，再回答）

### `AddCommittedTimestamps()`方法

参数是一个时间戳列表。

在构造snapshot时，就会决定该snapshot的`all_committed_before_`和`none_committed_at_or_after_`属性，但是后续仍然还会添加一些 “已提交”的时间戳，会放在`committed_timestamps_`中。

该函数，会检查并更新 `none_committed_at_or_after_`（这是必需的，为了能够让`none_committed_at_or_after_`维持在正确的状态，在有新事物的提交时间戳 大于当前的`none_committed_at_or_after_`值时，是必须要 提升 该值的）。

使用场景：  
在`flush`的时候使用。  
一个“已提交”的事务集合被刷入到文件中时，在MVCC的角度看，不再是一致的snapshot。 因此，需要将这些 “已提交”的事务，添加到 snapshot中。

>> 问题： 这个“使用场景”，不是很明白？ 具体是什么场景会破坏snapshot的一致性？

### `ToString()`方法

该方法只会打印 处于“已提交”状态的事务列表。

### `Enquals()`方法

比较两个`MvccSnapshot`对象。 只有当它们所有的“3个关于时间的成员”都完全相同时，才会认为是相等的。

注意：其中，`committed_timestamps`是一个`vector`类型，这个方法要求这个vector的所有成员都相等（即顺序也要相等），才会认为是相等的。

## `private enum MvccManager::TxnState`

枚举类型，就两个值。

事务的开始状态是`RESERVER`，在准备提交时，需要先切换为`APPLYING`，然后才能进行提交。

```
  enum TxnState {
    RESERVED,
    APPLYING
  };
```

## `private enum MvccManager::WaitFor`

表示 等待的类型。 
1. 等待，直到所有事务都被结束（都被提交，或都被回滚）。
2. 等待，直到没有处于`APPLYING`状态的事务。

```
  enum WaitFor {
    ALL_COMMITTED,
    NONE_APPLYING
  };

```

## `private struct MvccManager::WaitingState`

封装一个等待信息，包括
1. 要等待的事务时间戳；
2. “信号量”（它的`wait()`和`CountDown()`方法用来触发“阻塞等待”和“信号通知”）；
3. 等待类型

```
  struct WaitingState {
    Timestamp timestamp;
    CountDownLatch* latch;
    WaitFor wait_for;
  };
```
## `class MvccManager`

所有"MVCC txns"的协调者。  

每个tablet都有唯一的`MvccManager`对象，用来管理当前`tablet`的MVCC状态。

如果希望进行某些修改操作的线程，都会使用该`MvccManager`去获取一个“唯一的时间戳”，通常是通过`ScopedTransaction`类。

**使用`MVCC`是为了将修改延迟到`commit_time`，并且允许 迭代器(`iterator`)可以在一个 只含有“已提交”事务的 snapshot上进行操作。**  

对于一个"mvcc txn"，有两个有效的路径：
1. `StartTransaction()` -> `StartApplyingTransaction()` -> `CommitTransaction()`;
2. `StartTransaction()` -> `AbortTransaction()`;

当一个"mvcc txn"准备修改内存结构时，它应该调用`StartApplyingTransaction()`（把状态修改为`APPLYING`状态），
然后，这个"mvcc txn"可以进行内存结构的修改，并且在 有限的时间内进行commit。

“有限的时间”是指：它不应该再等待外部的输入，比如等待一个RPC。

**注意：**    
无法对内存结构的修改进行回滚（`rollback`）。所以，一旦调用了`StartApplyingTransaction()`，那么这个事务就最终必须提交。

### 成员

#### **`timestamps_in_flight_`**  
记录当前正在运行的事务列表。

是一个`unorder_map`类型，其中key是事务的时间戳（也是事务id），value是事务的状态（`enum TxnState`类型）。

#### **`safe_time_`**  
一个时间戳，后续到达的事务，时间戳一定大于它（不能“小于或等于”）。

初始值为`Timestamp::kMin`。

注意：它**只是限制后续达到的事务的时间戳，而没有限制已经达到的事务，更没有对状态做出任何限制**。

即当前已经到达的事务中：
1. 时间戳可以小于该“时间戳”；
2. 并且 事务的状态可以为“已提交”，也可以“正在运行”。

#### **`earliest_in_flight_`**  
记录在`timestamps_in_flight_`中的最小时间。 如果`timestamps_in_flight_`为空，那么该变量值为`Timestamp::kMax`。  

单独记录该值的目的：  
可以避免在每次commit的时候，都遍历整个`timestamps_in_flight_`。

也就是说，这个变量，就是为了实现上的高效率，进行的一个缓存。

既然是缓存，那么就牵扯到更新它的问题。

**有两个地方会对`earliest_in_flight_`进行更新**：
1. start一个新事务的时候：如果一个新事务的时间戳，小于该值，那么在将新的时间戳插入到`timestamps_in_flight_`的同时，要更新该值。
2. commit和abort一个事务的时候。 如果对应事务的时间戳，正好等于当前的`earliest_in_flight_`，那么就要更新该变量。注意：这时进行的更新，需要遍历`timestamps_in_flight_`列表，因为只有全部遍历，才能找到一个新的最小的值。

#### **`waiters_`**

有的时候，会给定一个时间戳，并且给定一个等待类型，然后进行等待。

这些“等待”，会被添加到这个`waiters_`成员中。

要主动通知这些`waiters_`的场景：
1. 关闭当前`MvccManager`对象的时候；
2. 修改`all_committed_before_`的时候；

#### **`open_`**  
用来标记当前`MvccManager`是否为“开启”状态。

```
  typedef simple_spinlock LockType;
  mutable LockType lock_;

  MvccSnapshot cur_snap_;

  typedef std::unordered_map<Timestamp::val_type, TxnState> InFlightMap;
  InFlightMap timestamps_in_flight_;

  Timestamp safe_time_;

  // The minimum timestamp in timestamps_in_flight_, or Timestamp::kMax
  // if that set is empty. This is cached in order to avoid having to iterate
  // over timestamps_in_flight_ on every commit.
  Timestamp earliest_in_flight_;

  mutable std::vector<WaitingState*> waiters_;

  std::atomic<bool> open_;
```

### 事务流程相关方法

对于一个"mvcc txn"，有两个有效的路径：
1. `StartTransaction()` -> `StartApplyingTransaction()` -> `CommitTransaction()`;
2. `StartTransaction()` -> `AbortTransaction()`;

#### `StartTransaction()`

参数是一个时间戳，表示开启一个事务（参数就是该事务的时间戳）。

要求：  
1. 所给定的时间戳（对应的事务），是“未提交的”；
2. 所给定的时间戳，必须 大于 `safe_time_`;（也不能 等于）；

```

void MvccManager::StartTransaction(Timestamp timestamp) {
  MAYBE_INJECT_RANDOM_LATENCY(FLAGS_inject_latency_ms_before_starting_txn);
  std::lock_guard<LockType> l(lock_);
  CHECK(!cur_snap_.IsCommitted(timestamp)) << "Trying to start a new txn at an already-committed"
                                           << " timestamp: " << timestamp.ToString()
                                           << " cur_snap_: " << cur_snap_.ToString();
  CHECK(InitTransactionUnlocked(timestamp)) << "There is already a transaction with timestamp: "
                                            << timestamp.value() << " in flight or this timestamp "
                                            << "is before than or equal to \"safe\" time."
                                            << " Current Snapshot: " << cur_snap_.ToString()
                                            << " Current safe time: " << safe_time_;
}
```

说明：  
`private InitTransactionUnlocked()`方法，就是将 参数的时间戳，添加到 `timestamps_in_flight_`中。 

在添加之前，会检查下是否需要更新`earliest_in_flight_`（如果该时间戳 小于 `earliest_in_flight_`）。

#### `StartApplyingTransaction()`  

参数中传入一个“时间戳”，该方法标记 该时间戳（所对应的事务）要开始进行提交了。

标识方法是： 将对应事务的状态，修改为`APPLYING`。

注意：  
1. 该方法一定要在`CommitTransaction()`之前调用；
2. 在调用完该方法后，一定不能再调用`AbortTransaction(timestamp)`；

```
void MvccManager::StartApplyingTransaction(Timestamp timestamp) {
  std::lock_guard<LockType> l(lock_);
  auto it = timestamps_in_flight_.find(timestamp.value());
  if (PREDICT_FALSE(it == timestamps_in_flight_.end())) {
    LOG(FATAL) << "Cannot mark timestamp " << timestamp.ToString() << " as APPLYING: "
               << "not in the in-flight map.";
  }

  TxnState cur_state = it->second;
  if (PREDICT_FALSE(cur_state != RESERVED)) {
    LOG(FATAL) << "Cannot mark timestamp " << timestamp.ToString() << " as APPLYING: "
               << "wrong state: " << cur_state;
  }

  it->second = APPLYING;
}
```

#### `CommitTransaction()`

参数中传入一个“时间戳”，该方法 用来提交 该时间戳（所对应的事务）。

注意：  
1. 该时间戳 必须在`timestamps_in_flight`中。 如果不在，有两种情况：
  + 该事务可能还没有被start，这是不允许的；
  + 该事务可能已经被commit或者abort过（因为commit或abort时会从`timestamps_in_flight`中删除），这次是再次提交。   
  **一个事务只能被commit或abort一次。不允许将一个事务 1) 重复commit; 2)重复abort; 3) 既commit又abort。**；
2. 该方法会保证，将当前时间戳从`timestamps_in_flight`中删除掉。
3. 该时间戳（对应事务）的状态，必须是`APPLYING`。也就是已经调用过`StartApplyingTransaction()`。 否则，会直接打印`FATAL`日志并退出进程。
4. 如果当前时间戳等于`earliest_in_flight_`，那么需要更新下`earliest_in_flight_`。

>> 问题： `StartApplyingTransaction()`只是简单的修改一下状态，而且是一个内存状态，为什么一定要存在？ 比如宕机重启后这个内存状态也是会丢失的。

```
void MvccManager::CommitTransaction(Timestamp timestamp) {
  std::lock_guard<LockType> l(lock_);

  // Commit the transaction, but do not adjust 'all_committed_before_', that will
  // be done with a separate AdjustSafeTime() call.
  bool was_earliest = false;
  CommitTransactionUnlocked(timestamp, &was_earliest);

  if (was_earliest && safe_time_ >= timestamp) {
    // If this transaction was the earliest in-flight, we might have to adjust
    // the "clean" timestamp.
    AdjustCleanTime();
  }
}
```

说明1：  
`CommitTransactionUnlocked()`的工作内容：
1. 将当前时间戳 从`timestamps_in_flight_`中移除；
2. 并将 当前时间戳，作为 commit timestamp， 添加到 当前tablet的snapshot中。
3. 检查并更新`earliest_in_flight_`。

注意：不是每次commit都需要更新`earliest_in_flight_`：  
只有当本次提交的时间戳，就是当前的`earliest_in_flight_`时，需要进行更新。（这是很正常的，因为`earliest_in_flight_`已经被提交了，所以要对它进行更新）

另外，如果当前提交事务的时间戳，就是`earliest_in_flight_`，那么还需要检查是否需要更新`clean time`（即当前snapshot的`all_committed_before_`）。

是否需要更新`clean time`的判断依据：当前的`earliest_in_flight_`是否小于`safe_time_`。如果当前事务的时间戳（即`earliest_in_flight_`）是小于`safe_time_`的，因为后续事务的时间戳都一定大于`safe_time_`，所以后续到来的事务的时间戳也一定都大于`earliest_in_flight_`，那么也就是说，当前snapshot的`all_committed_before_`至少可以提升到 当前该事务的时间戳。

更新`clean time`的具体内容，参见`AdjustCleanTime()`。

注意：因为在`CommitTransactionUnlocked()`中，已经提升了当前的`earliest_in_flight_`，所以在`AdjustCleanTime()`中，看到的`earliest_in_flight_`已经是被提升过的时间戳了。

#### `AbortTransaction()`
参数中传入一个“时间戳”，该方法 用来 **回滚掉** 该时间戳（所对应的事务）

注意： 
1. 和`CommitTransaction()`一样的，该方法也要求：  
   1) 该时间戳 必须在`timestamps_in_flight`中。 如果不在，有两种情况：
      + 该事务可能还没有被start，这是不允许的；
      + 那么该事务可能已经被commit或者abort过（因为commit或abort时会从`timestamps_in_flight`中删除），这次是再次提交。    
    **一个事务只能被commit或abort一次。不允许将一个事务 1) 重复commit; 2)重复abort; 3) 既commit又abort。**；
   2) 该方法也会保证，将当前时间戳从`timestamps_in_flight`中删除掉。
   3) 如果当前时间戳等于`earliest_in_flight_`，那么需要更新下`earliest_in_flight_`。
2. 和`CommitTransaction()`不一样的是，该时间戳（对应事务）的状态，必须 **不是** `APPLYING`（即只能是初始的`RESERVED`状态）。也就是不能abort掉一个已经调用过`StartApplyingTransaction()`的事务。 否则，会直接打印`FATAL`日志并退出进程。

```
void MvccManager::AbortTransaction(Timestamp timestamp) {
  std::lock_guard<LockType> l(lock_);

  // Remove from our in-flight list.
  TxnState old_state = RemoveInFlightAndGetStateUnlocked(timestamp);

  // If the tablet is shutting down, we can ignore the state of the
  // transactions.
  if (PREDICT_FALSE(!is_open())) {
    LOG(WARNING) << "aborting transaction with timestamp " << timestamp.ToString()
        << " in state " << old_state << "; MVCC is closed";
    return;
  }

  CHECK_EQ(old_state, RESERVED) << "transaction with timestamp " << timestamp.ToString()
                                << " cannot be aborted in state " << old_state;

  // If we're aborting the earliest transaction that was in flight,
  // update our cached value.
  if (earliest_in_flight_ == timestamp) {
    AdvanceEarliestInFlightTimestamp();
  }
}
```

见`commit_id:adfab44c04ea734831f052b8e7cb9e4abb7848d0` 的commit message。 它限定了“**一个tablet只有在被删除的时候，才会被关闭**”。  
所以这时，它不是很关心在该tablet上正在运行的事务。 它把一些原来会crash的场景，改为了不再crash，仅仅会打印一条warning日志。

包括这里的，如果一个tablet被关闭（删除）了，那么在abort时，即使它的状态不是`RESERVED`，那么就打印一条warning日志，不会crash掉进程。

### 修改时间变量的函数
#### `AdjustSafeTime()`

参数表示新的`safe_time`。该方法的作用是：调整当前的`safe_time_`，然后`MvccManager`就可以利用新的`safe_time_`清理掉内部的一些旧状态。

清理的“旧状态”，指的是要更新 snapshot中的 状态（其中的3个时间戳相关的属性），参见`AdjustCleanTime()`。

说明：
任何一个新到来的事务，它的时间戳 一定大于当前的`safe_time_`。

所以，调用这个方法后，一定要保证 任何新的事务，都不会再 小于或等于 指定的`safe_time`。

```
void MvccManager::AdjustSafeTime(Timestamp safe_time) {
  std::lock_guard<LockType> l(lock_);
  // No more transactions will start with a ts that is lower than or equal
  // to 'safe_time', so we adjust the snapshot accordingly.
  if (PREDICT_TRUE(safe_time_ <= safe_time)) {
    DVLOG(4) << "Adjusting safe time to: " << safe_time;
    safe_time_ = safe_time;
  } else {
    // Note: Getting here means that we are about to apply a transaction out of
    // order. This out-of-order applying is only safe because concurrrent
    // transactions are guaranteed to not affect the same state based on locks
    // taken before starting the transaction (e.g. row locks, schema locks).
    KLOG_EVERY_N(INFO, 10) << Substitute("Tried to move safe_time back from $0 to $1. "
                                         "Current Snapshot: $2", safe_time_.ToString(),
                                         safe_time.ToString(), cur_snap_.ToString());
    return;
  }

  AdjustCleanTime();
}
```

>> 问题： 这个safe_time在什么场景下会倒退？

#### `private AdjustCleanTime()`

**"clean time"的含义：** 就是snapshot中的`all_committed_before_`。 

这个时间戳的用途是：用来标识 所有小于该时间戳的事务，状态都是确定的（"commtted"或"aborted"）。

注意：在commit和abort事务的时候，进入该方法的条件，都包含 对应事务的时间戳，是当前snapshot的`earliest_in_flight_`，但是在进入该方法前，都已经将该值提升过了（比如：在`CommitTransactionUnlocked()`中，已经提升了当前的`earliest_in_flight_`），所以在`AdjustCleanTime()`中，看到的`earliest_in_flight_`已经是被提升过的时间戳了。

**根据什么进行调整：**  
1. 当前 处于 flight状态的所有事务的时间戳（主要是`earliest_in_flight_`）；
2. 当前的`safe_time_`

调整的内容：
1. 调整`all_committed_before_`；
2. 将`committed_timestamps_`中小于 新`all_committed_before_` 的时间戳删除；
3. 调整`none_committed_at_or_after_`。 （当清理后的`committed_timestamps_`为空，说明这时没有正在提交的事务，这时对`none_committed_at_or_after_`，从而可以提高该snapshot对一些场景的判断效率）；

当然，因为调整了snapshot的状态，所以可能需要激活一些snapshot上的“等待者”。

**调整`all_committed_before_`可能遇到的两种场景**：  
1. 当前的`earliest_in_flight_` 小于 `safe_time_`，即当前有正在运行的事务，并且时间戳小于`safe_time_`。 这时，应该将`all_committed_before_`（低水位）更新为 `earliest_in_flight_`。
2. 否则,（即当前的`earliest_in_flight_` 大于等于 `safe_time_`)，因为任何后续达到的事务，时间戳一定大于`safe_time_`（但是仍然可能是小于当前的`earliest_in_flight_`），所以这时，只能将将`all_committed_before_`（低水位）更新为 `safe_time_`）。

```
void MvccManager::AdjustCleanTime() {
  // There are two possibilities:
  //
  // 1) We still have an in-flight transaction earlier than 'safe_time_'.
  //    In this case, we update the watermark to that transaction's timestamp.
  //
  // 2) There are no in-flight transactions earlier than 'safe_time_'.
  //    (There may still be in-flight transactions with future timestamps due to
  //    commit-wait transactions which start in the future). In this case, we update
  //    the watermark to 'safe_time_', since we know that no new
  //    transactions can start with an earlier timestamp.
  //
  // In either case, we have to add the newly committed ts only if it remains higher
  // than the new watermark.

  if (earliest_in_flight_ < safe_time_) {
    cur_snap_.all_committed_before_ = earliest_in_flight_;
  } else {
    cur_snap_.all_committed_before_ = safe_time_;
  }

  DVLOG(4) << "Adjusted clean time to: " << cur_snap_.all_committed_before_;

  // Filter out any committed timestamps that now fall below the watermark
  FilterTimestamps(&cur_snap_.committed_timestamps_, cur_snap_.all_committed_before_.value());

  // If the current snapshot doesn't have any committed timestamps, then make sure we still
  // advance the 'none_committed_at_or_after_' watermark so that it never falls below
  // 'all_committed_before_'.
  if (cur_snap_.committed_timestamps_.empty()) {
    cur_snap_.none_committed_at_or_after_ = cur_snap_.all_committed_before_;
  }

  // it may also have unblocked some waiters.
  // Check if someone is waiting for transactions to be committed.
  if (PREDICT_FALSE(!waiters_.empty())) {
    auto iter = waiters_.begin();
    while (iter != waiters_.end()) {
      WaitingState* waiter = *iter;
      if (IsDoneWaitingUnlocked(*waiter)) {
        iter = waiters_.erase(iter);
        waiter->latch->CountDown();
        continue;
      }
      iter++;
    }
  }
}
```

#### `private AdvanceEarliestInFlightTimestamp()`

根据当前的正在进行的事务列表，调整`timestamps_in_flight_`的值。

这里更新`earliest_in_flight_`的方法：
1. 如果`timestamps_in_flight_`为空，即当前已经没有正在运行的事务，那么将`earliest_in_flight_`重置为`kMax`（初始值）；
2. 否则，需要遍历下`timestamps_in_flight_`列表，找到最小的时间戳，并赋值给`earliest_in_flight_`；

### 创建新的snapshot

#### `TakeSnapshot()`

将该tablet当前的状态，创建一个snapshot，通过参数返回。

通过这个snapshot，可以访问当前已经提交的事务。

```
void MvccManager::TakeSnapshot(MvccSnapshot *snap) const {
  std::lock_guard<LockType> l(lock_);
  *snap = cur_snap_;
}
```

#### `WaitForSnapshotWithAllCommitted()`  

给定一个时间戳，返回一个snapshot(通过指针参数)，满足在该snapshot中，所有小于该“时间戳”的事务，状态都是确定的(committed或aborted)。

**注意：该方法可能需要等待**，详见`WaitUntil()`。

#### `WaitForApplyingTransactionsToCommit()`

等待，直到当前所有处于`APPLYING`状态的事务都结束。

逻辑：
1. 遍历当前的`timestamps_in_flight_`，找到当前处于`APPLYING`状态的最大的时间戳，构造出一个`WaitingState`对象；
2. 以`NONE_APPLYING`的等待形式，等待该时间戳结束；

**注意：**  
即使这个方法正确返回后，如果再去检查所有当前正在运行的事务的状态，依然可能会有 一些时间戳小于指定时间戳的事务，状态为`APPLYING`。因为这个方法，只保证在调用的那个时间点，处于`APPLYING`状态的事务，在函数返回后，已经结束。   

但是有可能在等待的过程中，有新增的APPLYING状态的事务，这些新增的`APPLYING`状态的事务，是不会被等待的。

#### `private WaitUntil()`

给定一个“时间戳”和“deadline”，等待"该时间戳之前的"所有事务，都处于特定的状态，有最长阻塞时间限制（最长不超过`deadline`）。

`WaitFor`决定了等待的最终状态：1. 全结束；2. 都不处于`APPLYING`状态。

**返回的条件：**

如果等待类型是`ALL_COMMITTED`: （参见`AreAllTransactionsCommittedUnlocked()`）
1. 给定的时间戳 小于 `all_committed_before_`; 因为这种场景，所有的事务的状态一定都是确定的了。
2. 给定的时间戳，小于 `earliest_in_flight_`。 

如果等待类型是`NONE_APPLYING`: （参见`AnyApplyingAtOrBeforeUnlocked()`）
要等到`timestamps_in_flight_`中的所有时间戳，都是大于指定时间戳的。

注意：`AnyApplyingAtOrBeforeUnlocked()`比较的是“小于或等于”指定的时间戳。在实际使用的时候，在该方法的结果上取反。即`!AnyApplyingAtOrBeforeUnlocked()`。

`AnyApplyingAtOrBeforeUnlocked()`的实现方法：即依次判断所有正在进行的事务，如果任何一个时间戳 小于或等于 指定时间戳，那么返回true。

>> 说明：目前这块的实现，仅仅比较了时间戳大小，并没有比较`APPLYING`状态，所以这里可能需要修改。待理清楚后，提交个PR。

### 总结

>> 问题： 本文件中，有很多时间戳，这些时间戳 在发生什么事件时 会进行更新？ 更新的规则是？ 有规律吗？









