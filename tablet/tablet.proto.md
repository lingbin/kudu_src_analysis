[TOC]

文件： `src/kudu/tablet/tablet.proto`

## `message MemStoreTargetPB`

表示如下两种场景之一：
1. 对于要在`MRS`上做的`insert`和`mutate`：保存`MemRowSet`的`id`（）。
2. 对于要在`DiskRowSet`上要做`mutate`: 保存`DiskRowSet`的id、以及`delta`的`id`。

**注意：**  
因为要么是“场景1”，要么是“场景2”，所以其中的属性，根据相应的场景，设置对应的一部分。

```
message MemStoreTargetPB {
  // -1 defaults here are so that, if a caller forgets to check has_mrs_id(),
  // they won't accidentally see real-looking (i.e 0) IDs.

  // Either this field...
  optional int64 mrs_id = 1 [ default = -1];

  // ... or both of the following fields are set.
  optional int64 rs_id = 2 [ default = -1 ];
  optional int64 dms_id = 3 [ default = -1 ];
}
```


## `message OperationResultPB`

一个"insert"或者"mutate"操作的结果。

### `skip_on_replay`属性

在回放日志的时候，那么该属性会被设置为`true`。 表示，当前操作的状态
1. 如果已经被`Flush`过了; 
2. 之前已经失败过了，不应该再次进行`apply`。

### `failed_status`属性
如果当前操作失败了，那么要设置这个属性。

### `mutated_stores`属性

注意是`repeated`修饰符，可能会有多个。

当前操作所涉及的`MemStore`：
1. 对于`INSERT`: 只会有一个store。
2. 对于`MUTATE`：可能会有多个。 发生的场景是：如果在进行`compaction`的过程中，该修改到达，那么可能会关联多个`RowSet`。

```
message OperationResultPB {
  optional bool skip_on_replay = 1 [ default = false ];

  optional kudu.AppStatusPB failed_status = 2;

  repeated MemStoreTargetPB mutated_stores = 3;
}
```

## `message TxResultPB`

描述一个事务的最终结果。

因为一个事务中，可能包含多行修改，所以这里就是包含多个`OperationResultPB`成员。(`repeated`修饰符)

```
message TxResultPB {
  repeated OperationResultPB ops = 1;
}
```

## `message DeltaStatsPB`

在`flush`一个"delta store"时，一些统计信息。

```
// Delta statistics for a flushed deltastore
message DeltaStatsPB {
  // Number of deletes (deletes result in deletion of an entire row)
  required int64 delete_count = 1;

  // Number of reinserts.
  // Optional for data format compatibility.
  optional int64 reinsert_count = 6 [ default = 0];

  // REMOVED: replaced by column_stats, which maps by column ID,
  // whereas this older version mapped by index.
  // repeated int64 per_column_update_count = 2;

  // The min Timestamp that was stored in this delta.
  required fixed64 min_timestamp = 3;
  // The max Timestamp that was stored in this delta.
  required fixed64 max_timestamp = 4;

  // Per-column statistics about this delta file.
  message ColumnStats {
    // The column ID.
    required int32 col_id = 1;
    // The number of updates which refer to this column ID.
    optional int64 update_count = 2 [ default = 0 ];
  }
  repeated ColumnStats column_stats = 5;
}

```

## `message TabletStatusPB` 

`tablet`的状态。

```
message TabletStatusPB {
  required string tablet_id = 1;
  required string table_name = 2;
  optional TabletStatePB state = 3 [ default = UNKNOWN ];
  optional tablet.TabletDataState tablet_data_state = 8 [ default = TABLET_DATA_UNKNOWN ];
  required string last_status = 4;
  // DEPRECATED.
  optional bytes start_key = 5;
  // DEPRECATED.
  optional bytes end_key = 6;
  optional PartitionPB partition = 9;
  optional int64 estimated_on_disk_size = 7;
  repeated bytes data_dirs = 10;
}
```










