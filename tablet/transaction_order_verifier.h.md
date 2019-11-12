[TOC]

文件：`src/kudu/tablet/transaction_order_verifier.h`

## 概述

每个`TabletReplica`对象都有一个`TransactionOrderVerifier`成员。

用来验证：
1. 多个操作是否都是按照 index递增 的顺序 进行`apply`的；
2. 是否都是以相同的顺序，进入`Prepare()`阶段的；

说明：这个类本质上是一个“错误检查类”，也就是说，它实际上是可以不存在的。

但是目前，即便是在`RELEASE`模式下，也是开启的。如果发现它成为瓶颈，那么可以将它修改为仅在`DEBUG`模式下才打开。

>> extraordinarily: 极端地；极其；

**这个类不是“线程安全”的，** 线程间互斥是在外部 来保证的（见下面的分析）。

该类仅仅提供一个接口: `CheckApply()`。

总是在`PREPARE`和`REPLICATE`阶段结束后，才会调用该类（进行检查）。

### 线程同步时的注意事项

可能会有多个线程同时调用该类进行检查。

比如说：
+ prepare pool中的一个线程（`RPC handler`），正在处理一个tablet上的多个`UpdateConsensus()`时；
+ 另外一个线程，当`leader`收到了来自（同一个raft组的）replica的回复消息时；

但是可以保证的是，一定不会有多个线程，同时进入`CheckApply()`方法。

**为什么可以做到互斥：**  
1. `CheckApply(N)`只会在`Prepare(N)`和`Replicate(N)`都完成以后，才会被执行。  
    并且，执行`CheckApply(N)`的线程，要么是执行`Prepare(N)`的线程，要么是执行`Replicate(N)`的线程。
2. `Prepare(N-1)`的完成，一定是先于`Prepare(N)`的，因为`Prepare`是在单个线程内做的。
3. `Replicate(N-1)`的完成，一定是先于`Replicate(N)`的，这时由`raft的协议`保证的。

也就是说：只要`Prepare(N-1)`和`Replicate(N-1)`都完成，那么`CheckApply(N-1)`就完成了。

所以：`CheckApply(N-1)`不会和`CheckApply(N)`同时存在。

>> 问题：这里如何保证不会并行的，还不是太清楚？能否画个图说明下？

>> 问题：这里提到`Prepare()`是单线程的？ 这个是怎么做到的？


### 成员



### `CheckApply()`方法

给定一个`raft index`和“prepare时间戳”，验证是否满足：
1. `index`必须是单调递增的，同时也是“连续”的（“连续”的含义是：不能有空洞）；
2. `prepare`时间戳 必须是单调递增的。

注意1：这里“prepare时间戳”，是当前机器本地的`monotonic timestamp`，并不是`kudu tiemstamp`。 因为这里，只要求要验证的时间满足“在本地是单调递增”即可，所以使用“local real time”即可。

注意2：如果检查不过，进程会直接crash。
























