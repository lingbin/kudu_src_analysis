[TOC]

文件: `src/kudu/util/throttler.h`

逻辑比较简单，限制每个tablet的写入压力，**只针对“写入”，不限制“读取”**。

一些基本介绍，参见 commit_id: f8a677b1fea197bb88672fb944e89b39b1b80829 

默认情况下，是不进行限制的。

## 成员

```
  MonoTime next_refill_;  
  uint64_t op_refill_;
  uint64_t op_token_;
  uint64_t op_token_max_;
  uint64_t byte_refill_;
  uint64_t byte_token_;
  uint64_t byte_token_max_;
  simple_spinlock lock_;
```

`next_refill_`: 下次进行重置的时间；

`op_refill_`： 根据指定的QPS阈值，在指定的重置周期内（默认是100ms），允许的最大请求数目；

`op_token_`：剩余的token数。（注意：这个数是会累积的，即上一个周期没有消耗完的，会累积到下一个周期。最大不超过 `op_token_max_`）；

`op_token_max_`: 每个重置周期内，允许的最大的token数。 `op_refill_ * burst_rate`。 

`byte_refill_`：根据指定的“数据大小”阈值，在指定的重置周期内（默认是100ms），允许的最大数据量；

`byte_token_`：剩余的token数。（注意：这个数是会累积的，即上一个周期没有消耗完的，会累积到下一个周期。最大不超过 `byte_token_max_`）；

`byte_token_max_`: 每个重置周期内，允许的最大的数据量。 `byte_refill_ * burst_rate`。 

## 构造函数

有两个维度进行限流（可以同时指定，也可以只限定 其中一个）：
1. 写入请求的`QPS`，对应`FLAGS_tablet_throttler_rpc_per_sec`
2. 每秒写入的`数据大小`，对应`FLAGS_tablet_throttler_bytes_per_sec`。

如果`op_per_sec`或`byte_per_sec`的值为0，表示不按照对应的维度进行限流。

**这里可以指定“突发流量”**  
对应`FLAGS_tablet_throttler_burst_factor`，默认值为`1.0f`。 

注意：如果要设置`FLAGS_tablet_throttler_burst_factor`，它的值应该是 大于 1 的。

```
  Throttler(MonoTime now, uint64_t op_per_sec, uint64_t byte_per_sec, double burst_factor);
```

实现如下：

```
Throttler::Throttler(MonoTime now, uint64_t op_rate, uint64_t byte_rate, double burst_factor) :
    next_refill_(now) {
  op_refill_ = op_rate / (MonoTime::kMicrosecondsPerSecond / kRefillPeriodMicros);
  op_token_ = 0;
  op_token_max_ = static_cast<uint64_t>(op_refill_ * burst_factor);
  byte_refill_ = byte_rate / (MonoTime::kMicrosecondsPerSecond / kRefillPeriodMicros);
  byte_token_ = 0;
  byte_token_max_ = static_cast<uint64_t>(byte_refill_ * burst_factor);
}
```

注意1：虽然目前`kRefillPeriodMicros`是在代码中硬编码的（值为`10^5`），但是如果要修改`kRefillPeriodMicros`的值，一定要小心。  
因为根据上面的代码计算`op_refill_`的部分，因为`(MonoTime::kMicrosecondsPerSecond / kRefillPeriodMicros)`作为分母，**所以要求 `kRefillPeriodMicros` 的值一定要 小于 `MonoTime::kMicrosecondsPerSecond`**(否则，分母会为0)。也就是`kRefillPeriodMicros`的值一定要小于`10^6`。

也就是说：`kRefillPeriodMicros`的取值范围是`[0, 10^6]`;

## `refill()`方法

即对于流量（QPS和数据量）的计算，每隔这么一段时间，就重置一次计数器。

进行重置内部的计数器，默认是`每100ms`重置一次。

```
void Throttler::Refill(MonoTime now) {
  int64_t d = (now - next_refill_).ToMicroseconds();
  if (d < 0) {
    return;
  }
  uint64_t num_period = d / kRefillPeriodMicros + 1;
  next_refill_ += MonoDelta::FromMicroseconds(num_period * kRefillPeriodMicros);
  op_token_ += num_period * op_refill_;
  op_token_ = std::min(op_token_, op_token_max_);
  byte_token_ += num_period * byte_refill_;
  byte_token_ = std::min(byte_token_, byte_token_max_);
}
```

注意：**`op_token_`和`byte_token_`是累积的**。  
即如果连续几个周期内，流量都很小，那么在下次更新的时候，是会被累积起来的。不过最大不超过相应的 `token_max_`值。

正是因为累积的，所以可以用来应对突发流量。也就是说，如果一段时间内流量都比较小，那么最大可以到达的流量上限值（`rate` * `burst_factor`)。  
但是如果一段流量很大，已经达到了“突发流量上限”（即前面累积的值已经被消耗完了），那么接下来的 重置周期内，最大流量是不能超过默认的流量上限（即`op_token`和`byte_token`）的。

### 单测-突发流量

```
TEST_F(ThrottlerTest, TestBurst) {
  // Check IO rate throttling
  MonoTime now = MonoTime::Now();
  Throttler t0(now, 2000, 1000*1000, 5);
  // Fill up bucket
  now += MonoDelta::FromMilliseconds(2000);
  for (int i = 0; i < 100; i++) {
    now += MonoDelta::FromMilliseconds(1);
    ASSERT_TRUE(t0.Take(now, 1, 5000));
  }
  ASSERT_TRUE(t0.Take(now, 1, 100000));
  ASSERT_FALSE(t0.Take(now, 1, 1));
}
```

这里是通过触发“数据大小”的阈值。

初始化的时候，每秒最大为 1000*1000 = 1MB，突发流量系数为5，所以最大突发流量（经过累积）为5MB。

因为100ms更新一次，所以100ms内的正常的阈值为100KB，突发流量最大值为 0.5MB = 500KB。

因为在开始循环前，增加了2s（即20个重置周期），将初始化的时间记为`start`，那么循环的第一次，会将 `next_refill_`修改为`start + 2100ms`。

第一次调用`Take()`，传入的时间为`start + 2000ms + 1ms`;

循环的第1次，是发生在`Throttler`对象初始化后2s，所以第一次调用`Take()`时，已经累积到了最大值。

然后经过100个循环，每个循环5KB，共写入了500KB，即累积的突发流量已经写完。

循环中的最后1次，  
+ 在调用`Take()`前，`byte_token_`的值为5KB；
+ 在调用`Take()`后，会将`next_refill_`修改为 `start + 2200ms`。同时会重置 `byte_token_`，内部的`Refill()`会将`byte_token_`修改为 5KB + 100KB = 105KB，然后会再减去当前的消耗5KB，即 105KB - 5KB = 100KB。

在循环体外，第一个调用`Take()`时，就消耗了100KB。 至此，该重置周期内的数据大小的阈值消耗完毕，不能再接受新的写请求；

所以，最后一次调用`Toke()`时，虽然只消耗 1B，也是不允许的，即最后一次调用`Take()`会返回`false`。

## `take()`方法

提供一个`take()`方法，传入 当前操作的`op数`和“数据大小”，用来判断是否需要对该请求进行限流。

如果返回`true`，表示还有余量（没有达到阈值），不需要进行限流；

注意：在`Throttler`内部，**没有专门的线程**专门进行后台统计。  
而是在`Throttler`内部进入一个“下一次重试的时间点”，然后在每次调用`take()`的时候，通过传入“当前时间”，这样在`Throttler`内部，就可以综合使用“当前时间”和“内部记录的 下次重置时间点”，进行判断是否应该进行重置，以及是否超过了对应的阈值（如果超过，就应该进行限流，给客户端返回失败(`false`)）。

```
bool Throttler::Take(MonoTime now, uint64_t op, uint64_t byte) {
  if (op_refill_ == 0 && byte_refill_ == 0) {
    return true;
  }
  std::lock_guard<simple_spinlock> lock(lock_);
  Refill(now);
  if ((op_refill_ == 0 || op <= op_token_) &&
      (byte_refill_ == 0 || byte <= byte_token_)) {
    if (op_refill_ > 0) {
      op_token_ -= op;
    }
    if (byte_refill_ > 0) {
      byte_token_ -= byte;
    }
    return true;
  }
  return false;
}
```

>> 问题：代码中来回判断了多次，是否可以优化？  比如按照“短路优先返回”的规则，进行优化？
