[TOC]

文件：`src/kudu/common/row.h`

## 名字解释

`cell`: 指一行数据中的一个列，即一个table中的一个“单元格”。

## `struct SimpleConstCell`

描述一个常量的“单元格”。

成员包括该“单元格”所对应的“元信息”和“值”，即`ColumnSchema`和`value`。

注意：`col_schema_`和`value_`都是指针，所以调用者需要保证，在该`SimpleConstCell`的生命周期类，它包含的`col_schema_`和`value_`都是有效的。

```
struct SimpleConstCell {
 public:
  // Both parameters must remain valid for the lifetime of the cell object.
  SimpleConstCell(const ColumnSchema* col_schema,
                  const void* value)
    : col_schema_(col_schema),
      value_(value) {
  }

  const TypeInfo* typeinfo() const { return col_schema_->type_info(); }
  size_t size() const { return col_schema_->type_info()->size(); }
  bool is_nullable() const { return col_schema_->is_nullable(); }
  const void* ptr() const { return value_; }
  bool is_null() const { return value_ == NULL; }

 private:
  const ColumnSchema* col_schema_;
  const void* value_;
};
```

注意1：在这里`value_`的类型是`void*`类型。它所指向的实际类型，是由该“单元格”的类型决定的。 具体的，就是每个类型所对应的`DataTypeTraits<T>::cpp_type`类型。

这里要注意的是：对于`BINARY`和`STRING`类型，他们对应的类型是`Slice`类型。

注意2: 要区分 “schema信息中的`is_nullable()`”和“数据中的`is_null()`”。  
+ `is_nullable()`是描述 元信息，表示当前列是否“可以为空”；
+ `is_null()`是在描述 数据，表示当前“单元格”是否为空。

只有一个 元信息 中 `nullable==true`的列，它的单元格的`is_null()` 才可能为`true`。

但是`nullable == true`，不表示`is_null` 一定为"true"。

注意3： 对于“可以被修改”（mutable）的“单元格”，还有一个成员方法是`mutable_ptr()`。 但是因为`SimpleConstCell`是不可以被修改的（它的值，只有在创建的时候指定。一旦指定，就不可以再被修改），所以它没有这个成员方法。

## `CopyCellData()`  -- 全局函数

模板函数。将“单元格”的内容，从`src`拷贝到`dst`。

模板删除类型(`Cell`)，需要支持的接口，参见上面的`SimpleConstCell`类。

如果值的类型是`Slice`对象（即当前column的类型为`STRING`或`BINARY`）:
1. 如果参数`dst_arena`不是`nullptr`，那么会重新会为`Slice`和`指向的数据`重新分配内存;
2. 如果参数`dst_arena`为`nullptr`，那么不进行重新分配内容，而是直接将`dest Slice`赋值给`src Slice`；在赋值以后，"src Slice"中的指针，就和"desc Slice"指向相同的位置。所以调用者，必须保证指针的有效性。   
  一般这种使用方式，都是发生在“src Slice”的生命周期和"desc Slice"是相同的。所以就可以通过不重新分配内存，来提高效率。

>> stick around: 在附近逗留； 这里指，在赋值以后，"src slice"和"desc slice"的生命周期是一样的。

### `CopyCellData()` VS `CopyData()`  
1. `CopyCellData()`: 只拷贝数据，但是并不拷贝"null"状态;
2. `CopyData()`: 即拷贝数据，也拷贝"null"状态。

>> 问题： 为什么要分为两种情况？ 难道拷贝不是要么就全部拷贝（包括null状态），要么就不拷贝null状态吗？ 也就是不是应该所有的地方都统一的吗？  难道是为了，在从schema可以判断列不可以为空（`is_nullable==false`）的情况下，就可以直接调用`CopyCellData()`，减少了在内部对`null`的判断？ 

说明：需要分为两种情况的原因。

实际上，在现在的代码实现中，值在`CopyCell()`中调用了`CopyCellData()`（在判断完当前`cell`不为`null`以后）。

[猜测]所以，这里分为两种情况，应该有两个原因：
为了在已经明确`cell`不会为`null`的地方（即已经明确`is_nullable==false`），可以直接调用`CopyCellData()`；（但是如上所述，实际上现在没有这么用）。

目前的实现中，所有行的开始位置，都会有有一个`null bitmap`，即使这个`Row`中所有列都不可为`null`。

```
template <class SrcCellType, class DstCellType, class ArenaType>
Status CopyCellData(const SrcCellType &src, DstCellType* dst, ArenaType *dst_arena) {
  DCHECK_EQ(src.typeinfo()->type(), dst->typeinfo()->type());

  if (src.typeinfo()->physical_type() == BINARY) {
    const Slice *src_slice = reinterpret_cast<const Slice *>(src.ptr());
    Slice *dst_slice = reinterpret_cast<Slice *>(dst->mutable_ptr());
    if (dst_arena != NULL) {
      if (PREDICT_FALSE(!dst_arena->RelocateSlice(*src_slice, dst_slice))) {
        return Status::IOError("out of memory copying slice", src_slice->ToString());
      }
    } else {
      // Just copy the slice without relocating.
      // This is used by callers who know that the source row's data is going
      // to stick around for the scope of the destination.
      *dst_slice = *src_slice;
    }
  } else {
    memcpy(dst->mutable_ptr(), src.ptr(), src.size()); // TODO: inline?
  }
  return Status::OK();
}

```

>> 问题： 这里在`memcpy()`的那一行后面，有个`TODO: inline`的注释？ 这里怎么样才能将 这一个分支（即pyhsical_type不是`BINARY`的分支）搞成`inline`的？  是直接封装成一个函数？ 还是说要利用模板编程，把非`BINARY`类型的函数特化版本 写成 inline的方式？ 

## `CopyCell()`  -- 全局函数

模板函数。也是拷贝“单元格”的内容。但是会检查“是否可为`null`”。

如果为null,那么不需要进行任何拷贝。

```
template <class SrcCellType, class DstCellType, class ArenaType>
Status CopyCell(const SrcCellType &src, DstCellType* dst, ArenaType *dst_arena) {
  if (src.is_nullable()) {
    dst->set_null(src.is_null());
    if (src.is_null()) {
      return Status::OK();
    }
  }

  return CopyCellData(src, dst, dst_arena);
}
```

## `CopyRow()`

模板函数。拷贝整行数据。

注意："src row"和"dest row"必须具有相同的`Schema`。

>> 扩展： 如果两个`Row`的schema不同，那么只能用`ProjectRow()`来进行数据拷贝（不能用`CopyRow()`）。

对于值为`Slice`的类型，那么和`CopyData()`一样，如果参数中提供的“内存池”不是`nullptr`，那么会在其中重新分配内存，并进行数据拷贝。

>> 问题："This can be used to translate between columnar and row-wise layout, for example."? 这句话是什么意思? 在“行式”和“列式”之间转换？

## `Clss RowProjector`

用来描述在特定`Schema`上的一个投影。

在创建该对象的时候，会传递两个`Schema`对象进来。1) base schema; 2) 投影的`Schema`。

有了该`RowProjector`对象以后，给定一个`Row`（对应`base schema`），可以将它的数据，投影到一个新的`Row`上（参见`ProjectRow()`方法）。

使用场景举例：
在读取的时候，用户提供的schema作为“投影schema”, 引擎中表的shema是“base schema”。在读取的时候，在引擎中存储的每一行数据（对应`base schema`）,使用该类将就是将“引擎中的`Row`” 映射到 “projection schema”对应的`Row`上(查询结果)。

问题：在写入的时候，用户提供的schema，和 引擎中表的schema ，谁作为作为“base schema”，谁是“投影schema”？

>> 问题：确认下上面的“使用场景”是否正确？

投影中的列，与`base schema`中的列相比，可能会有以下3种情况：
1. 投影中的列，在`base schema`中也存在，并且类型也一样；
2. 投影中的列，在`base schema`中也存在，但是“类型”是不同的。  
    这种情况下，需要用一个`adapter`进行类型的转换。比如说：将`INT8`转换为`INT64`，`INT8`转换为`STRING`等；（因为目前`Kudu`并不支持`adapter`，所以这种情况会报错。）
3. 投影中的列，在`base schema`中并不存在。
    这种情况下，投影中的列，使用它的“默认值”。

### 使用举例

```
// Example:
RowProjector projector(base_schema, projection);
projector.Init();
projector.ProjectRow(row_a, &row_b, &row_b_arena);
```

>> 注意： “投影”本质上也是一个`Schema`对象。

### 成员

注意：因为`base_schema_`和`projection_`都是指针类型，所以“调用者”要保证在当前`RowProjector`对象的生命周期内，这两个指针的有效性。

#### `base_schema_`和`projection_`

类型都是`Schema*`。表示“投影”的`schema`和其对应的`base Schema`。

#### `is_identity_`

表示`base_schema_`和`projection_`是否相等。

通过在构造该对象的时候，缓存是否相等，那么后续在判断的时候，就不需要每次在重新比较。

>> 问题：“相等”和“不相等”，处理方式有什么不同？

#### `base_cols_mapping_`

`std::vector<std::pair<size_t, size_t>>`类型，用来表示一个映射关系：从“投影”中的列 映射到 “base schema”中的列。
1. key是在“投影”(`Schema`对象)中的 column 序号；
2. value是在`base Schema`对象中的 column 序号；

注意：只有在`projection schema`和`base schema`中完全相同的列(`ColumnSchema`)，才会被添加到`base_cols_mapping_`中。

比较两列的方式是，比较两个`ColumnSchema`的类型（不比较`ColumnSchema`的其它成员）。

#### `projection_defaults_`

`std::vector<size_t>`类型。

一个列，会被添加进来的场景：1) 该列在当前`projection schema`中存在，但是在`base schema`中不存在；2) 该列在`projection schema`中有“默认值” 或者 “可以为空”。

其中每个元素，表示的是 这个列在 `projection schema`中的序号。（因为这个column只在`projection schema`中存在，所以这个序号不可能是在`base schema`中的序号，一定是在`projection schema`中的）

**注意：`projection_defaults_`和`base_cols_mapping_`中的成员是可以覆盖在`projection schema`中的所有列，并且两者之间 没有重复**  

因为这两个成员 所对应的场景是不同的。而`projection schema`中的每一列，只能属于一种情况。即一个列，会且仅会 被添加到其中一个成员中。

```
  typedef std::pair<size_t, size_t> ProjectionIdxMapping;
  std::vector<ProjectionIdxMapping> base_cols_mapping_;
  std::vector<size_t> projection_defaults_;

  const Schema* base_schema_;
  const Schema* projection_;
  bool is_identity_;
```

### 接口列表

#### `ProjectRowForRead()`

在**读取场景**下，按照当前“投影”的schema，将一个`Row` 投影到 另一个`Row`中。

内部是调用`ProjectRow()`来实现的。

#### `ProjectRowForWrite()`

在**写入场景**下，按照当前“投影”的schema，将一个`Row` 投影到 另一个`Row`中。

内部是调用`ProjectRow()`来实现的。

#### `private ProjectBaseColumn()`

在“投影”和“base schema”中都存在的列，将这个映射关系添加到`base_cols_mapping_`中。

#### `private ProjectDefaultColumn()`

在“投影”中的列，在`base schema`中不存在。并且该列 有“默认值”，或者是"nullable"的。

虽然在“base schema”中不存在，但是因为有默认值，或者可以为"null"，所以在“投影”时，仍然是可以在“dest row”中给这个列填充值的。

将这个列的序号，添加到`projection_defaults_`中。

#### `private ProjectExtraColumn()`

在“投影”中的列，在`base schema`中不存在。并且该列既没有默认值，也不是"nullable"的。

因为列不存在，而且也没有默认值，也不是"nullable"的，所以就没有办法在“目标row”中进行填充它的值。**所以如果出现这种情况，就会报错**。

#### `private ProjectRow()`

模板函数，也是本类最关键的函数。

按照当前“投影”的schema，将一个`Row` 投影到 另一个`Row`中。

该函数的其中一个模板是`bool FOR_READ`，根据是否为用来读取，该函数会被两个函数调用：`ProjectRowForRead()`和`ProjectRowForWrite()`。

**说明：为什么需要区别是“在读取场景”还是“在写入场景”**  
因为默认值有两种：`read_default_value`和`write_default_value`。 同时在需要默认值进行填充时，说明该列一定属于`projectiong_defaults_`成员中。
+ 在读取场景：使用`read_default_value`；
+ 在写入场景：使用`write_default_value`；

如果提供了`dst arena`，那么间接引用的数据，会进行拷贝。（`STRING`和`BINARY`类型中，所保存的具体数据）

```
  template<class RowType1, class RowType2, class ArenaType, bool FOR_READ>
  Status ProjectRow(const RowType1& src_row, RowType2 *dst_row, ArenaType *dst_arena) const {
    DCHECK_SCHEMA_EQ(*base_schema_, *src_row.schema());
    DCHECK_SCHEMA_EQ(*projection_, *dst_row->schema());

    // Copy directly from base Data
    for (const auto& base_mapping : base_cols_mapping_) {
      typename RowType1::Cell src_cell = src_row.cell(base_mapping.second);
      typename RowType2::Cell dst_cell = dst_row->cell(base_mapping.first);
      RETURN_NOT_OK(CopyCell(src_cell, &dst_cell, dst_arena));
    }

    // Fill with Defaults
    for (auto proj_idx : projection_defaults_) {
      const ColumnSchema& col_proj = projection_->column(proj_idx);
      const void *vdefault = FOR_READ ? col_proj.read_default_value() :
                                        col_proj.write_default_value();
      SimpleConstCell src_cell(&col_proj, vdefault);
      typename RowType2::Cell dst_cell = dst_row->cell(proj_idx);
      RETURN_NOT_OK(CopyCell(src_cell, &dst_cell, dst_arena));
    }

    return Status::OK();
  }
```

如上面所述：`projection_defaults_`和`base_cols_mapping_`中的成员是可以覆盖在`projection schema`中的所有列，并且两者之间 没有重复。

说明：这里的逻辑，使用`CopyCell()`，来分别将 `projection_defaults_`和`base_cols_mapping_` 中的列都进行拷贝。

所以在将这两部分都调用`CopyCell()`以后，所有的列都会被赋值(“投影”)到“目标
`Row`”中。并且，因为两个成员之间是不重复的，所以任何一列都不会被拷贝两次。

## `class DeltaProjector`

>> 问题：待补充


## `RelocateIndirectDataToArena()`  -- 全局函数

将一个`Row`中 间接引用的数据（例如`STRING`类型指向的数据），拷贝到指定的`内存池`中。

注意1：在执行该方法后，参数中的`Row`会被修改：它所 间接引用数据的指针，会指向新申请的地址。

注意2：在实现该方法时，应该先计算出总共要申请的内存大小（通过遍历，找到所有`physical_type`为`BINARY`的列，实际类型为`BINARY`或`STRING`），然后一次性申请。而不是对于每个需要拷贝的列，都逐个单独申请。


**说明：为什么要“先计算总大小，然后一次性申请”**  
虽然从“内存池”中申请内存的代价很低，每次申请基本上有一个`CAS`操作。

因为`Row`中可能含有数百个`STRING`的列，如果每个列都单独申请，尤其在多个线程同时操作同一个“内存池”的情况下，竞争也就不可忽略了。

所以，应该是“先计算出总大小，然后一次性申请”，这也能很大程度上减少竞争。

```
template <class RowType, class ArenaType>
inline Status RelocateIndirectDataToArena(RowType *row, ArenaType *dst_arena) {
  const Schema* schema = row->schema();
  // First calculate the total size we'll need to allocate in the arena.
  int size = 0;
  for (int i = 0; i < schema->num_columns(); i++) {
    typename RowType::Cell cell = row->cell(i);
    if (cell.typeinfo()->physical_type() == BINARY) {
      if (cell.is_nullable() && cell.is_null()) {
        continue;
      }

      const Slice *slice = reinterpret_cast<const Slice *>(cell.ptr());
      size += slice->size();
    }
  }
  if (size == 0) return Status::OK();

  // Then allocate it in one shot and copy the actual data.
  // Even though Arena allocation is cheap, a row may have hundreds of
  // small string columns and each operation is at least one CAS. With
  // many concurrent threads copying into a single arena, this avoids
  // a lot of contention.
  uint8_t* dst = static_cast<uint8_t*>(dst_arena->AllocateBytes(size));
  for (int i = 0; i < schema->num_columns(); i++) {
    typename RowType::Cell cell = row->cell(i);
    if (cell.typeinfo()->physical_type() == BINARY) {
      if (cell.is_nullable() && cell.is_null()) {
        continue;
      }

      Slice *slice = reinterpret_cast<Slice *>(cell.mutable_ptr());
      slice->relocate(dst);
      dst += slice->size();
    }
  }
  return Status::OK();
}
```

## `class ContiguousRowHelper`

>> contiguous: 连续的，相连的；邻近的；  

工具类。

该方法中定义了一系列的`static`方法。调用者可以通过传入“`Schema`对象”和一些其它参数（比如 一个`Row`的数据所对应的一段连续内存的首指针，列的序号等），来调用这些`static`方法。

### 一行数据的编码内容

对应的方法为`ContiguousRowHelper::row_size()`，其中包含两部分: 
1. `null bitmap`的大小； 
2. `Schema`本身需要的大小；

注意：如果一个`Schema`中所有列都不可为`null`，那么这行数据是没有`null bitmap`的。(参见`ContiguousRowHelper::null_bitmap_size()`)。

### `null bitmap`在“行数据”的尾部

参见`ContiguousRowHelper::null_bitmap_ptr()`，计算方法是`row_data + schema.byte_size()`，即`null bitmap`的偏移量，是在“行数据”的后面。

### 如何存储值为`null`的列

如果一列为`null`，那么该列在数据中就什么都不存，只在`null bitmap`中有一个标识。

参见`ContiguousRowHelper::nullable_cell_ptr()`

**注意： 单就“数据段”而言，它的长度仅仅由 列的类型决定，与“是否为`nullable`”是无关的。**

所以，即使在存储的时候，对于一行数据，对于其中为`null`的列，**在“数据段”中仍然有它的内存位置，只是未被初始化**（因为在访问时，会先通过`null bitmap`进行判断是否为`null`，如果为`null`，就直接返回了）。

注意： 对于`nullable`的列，获取它所对应的“单元格”(`cell`)的指针时，一定要使用`nullable_cell_ptr()`，不能直接用`cell_ptr()`。

如果一列的值为`null`，但这时却使用`cell_ptr()`来获取，那么会获取到一段“没有被初始化的”地址空间，这是错误的。

如上，也是`nullable_cell_ptr()`和`cell_ptr()`在用法上的区别。

```
      | <----------- 数据段 ------------->|<- bitmap -> |
                                            _1 _2 _3 _4
      |--------|xxxxxxxx|--------|xxxxxxxx|  0  1  0  1 |
      ^        ^        ^        ^
      |        |        |        |
    col_1   col_2     col_3    col_4
   
   "_1"表示 col_1 在null bitmap中对应的bit位
   
   因为col_2 和 col_4 的值为null，所以在“数据段”中它们对应的位置是“未初始化”的。
```

## `class ContiguousRowCell`

模板类。定义了 一个`ContiguousRow`类型的row的“单元格”。

模板参数是`RowType`，即会有多个种类的`Row`类型。

### 参数

#### `row_`

该“单元格”所属的`ContiguousRowType`指针。

#### `col_idx_`

当前“单元格”在`Row`中的序号（顺序号，下标）。

```
  const ContiguousRowType* row_;
  int col_idx_;
```

### 接口列表

提供了获取该“单元格”的 元信息 和 数据内容的多个接口。

注意：相比于`SimpleConstCell`类，多了两个方法：1) `mutable_ptr()`; 2) `set_null()`；

原因是：因为这两个方法，都是要修改当前“单元格”的内容的。 而`SimpleConstCell`类是不可修改的，所以没有这两个方法。

### 对于模板参数的要求

模板参数是一个`Row`对象，表示“一行数据”。

当前的`Row`有以下几种：
1. `ContiguousRow`;
2. `ConstContiguousRow`
2. `MRSRow`;

因为是模板参数，所以这些名字是“固定的”。

接口         | 说明
----------   |---
`cell_ptr()` | 返回“单元格”的值的指针。返回的是`const void*`，不可修改。
`mutable_cell_ptr()` |  返回“单元格”的值的指针。返回的是`void*`，是可以修改的。
`schema()`   | 返回一个`Schema`指针。当前“单元格”所属的`Row`的schema。
`is_null()`  | 当前“单元格”是否为`null`
`set_null()` | 将当前“单元格”的值设置为`null` 

## `class ContiguousRow`

一种`Row`，在其中，所有列的数据都在内存中。 每列数据的偏移量可以通过`schema.column_offset()`查看。

直观上，在一个任意类型的`Row`中，都一定会有两个属性: 1) 当前row的schema信息；2) 具体的数据；

当前类`ContiguourRow`，作为最简单的`Row`，直接使用两个成员来表示这两个信息：
1. `schema_`: 表示当前行所对应的`Schema`信息。
2. `row_data_`: 具体包含的数据。

说明：关于`row_data_`中数据的编码方法，参见`ContiguousRowHelper`部分的说明。

注意：对于`BINARY`和`STRING`类型的列，他们所间接引用的数据地址，并没有包含在该类中。

在当前类型的row(`ContiguousRow`)中的“单元格”，就是`ContiguousRowCell`类型。

在该类中，定义了很多方法，基本都是读取或者设置“数据”的，基本都是通过`ContiguousRowHelpter`类的`static`方法来实现的。

注意：在这里，只能修改“数据”，是不会再去修改`schema`对象了（`schema`属于“元数据”，不属于“数据”）。  

因为无论是“读流程”还是“写流程”，都是要**1) 先准备schema、然后 2) 进行读写数据**。  
一旦到了“要读写数据”的阶段，那么`schema`一定是已经准备好了的
即这时`schema`都是固定的了，不会被修改（所以在`ContiguourRowHelper`类中的函数中，都是传递的`const Schema& schema`类型，即常量类型）。

### `ContiguousRowCell<T>`需要的方法

参见上面的`ContiguousRowCell<T>`部分。

### 其它需要的方法

这些方法是所有类型的`Row`都要提供的。

接口         | 说明
----------   |---
`nullable_cell_ptr()` | 该单元格如果为`null`，那么返回`null`; <br/> 否则返回“单元格”的值的指针。返回的是`const void*`，不可修改。
`cell()` | 返回一个`ContiguousRowCell<T>`对象

## `class ConstContiguousRow`

这个类和`ContiguousRow`很像。 区别在于它引用的内存是一个常量区域，即**这个`Row`的数据不会被修改**。

注意：在实现时，因为`ContiguousRowCell<RowType>`是一个模板类，代码中利用了一个技巧，来防止在将`ConstContiguousRow`错误的作为`ContiguousRowCell<T>`的模板参数。（在`ContiguousRowCell<T>`中，是提供对`Cell`尽心修改的方法的）。

技巧是：对于那些能修改内容的方法，在`ConstContiguousRow`中压根没有声明。所以，如果其它地方想调用`ConstContiguousRow>`的这些方法，在编译时就会报错，因为不存在。

说明：可能代码中有一些“间接”的调用，比如有些“模板类”的模板参数是一个`RowType`，其中该模板内的部分方法会调用`RowType`的一些方法，比如`set_null()`。也就是说：既想让`ConstContiguousRow`能够作为这些“模板类”的模板参数，又不想在`ConstContiguousRow`中提供会修改其内容的方法。

另外，在当前的代码中，也通过“显式声明模板特化，但不提供实现”的方式，对相关的方法进行了"delete"，如果代码中错误的使用了，那么会编译会报错。

```
// Delete functions from ContiguousRowCell that can mutate the cell by
// specializing for ConstContiguousRow.
template<>
void* ContiguousRowCell<ConstContiguousRow>::mutable_ptr() const;
template<>
void ContiguousRowCell<ConstContiguousRow>::set_null(bool null) const;
```

说明：上述代码声明，显式的进行了模板特化的声明：将`ConstContiguousRow`作为模板参数，传递给`ContiguousRowCell`。但是对于这两个方法，并没有提供具体的实现，这样可以避免被误用。 

如果代码中调用了`ContiguousRowCell<ConstContiguousRow>::mutable_ptr()`, 那么会在编译时因为找不到具体的函数实现，而报错。

>> 问题：如果不显式声明上述两个方法，那么`ContiguousRowCell<ConstContiguousRow>`是不是会特化失败？

在`ContiguousRowCell`类中，有两个方法可能会修改对应的`Row`。
1. `mutable_ptr()`;
2. `set_null()`;

所以在`ConstContiguousRow`类中，没有的函数是：
1. `mutable_cell_ptr()`
2. `set_null()`

## `class Builider`

工具类。 用来按照跟定的`Schema`对象，构建出来`Row`对象。

这个类，只在 单测 中使用，不详细说了。







