[TOC]

文件：`src/kudu/common/columnblock.h`

# `class ColumnBlock`

职责：用来封装 某个列 的多个“值”。

简单的说：就是内部维护这一个`buffer`（用来真正的存放这些“值”），然后加上一些描述信息。

对比：“使用该类来封装‘这组值’” VS “一个普通的`buffer`”.

1. 在封装以后，可以添加一些关于该`buffer`的“描述信息”；
2. 而如果只使用一个`buffer`，因为它内部只是保存值，就没有任何“描述信息”了，这样不方便使用。

说明：这里的描述信息有：
1. 其中保存的‘值’的个数；
2. 该列的“数据类型”
3. 该列是否“可以为空”；

## 类型别名

```
typedef ColumnBlockCell Cell;
```

在该类中，给`ColumnBlockCell`定义了一个别名`Cell`。（应该主要是为了降低变量名的长度，方便写代码）

## 成员

本类的所有`5`个成员都是“指针”，所以本类是比较小的。 在使用该类的时候，进行“值拷贝”的代价很低。

实际上，参见`ColumnBlockCell`类，在使用本类时，就是直接进行的“值拷贝”。

### `type_`
当前`ColumnBlock`所对应的列类型。

### `null_bitmap_`
对象的`null bitmap`。

如果当前列不能为空，那么该变量的值为`nullptr`;

### `data_`
真正的“数据段”。

如果某行数据“不为`null`”，那么就可以在`data_`中读取它的值。

### `nrows_`
数据的个数（因为每行数据会保存一个‘值’，所以该值也表示“数据的行数”）。

### `arena_`
对应的“内存池”。

对于`BINARY`和`STRING`等会“间接引用”数据的类型，在`data_`中保存的是`Slice`对象，它们所“间接引用”的数据，在该“内存池”中。

```
  const TypeInfo *type_;
  uint8_t *null_bitmap_;

  uint8_t *data_;
  size_t nrows_;

  Arena *arena_;
```

## 接口列表

### 构造函数

```
  ColumnBlock(const TypeInfo* type,
              uint8_t *null_bitmap,
              void *data,
              size_t nrows,
              Arena *arena)
    : type_(type),
      null_bitmap_(null_bitmap),
      data_(reinterpret_cast<uint8_t *>(data)),
      nrows_(nrows),
      arena_(arena) {
    DCHECK(data_) << "null data";
  }
```

### `SetCellIsNull()`
将“指定行号”的数据，设置为`null`。

其实就是在“null 位图”中标记一下。

```
  void SetCellIsNull(size_t idx, bool is_null) {
    DCHECK(is_nullable());
    BitmapChange(null_bitmap_, idx, !is_null);
  }
```
### `SetCellValue()`
将“指定行号”的数据，设置为“指定的值”。

```
  void SetCellValue(size_t idx, const void *new_val) {
    strings::memcpy_inlined(mutable_cell_ptr(idx), new_val, type_->size());
  }
```

说明：所要赋的值，是通过用`void*`来传进来的。 而赋值，是直接进行“内存拷贝”（通过`strings::memcpy_inlined()`）。

也就是说，这里并没有采用如下流程：
1. 先没有将相应的值，转化为相应的类型；
2. 然后根据类型，进行赋值。

**这样“直接进行内存拷贝”的好处： 会省掉很多“分支判断”。**  

因为要支持的类型很多，如果采用上面“先转化类型，然后再‘按照类型’进行赋值”的方式，那么必然要添加很多“分支判断”。

而分支判断的坏处是：会极大的影响CPU的流水线执行效率。

### `OverwriteWithPattern()`

只在`NDEBUG`模式下使用。

> 扩展：在整个项目中，有很多类中（主要是和“内存操作”相关的类）提供了这个方法。  

该方法的操作是：手动地给一段内存进行赋值。  

**可以理解为：手动的将一段内存写脏，从而更容易的发现错误。**  

```
#ifndef NDEBUG
  void OverwriteWithPattern(size_t idx, StringPiece pattern) {
    char *col_data = reinterpret_cast<char *>(mutable_cell_ptr(idx));
    kudu::OverwriteWithPattern(col_data, type_->size(), pattern);
  }
#endif
```

### `cell_ptr()`
返回指定“序号”的‘值’。

注意该方法的使用场景：如果能够确定该列的值一定不为`null`（即`nullalbe == false`），那么可以直接使用该方法。

但是，如果不能确定该列的`nullable`的值，那么必须先判断“该序号的‘值’”是否为`null`。只要不为`null`的时候，才能调用该方法去获取它的值。（对于这种场景，更方便的方法是：直接调用`nullable_cell_ptr()`）

```
  const uint8_t *cell_ptr(size_t idx) const {
    DCHECK_LT(idx, nrows_);
    return data_ + type_->size() * idx;
  }
```

### `nullable_cell_ptr()`
返回指定“序号”的‘值’。 如果这个‘值’为`null`，那么也返回`null`。

```
  const uint8_t *nullable_cell_ptr(size_t idx) const {
    return is_null(idx) ? NULL : cell_ptr(idx);
  }
```

### `cell()`
获取对应的`Cell`对象（即`ColumnBlockCell`对象）

```
inline ColumnBlockCell ColumnBlock::cell(size_t idx) const {
  return ColumnBlockCell(*this, idx);
}
```

说明：这个方法既然已经显式定义为`inline`了，而且方法体也非常短，那么为什么要在类外单独定义呢？
原因是：该类使用到了`ColumnBlockCell`对象，所以要知道`ColumnBlockCell`的完整定义。  
而在本文件中，同时定义了`ColumnBlock`和`ColumnBlockCell`（其中，`ColumnBlockCell`的定义在后面）。  
即在`ColumnBlock`类定义的时候，是不知道`ColumnBlockCell`的完整定义的。  

如果在`ColumnBlock`类中直接写上该函数体，编译会报错。因为还不知道`ColumnBlockCell`的完整定义。

### `CopyTo()`
将当前`ColumnBlock`中封装的“这一组‘值’”，拷贝到另一个`ColumnBlock`中。

参数说明：
1. 在“源`ColumnBlock`”中的“起点”由参数`src_cell_off`指定；
2. 在“目的`ColumnBlock`”中的“起点”有参数`dst_cell_off`指定；
3. 拷贝的“‘值’的个数”由`null_cells`指定；

注意：这里的参数中，还会包含一个`SelectionVector`对象（`sel_vec`）。

背景说明：只有在`RowBlock`中，才会去使用`ClolumnBlock`对象。

**说明：要有这个参数（`sel_vec`）的作用：**  
对于那些有“间接引用”数据的类型（比如`BINARY`），在拷贝的时候，也同时需要拷贝“间接引用”的数据。

而当一个列的类型是`BINARY`，如果它的一个‘值’，是“没有被选择”的。那么，它所对应的`Slice`对象本身，
1. 可能就是一个无效的`Slice`对象。（如果被强制转化为一个`Slice`对象，那么它的指针也一定是“无效的”）；  
2. 即使是一个有效的`Slice`对象，它所“间接引用”的内存空间，也可能已经被回收了；  

所以如果这个‘值’是“没有被选择”，那么就不能去拷贝它所“间接引用”的数据。  

而如何判断一个‘值’是否为“被选择了”，就是`sel_vec`参数的作用。

综上，对于有“间接引用”的类型，只有当‘值’是被选择的时候，才会被拷贝。  
因为这里的每个‘值’，实际上就对应一个数据行。  
**也就是说，对于`BINARY`等类型，只有那些“被选择了的”数据行，才会被拷贝。**

注意：上面说的是 检查这个值 “是否被选择”了， 而不是 检查这个值 “是否为`null`”。

```
Status ColumnBlock::CopyTo(const SelectionVector& sel_vec,
                           ColumnBlock* dst, size_t src_cell_off,
                           size_t dst_cell_off, size_t num_cells) const {
  DCHECK_EQ(type_, dst->type_);
  DCHECK_EQ(is_nullable(), dst->is_nullable());
  DCHECK_GE(nrows_, src_cell_off + num_cells);
  DCHECK_GE(dst->nrows_, dst_cell_off + num_cells);

  // Columns with indirect data need to be copied cell-by-cell in order to
  // perform arena relocation. Deselected cells must be skipped; the source
  // content could be garbage so it'd be unsafe to access it as indirect data.
  if (type_->physical_type() == BINARY) {
    for (size_t cell_idx = 0; cell_idx < num_cells; cell_idx++) {
      if (sel_vec.IsRowSelected(src_cell_off + cell_idx)) {
        Cell s(cell(src_cell_off + cell_idx));
        Cell d(dst->cell(dst_cell_off + cell_idx));
        RETURN_NOT_OK(CopyCell(s, &d, dst->arena())); // Also copies nullability.
      }
    }
  } else {
    memcpy(dst->data_ + (dst_cell_off * type_->size()),
           data_ + (src_cell_off * type_->size()),
           num_cells * type_->size());
    if (null_bitmap_) {
      BitmapCopy(dst->null_bitmap_, dst_cell_off,
                 null_bitmap_, src_cell_off,
                 num_cells);
  }
}
```

说明1：进行拷贝的两个`ColumnBlock`对象，要求基本属性是一样的：
1. 数据类型；
2. `nullable`;

说明2：还有一些安全检查: “源`ColumnBlock`” 和 “目标`ColumnBlock`”中要有足够数量的‘值’ 供拷贝。  

说明3：不同的类型，有不同的拷贝方法：
1. 对于`BINARY`类型，因为同时要拷贝“间接引用”的数据，所以只能 逐个‘值’的进行拷贝。  
2. 对于其它类型，直接进行“内存拷贝”即可。  

说明4：这里的“拷贝”，要包含两部分：
1. 这个‘值’本身；
2. 还要考虑是否为`null`，来设置“`null`位图”。

### `ToString()`
将当前对象表示的“这一组‘值’”打印成字符串。

```
  std::string ToString() const {
    std::string s;
    for (int i = 0; i < nrows(); i++) {
      if (i > 0) {
        s.append(" ");
      }
      if (is_nullable() && is_null(i)) {
        s.append("NULL");
      } else {
        type_->AppendDebugStringForValue(cell_ptr(i), &s);
      }
    }
    return s;
  }
```

### 信息获取函数
```
  bool is_nullable() const {
    return null_bitmap_ != NULL;
  }

  bool is_null(size_t idx) const {
    DCHECK(is_nullable());
    DCHECK_LT(idx, nrows_);
    return !BitmapTest(null_bitmap_, idx);
  }
  
  const size_t stride() const { return type_->size(); }

```

### `getter`方法

```
  uint8_t *null_bitmap() const {
    return null_bitmap_;
  }
  
  const uint8_t * data() const { return data_; }
  uint8_t *data() { return data_; }
  
  const size_t nrows() const { return nrows_; }
  
  Arena *arena() { return arena_; }

  const TypeInfo* type_info() const {
    return type_;
  }
```

### `private mutable_cell_ptr()`
工具方法； 获得指定序号的`cell`值的指针。

```
  uint8_t *mutable_cell_ptr(size_t idx) {
    DCHECK_LT(idx, nrows_);
    return data_ + type_->size() * idx;
  }
```

# 全局函数

## 比较两个`ColumnBlock`是否相同

```
inline bool operator==(const ColumnBlock& a, const ColumnBlock& b) {
  // 1. Same number of rows.
  if (a.nrows() != b.nrows()) {
    return false;
  }

  // 2. Same nullability.
  if (a.is_nullable() != b.is_nullable()) {
    return false;
  }

  // 3. If nullable, same null bitmap contents.
  if (a.is_nullable() &&
      !BitmapEquals(a.null_bitmap(), b.null_bitmap(), a.nrows())) {
    return false;
  }

  // 4. Same data. We can't just compare the raw data because some entries may
  //    be pointers to the actual data elsewhere.
  for (int i = 0; i < a.nrows(); i++) {
    if (a.is_nullable() && a.is_null(i)) {
      continue;
    }
    if (a.type_info()->Compare(a.cell_ptr(i), b.cell_ptr(i)) != 0) {
      return false;
    }
  }

  return true;
}

inline bool operator!=(const ColumnBlock& a, const ColumnBlock& b) {
  return !(a == b);
}
```

说明：在比较两个`ColumnBlock`中所有的 “具体‘值’” 是否相等的时候，不能直接对`data_`进行“内存比较”。

原因是：如果它的类型是有“间接引用”的数据，那么其中的`Slice`对象可能分别引用的两个地址，但是具体的值是相同的。  
这种场景，应该被认为是“相等的”。但是这个时候，如果直接比较`data_`，会认为是“不相等”的。

# `class ColumnBlockCell`
该类的含义：表示在`ColumnBlock`中的一个‘值’。

注意：该类是“可拷贝的”。

因为本类是非常小的，所以在`ColumnBlock`中使用该类时，都是直接“值拷贝”的。（其实: `ColumnBlock`类也比较小，这里使用`ColumnBlock`也是直接进行“值拷贝”）。

## 成员

### `block_`
当前`cell`所对应的`ColumnBlock`对象；

### `row_idx_`
当前`cell`的序号。

```
  ColumnBlock block_;
  size_t row_idx_;
```

## 接口列表

参见`row.h`，该类就是一种`Cell`，所以需要提供对于`Cell`要提供的方法。

其实，在`ColumnBlock::CopyTo()`中，会将“本类的对象：作为使用了`CopyCell()`的参数。

```
class ColumnBlockCell {
 public:
  ColumnBlockCell(ColumnBlock block, size_t row_idx)
      : block_(block), row_idx_(row_idx) {}

  const TypeInfo* typeinfo() const { return block_.type_info(); }
  size_t size() const { return block_.type_info()->size(); }
  const void* ptr() const {
    return is_nullable() ? block_.nullable_cell_ptr(row_idx_)
      : block_.cell_ptr(row_idx_);
  }
  void* mutable_ptr() { return block_.mutable_cell_ptr(row_idx_); }
  bool is_nullable() const { return block_.is_nullable(); }
  bool is_null() const { return block_.is_null(row_idx_); }
  void set_null(bool is_null) { block_.SetCellIsNull(row_idx_, is_null); }
  
 protected:

};
```

# `class ColumnDataView`

本类的职责：封装一个`ColumnBlock`对象中的“一段数据”。

类似于：`Slice`封装了`std::string`中的一段数据。

只不过，在一个`ColumnDataView`对象中，只有“起始点”，没有指定“终点”，即是封装了在一个`ColumnBlock`对象中，从指定的“起始点”，到终点的数据。

```
// 1. `Slice`可以表示在`std::string`的任意一段值。

   std::string的值          |----------------------------------|
                            |<-- slice_1 -->|  
                                    |<-- slice_2 -->|
                                               |<-- slice_3 -->| 
                                               
// 2. ColumnDataView 表示的是 在 ColumnBlock 中 从“指定位置” 到 终点 的值。
   
   ColumnBlock中的‘值’:     |----------------------------------|
                                |<-- column_block_view_1------>|  
                                   |<-- column_block_view_2 -->|  
```

**说明：这个类的使用场景：**  
在从`cfile`中“读取”和“写入”数据时，会使用这个类。

>> 问题：如何使用的？

## 成员

### `column_block_`
对应的`ColumnBlock`对象；

### `row_offset_`
当前对象所表示的‘值’，在`ColumnBlock`对象中的 起点下标。

```
  ColumnBlock *column_block_;
  size_t row_offset_;
```

## 接口列表

### `构造函数`
```
  explicit ColumnDataView(ColumnBlock *column_block, size_t first_row_idx = 0)
    : column_block_(column_block), row_offset_(0) {
    Advance(first_row_idx);
  }
```

### `Advance()`
前进“指定的长度”，即跳过“指定数量”的‘值’。

```
  void Advance(size_t skip) {
    // Check <= here, not <, since you can skip to
    // the very end of the data (leaving an empty block)
    DCHECK_LE(skip, column_block_->nrows());
    row_offset_ += skip;
  }
```

### `SetNullBits()`
设置“一段‘值’”是否为`null`。

在实现时，只需要修改“`null`位图”。

```
  // Set 'nrows' bits of the the null-bitmap to "value"
  // true if null, false if not null.
  void SetNullBits(size_t nrows, bool value) {
    BitmapChangeBits(column_block_->null_bitmap(), row_offset_, nrows, value);
  }
```

### 信息获取函数

```
  size_t first_row_index() const {
    return row_offset_;
  }

  uint8_t *data() {
    return column_block_->mutable_cell_ptr(row_offset_);
  }

  const uint8_t *data() const {
    return column_block_->cell_ptr(row_offset_);
  }
  
  Arena *arena() { return column_block_->arena(); }

  size_t nrows() const {
    return column_block_->nrows() - row_offset_;
  }

  const size_t stride() const {
    return column_block_->stride();
  }

  const TypeInfo* type_info() const {
    return column_block_->type_info();
  }
```

# `template<> ScopedColumnBlock`
工具类。实际上就是一个内存是由自己维护的`ColumnBlock`类。

在该类内部，会申请并维护一段内存，用来模拟`ColumnBlock`中“指针成员”所引用的数据。

在该类析构时，会自动释放这些内存。（这也是该类名字中有`"Scoped"`的原因）。

说明：这个类，是为了方便写单测（当前也只在“单测”中使用）。  
因为它的内存是自己维护的，不是从“内存池”中申请的。

**说明：本类的实现方式**  
相比于`ColumnBlock`类（它的成员都是“指针”），这些成员所指向的“数据地址”都是外部先构建好（一般是在“内存池”中构建的），然后传递个`ColumnBlock`的。

而在本类中，所有的内存都是自己维护的。

为了能够和`ColumnBlock`有相同的接口，这个类是通过“继承`ColumnBlock`”来实现的。

在继承以后，自然也就有了父类`ColumnBlock`的所有公开方法。 

但是因为在`ColumnBlock`中，它的方法，都是访问`ColumnBlock`自己的成员属性。而这里为了让这些父类的方法，能够使用 自己维护内存的成员，**将这些属性，使用相同的变量名字，又都重新定义了一遍**。这样 子类中定义的这些成员属性，就覆盖掉了在父类中的成员属性。 在父类的方法进行访问一个变量时（加入名字为`data_`），实际上访问的“子类”中定义的那一个。

总结：使用了`2`个技巧：  
1. 通过继承`ColumnBlock`类，拥有了相同的接口；
2. 通过 使用相同的变量名，覆盖掉了 父类的相关属性；（那些需要“维护内存”的属性）

```
template<DataType type>
class ScopedColumnBlock : public ColumnBlock {
 public:
  typedef typename TypeTraits<type>::cpp_type cpp_type;

  explicit ScopedColumnBlock(size_t n_rows, bool allow_nulls = true)
    : ColumnBlock(GetTypeInfo(type),
                  allow_nulls ? new uint8_t[BitmapSize(n_rows)] : nullptr,
                  new cpp_type[n_rows],
                  n_rows,
                  new Arena(1024)),
      null_bitmap_(null_bitmap()),
      data_(reinterpret_cast<cpp_type *>(data())),
      arena_(arena()) {
    if (allow_nulls) {
      // All rows begin not null.
      BitmapChangeBits(null_bitmap(), /*offset=*/ 0, n_rows, /*value=*/ false);
    }
  }

  const cpp_type &operator[](size_t idx) const {
    return data_[idx];
  }

  cpp_type &operator[](size_t idx) {
    return data_[idx];
  }

 private:
  gscoped_array<uint8_t> null_bitmap_;
  gscoped_array<cpp_type> data_;
  gscoped_ptr<Arena> arena_;

};
```



