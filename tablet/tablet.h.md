[TOC]

文件: `src/kudu/tablet/tablet.cc`

## `Tablet`


### 成员


### 接口列表

#### `ApplyRowOperation()`

应用("apply")一个行操作。

注意：这个“行操作”必须已经经过`prepared`。

“执行结果”会被放入到`row_op->result`中。

执行逻辑：
1. 先执行一些安全检查(参见`ValidateOpOrMarkFailed()`);
2. 检查当前`Tablet`的状态(`CheckHasNotBeenStoppedUnlocked()`)
     + 如果已经被停止(`kStopped`或`kShutdown`)，那么就返回失败；
     + 当前的状态应该是`kOpen`或`kBootstrapping`;  
    
   说明：在检查`Tablet`状态的过程，应该加上`state_lock_`锁。
3. 




#### `private ValidateOpOrMarkFailed()`

检查当前“行操作”是否有效。

1. 当前`RowOp`对象已经被设置（一定是一个错误的状态），那么直接返回`false`。
2. 调用`ValidateOp()`，针对不同的操作类型进行分别检查。

#### `private ValidateOp()`

针对不同的类型，分别进行检查是否为有效的操作。

+ 对于`INSERT`和`UPSERT`: 目前只检查，经过编码的key的长度，是否超过了阈值(`FLAGS_max_encoded_key_size_bytes`)。 如果超过了，就报错。
+ 对于`UPDATE`和`DELETE`: 不可以是`re-insert`操作（`re-insert`操作，应该当做`INSERT`类型来处理）。

#### `private CheckRowInTablet()`

给定一个`row key`（是`ConstContiguourRow`类型），检查它是否属于当前的`Tablet`。

判断方法：通过当前`Tablet`的元信息，就可以进行判断。


#### `private FindRowSetsToCheck()`

给定一个"行操作"，查找它可能属于的`RowSet`集合。


























