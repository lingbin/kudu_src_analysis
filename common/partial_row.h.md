[TOC]

文件：`src/kudu/common/partial_row.h`

# `class KuduPartialRow`

和`ConstContiguousRow`一样，是一种`Row`。

注意：正如该类的名字，这是一个“不完整”的`Row`。所以
1. 对于对于接受模板参数为`RowType`的函数，一般是不可以直接传入该类型的；
2. 在对其中的数据进行修改时，一般是通过临时构建出`ContiguousRow`对象，然后利用`ContiguousRowCell<ContiguousRow>`对它进行修改。

和`ContiguourRow`不同的是，在`KuduPartialRow`中，可能只包含“部分列”的数据（即有些列的值是存在的，有些列还没有被赋值）

所以，在该列中，除了像`ContiguourRow`那样，要包含一个`uint8_t* row_data_`以外，还包含一个`isset_bitmap_`（表示哪些列已经被赋值）。

另外，在当前类中，对于“变长类型”的列(只有两种类型：`BINARY`或者`STRING`)，可能会直接维护那些“间接引用的数据”。（参见`owned_strings_bitmap_`）。 （如果直接维护，即这些“间接引用”的数据，是直接在本类中`new`出来的，所以要负责这些数据的析构）

## 成员

### `schema_`
表示当前行所对应的`Schema`信息。

### `isset_bitmap_`
一个`位图`，表示其中的列是否已经被赋值。

这个“位图”的长度，是由当前`Schema`对象中column的数量决定的。

**这个变量的用途：**  
为了与`NULL`值进行区分。

如果一列的值为`null`：
1. 如果在`isset_bitmap_`中对应的位值为`0`，即它没有被显式赋值，那么在`server`端处理“插入”请求时，会使用“默认值”。
2. 如果在`isset_bitmap_`中对应的位值为`0`，即它是被显式赋值的，那么那么在`server`端处理“插入”请求时，就使用`null`。

说明：在其中的每个列，设置时是根据“该列在`Schema`中的顺序号”进行设置的。

**和`row_data_`中的`null bitmap`的区别**  
在`row_data`中，也有一个`null bitmap`，对于一个“可为`null`”的列，如果它的值为`null`,那么在`null bitmap`中的相应bit位，会被设置为`1`。

这两者的区别是：
+ `isset_bitmap_`强调的是，该列的值“是否已经设置”。即使被设置为`null`，该列的值也是被设置过了，即在`isset_bitmap_`中的相应bit位的值会为1。
+ `row_data_`中的`null bitmap`，强调的是，该列的是否为`null`。

### `owned_strings_bitmap_`

一个`位图`，对于“变长”的列，表示它们所间接引用的数据，是否在由当前`PartitionRow`对象所维护。

这个“位图”的长度，是由当前`Schema`对象中column的数量决定的。和`isset_bitmap_`的大小是相同的。

注意：对于在这个位图中设置为`1`的列，即这些变长的列，所间接应用的数据，是有当前对象来维护，所以
1. 在对这些列进行重新赋值的时候：需要将当时“间接引用”数据析构掉；并且重新申请内存。
2. 在析构`KuduPartialRow`对象时，也需要将这些列，所间接引用的数据，析构掉(参见`DeallocateOwnedStrings()`)。

说明1：和`isset_bitmap_`一样，在其中的每个列，设置时是根据“该列在`Schema`中的顺序号”进行设置的。

说明2: 如果一个列在`isset_bitmap_`中设置，那么在`isset_bitmap_`中一定是已经设置过了。

### `row_data_`
`Row`的具体数据内容。

```
  const Schema* schema_;

  uint8_t* isset_bitmap_;

  // 1-bit set for any variable length columns whose memory is managed by this instance.
  // These strings need to be deallocated whenever the value is reset,
  // or when the instance is destructed.
  uint8_t* owned_strings_bitmap_;

  // The normal "contiguous row" format row data. Any column whose data is unset
  // or NULL can have undefined bytes.
  uint8_t* row_data_;
```

## 接口列表

### 构造函数

```
所申请内存的分布方式：

        |<---------------- 申请的内存 --------------------------->|  
        |                                                         |
        |-------------+-------------+-----------------------------|
        ^             ^             ^
        |             |             |
    isset_bitmap_     |          row_data_
                  owned_strings_bitmap_
```

```
KuduPartialRow::KuduPartialRow(const Schema* schema)
  : schema_(schema) {
  DCHECK(schema_->initialized());
  size_t column_bitmap_size = BitmapSize(schema_->num_columns());
  size_t row_size = ContiguousRowHelper::row_size(*schema);

  auto dst = new uint8_t[2 * column_bitmap_size + row_size];
  isset_bitmap_ = dst;
  owned_strings_bitmap_ = isset_bitmap_ + column_bitmap_size;

  memset(isset_bitmap_, 0, 2 * column_bitmap_size);

  row_data_ = owned_strings_bitmap_ + column_bitmap_size;
#ifndef NDEBUG
  OverwriteWithPattern(reinterpret_cast<char*>(row_data_),
                       row_size, "NEWNEWNEWNEWNEW");
#endif
  ContiguousRowHelper::InitNullsBitmap(
    *schema_, row_data_, ContiguousRowHelper::null_bitmap_size(*schema_));
}
```

### 析构函数

```
KuduPartialRow::~KuduPartialRow() {
  DeallocateOwnedStrings();
  // Both the row data and bitmap came from the same allocation.
  // The bitmap is at the start of it.
  delete [] isset_bitmap_;
}
```

**为什么这里只`delete[]`掉`isset_bitmap_`, 而不用清理`owned_strings_bitmap_`和`row_data_`**    
参见`构造函数`，因为 `isset_bitmap_`, `owned_strings_bitmap_`, `row_data_`这3个成员的内存，是一次性分配出来的，不是分3次申请出来的。

所以在释放的时候，只需要用“所申请内存的首地址”来手动释放一次即可。

而`isset_bitmap_`指向的，就是这块地址的首地址。所以只需要`delete[] isset_bitmap_`即可。

### `private DeallocateOwnedStrings()`

对于那些变长类型的列，如果“间接引用”的数据 是由当前`KuduPartialRow`对象来维护的，由这个函数来负责将它们析构掉。

```

void KuduPartialRow::DeallocateStringIfSet(int col_idx, const ColumnSchema& col) {
  if (BitmapTest(owned_strings_bitmap_, col_idx)) {
    ContiguousRow row(schema_, row_data_);
    const Slice* dst;
    if (col.type_info()->type() == BINARY) {
      dst = schema_->ExtractColumnFromRow<BINARY>(row, col_idx);
    } else {
      CHECK(col.type_info()->type() == STRING);
      dst = schema_->ExtractColumnFromRow<STRING>(row, col_idx);
    }
    delete [] dst->data();
    BitmapClear(owned_strings_bitmap_, col_idx);
  }
}

void KuduPartialRow::DeallocateOwnedStrings() {
  for (int i = 0; i < schema_->num_columns(); i++) {
    DeallocateStringIfSet(i, schema_->column(i));
  }
}
```

注意：该方法只会修改`owned_strings_bitmap_`，但是**不会修改** `isset_bitmap_`。

### 各种类型的`Set`方法

作用是：指定一个列，根据它的类型，向它赋值。

注意：“指定一个列”的方式有两种：1) 指定`column_name`; 2) 指定列的顺序号(`column_idx`)（注意：不是`column id`）。

#### 根据`column_name`进行赋值
```
  Status SetBool(const Slice& col_name, bool val) WARN_UNUSED_RESULT;

  Status SetInt8(const Slice& col_name, int8_t val) WARN_UNUSED_RESULT;
  Status SetInt16(const Slice& col_name, int16_t val) WARN_UNUSED_RESULT;
  Status SetInt32(const Slice& col_name, int32_t val) WARN_UNUSED_RESULT;
  Status SetInt64(const Slice& col_name, int64_t val) WARN_UNUSED_RESULT;

  Status SetUnixTimeMicros(const Slice& col_name,
                           int64_t micros_since_utc_epoch) WARN_UNUSED_RESULT;

  Status SetFloat(const Slice& col_name, float val) WARN_UNUSED_RESULT;
  Status SetDouble(const Slice& col_name, double val) WARN_UNUSED_RESULT;
  
#if KUDU_INT128_SUPPORTED
  Status SetUnscaledDecimal(const Slice& col_name, int128_t val) WARN_UNUSED_RESULT;
#endif

  Status SetBinary(const Slice& col_name, const Slice& val) WARN_UNUSED_RESULT;
  Status SetString(const Slice& col_name, const Slice& val) WARN_UNUSED_RESULT;
  
  Status SetBinaryCopy(const Slice& col_name, const Slice& val) WARN_UNUSED_RESULT;
  Status SetStringCopy(const Slice& col_name, const Slice& val) WARN_UNUSED_RESULT;

  Status SetBinaryNoCopy(const Slice& col_name, const Slice& val) WARN_UNUSED_RESULT;
  Status SetStringNoCopy(const Slice& col_name, const Slice& val) WARN_UNUSED_RESULT;
  
  Status SetNull(const Slice& col_name) WARN_UNUSED_RESULT;
  
  
```

#### 根据`column_idx`进行赋值


```
  Status SetBool(int col_idx, bool val) WARN_UNUSED_RESULT;

  Status SetInt8(int col_idx, int8_t val) WARN_UNUSED_RESULT;
  Status SetInt16(int col_idx, int16_t val) WARN_UNUSED_RESULT;
  Status SetInt32(int col_idx, int32_t val) WARN_UNUSED_RESULT;
  Status SetInt64(int col_idx, int64_t val) WARN_UNUSED_RESULT;
  
  Status SetUnixTimeMicros(int col_idx, int64_t micros_since_utc_epoch) WARN_UNUSED_RESULT;

  Status SetFloat(int col_idx, float val) WARN_UNUSED_RESULT;
  Status SetDouble(int col_idx, double val) WARN_UNUSED_RESULT;
#if KUDU_INT128_SUPPORTED
  Status SetUnscaledDecimal(int col_idx, int128_t val) WARN_UNUSED_RESULT;
#endif

  Status SetBinary(int col_idx, const Slice& val) WARN_UNUSED_RESULT;
  Status SetString(int col_idx, const Slice& val) WARN_UNUSED_RESULT;
  
  Status SetStringCopy(int col_idx, const Slice& val) WARN_UNUSED_RESULT;
  Status SetBinaryCopy(int col_idx, const Slice& val) WARN_UNUSED_RESULT;
  
  Status SetBinaryNoCopy(int col_idx, const Slice& val) WARN_UNUSED_RESULT;
  Status SetStringNoCopy(int col_idx, const Slice& val) WARN_UNUSED_RESULT;
  
  Status SetNull(int col_idx) WARN_UNUSED_RESULT;
```

**说明1：**  
这里是没有对`无符号整型`的处理，是因为`kudu`为了和`java 客户端`兼容，即“java客户端”和“c++客户端”有一样的行为，已经去除掉了“无符号整型”。  
参见`commit_id: 7ba706e85d50f627d39f287c6dfc9a54e51508aa`的commit_msg。

**说明2：**  
对于`SetUnscaledDecimal()`方法，之所以要用`KUDU_INT128_SUPPORTED`宏包围起来。是因为其中使用了`__int128`，但是`__int128`是在`gcc4.6`以后才可以使用。但是对于`kudu client`来说(在上层用户用户代码中，需要包含这个头文件，才能进行编译)，是希望在`gcc4.4`也能够编译的。所以，这里在头文件中，是用了`KUDU_INT128_SUPPORTED`来进行包围，即对于`gcc4.4`来说，是不能使用这个方法的。  

参见：`ba33f0346877c8d606ae97a42a6403a001e384bb`的commit_msg。

另外，这里没有在`.cpp`文件中使用`KUDU_INT128_SUPPORTED`宏。  
因为对于上层用户代码来说，在编译时，是通过链接相应的`.so`文件的（服务端会编译出来`.so`文件）。 如果在用户的代码中`.h`文件中，没有对应的函数，那么只要不链接这些函数就行了。即，不需要在`.cpp`文件中使用这些宏。

#### 具体的实现

对于‘大部分类型’对应的`SetXXX()`方法，都是通过直接调用相应的模板方法 `Set()`来实现的。

例外的类型是：
1. `Decimal`类型: 入口方法是`SetUnscaleDecimal()`（不是`Set` + `类型`）。最终也是通过调用`Set()`来实现的。
2. 以及变长的`STRING`和`BINARY`类型：

##### `private Set()`  -- 模板方法 

和上面一样，有两个：
1. 根据`column name`进行设置；
2. 根据`column idx`进行设置；

```
// 函数声明
template<typename T>
Status Set(const Slice& col_name, const typename T::cpp_type& val,
             bool owned = false);
template<typename T>
Status Set(int col_idx, const typename T::cpp_type& val,
             bool owned = false);
             
// 函数实现             
template<typename T>
Status KuduPartialRow::Set(const Slice& col_name,
                           const typename T::cpp_type& val,
                           bool owned) {
  int col_idx;
  RETURN_NOT_OK(schema_->FindColumn(col_name, &col_idx));
  return Set<T>(col_idx, val, owned);
}

template<typename T>
Status KuduPartialRow::Set(int col_idx,
                           const typename T::cpp_type& val,
                           bool owned) {
  const ColumnSchema& col = schema_->column(col_idx);
  if (PREDICT_FALSE(col.type_info()->type() != T::type)) {
    // TODO: at some point we could allow type coercion here.
    return Status::InvalidArgument(
      Substitute("invalid type $0 provided for column '$1' (expected $2)",
                 T::name(),
                 col.name(), col.type_info()->name()));
  }

  ContiguousRow row(schema_, row_data_);

  // If we're replacing an existing STRING/BINARY value, deallocate the old value.
  if (T::physical_type == BINARY) DeallocateStringIfSet(col_idx, col);

  // Mark the column as set.
  BitmapSet(isset_bitmap_, col_idx);

  if (col.is_nullable()) {
    row.set_null(col_idx, false);
  }

  ContiguousRowCell<ContiguousRow> dst(&row, col_idx);
  memcpy(dst.mutable_ptr(), &val, sizeof(val));
  if (owned) {
    BitmapSet(owned_strings_bitmap_, col_idx);
  }
  return Status::OK();
}
```

**注意：“根据`column_idx`进行设置”的性能会好一些**  
因为“根据`column_idx`进行设置”，相比于“根据`column name`进行设置”，去不需要通过“在`hash_map`中通过根据`column_name`去查找`column_idx`”。

说明1：  
在实现的时候，在“栈”上构建了`ContiguousRow`对象，然后利用`ContiguousRow`作为模板参数 传给`ContiguousRowCell`，并对其进行修改。

注意：`ContiguousRow`对象只有两个属性(`const Schema* schema_`和`uint8_t *row_data_`)，而且都比较简单，而且这里是在“栈上”构建，所以开销是非常小的。

说明2:  
对于`physical_type == BINARY`的类型(`BINARY`和`STRING`)，那么如果之前已经设置过了对应的值（即当前对象维护的“间接引用”的值），那么在赋值前需要先释放之前“间接引用”的内存。

说明3： `owned`参数

表示是否需要拷贝“间接引用”的数据。

这个参数的默认值为`false`，只有在`SetSliceCopy()`中调用时才会为`true`。

说明：可能正是因为只有在`SetSliceCopy()`中调用时，`owned`参数的值才会`true`，所以在`Set()`方法的实现中，如果“`owned==true`”的时候，并没有再去检查对应的类型是否为`BINARY`或`STRING`。（只有这两种类型，才会使用到`SetSliceCopy()`函数进行赋值）

##### `SetUnscaledDecimal()`  -- `DECIMAL`类型

**注意：**  对于任意的`DECIMAL`类型(`DECIMAL32/DECIMAL64/DECIMAL128`)，赋值都是用这个方法。

之所以会都用这一个方法，是因为这个方法，只在用户代码在调用`kudu client`时使用。  
另外，虽然`DECIMAL`类型，在`server`端，实际上有3种类型，但是对于“上层用户”看来，只有一种类型，就是`DECIMAL`。（参见`KuduColumnSchema`类，在文件`src/kudu/client/schema.h`中）。

**说明1：名字中的`unscaled`的含义**  
因为在本函数的时间中，只考虑了`precision`的值，而没有考虑`scale`的值。

`precision`的值，用来判断所传入的值是否合法（即，是否在有效的范围内）。

**说明2： 为什么传入的`val`值是`int128`类型。**  
【猜测】：在`kudu`的实现时，对于`decimal(precision, scale)`，在实际存储的时候，是按照一个`precision`位的整型进行存储的。

比如说`Decimal(8, 5)`，在实际存储的时候，是一个`8`位的整数。因为这个生命，能够表示类似`123.45678`的数，但是在实际存储的时候，存储为12345678，然后计算的时候，也是采用这个形式，只是最终返回给用户的时候，会按照`Schema`中的`scale`值，再进行转化。

>> 问题：上述猜测的是正确的吗？

**说明3: **  
该函数，最终也是调用 模板函数`Set()`来实现的。

```
Status KuduPartialRow::SetUnscaledDecimal(int col_idx, int128_t val) {
  const ColumnSchema& col = schema_->column(col_idx);
  const DataType col_type = col.type_info()->type();
  switch (col_type) {
    case DECIMAL32:
      RETURN_NOT_OK(CheckDecimalValueInRange(col, val));
      return Set<TypeTraits<DECIMAL32> >(col_idx, static_cast<int32_t>(val));
    case DECIMAL64:
      RETURN_NOT_OK(CheckDecimalValueInRange(col, val));
      return Set<TypeTraits<DECIMAL64> >(col_idx, static_cast<int64_t>(val));
    case DECIMAL128:
      RETURN_NOT_OK(CheckDecimalValueInRange(col, val));
      return Set<TypeTraits<DECIMAL128> >(col_idx, static_cast<int128_t>(val));
    default:
      return Status::InvalidArgument(
          Substitute("invalid type $0 provided for column '$1' (expected decimal)",
                     col.type_info()->name(), col.name()));
  }
}
```

###### `CheckDecimalValueInRange()`  -- 工具方法
注意：不是类的方法，仅在`.cpp`中定义，是个工具方法。

说明：每个`Decimal`类型，都有一个`precision`和`scale`属性(其中`precision`的最大值为`38`，原因是 最大用一个`__int128`来表示一个`Decimal`类型，而一个`__int128`的类型，取值返回最大是`39`位的整数,即`int128`类型能够完全涵盖的，只能是`38`位)。

这个方法的作用，是检查所给的`decimal`数值，是否超过了定义的“精度范围”。

```
Status CheckDecimalValueInRange(ColumnSchema col, int128_t val) {
  int128_t max_val = MaxUnscaledDecimal(col.type_attributes().precision);
  int128_t min_val = -max_val;
  if (val < min_val || val > max_val) {
    return Status::InvalidArgument(
        Substitute("value $0 out of range for decimal column '$1'",
                   DecimalToString(val, col.type_attributes().scale), col.name()));
  }
  return Status::OK();
}
```

**说明1: `MaxUnscaledDecimal()`的作用**  
给定一个`precision`，返回它说所能表示的最大值。

比如：加入该表定义时，该列的`precision`是`8`, 那么`MaxUnscaledDecimal()`的返回结果是`999 999 99`。

##### `SetString()`和`SetBinary()`

因为有“间接引用”的数据，根据是否在`set`的同时进行数据拷贝，区分为两类方法：1) `SetXxxCopy()`; 2) `SetXxxNoCopy()`;

>> 这里的`Xxx`表示`String`或`Binary`。

说明：对于`STRING`和`BINARY`类型，因为在`C++`中都是对应的二进制字符串(`std::string`)，所以它们的处理逻辑是相同的。

###### `SetString()`和`SetBinary()`方法，是等价于`SetXxxCopy()`。

参见`commit_id: 48766a4ce17d422ced9a6ec78c9a9969ac44d8c9`的commit_msg。  
在很早之前的`kudu`版本中，`SetXxx()`，是等价于`SetXxxNoCopy()`的，即是不会拷贝数据的。但是这会导致上层用户比较困惑，因为其它的`SetYYY()`接口，都是数据安全的（比如：`SetInt32()`/`SetDouble()`，上层用户可能会假定，所有的`SetXxx()`都是安全的，即无论是`SetString()`还是`SetInt32()`，在调用完之后，就不用再维护传入进来的`Slice val`参数，即用户可能会立即释放该`Slice`对象说间接引用的内存）

为了解决这个问题，从上面的`commit_id`开始，将`SetString()`修改为等价于`SetStringCopy()`。

注意：这种修改，如果用户的使用场景上，是不需要进行拷贝的，是会有性能下降的，因为多了拷贝。

所以，对于高级用户，如果用户自己能够确定“不需要进行拷贝”，那么可以手动调用`SetXxxNoCopy()`方法。

对于`SetXxxCopy()`方法，内部是调用`SetSliceCopy()`方法来实现的。

###### `SetXxxNoCopy()`

因为不用拷贝`Slice`对象中的“间接引用”的数据，所以只需要拷贝在`row_data_`中中的“`Slice`对象本身”即可。

内部直接调用“模板函数`Set()`方法”，即`private Set<TypeTraits<STRING>>()`和`private Set<TypeTraits<BINARY>>()`

###### `private SetSliceCopy()`

是`SetXxxCopy()`的真正实现。

分为两步：
1. 先申请内存，并拷贝数据；
2. 调用`Set()`方法来拷贝`Slice`对象。

```
Status KuduPartialRow::SetBinary(const Slice& col_name, const Slice& val) {
  return SetBinaryCopy(col_name, val);
}
Status KuduPartialRow::SetString(const Slice& col_name, const Slice& val) {
  return SetStringCopy(col_name, val);
}
Status KuduPartialRow::SetBinary(int col_idx, const Slice& val) {
  return SetBinaryCopy(col_idx, val);
}
Status KuduPartialRow::SetString(int col_idx, const Slice& val) {
  return SetStringCopy(col_idx, val);
}


Status KuduPartialRow::SetBinaryCopy(const Slice& col_name, const Slice& val) {
  return SetSliceCopy<TypeTraits<BINARY> >(col_name, val);
}
Status KuduPartialRow::SetStringCopy(const Slice& col_name, const Slice& val) {
  return SetSliceCopy<TypeTraits<STRING> >(col_name, val);
}
Status KuduPartialRow::SetBinaryCopy(int col_idx, const Slice& val) {
  return SetSliceCopy<TypeTraits<BINARY> >(col_idx, val);
}
Status KuduPartialRow::SetStringCopy(int col_idx, const Slice& val) {
  return SetSliceCopy<TypeTraits<STRING> >(col_idx, val);
}


Status KuduPartialRow::SetBinaryNoCopy(const Slice& col_name, const Slice& val) {
  return Set<TypeTraits<BINARY> >(col_name, val, false);
}
Status KuduPartialRow::SetStringNoCopy(const Slice& col_name, const Slice& val) {
  return Set<TypeTraits<STRING> >(col_name, val, false);
}
Status KuduPartialRow::SetBinaryNoCopy(int col_idx, const Slice& val) {
  return Set<TypeTraits<BINARY> >(col_idx, val, false);
}
Status KuduPartialRow::SetStringNoCopy(int col_idx, const Slice& val) {
  return Set<TypeTraits<STRING> >(col_idx, val, false);
}


template<typename T>
Status KuduPartialRow::SetSliceCopy(const Slice& col_name, const Slice& val) {
  auto relocated = new uint8_t[val.size()];
  memcpy(relocated, val.data(), val.size());
  Slice relocated_val(relocated, val.size());
  Status s = Set<T>(col_name, relocated_val, true);
  if (!s.ok()) {
    delete [] relocated;
  }
  return s;
}

template<typename T>
Status KuduPartialRow::SetSliceCopy(int col_idx, const Slice& val) {
  auto relocated = new uint8_t[val.size()];
  memcpy(relocated, val.data(), val.size());
  Slice relocated_val(relocated, val.size());
  Status s = Set<T>(col_idx, relocated_val, true);
  if (!s.ok()) {
    delete [] relocated;
  }
  return s;
}

```

#### `SetNull()`

和其它`SetXxx()`方法一样，即可以通过`column_name`进行设置，也可以通过`column_idx`进行设置。

注意：只有该列是“是`nullable`”的，这个方法才可能成功。

```
Status KuduPartialRow::SetNull(const Slice& col_name) {
  int col_idx;
  RETURN_NOT_OK(schema_->FindColumn(col_name, &col_idx));
  return SetNull(col_idx);
}

Status KuduPartialRow::SetNull(int col_idx) {
  const ColumnSchema& col = schema_->column(col_idx);
  if (PREDICT_FALSE(!col.is_nullable())) {
    return Status::InvalidArgument("column not nullable", col.ToString());
  }

  if (col.type_info()->physical_type() == BINARY) DeallocateStringIfSet(col_idx, col);

  ContiguousRow row(schema_, row_data_);
  row.set_null(col_idx, true);

  // Mark the column as set.
  BitmapSet(isset_bitmap_, col_idx);
  return Status::OK();
}
```

注意1:   
如果`physical_type == BINARY`（即是变长类型，`STRING`或`BINARY`），那么需要先将“间接引用”的内存清理掉。

### `Unset()`

将一列的值，恢复为使用“默认值”。

注意：这个方法和`SetNull()`的作用 是不同的。
1. `SetNull()`： 会设置`isset_bitmap_`中的相应bit位；（因为进行了显式赋值，所以该列在`server`端就会填充`null`）
2. `Unset()`： 会将`isset_bitmap_`中的相应bit位清空（表示该列没有显式赋值，那么在`server`端会用 该列的默认值进行填充。）；

```
  Status Unset(const Slice& col_name) WARN_UNUSED_RESULT;
  Status Unset(int col_idx) WARN_UNUSED_RESULT;

Status KuduPartialRow::Unset(const Slice& col_name) {
  int col_idx;
  RETURN_NOT_OK(schema_->FindColumn(col_name, &col_idx));
  return Unset(col_idx);
}

Status KuduPartialRow::Unset(int col_idx) {
  const ColumnSchema& col = schema_->column(col_idx);
  if (col.type_info()->physical_type() == BINARY) DeallocateStringIfSet(col_idx, col);
  BitmapClear(isset_bitmap_, col_idx);
  return Status::OK();
}
```

注意：因为这里只是简单的将`isset_bitmap_`中的相应的bit位清零。所以，在检查一个列是否为`null`的时候，必须要先检查`isset_bitmap_`中相应的bit位，才能到`row_data_`中去检查`null bitmap`。

比如说这样一个操作序列：
1. 通过`SetNull()`将该列设置为`null`。  
    这时,`isset_bitmap_`中相应bit位被赋值为`1`； `row_data_`中`null bitmap`的相应bit位 也被赋值为`1`;
2. 使用`Unset()`将该列的值清空。  
    这时，只有`isset_bitmap_`中相应bit位被重置为`0`；（ `row_data_`中`null bitmap`的相应bit位 仍然为`1`;）
3. 调用`isNull()`来检查，该列的是知否为`null`。
    这时，如果直接去`row_data_`中的`null bitmap`中去检查相应的bit位，因为值为`1`，而认为该列的值为`null`。但是这时错误的，因为该列的值还没有被赋值（`isset_bitmap_`中对应的bit位，值为`0`）。

### `IsColumnSet()`

检查给定的列，是否已经被赋值。（即，`isset_bitmap_`中对应的bit位 是否为`1`）

```
  bool IsColumnSet(const Slice& col_name) const;
  bool IsColumnSet(int col_idx) const;
```

### `IsNull()`

```
  bool IsNull(const Slice& col_name) const;
  bool IsNull(int col_idx) const;
  
  
bool KuduPartialRow::IsNull(int col_idx) const {
  const ColumnSchema& col = schema_->column(col_idx);
  if (!col.is_nullable()) {
    return false;
  }

  if (!IsColumnSet(col_idx)) return false;

  ContiguousRow row(schema_, row_data_);
  return row.is_null(col_idx);
}

bool KuduPartialRow::IsNull(const Slice& col_name) const {
  int col_idx;
  CHECK_OK(schema_->FindColumn(col_name, &col_idx));
  return IsNull(col_idx);
}
```

如果该列尚未被赋值，那么返回`false`，即认为 该列的值不为`null`。

注意：一定要先确定“该列已经被赋值”的情况下，再去检查`row_data_`中的`null bitmap`。（原因 参见上面的`Unset()`函数的说明）

### 各种类型的`Get()`方法

#### 根据`column_name`进行获取

```
  Status GetBool(const Slice& col_name, bool* val) const WARN_UNUSED_RESULT;

  Status GetInt8(const Slice& col_name, int8_t* val) const WARN_UNUSED_RESULT;
  Status GetInt16(const Slice& col_name, int16_t* val) const WARN_UNUSED_RESULT;
  Status GetInt32(const Slice& col_name, int32_t* val) const WARN_UNUSED_RESULT;
  Status GetInt64(const Slice& col_name, int64_t* val) const WARN_UNUSED_RESULT;
  Status GetUnixTimeMicros(const Slice& col_name,
                      int64_t* micros_since_utc_epoch) const WARN_UNUSED_RESULT;

  Status GetFloat(const Slice& col_name, float* val) const WARN_UNUSED_RESULT;
  Status GetDouble(const Slice& col_name, double* val) const WARN_UNUSED_RESULT;
  
#if KUDU_INT128_SUPPORTED
  // NOTE: The non-const version of this function is kept for backwards compatibility.
  Status GetUnscaledDecimal(const Slice& col_name, int128_t* val) WARN_UNUSED_RESULT;
  Status GetUnscaledDecimal(const Slice& col_name, int128_t* val) const WARN_UNUSED_RESULT;
#endif

  Status GetString(const Slice& col_name, Slice* val) const WARN_UNUSED_RESULT;
  Status GetBinary(const Slice& col_name, Slice* val) const WARN_UNUSED_RESULT;
```

#### 根据`column_idx`进行获取

```
  Status GetBool(int col_idx, bool* val) const WARN_UNUSED_RESULT;

  Status GetInt8(int col_idx, int8_t* val) const WARN_UNUSED_RESULT;
  Status GetInt16(int col_idx, int16_t* val) const WARN_UNUSED_RESULT;
  Status GetInt32(int col_idx, int32_t* val) const WARN_UNUSED_RESULT;
  Status GetInt64(int col_idx, int64_t* val) const WARN_UNUSED_RESULT;
  Status GetUnixTimeMicros(int col_idx, int64_t* micros_since_utc_epoch) const WARN_UNUSED_RESULT;

  Status GetFloat(int col_idx, float* val) const WARN_UNUSED_RESULT;
  Status GetDouble(int col_idx, double* val) const WARN_UNUSED_RESULT;
#if KUDU_INT128_SUPPORTED
  // NOTE: The non-const version of this function is kept for backwards compatibility.
  Status GetUnscaledDecimal(int col_idx, int128_t* val) WARN_UNUSED_RESULT;
  Status GetUnscaledDecimal(int col_idx, int128_t* val) const WARN_UNUSED_RESULT;
#endif

  Status GetString(int col_idx, Slice* val) const WARN_UNUSED_RESULT;
  Status GetBinary(int col_idx, Slice* val) const WARN_UNUSED_RESULT;
```

#### 具体实现

##### `private Get()` -- 模板方法

和模板方法`Set()`一样，有两个：

1. 根据`column name`进行获取；
2. 根据`column idx`进行获取；

```
template<typename T>
Status KuduPartialRow::Get(const Slice& col_name,
                           typename T::cpp_type* val) const {
  int col_idx;
  RETURN_NOT_OK(schema_->FindColumn(col_name, &col_idx));
  return Get<T>(col_idx, val);
}

template<typename T>
Status KuduPartialRow::Get(int col_idx, typename T::cpp_type* val) const {
  const ColumnSchema& col = schema_->column(col_idx);
  if (PREDICT_FALSE(col.type_info()->type() != T::type)) {
    // TODO: at some point we could allow type coercion here.
    return Status::InvalidArgument(
      Substitute("invalid type $0 provided for column '$1' (expected $2)",
                 T::name(),
                 col.name(), col.type_info()->name()));
  }

  if (PREDICT_FALSE(!IsColumnSet(col_idx))) {
    return Status::NotFound("column not set");
  }
  if (col.is_nullable() && IsNull(col_idx)) {
    return Status::NotFound("column is NULL");
  }

  ContiguousRow row(schema_, row_data_);
  memcpy(val, row.cell_ptr(col_idx), sizeof(*val));
  return Status::OK();
}
```

说明1： 首先会检查“用户使用的方式”和“该列真正的类型”是否匹配。（比如：不能用`GetInt32`去获取“类型为`INT64`的列的值”）。

说明2： 只有已经被赋过值的列，才能调用该方法。否则返回`Status::NotFound`。

说明3: 如果该列的值为`null`(已经被赋值，并且值为`null`)，也返回`Status::NotFound`；

注意： 和`Set()`类似，对于“使用`column idx`的方式”，因为不需要通过`hash map`来查找`column idx`的步骤，所以会快过于“使用`column name`进行获取的方式”。所以，在对性能较为明爱的程序中，应该优先使用“使用`column idx`进行获取”的方式。

##### `GetUnscaledDecimal()`  -- `DECIMAL`类型

**注意：**  对于任意的`DECIMAL`类型(`DECIMAL32/DECIMAL64/DECIMAL128`)，都是用这个方法。

和`SetUnscaledDecimal()`一样，之所以会都用这一个方法，是因为这个方法，只在用户代码在调用`kudu client`时使用。  
另外，虽然`DECIMAL`类型，在`server`端，实际上有3种类型，但是对于“上层用户”看来，只有一种类型，就是`DECIMAL`。（参见`KuduColumnSchema`类，在文件`src/kudu/client/schema.h`中）。

**说明1: **  
该函数，最终也是调用 模板函数`Get()`来实现的。

##### `GetString()`和`GetBinary()`

因为这两个函数的声明中，返回的对象就是`Slice`对象。 而用户一旦看到“`Slice`对象”，就应该知道需要注意“间接引用的内存”要注意保持有效的问题。

所以，和`SetXxx`时不同，在读取时不需要“是否拷贝”的问题。

### `EncodeRowKey()`

将 该行数据的`主键`列的值进行编码，结果放入到`std::string encoded_key`参数中。

使用场景：
对于一个`KuduPartialRow`对象，当用户填充了部分列的数据后，调用该函数，获取当前行的“编码过的主键”。

```
Status KuduPartialRow::EncodeRowKey(string* encoded_key) const {
  // Currently, a row key must be fully specified.
  // TODO: allow specifying a prefix of the key, and automatically
  // fill the rest with minimum values.
  for (int i = 0; i < schema_->num_key_columns(); i++) {
    if (PREDICT_FALSE(!IsColumnSet(i))) {
      return Status::InvalidArgument("All key columns must be set",
                                     schema_->column(i).name());
    }
  }

  encoded_key->clear();
  ContiguousRow row(schema_, row_data_);

  for (int i = 0; i < schema_->num_key_columns(); i++) {
    bool is_last = i == schema_->num_key_columns() - 1;
    const TypeInfo* ti = schema_->column(i).type_info();
    GetKeyEncoder<string>(ti).Encode(row.cell_ptr(i), is_last, encoded_key);
  }

  return Status::OK();
}
```

注意：目前的实现中，要求所有的`主键的所有列`都必须已经被赋值。

### `ToEncodedRowKeyOrDie()`

和`EncodeRowKey()`功能相同，但是不允许失败。 如果失败，直接触发`CHECK()`掉。

### 检查是否已经被赋值  -- 工具函数
#### `IsKeySet()`
检查是否“所有的key列”已经都被赋值。

#### `AllColumnsSet()`
检查 是否所有列 都已经被赋值。

```
bool KuduPartialRow::AllColumnsSet() const {
  return BitmapIsAllSet(isset_bitmap_, 0, schema_->num_columns());
}

bool KuduPartialRow::IsKeySet() const {
  return BitmapIsAllSet(isset_bitmap_, 0, schema_->num_key_columns());
}
```

### `ToString()`

>> 问题：目前这个方法，对于没有设置的列，会直接不打印。 是不是打印个标识比较好，比如`"NOT_SET"`。







