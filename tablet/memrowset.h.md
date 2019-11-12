[TOC]

文件： `src/kudu/tablet/memrowset.h`


## `class MemRowSet`

### 实现的注意事项

`MemRowSet`是一个 并发的`b-tree`。 用来保存被插入、但还没有被flush的数据。

为了提供快照一致性，数据在插入以后，就不会在原地修改；而是被添加到“修改链表”中。 这个“修改链表”就相当于该行的“redo log”。

>> 名词解释  
>> `CBTree`: 并发的`b-tree`，是`MemRowSet`的存储结构。

每一行数据，都会作为`CBTree`的一个条目。   
+ key是：该行数据的“主键”编码后的格式，是会按照“字典序”排列的；
+ value是：是一个`MRSRow`对象；

**注意：**  
在`MemRowSet`中的所有内存申请操作，都是在它内部的“内存池”中申请的。然后在该`MemRowSet`被析构的时候，一起被释放。

## `class MRSRow`

是`CBTree`的值。

注意：（在`CBTree`中实际存储的条目）  
实际上，在`CBTree`中存储的条目，是`Slice`类型。是用来构造`MRSRow`对象时传入的`Slice`（参见`构造函数`）。 在这个`Slice`中，包含两个部分：
1. `Header`对象；
2. 该行的具体值 包括1) 表示是否为`null`的bitmap; 2) 具体的数据；

`CBTree`中的条目(`Slice`对象)，它们所间接引用的数据，都指向`MemRowSet`的“内存池”。

该类的接口类似于`ContiguousRow`（参见`src/kudu/common/row.h`）。

因为和`ContiguousRow`一样，都会作为`ContiguousRowCell<T>`模板参数，所以一定要提供一些该模板要求的接口。（参见： `src/kudu/common/row.h`中的`ContiguousRowCell<T>`）

### `struct MRSRow::Header`

链表的头结点。

分别指向“修改列表”(`redo记录`)链表的头指针。

注意：在这里，也记录了一个“尾指针”。

#### `insertion_timestamp`
被插入记录的时间戳。

注意：对于一行数据，第一次操作一定是“插入”操作。

每个扫描器（迭代器），都有一个快照时间戳。 如果小于`insertion_timestamp`，那么这条记录会被忽略掉。

#### `redo_head`
头指针，指向“修改链表”的头部；

#### `redo_tail`
尾指针，指向“修改链表”的尾部；

```
  struct Header {
      Timestamp insertion_timestamp;

      Mutation* redo_head;
      Mutation* redo_tail;
  };
};
```

说明：这里之所以要将`tail`指针也保留下来，是为了提高“插入”的效率。因为“插入”是在末尾进行的。

>> 附录：关于尾指针的引入  
>> 参见：the tail link for redo log in MemRowSet: https://gerrit.cloudera.org/#/c/13412/ and optimization on DiskRowSet: https://gerrit.cloudera.org/#/c/13516/ ; https://gerrit.cloudera.org/#/c/13661/.

### 成员

#### `header_`
“修改记录链表”的头指针。

类型是一个`Header*`，具体指向的位置，构造`MRSRow`对象时传入的`Slice`对象的头部。（参见`构造函数`的代码）

在构建当前`MRSRow`对象时，会传入一个`Slice`对象。在传入的`Slice`对象的头部，是一个`Header`对象。 然后才是该行数据的值。

注意：`row_slice_`成员所指向的值，是不包含`Header`对象的。（因为在构造函数中，已经通过`Slice::remove_prefix()`将该部分去掉了）

#### `row_slice_`
该行数据的具体内容。

`Slice`内部的指针，指向当前`MemRowSet`的“内存池”。

其中值的编码格式，与`ContiguousRow`是相同的：
1. 每个列占用的内存长度，与它的类型是相关的；
2. 是否有`null bitmap`，取决于是否有`nullable == true`的列；

#### `memrowset_`
当前`MRSRow`所属的`MemRowSet`对象指针。

```
  Header *header_;

  // Actual row data.
  Slice row_slice_;

  const MemRowSet *memrowset_;
```

### 接口列表

#### `MRSRow`构造函数

当前`MRSRow`中的数据，来自于传入的`Slice`对象。（在`MRSRow`类中，并不会进行内存大 申请和释放）。

```
  MRSRow(const MemRowSet *memrowset, const Slice &s) {
    DCHECK_GE(s.size(), sizeof(Header));
    row_slice_ = s;
    header_ = reinterpret_cast<Header *>(row_slice_.mutable_data());
    row_slice_.remove_prefix(sizeof(Header));
    memrowset_ = memrowset;
  }
```

注意：在传入的`Slice`对象的头部，是一个`MRSRow::Header`对象。但是在赋值给`row_slice_`之前，已经将这部分移除（所以，在`row_slice_`成员中，只包含该行数据的“修改内容”）。

注意：在这个`row_slice_`中，是包含“所有列”的数据的（也包括key列的）。

对比：`RowChangeList`中的“内容”，是不包括`key`列值的。也就是`Mutation`中的`Slice`部分。

```
//   |---------- 传入的Slice对象 ---------------------|
//
//   |----------------|-------------------------------|
//   
//   |<--- Header --->|<--------- row_slice_成员----->| 
```

说明：`row_slice_`部分，即“行的数据”部分，内部的编码方式与`ContiguousRow`是相同的。包含了“所有列”以及一个可选的`null bitmap`。

对于插入到`MemRowSet`的`MRSRow`，其间接引用的数据（比如`STRING`类型），在当前`MemRowSet`的内存池中。

#### 作为`ContiguousRowCell<T>`中模板参数需要的接口

##### `schema()`
当前“单元格”所属的Row的schema。

```
inline const Schema* MRSRow::schema() const {
  return &memrowset_->schema_nonvirtual();
}
```

说明：在代码的实现时，并没有在`MRSRow::schema()`声明的地方直接实现，而是`.h`文件的最后，进行了实现，这是因为在声明`MRSRow::schema()`的时候，还没有定义`MemRowSet`，所以是无法找到`MemRowSet::schema_nonvirtual()`方法的（即编译会报错，提示“不完整的类型”）。

而在文件最后进行定义，因为之前已经定义了`MemRowSet`,所以是没有问题的。

##### `cell_ptr()`
返回“单元格”的值的指针。返回的是`const void*`，不可修改。

##### `mutable_cell_ptr()`
返回“单元格”的值的指针。返回的是`void*`，是可以修改的。

##### `is_null()`
当前“单元格”是否为`null`

##### `set_null()`
将当前“单元格”的值设置为`null`

```
  bool is_null(size_t col_idx) const {
    return ContiguousRowHelper::is_null(*schema(), row_slice_.data(), col_idx);
  }

  void set_null(size_t col_idx, bool is_null) const {
    ContiguousRowHelper::SetCellIsNull(*schema(),
      const_cast<uint8_t*>(row_slice_.data()), col_idx, is_null);
  }

  const uint8_t *cell_ptr(size_t col_idx) const {
    return ContiguousRowHelper::cell_ptr(*schema(), row_slice_.data(), col_idx);
  }

  uint8_t *mutable_cell_ptr(size_t col_idx) const {
    return const_cast<uint8_t*>(cell_ptr(col_idx));
  }
```

#### 作为`RowType`需要的其它方法

##### `nullable_cell_ptr()`

**说明：`nullable_cell_ptr()`和`cell_ptr()`的区别**  
在`nullable_cell_ptr()`中，会先检查当前“单元格”是否为`null`。
+ 如果为`null`：那么直接返回`nullptr`;
+ 如果不为`null`，那么范围`cell_ptr()`；

```
  const uint8_t *nullable_cell_ptr(size_t col_idx) const {
    return ContiguousRowHelper::nullable_cell_ptr(*schema(), row_slice_.data(), col_idx);
  }
```

关于如何存储值为`null`的列，参见`ContiguousRowHelper::nullable_cell_ptr()`的说明。

##### `cell()`

返回一个`ContiguousRow<MRSRow>`对象。

#### 一些`getter`方法
##### `insertion_timestamp()`
从当前记录的`Header`中，获取“插入时间戳”。

```
  Timestamp insertion_timestamp() const { 
    return header_->insertion_timestamp; 
  }
```

##### `redo_head()`
获取当前的“redo记录”的 头指针。

```
  Mutation* redo_head() { return header_->redo_head; }
  const Mutation* redo_head() const { return header_->redo_head; }
```

##### `acquire_redo_head()`
使用`Acquire`指令来加载“redo记录”的头指针。

>> 问题： 为什么要使用`Acquire`指令？ 使用场景是？ 不用的话，会有什么问题？

```
  // Load 'redo_head' with an 'Acquire' memory barrier.
  Mutation* acquire_redo_head() {
    return reinterpret_cast<Mutation*>(
        base::subtle::Acquire_Load(reinterpret_cast<AtomicWord*>(&header_->redo_head)));
  }
```

##### `row_slice()`和`row_data()`

```
  const Slice &row_slice() const { return row_slice_; }
  const uint8_t* row_data() const { return row_slice_.data(); }
```

#### 该类型的`Row`所特有的方法

##### `isGhost()`

如果一行数据，最后一个修改是“DELETE”，那么就说这行数据是`ghost`。

因为保存了“尾指针”，所以可以快速定位到尾部。

注意：因为该行记录，第一条插入，一定是`INSERT`。也就是说，如果有`DELETE`，只可能存在与“修改链表”上。

```
bool MRSRow::IsGhost() const {
  const Mutation *mut_tail = header_->redo_tail;
  if (mut_tail == nullptr) {
    return false;
  }
  RowChangeListDecoder decoder(mut_tail->changelist());
  Status s = decoder.Init();
  if (!PREDICT_TRUE(s.ok())) {
    LOG(FATAL) << Substitute("Failed to decode: $0 ($1)",
                             mut_tail->changelist().ToString(*schema()),
                             s.ToString());
  }
  return decoder.is_delete();
}

```

## `MSBTreeTraits`

定义一个`BTree`类型，将其中的“内存池”类型，修改为`ThreadSafeMemoryTrackingArena`。 

这个“内存池”类型，可以线程安全的申请内存，还可以跟踪内存的分配情况。

```
struct MSBTreeTraits : public btree::BTreeTraits {
  typedef ThreadSafeMemoryTrackingArena ArenaType;
};
```

## `DEFINE_MRSROW_ON_STACK`宏

在“栈”上定义一个`MRSRow`对象。变量的名称就是 宏参数`varname`。

说明：其中在栈上还会声明一个`Slice`对象（名字是宏参数`slice_name`），它指向的内存是栈上的一个数组。

宏的实现方法：
1. 先计算所需内存的总大小。 `Header`对象需要的内存 + 一行数据需要的内存大小（`Row::byte_size()`）。
2. 在栈上分配一个数组(长度为上面计算的值)。
3. 新建一个`Slice`对象，内容指向刚刚的数组。
4. 创建`MRSRow`对象；

```
#define DEFINE_MRSROW_ON_STACK(memrowset, varname, slice_name) \
  size_t varname##_size = sizeof(MRSRow::Header) + \
                           ContiguousRowHelper::row_size((memrowset)->schema_nonvirtual()); \
  uint8_t varname##_storage[varname##_size]; \
  Slice slice_name(varname##_storage, varname##_size); \
  ContiguousRowHelper::InitNullsBitmap((memrowset)->schema_nonvirtual(), slice_name); \
  MRSRow varname(memrowset, slice_name);
```

## `class MemRowSet`

**注意1：在其中的数据，是按照“行式”组织的，不是“列式”的。**  

**注意2：其中的数据是有序的(按照“主键”的升序)。**

### 成员

#### `id_`
当前`MemRowSet`的`id`，每个`MemRowSet`有唯一的id。

#### `schema_`
当前`MemRowSet`的`schema`。

**注意1：这是一个对象成员，不是指针，所以需要拷贝**  
注意它是一个对象，不是一个指针。在构建的时候，会传入一个`Schema`，会被拷贝到`MemRowSet`中。

**注意2：这个成员是`const`的，所以不可以修改**  

>> 问题：既然一个`MemRowSet`中`Schema`是不可以被修改的，那么如果发生了"online schema change"，然后再读取时会不会选错`Schema`？

#### `allocator_`
类型是`std::shared_ptr<MemoryTrackingBufferAllocator>`。

内存分配器，可以监控内存的分配情况。

#### `arena_`
类型是`std::shared_ptr<ThreadSafeMemoryTrackingArena>`

每个`MemRowSet`中，都会维护一个“内存池”。所有需要的内存，都是从这个“内存池”中申请。在当前`MemRowSet`中被析构的时候，统一释放“内存池”中所有的内存。

#### 子类型`MSBTIter`
`CBTree`的迭代器类型。

#### `tree_`
内部存储所有记录的数据结构：“并发的`b-tree`”。

>> 备注：有时候简称为`CBTree`.  

#### `debug_insert_count_`和`debug_update_count_`
一个大概的计数变量。

注意1：它们并不是原子修改的，只是简单的使用`volatile`修饰符。

所以在任何需要精确值的时候，都不应该使用这两个变量。

>> sanity check: 合理性检查；健全性检查；  

它的用处是：仅仅在`flush`的时候，做一个健全性检查。

#### `compact_flush_lock_`

一个互斥锁。用来防止出现多个任务同时flush。

#### `anchorer_`

用来记录当前`MemRowSet`中，尚未刷盘的最小的 `raft log index`。

>> 问题：记录这个值，有什么用？

#### `has_been_compacted_`

用来标识当前`RowSet`是否已经从`rowset tree`中删除掉。

如果已经被删除掉，那么在后续的`compaction`作业中，就不应该再包含当前`MemRowSet`。

#### `live_row_count_`
当前`MemRowSet`中，还存有的行数（不包含已经被delete掉的行）

注意：和`debug_insert_count_`、`debug_update_count_`是不同的，这个变量是`AtomicInt<int64_t>`类型，是一个精确的值。

```
  typedef btree::CBTree<MSBTreeTraits> MSBTree;

  int64_t id_;

  const Schema schema_;
  std::shared_ptr<MemoryTrackingBufferAllocator> allocator_;
  std::shared_ptr<ThreadSafeMemoryTrackingArena> arena_;

  typedef btree::CBTreeIterator<MSBTreeTraits> MSBTIter;

  MSBTree tree_;

  volatile uint64_t debug_insert_count_;
  volatile uint64_t debug_update_count_;

  std::mutex compact_flush_lock_;

  log::MinLogIndexAnchorer anchorer_;

  std::atomic<bool> has_been_compacted_;

  AtomicInt<int64_t> live_row_count_;
```

### 接口列表

该类继承自`RowSet`类，有很多接口需要去实现。

注意：`MemRowSet`类没有提供 `public`的构造函数，所以只能通过 工厂方法`Create()`进行创建。

#### `static Create()  -- 工厂方法`

创建出来的`MemRowSet`对象，通过参数返回。

```
Status MemRowSet::Create(int64_t id,
                         const Schema &schema,
                         LogAnchorRegistry* log_anchor_registry,
                         shared_ptr<MemTracker> parent_tracker,
                         shared_ptr<MemRowSet>* mrs) {
  shared_ptr<MemRowSet> local_mrs(new MemRowSet(
      id, schema, log_anchor_registry, std::move(parent_tracker)));

  mrs->swap(local_mrs);
  return Status::OK();
}
```

#### `private 构造函数`

会同时创建一些基本对象，比如`MemTracker`（名称是`MemRowSet-$id`）

```
MemRowSet::MemRowSet(int64_t id,
                     const Schema& schema,
                     LogAnchorRegistry* log_anchor_registry,
                     shared_ptr<MemTracker> parent_tracker)
  : id_(id),
    schema_(schema),
    allocator_(new MemoryTrackingBufferAllocator(
        HeapBufferAllocator::Get(),
        CreateMemTrackerForMemRowSet(id, std::move(parent_tracker)))),
    arena_(new ThreadSafeMemoryTrackingArena(kInitialArenaSize, allocator_)),
    tree_(arena_),
    debug_insert_count_(0),
    debug_update_count_(0),
    anchorer_(log_anchor_registry, Substitute("MemRowSet-$0", id_)),
    has_been_compacted_(false),
    live_row_count_(0) {
  CHECK(schema.has_column_ids());
  ANNOTATE_BENIGN_RACE(&debug_insert_count_, "insert count isnt accurate");
  ANNOTATE_BENIGN_RACE(&debug_update_count_, "update count isnt accurate");
}
```

注意：这里传入的`Schema`对象，一定是已经初始化过了`column_id`相关的属性的（因为后续插入就要按照这个`Schema`进行数据处理，所以必然是需要已经初始化过了`column_id`的）。

#### 数据操作接口

##### `Insert()`

插入数据时，用户传入的数据，最终会组成一个`ConstContiguousRow`（即一个常量的`Row`，不可以修改其中的值）。

```
Status MemRowSet::Insert(Timestamp timestamp,
                         const ConstContiguousRow& row,
                         const OpId& op_id) {
  CHECK(row.schema()->has_column_ids());
  DCHECK_SCHEMA_EQ(schema_, *row.schema());

  {
    faststring enc_key_buf;
    schema_.EncodeComparableKey(row, &enc_key_buf);
    Slice enc_key(enc_key_buf);

    btree::PreparedMutation<MSBTreeTraits> mutation(enc_key);
    mutation.Prepare(&tree_);

    // TODO: for now, the key ends up stored doubly --
    // once encoded in the btree key, and again in the value
    // (unencoded).
    // That's not very memory-efficient!

    if (mutation.exists()) {
      // It's OK for it to exist if it's just a "ghost" row -- i.e the
      // row is deleted.
      MRSRow ms_row(this, mutation.current_mutable_value());
      if (!ms_row.IsGhost()) {
        return Status::AlreadyPresent("key already present");
      }

      // Insert a "reinsert" mutation.
      return Reinsert(timestamp, row, &ms_row);
    }

    // Copy the non-encoded key onto the stack since we need
    // to mutate it when we relocate its Slices into our arena.
    DEFINE_MRSROW_ON_STACK(this, mrsrow, mrsrow_slice);
    mrsrow.header_->insertion_timestamp = timestamp;
    mrsrow.header_->redo_head = nullptr;
    mrsrow.header_->redo_tail = nullptr;
    RETURN_NOT_OK(mrsrow.CopyRow(row, arena_.get()));

    CHECK(mutation.Insert(mrsrow_slice))
    << "Expected to be able to insert, since the prepared mutation "
    << "succeeded!";
  }

  anchorer_.AnchorIfMinimum(op_id.index());

  debug_insert_count_++;
  live_row_count_.Increment();
  return Status::OK();
}
```

说明1：  
在实现时，会先定义一个`faststring`对象，然后将“复合主键”中各列的值编码进去（注意：这个编码之后的结果，是可以按照字典序排序的）。 而这段代码的后面实现中，并没有异步的逻辑，也就是在完成修改之前，这个`faststring`对象都一定是有效的。

在这个代码块退出时（写入已经完成），会析构这个`faststring`对象。

说明2：  
对于`b-tree`的操作，需要先进行`Prepare()`操作。在`Prepare()`之后，就可以判断该行数据是否已经存在了。

说明3:  
对于存在的行，可以通过`mutation.current_mutable_value()`获取到该行数据在`CBBTree`中的值（通过`MRSRow row(this, mutation.current_mutable_value());`）就可以构造出对应的`MRSRow`对象。

因为构建这个`MRSRow`对象时，所使用的`Slice`对象实际指向的内存，就是在`CBTree`中的条目所指向的内存，所以修改这个`MRSRow`对象，就是在修改`CBTree`中的条目。

说明4:   
对于在`b-tree`中判断结果为“已经存在”的row，还需要判断下这行数据是否已经是`ghost`(即已经被删除)。
+ 如果不是`ghost`，就要报错（因为一行数据不允许插入两次以上）。
+ 如果是`ghost`，那么还是可以插入的；这时调用`Reinsert()`函数。因为这时不是向`b-tree`进行插入，而是要将本次修改添加到“修改链表”(`redo链表`)中。

说明5：  
在最终判断可以插入之后，需要先将“间接引用”的数据，拷贝到当前`MemRowSet`的“内存池”中。

因为根据用户传入的数据构造的`ConstContiguousRow`对象中，“间接引用”的数据，在当前插入请求处理结束以后，就会被释放。

##### `private Reinsert()`

对于一个“插入”操作，如果发现它在`CBTree`中是存在，但是它是一个`ghost`(已经被删除)，那么调用`Reinsert()`进行重新插入。

因为是在`CBTree`中，已经有该行数据，所以最终会将这次写入操作，插入到该行数据的“redo 链表”中。

```
Status MemRowSet::Reinsert(Timestamp timestamp, const ConstContiguousRow& row, MRSRow *ms_row) {
  DCHECK_SCHEMA_EQ(schema_, *row.schema());

  // Encode the REINSERT mutation
  faststring buf;
  RowChangeListEncoder encoder(&buf);
  encoder.SetToReinsert(row);

  // Move the REINSERT mutation itself into our Arena.
  Mutation *mut = Mutation::CreateInArena(arena_.get(), timestamp, encoder.as_changelist());

  // Append the mutation into the row's mutation list.
  // This function has "release" semantics which ensures that the memory writes
  // for the mutation are fully published before any concurrent reader sees
  // the appended mutation.
  mut->AppendToListAtomic(&ms_row->header_->redo_head, &ms_row->header_->redo_tail);

  live_row_count_.Increment();
  return Status::OK();
}
```

说明1：  
因为在“redo 链表”中，保存的数据都是`RowChangeList`格式，所以要先将用户传入的`ConstContiguousRow`转化为`RowChangeList`格式。

说明2：  
因为是要插入到“redo 链表”中，所以要构造出一个`Mutation`对象。其中包含的数据，就是上面构造的`RowChangeList`的内容。

说明3:  
因为当前“reinsert”操作是要添加到“redo 链表”的末尾的，所以要修改"redo 链表"的尾指针。

>> 问题：这里为什么没有像`Insert()`那样，在操作完成以后增加`debug_insert_count_`, 也没有修改`anchorer_.AnchorIfMinimun()`?

##### `MutateRow()`
针对`Update`和`Delete`操作。

```
Status MemRowSet::MutateRow(Timestamp timestamp,
                            const RowSetKeyProbe &probe,
                            const RowChangeList &delta,
                            const consensus::OpId& op_id,
                            const IOContext* /*io_context*/,
                            ProbeStats* stats,
                            OperationResultPB *result) {
  {
    btree::PreparedMutation<MSBTreeTraits> mutation(probe.encoded_key_slice());
    mutation.Prepare(&tree_);

    if (!mutation.exists()) {
      return Status::NotFound("not in memrowset");
    }

    MRSRow row(this, mutation.current_mutable_value());

    // If the row exists, it may still be a "ghost" row -- i.e a row
    // that's been deleted. If that's the case, we should treat it as
    // NotFound.
    if (row.IsGhost()) {
      return Status::NotFound("not in memrowset (ghost)");
    }

    // Append to the linked list of mutations for this row.
    Mutation *mut = Mutation::CreateInArena(arena_.get(), timestamp, delta);

    // This function has "release" semantics which ensures that the memory writes
    // for the mutation are fully published before any concurrent reader sees
    // the appended mutation.
    mut->AppendToListAtomic(&row.header_->redo_head, &row.header_->redo_tail);

    MemStoreTargetPB* target = result->add_mutated_stores();
    target->set_mrs_id(id_);
  }

  stats->mrs_consulted++;

  anchorer_.AnchorIfMinimum(op_id.index());
  debug_update_count_++;
  if (delta.is_delete()) {
    live_row_count_.IncrementBy(-1);
  }
  return Status::OK();
}
```

说明1： 
因为是要进行`update`或者`delete`，所以就要求该行数据一定是存在的。  

一行数据不存在，包含两种情况（如下两种情况都会返回错误）：
1. 在`CBTree`中不存在；
2. 在`CBTree`中存在，但是是一个`ghost`；

说明2:  
因为`CBTree`中一定存在该行数据，所以直接利用`RowChangeList`构建出`Mutation`对象，插入到“redo 链表”中即可，同时会修改该链表的`尾指针`。

##### `CheckRowPresent()`

检查一行数据是否存在。

注意：不能 只检查在`CBTree`中是否存在，还要检查它是不是一个`ghost`。

如果是一个`ghost`，那么就相当于它压根从来没出现过。即也会返回`Status::OK`，并且`*present = false`。

```
Status MemRowSet::CheckRowPresent(const RowSetKeyProbe &probe, const IOContext* /*io_context*/,
                                  bool* present, ProbeStats* stats) const {
  // Use a PreparedMutation here even though we don't plan to mutate. Even though
  // this takes a lock rather than an optimistic copy, it should be a very short
  // critical section, and this call is only made on updates, which are rare.

  stats->mrs_consulted++;

  btree::PreparedMutation<MSBTreeTraits> mutation(probe.encoded_key_slice());
  mutation.Prepare(const_cast<MSBTree *>(&tree_));

  if (!mutation.exists()) {
    *present = false;
    return Status::OK();
  }

  // TODO(perf): using current_mutable_value() will actually change the data's
  // version number, even though we're not going to do any mutation. This would
  // make concurrent readers retry, even though they don't have to (we aren't
  // actually mutating anything here!)
  MRSRow row(this, mutation.current_mutable_value());

  // If the row exists, it may still be a "ghost" row -- i.e a row
  // that's been deleted. If that's the case, we should treat it as
  // NotFound.
  *present = !row.IsGhost();
  return Status::OK();
}

```

说明1：  
参见注释部分，虽然这个函数只是检查一行数据是否存在，完全不会进行任何修改。  
但是这里仍然使用了`bree::PreparedMutation<T>`（使用这个对象，在`Prepare()`时会进行加锁，在析构该对象的时候会进行解锁）。   

但是，`kudu`的作者认为这问题不大，原因是：
1. 这里的“关键区域”(`critical section`)非常小；
2. 当前函数`CheckRowPresent()`只在进行`mutate`操作时，才会被调用。而在`Kudu`的设计理念中，`更新`操作是非常少的。

说明2:  
即使在`CBTree`中存在，也不能说明该行数据是存在的，还要看该行数据是否为`ghost`。

#### 获取信息的接口

##### `CountRows()`

实际就是调用`btree`的`count()`接口。

注意1：这个方法需要遍历`btree`的所有条目，所以是比较慢的。

注意2: 这个方法只考虑在`CBTree`中的条目，而不考虑其中为`ghost`的条目。所以该方法获取的数量，实际上是会大于`live_row_count_`的。

##### `CountLiveRows()`

因为在`MemRowSet`中，记录了`live_row_count_`变量，所以该方法就是直接返回该变量的值。

##### `GetBounds()`

因为对于`MemRowSet`来说，后续还会不断的有数据进行插入，所以它的“上下边界”实际上是不能确定的，所以这个`MemRowSet`是不支持这个方法的（返回`Status::NotSupport("")`）;

##### `OnDiskSize()`
##### `OnDiskBaseDataSize()`
##### `OnDiskBaseDataSizeWithRedos()`
##### `OnDiskBaseDataColumnSize()`

上面连续4个方法，都是获取在“磁盘”上的数据大小。因为对于`MemRowSet`来说，所有数据都在内存中，所以这4个方法，都是返回`0`。


#### 和`compaction`互斥控制 相关的函数

##### `compact_flush_lock()`
获取该`MemRowSet`中，用来保护`compaction`和`flush`的互斥锁。

因为目前`MemRowSet`不支持`compaction`，所以这个锁实际上只用来保护`flush`操作的。

##### `has_been_compacted()`
检查当前`MemRowSet`是否已经进行过了`compaction`。

在需要对该`MemRowSet`进行`compaction`的时候，会检查这个值，如果已经进行过，那么就不能再对它重复进行`compaction`。

##### `set_has_been_compacted()`
设置`has_been_compacted_`的值为`true`，表示当前`MemRowSet`已经进行过了`Compaction`。

一个`MemRowSet`，一旦经过`compaction`，那么就会从`rowset tree`中移除，后续的`compaction`就不会再考虑当前`MemRowSet`。

##### `IsAvailableForCompaction()`

在当前的`Kudu`版本，`MemRowSet`是不支持`compaction`的，所以该方法永远返回`false`。

说明：因为目前`MemRowSet`不支持`compaction`，所以上面的`has_been_compacted()`/`set_has_been_compacted()`是不会被调用的。

#### 后台操作(`compaction`和`flush`)相关的操作


#### 其它接口

#### `NewCompactionInput()`

##### `()`
##### `()`
##### `()`
##### `()`


## `class MemRowSet::Iterator`

继承自`RowwiseIterator`。用来遍历在`MemRowSet`中的数据。

因为当前结构会维护一个指向`MemRowSet`的指针，所以在当前`Iterator`的生命周期内，一定不能将`MemRowSet`析构掉。

注意：这个`Iterator`不是一个完整的快照，但是对于其中单独的行，能够保证一致性。 并且，在并发修改的同时进行迭代，也是安全的。  

注意：“保证一致性”是说，它会至少返回在它创建时，已经存在的所有行。（可能更多）。

注意：其中的每一行，至少是在创建的时刻之前已经插入的，也可能会更靠前。

>> 问题：这段注释应该怎么理解？

注意：这个类的定义方式比较特别。

### 注意该类的定义方式

类似如下的形式：

```
class MemRowSet {
    class Iterator;
}

class MemRowSet::Iterator {
    
}
```

即在`MemRowSet`中，先前置声明了一下`Iterator`类（因为是在类`MemRowSet`内部声明的，所以`Iterator`的“完整命名”是`MemRowSet::Iterator`）。 然后在内的外部，对其进行了实现。

所以在定义时，类名部分应该写“完整的命名”，即`MemRowSet::Iterator`（不能只写`Iterator`，那样会被认为是`::Iterator`，即命名空间不正确）。

### `enum MemRowSet::Iterator::ScanState`

表示一个`Iterator`对象的状态。

**`kScanning`**  
还会继续通过 该`Iterator`对象 获取和返回数据；

**`kFinished`**  
迭代已经结束，不会再通过该`Iterator`对象获取数据。

```
  enum ScanState {
    // Enumerated constants to indicate the iterator state:
    kUninitialized = 0,
    kScanning = 1,  // We may continue fetching and returning values.
    kFinished = 2   // We either know we can never reach the lower bound, or
                    // we've exceeded the upper bound.
  };
```

### `enum MemRowSet::Iterator::ApplyStatus`

表示一个写操作的结果。

在遍历`Mutation链表`，并应用(`apply`)需要的条目，结果保存在`dst_row`中（任何需要的内存申请，都从`dst_arena`中申请）。

>> 问题：待详解

```
  // Walks the mutations in 'mutation_head', applying relevant ones to 'dst_row'
  // (performing any allocations out of 'dst_arena'). 'insert_excluded' is true
  // if the row's original insertion took place outside the iterator's time range.
  //
  // On success, 'apply_status' summarizes the application process.
  enum ApplyStatus {
    // No mutations were applied to the row, either because none existed or
    // because none were relevant.
    NONE_APPLIED,

    // At least one mutation was applied to the row.
    APPLIED_AND_PRESENT,

    // At least one mutation was applied to the row, and the row's final state
    // was deleted (i.e. the last mutation was a DELETE).
    APPLIED_AND_DELETED,

    // Some mutations were applied, but the sequence of applied mutations was
    // such that clients should never see this row in their output (i.e. the row
    // was inserted and deleted in the same timestamp).
    APPLIED_AND_UNOBSERVABLE,
  };
```

### 成员

#### `memrowset_`
所属的`MemRowSet`对象。

注意`const std::shared_ptr<const MemRowSet>`类型，有两点要说明的：
1. 因为是`std::shared_ptr<>`，所以可以避免失效的问题；
2. 因为是`const`，所以无法对`MemRowSet`进行修改。

>> 问题：如何方式在当前`Iterator`正在被使用的时候，`MemRowSet`对象被析构掉？

#### `iter_`
对`CBTree`的迭代器。

#### `opts_`
迭代器的选项。

类型是`const RowIteratorOptions`，需要注意两点：
1. 是一个对象，所以在构造的时候，需要拷贝；
2. 是一个`const`的，所以该对象一旦构造，这个对象就确定了，不可再修改。

#### `projector_`
类型是`const gscoped_ptr<MRSRowProjector>`。

>> 问题：干嘛用的？

#### `delta_projector_`
类型是`DeltaProjector`.

>> 问题：干嘛用的？

#### `projection_vc_is_deleted_idx_`

>> 缩写：  
>> "vc": virtual column

在`projection schema`中，第一个类型为`IS_DELETE`的“虚拟列”的下标。

如果不存在这样的“虚拟列”，那么它的值为`kColumnNotFound`。

#### `delta_buf_`
类型是`faststring`，是一个buff，用来保存`RowChangeList`的内容。

#### `tmp_buf`
类型是`faststring`，也是一个buff，用来保存`seek`操作的目标。

#### `state_`
当前迭代器的状态。  

表示我们是否会继续扫描、是否已经扫描了最后的batch、是否已经达到了结尾、是否永远不会到达起始点 等；

#### `exclusive_upper_bound_`
扫描操作的上限边界。

类型是`boost::optional<const Slice &>`，即这个是不是一定存在的。

```
  const std::shared_ptr<const MemRowSet> memrowset_;
  gscoped_ptr<MemRowSet::MSBTIter> iter_;

  const RowIteratorOptions opts_;

  // Mapping from projected column index back to memrowset column index.
  // Relies on the MRSRowProjector interface to abstract from the two
  // different implementations of the RowProjector, which may change
  // at runtime (using vs. not using code generation).
  const gscoped_ptr<MRSRowProjector> projector_;
  DeltaProjector delta_projector_;

  const int projection_vc_is_deleted_idx_;

  faststring delta_buf_;

  faststring tmp_buf;

  ScanState state_;

  boost::optional<const Slice &> exclusive_upper_bound_;
```

### 接口列表












