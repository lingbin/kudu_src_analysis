[TOC]

文件: `src/kudu/common/wire_protocol.proto`

## `message AppStatusPB`

在`RPC`中传递的错误码。

任何一个可能会产生“应用级”错误的`RPC方法`，都应该在它的`Response`结构体中，将`AppStatusPB`结构作为一个`optional`的成员。

这个类，与代码中的`Status`类相对应：
1. 在`C++`中：对应`kudu::Status`类；
2. 在`Java`中：对应`org.apache.kudu.Status`类。

### `enum AppStatusPB::ErrorCode`

所有的错误码列表。

```
  enum ErrorCode {
    UNKNOWN_ERROR = 999;
    OK = 0;
    NOT_FOUND = 1;
    CORRUPTION = 2;
    NOT_SUPPORTED = 3;
    INVALID_ARGUMENT = 4;
    IO_ERROR = 5;
    ALREADY_PRESENT = 6;
    RUNTIME_ERROR = 7;
    NETWORK_ERROR = 8;
    ILLEGAL_STATE = 9;
    NOT_AUTHORIZED = 10;
    ABORTED = 11;
    REMOTE_ERROR = 12;
    SERVICE_UNAVAILABLE = 13;
    TIMED_OUT = 14;
    UNINITIALIZED = 15;
    CONFIGURATION_ERROR = 16;
    INCOMPLETE = 17;
    END_OF_FILE = 18;
    CANCELLED = 19;
  }
```

### 成员

注意：除了`Kudu`定义的错误码，还包括了`posix`的错误码。

```
message AppStatusPB {
  required ErrorCode code = 1;
  optional string message = 2;
  optional int32 posix_code = 4;
}
```

## `message NodeInstancePB`

用来唯一的标识 集群中的一台server。

### `byte permanent_uuid`

用来表示server的uuid。

在server第一次启动的时候被设置。注意该值会被保存在磁盘上。

### `int64 instance_seqno`

每次重新启动，都会将该值增加1。表示被重启的次数。

这个值的初始值为`0`。

这个值的用途： 方便其它机器检查到该机器是否进行了重启。（如果有重启，可能有一些内存状态需要被清理）

```
message NodeInstancePB {
  required bytes permanent_uuid = 1;
  required int64 instance_seqno = 2;
}
```

## `message ServerRegistrationPB`

同时在`master`和`tserver`上都存在的一些共同属性（也称为“注册信息”）。

注意：在机器被重启之前，这些属性都会被保证 不会被改变。

**`rpc_addresses`和`http_addresses`属性**  

注意：这两个属性都是`repeated`的，即可能会有多个`host:port`对。

>> 问题：为什么会有多个？

**`https_enabled`属性**  
标识是否开启了`http`服务。

如果开启了`http`服务，那么URL是会使用上面的`http_addresses`属性。

**`start_time`属性**  
标识当前进程启动的时间。取值为 从`epoch`到现在的秒数。

```
message ServerRegistrationPB {
  repeated HostPortPB rpc_addresses = 1;
  repeated HostPortPB http_addresses = 2;
  optional string software_version = 3;

  optional bool https_enabled = 4;

  optional int64 start_time = 5;
}
```

## `message ServerEntryPB`

每个机器上，都会保存一个其它机器的列表。其中，每台机器对应一条记录。

**`error`属性**  
如果和一台机器在交互时发生错误，会把错误信息记录在这里。

**`instance_id`属性**  
对方机器的唯一标识。

**`registration`属性**  
对方机器的 “注册信息”。

**`role`属性**  
对方机器，在`raft`组中的角色信息。

>> 问题：当前这个类的使用场景是？ 是每个机器中有一个关于其它机器的总列表？ 还是每个raft组中有一个属于当前raft组的其它机器的列表？

```
message ServerEntryPB {
  // If there is an error communicating with the server (or retrieving
  // the server registration on the server itself), this field will be
  // set to contain the error.
  //
  // All subsequent fields are optional, as they may not be set if
  // an error is encountered communicating with the individual server.
  optional AppStatusPB error = 1;

  optional NodeInstancePB instance_id = 2;
  optional ServerRegistrationPB registration = 3;

  // If an error has occured earlier in the RPC call, the role
  // may be not be set.
  optional consensus.RaftPeerPB.Role role = 4;
}
```

## `message RowwiseRowBlockPB`

表示一个`Row Block`。在其中，会连续的存放多个`Row`。

### `num_rows`属性

存储的行数。

说明：为什么要单独保存这个值。  
虽然通常情况下，这个值也可以通过`rows占用的总内存大小 / row_width` 来计算出来。 
但是，如果是一个空的投影，比如`count(*)`，那么就只能通过该属性来决定 返回多少行。(因为投影为空，即`rod_width==0`，无法使用上面的公式)。

### `rows_sidecar`属性

>> sidecar: （侧三轮摩托车的）挎斗  

在每个`sidecar`中，每个`Row`都是按照与`kudu::ContiguousRow`相同的内存格式进行保存（即一个原始的没有经过编码的数据段， 加上 一个表示是否为空的 bitmap）。

对于值为`null`的“单元格”，它的存储内容是“未定义”的。 通常情况下，是填充的`\x00`，但是并不保证一定是这样。

而且，client可能会将这些为`null`的单元格，初始化为任意的值。

注意1：将这些值设置为“常量”，确实是可以提高 RPC的压缩效率的。

注意2：其中的指针，都是保存的相对于`sidecar`首部的偏移量。

### `indirect_data_sidecar`属性

对于间接引用的数据，保存对应的`sidecar index`。

在`sidecar`中，在一个block中的所有“间接引用”的数据类型，都是被“连续存储”的。  

比如：在一个block中的`STRING`类型的值，存储的就是 按照在内存中的`Slice`的格式。 只不过其中的指针，不再是指向 `RAM`，而是表示相对于当前`protobuf`字段的偏移量。

>> 问题： 什么是`sidecar`? 是不是相当于`brpc`中的`attachment`?

```
message RowwiseRowBlockPB {
  optional int32 num_rows = 1 [ default = 0 ];

  optional int32 rows_sidecar = 2;

  optional int32 indirect_data_sidecar = 3;
}
```

## `message RowOperationsPB`

### `enum RowOPerationsPB::Type`
表示操作类型，包括如下内容：
1. 对一个表要做的一组操作：(`INSERT`, `UPDATE`, `UPSERT`, or `DELETE`);
2. 在`create/alter table`时，结合使用“SPLIT_ROW”类型 和 "range的上下边界" 来指定 区间范围；

>> 问题： `split row`是什么？  
>> 答：当用来表示partition的区间范围时，对应的类型是`SPLIT_ROW`，区间的上下边界值用 4个`BOUND` 分别来指定。

### range边界 的指定
1. 因为指定区间时，默认是“**左闭右开**”的，所以如果需要指定“左开”、或者“右闭”时，使用额外的属性，这里分别使用
  + `EXCLUSIVE_RANGE_LOWER_BOUND`: 左开；
  + `INCLUSIVE_RANGE_UPPER_BOUND`：右闭；
2. 正负无穷的指定方法：
  + 如果在指定区间时，指定 区间‘左边界’的两个属性都为空，那么表示该区间没有‘左边界’（即取值范围是 负无穷）；
  + 同理，如果 ‘右边界’的两个属性都为空，那么表示该区间没有‘右边界’；

### 编码 操作的数据

修改内容，是编码以后，保存在 二进制属性成员`rows`中的。

具体的编码方式为：
1. **操作类型**：1个字节。具体的值，对应`RowOPerationsPB::Type`的值；
2. **column isset bitmap**: 每个column对应一位（末尾可能会留有一部分空白）。 如果用户设置了某列的值，那么在这个bitmap中对应bit位的值为1；
3. **null bitmap**: 每个column对应一位（末尾可能会留有一部分空白）。如果对应的列为null，那么对应的bit位设置为1；
4. **column data**: 对于设置的每个column，并且它不为null，那么它的值就会被编码到这里。 具体的编码格式是各个类型的内存格式（小端）；对于字符串类型，在`rows`中保存的是指针，具体的值保存在`indirect_data`中。


条目 | 大小  |  说明
---|--- | ---
`operation type`      | 1个字节  |  具体的值，对应`RowOPerationsPB::Type`的值
`column isset bitmap` | 每个column对应一位（整字节对齐，末尾可能会留有一部分空白）  |  如果用户设置了某列的值，那么在这个bitmap中对应bit位的值为1；
`null bitmap`         | 每个column对应一位（整字节对齐，末尾可能会留有一部分空白）  |  如果对应的列为null，那么对应的bit位设置为1；
`column data`         | -------  |  对于设置的每个column，并且它不为null，那么它的值就会被编码到这里。 具体的编码格式是各个类型的内存格式（小端）；对于字符串类型，在`rows`中保存的是指针，具体的值保存在`indirect_data`中。

**注意：当有多行数据时，行与行之间是不留空白的**

> canonical: 典型的；标准的；

```
message RowOperationsPB {
  enum Type {
    UNKNOWN = 0;
    INSERT = 1;
    UPDATE = 2;
    DELETE = 3;
    UPSERT = 5;

    SPLIT_ROW = 4;
    RANGE_LOWER_BOUND = 6;
    RANGE_UPPER_BOUND = 7;
    EXCLUSIVE_RANGE_LOWER_BOUND = 8;
    INCLUSIVE_RANGE_UPPER_BOUND = 9;
  }

  optional bytes rows = 2 [(kudu.REDACT) = true];
  optional bytes indirect_data = 3 [(kudu.REDACT) = true];
}
```