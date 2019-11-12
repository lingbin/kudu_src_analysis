[TOC]

文件: `src/kudu/tablet/transactions/transaction_tracker.h`

## 概述

每个`TabletReplica`对象都有一个`TransactionTracker`成员，用来跟踪记录“正在执行”的事务。

每个`LeaderTransaction`， 通过`Add()`方法将 事务 注册进来；通过`Release()`方法取消事务的注册。

1. 在调用`TransactionDriver::Init()`的时候，会进行注册。
2. 在`TransactionDriver::Finalize()`和`TransactionDriver::HandlerFailure()`中，进行取消注册。


>> 问题：为什么要记录正在运行的事务？
>> 答：目前主要有几个作用：
>> 1. 进行一些metric统计；
>> 2. 限制 该tablet上的事务，运行时占用的总内存；
>> 3. 目前的实现，还包括了`TransactionDriver`对象的析构。

## `private struct TransactionTracker::Metrics`

封装了一些metric信息（事务有两种：`write_txn`和`alter_schema_txn`）：
1. `all_transactions_inflight`: 所有正在进行的事务数量；
2. `write_transactions_inflight`：正在进行的“写事务”的数量；
3. `alter_schema_transactions_inflight`：正在进行的“alter_schema”事务数量；
4. `transaction_memory_pressure_rejections`：因为系统压力大，被拒绝的事务数量；

注意前3个是 `GAUGE`类型；最后1个是`COUNTER` 类型。

```
struct Metrics {
    explicit Metrics(const scoped_refptr<MetricEntity>& entity);

    scoped_refptr<AtomicGauge<uint64_t> > all_transactions_inflight;
    scoped_refptr<AtomicGauge<uint64_t> > write_transactions_inflight;
    scoped_refptr<AtomicGauge<uint64_t> > alter_schema_transactions_inflight;

    scoped_refptr<Counter> transaction_memory_pressure_rejections;
  };
```

## `private struct TransactionTracker::State`

每个事务对应的一些信息，目前只有“内存占用”信息。

因为这个类(`TransactionTracker`)就是跟踪记录一个replica上正在进行的事务，所以一定有一个map成员来保存各个事务。 

**在这个map中，key是事务所对应的`TransactionDriver`对象，value是这个`State`对象**。

```
 // Per-transaction state that is tracked along with the transaction itself.
  struct State {
    State();

    // Approximate memory footprint of the transaction.
    int64_t memory_footprint;
  };
```

## `class TransactionTracker`类

### 成员

```
mutable simple_spinlock lock_;

// Protected by 'lock_'.
  typedef std::unordered_map<scoped_refptr<TransactionDriver>,
      State,
      ScopedRefPtrHashFunctor<TransactionDriver>,
      ScopedRefPtrEqualToFunctor<TransactionDriver> > TxnMap;
  TxnMap pending_txns_;

  gscoped_ptr<Metrics> metrics_;

  std::shared_ptr<MemTracker> mem_tracker_;
```

### `StartInstrumentation()`

因为每个replica对象都有一个`TransactionTracker`对象。如果系统开启了`metric`统计（replica本身也有一些`metric`信息），那么就会在当前 tracker对象中也要开启`metric`统计。  
打开metric统计的方法就是调用该方法`StartInstrumentation()`。

备注：目前实现是，在`TabletReplica::Start()`方法中，调用该方法。

### `StartMemoryTracking()`

和“metric统计”类似，通过该方法来开启对当前tablet的事务进行内存跟踪。

注意，和“metric统计”不同的是，只有在系统开启“metric统计”统计的时候，才会在tracker对象中也进行metric统计。但是 “事务占用的内存跟踪”是一定会开启的。

**相关的`GFLAG`是`FLAGS_tablet_transaction_memory_limit_mb`，默认大小是`64MB`。**  
+ 该值限制了一个tablet上，正在运行的事务占用的总内存大小。
+ 如果超过该值，那么新收到的事务会被拒绝（客户端会重试）。
+ 如果该值为 `-1`，那么表示不进行限制。

**限制内存的原理是：**  
通过创建1个`MemTracker`对象（会指定一个内存的阈值），然后在处理事务的时候，每次都调用`TryConsume()`来进行判断。如果返回`false`，则说明内存占用（所有正在进行的所有事务 所占用的）超过阈值，那么就拒绝当前请求。

>> 备注：  
>> 创建`MemTracker`时，如果传入的阈值为负数，那么就表示该`MemTracker`只进行内存跟踪，但不限制内存（即所有的`TryConsume()`都会返回`true`）。所以`FLAGS_tablet_transaction_memory_limit_mb == -1`时，当前Tracker就不会因为 内存占用太大 而拒绝请求。

### `Add()`方法

注册一个事务。

因为当前主要关注的是内存占用，所以在 当前方法中，也主要是对内存占用进行 跟踪和判断。  

如果内存占用超过了阈值，那么会 注册失败（请求会被拒绝，由客户端来进行重试）

**统计的内容**  
当前，对于每个写事务（对应一个`WriteRequest`的PB结构），只会统计它说对应的`WriteRequest`结构的大小（使用`ProtoBuf`的 `Message::SpaceUsed()`方法）

>> 问题：在protobuf 3.4.1 以后，`SpaceUsed()`方法已经是“过时的”，建议使用`SpaceUsedLong()`替换。  
>> 参见：https://abi-laboratory.pro/?view=changelog&l=protobuf&v=3.4.1 

>> 问题：为什么在protobuf的文档中说，`SpaceUsed()`比`ByteSize()`要慢，为什么Kudu中却没有使用`ByteSize()`? 为什么 protobuf要同时提供这两个接口，是不是各自有一些使用场景？（因为如果一个在任何场景都比另一个好，那么绝对没有理由再提供另外一个）  
>> 参见：https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.message#Message.SpaceUsed.details

```
Status TransactionTracker::Add(TransactionDriver* driver) {
  int64_t driver_mem_footprint = driver->state()->request()->SpaceUsed();
  if (mem_tracker_ && !mem_tracker_->TryConsume(driver_mem_footprint)) {
    if (metrics_) {
      metrics_->transaction_memory_pressure_rejections->Increment();
    }

    // May be null in unit tests.
    TabletReplica* replica = driver->state()->tablet_replica();

    string msg = Substitute(
        "transaction on tablet $0 rejected due to memory pressure: the memory "
        "usage of this transaction ($1) plus the current consumption ($2) "
        "exceeds the transaction memory limit ($3) or the limit of an ancestral "
        "memory tracker.",
        replica ? replica->tablet()->tablet_id() : "(unknown)",
        driver_mem_footprint, mem_tracker_->consumption(), mem_tracker_->limit());

    KLOG_EVERY_N_SECS(WARNING, 1) << msg << THROTTLE_MSG;

    return Status::ServiceUnavailable(msg);
  }

  IncrementCounters(*driver);

  // Cache the transaction memory footprint so we needn't refer to the request
  // again, as it may disappear between now and then.
  State st;
  st.memory_footprint = driver_mem_footprint;
  std::lock_guard<simple_spinlock> l(lock_);
  InsertOrDie(&pending_txns_, driver, st);
  return Status::OK();
}
```

### `Release()`方法

将当前事务移除，并清理对应的内存跟踪。

**注意：可能会触发对`TransactionDriver`对象的析构**

因为在map中，保存的是`scoped_refptr<TransactionDriver>`，如果从map中移除，该对象的引用计数可能会被减为0（就会触发对`TransactionDriver`对象的析构）。

>> 问题：这里从map中删除的时候，是加锁的。如果`TransactionDriver`的析构很费，那么这里就需要将进行下优化，使析构过程在加锁之外进行。  
>> 最简单的方法是，先重新拷贝`scoped_refptr`出来（即引用计数不会为0），然后在锁外进行释放。 

### `GetPendingTransactions()`方法

获取所有正在运行的事务列表。通过参数获取到 事务列表。

```
void GetPendingTransactions(std::vector<scoped_refptr<TransactionDriver> >* pending_out) const;
```

注意：因为获取的结果会放在参数中，参数是`scoped_refptr<TransactionDriver>`的数组，也就是说，对于这些`TransactionDriver`,如果作为返回结果的一部分，那么**它们的引用计数就会被增加**。  所以，即使它们在添加到这个vector中以后，立即被从map中删除，也不会触发这些`TransactionDriver`对象的析构。

### `WaitForAllToFinish()`方法

这里有个实现技巧：因为要等待的事务非常多，而且要做到两点：
1. 有一个超时时间，用来防止无限等待，如果超过该时间，就不再等待。
2. 希望在等待的过程中，不定期的打印出来相应的事务。这样不会误以为进程hang住了。

```
Status TransactionTracker::WaitForAllToFinish(const MonoDelta& timeout) const {
  static constexpr size_t kMaxTxnsToPrint = 50;
  int wait_time_us = 250;
  int num_complaints = 0;
  MonoTime start_time = MonoTime::Now();
  MonoTime next_log_time = start_time + MonoDelta::FromSeconds(1);

  while (1) {
    vector<scoped_refptr<TransactionDriver> > txns;
    GetPendingTransactions(&txns);

    if (txns.empty()) {
      break;
    }

    MonoTime now = MonoTime::Now();
    MonoDelta diff = now - start_time;
    if (diff > timeout) {
      return Status::TimedOut(Substitute("Timed out waiting for all transactions to finish. "
                                         "$0 transactions pending. Waited for $1",
                                         txns.size(), diff.ToString()));
    }
    if (now > next_log_time) {
      LOG(WARNING) << Substitute("TransactionTracker waiting for $0 outstanding transactions to"
                                 " complete now for $1", txns.size(), diff.ToString());
      LOG(INFO) << Substitute("Dumping up to $0 currently running transactions: ",
                              kMaxTxnsToPrint);
      const auto num_txn_limit = std::min(txns.size(), kMaxTxnsToPrint);
      for (auto i = 0; i < num_txn_limit; i++) {
        LOG(INFO) << txns[i]->ToString();
      }

      num_complaints++;
      // Exponential back-off on how often the transactions are dumped.
      next_log_time = now + MonoDelta::FromSeconds(1 << std::min(8, num_complaints));
    }
    wait_time_us = std::min(wait_time_us * 5 / 4, 1000000);
    SleepFor(MonoDelta::FromMicroseconds(wait_time_us));
  }
  return Status::OK();
}
```

说明：
1. 为了防止事务太多，全部打印没有必要。所以这里限制了，每次打印最多打印50个。
2. 因为每个事务的结束时间不一样，那么在一轮检测结束的时候，如果仍然存在 没有结束的事务，并且目前仍然没有超时，那么需要进行`sleep`一段时间，然后再重新进行检测。这里`sleep`的时间是逐渐递增的，最大为1秒。 起始的时间为默认是`250us`，每轮增加`1/4`.
3. 不需要在每轮检测的时候，都打印当前的事务列表。打印的时间间隔是指数级别递增的（根据不同的‘重试次数’(`num_complaints`)，重试次数越高，间隔越长）。但是设置了最大的重试间隔，即`1 << 8`s，即 `256秒`。第1次为1秒，以后的间隔是`1 << num_complaints`秒。
