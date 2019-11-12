[TOC]

文件： `src/kudu/tablet/metadata.proto`

# `ColumnDataPB`

```
message ColumnDataPB {
  required BlockIdPB block = 2;
  // REMOVED: optional ColumnSchemaPB OBSOLETE_schema = 3;
  optional int32 column_id = 4;
}
```

#
```


message DeltaDataPB {
  required BlockIdPB block = 2;
}

message RowSetDataPB {
  required uint64 id = 1;
  required int64 last_durable_dms_id = 2;
  repeated ColumnDataPB columns = 3;
  repeated DeltaDataPB redo_deltas = 4;
  repeated DeltaDataPB undo_deltas = 5;
  optional BlockIdPB bloom_block = 6;
  optional BlockIdPB adhoc_index_block = 7;
  optional bytes min_encoded_key = 8;
  optional bytes max_encoded_key = 9;

  // Number of live rows that have been persisted.
  optional int64 live_row_count = 10;
}
```