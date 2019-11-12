[TOC]

文件：`src/kudu/tablet/tablet_metadata.h`

# `class TabletMetadata`

用来管理一个`tablet`的所有`block`。

每个`tablet`都会有一个`TabletMetadata`成员。 当`tablet`创建`block`时，会调用`Tablet::Flush()`来将`block列表`持久化到磁盘上。

在启动的时候，`TSTabletManager`会从`tablets/`目录中的每个`super block`中，加载一个`TabletMetadata`对象，并且创建对应的`tablet`对象。

## `enum TabletMetadata::State`

描述当前`TabletMetadata`对象的状态。主要是用来区分是否已经“持久化到磁盘”。

```
  enum State {
    kNotLoadedYet,
    kNotWrittenYet,
    kInitialized
  };
```

## 成员

### `state_`
当前`TabletMetadata`的“持久化状态”。

### `data_lock_`
并发修改内部属性的“互斥锁”。

### `flush_lock_`
用来保护当前`tablet`的`flush`状态。即为了最多只能有一个线程对当前tablet进行`flush`。

注意：如果和`data_lock_`一起使用，那么必须先获取`flush_lock_`。

### `tablet_id_`
当前`tablet`的`id`.

### `table_id_`
当前`tablet`所属的`table`的`id`。

>> 问题：为什么`table_id_`和`tablet_id_`的类型，都是`std::string`类型？为什么不是`uint64_t`类型？

### `partition_`
当前`tablet`的分区信息。

类型是`Partition`，参见`src/kudu/commong/partition.h`。

### ``

### ``

### ``

```
  State state_;

  typedef simple_spinlock LockType;
  mutable LockType data_lock_;

  mutable Mutex flush_lock_;

  const std::string tablet_id_;
  std::string table_id_;

  Partition partition_;

  FsManager* const fs_manager_;
  RowSetMetadataVector rowsets_;

  base::subtle::Atomic64 next_rowset_idx_;

  int64_t last_durable_mrs_id_;

  // The current schema version. This is owned by this class.
  // We don't use gscoped_ptr so that we can do an atomic swap.
  Schema* schema_;
  uint32_t schema_version_;
  std::string table_name_;
  PartitionSchema partition_schema_;

  // Previous values of 'schema_'.
  // These are currently kept alive forever, under the assumption that
  // a given tablet won't have thousands of "alter table" calls.
  // They are kept alive so that callers of schema() don't need to
  // worry about reference counting or locking.
  std::vector<Schema*> old_schemas_;

  // Protected by 'data_lock_'.
  BlockIdSet orphaned_blocks_;

  // The current state of tablet copy for the tablet.
  TabletDataState tablet_data_state_;

  // Record of the last opid logged by the tablet before it was last
  // tombstoned. Has no meaning for non-tombstoned tablets.
  // Protected by 'data_lock_'.
  boost::optional<consensus::OpId> tombstone_last_logged_opid_;

  // Table extra config.
  boost::optional<TableExtraConfigPB> extra_config_;

  // Tablet's dimension label.
  boost::optional<std::string> dimension_label_;

  // If this counter is > 0 then Flush() will not write any data to
  // disk.
  int32_t num_flush_pins_;

  // Set if Flush() is called when num_flush_pins_ is > 0; if true,
  // then next UnPinFlush will call Flush() again to ensure the
  // metadata is persisted.
  bool needs_flush_;

  // The number of times metadata has been flushed to disk
  int flush_count_for_tests_;

  // A callback that, if set, is called before this metadata is flushed
  // to disk. Protected by the 'flush_lock_'.
  StatusClosure pre_flush_callback_;

  // The on-disk size of the tablet metadata, as of the last successful
  // call to Flush() or LoadFromDisk().
  std::atomic<int64_t> on_disk_size_;
  
  // The tablet supports live row counting if true.
  bool supports_live_row_count_;
```

## 接口列表

### `static CreateNew()`

创建一个新的`TabletMetadata`对象。

注意：这里的前提是：一个给定的`super block`，尚未被写入，并且这个方法会将这个`super block`写入到磁盘。





