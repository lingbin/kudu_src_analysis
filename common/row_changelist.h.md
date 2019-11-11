[TOC]

文件： `src/kudu/common/row_changelist.h`

## `class RowChangeList`

一个`RowChangeList`对象，就是封装了一个`Slice`对象，其中保存了一个“变化列表”。

注意1：一个`RowChangeList`对象，就是对 对 1个`Row`的一次修改。

注意2：在`RowChangeList`对象中，只包含`value列`的信息，并不包含所对应的`主键`信息。该对象表示的“修改操作”，是对应哪一个`Row`，是由使用场景的上下文中其它变量标识的。

注意3：之所以叫`list`，是因为
1. 可能修改了多个列；
2. 对同一个列，也可能修改多次；（目前是这样的，将来可能会改变，即将来一个`RowChangeList`中，对一个column可能只会修改一次）

可能包含如下3种类型:
1. `UPDATE`: 将一个或多个列， 修改为一个新的值；
2. `DELETE`: 删除一个`Row`;
3. `REINSERT`: 只针对`MemRowSet`，重新插入一行数据。

注意1：该类的使用方法：
1. 必须使用`RowChangeListEncoder`来构建`RowChangeList`对象；
2. 必须使用`RowChangeListDecoder`来读取`RowChangeList`对象；

注意2： 传递给`RowChangeListEncoder`和`RowChangeListDecoder`的`Schema`对象，应该是同一个。

### 序列化的格式

>> 说明： `RowChangeList`在代码中，常常缩写为`RCL`。

1. 第一个字节，表示`RCL`的类型。参见`RowChangeList::ChangeType`。
2. 对于不同的类型，有不同的编码方法。

如果`type == kDelete`，那么就没有更多数据了。 表示当前行已经被删除了。

如果`type == kReinsert`，那么会跟着一个`tuple-format`的数据行。
>> 备注：见代码注释，这个方式将来可能会被改变。
>>      例如下面链接提到的方式： https://gerrit.cloudera.org/975

>> 问题： 什么是`tuple-format`?

如果`type == kUpdate`：那么后面会跟着一个“更新序列”(即包含多个更新)。  
每个“更新项”的格式如下：

字段           | 长度     | 说明
---------------|--------- | ----------
`<column_id>`  | varint32 | 要更新的column的id
`<length + 1>` | varint32 | 如果是`0`，表示当前列的值为`null`; <br/> 否则，它的值为 对应数据的长度加1，即`data length + 1`
`<value>`      | 由前一个字段决定 | 该列的值。<br/> 如果是`STRING`类型，那么它的值就是具体的值。 <br/> 否则，它是一个固定长度的字段，它的长度由它的类型决定。<br/>（注意：它的长度是“对应类型长度 + 1”。比如，它的类型是`int32`，那么它的长度是`5`）

说明： 这里的长度是`length+1`，不是`length`。  
原因是对于`STRING`类型，可以区分“空字符串”和“`null`”。

使用`length + 1`作为长度，那么检测是否为`null`，就可以直接使用`if (length == 0)`来判断，而不再需要额外设置一个`is_null`的标识。

注意：  
虽然储存的长度是`<length + 1>`，但是在`<value>`中实际存储的数据，长度仍然是`length`（即并没有类似在尾部填充`\0`等的操作）。  
+ 对于`STRING`类型，存储的数据，就是字符串的原始值；
+ 对于其它定长类型，其中存储的数据就是它们对应的内存格式。比如`INT32`，虽然长度会存储`5`，但是值的部分，实际长度仍然是`4`。  

详见`RowChangeListEncoder::EncodeColumnMutationRaw()`和`RowChangeListDecoder::DecodeNext()`。

#### `UPDATE`类型举例

```
// 1) UPDATE SET [col_id 2] = "hello"
//   0x01    0x02      0x06          "h e l l o"
//   UPDATE  col_id=2  len(hello)+1  <raw value>
//   注意：这里的真正存储数据时，字符串所占的内存长度仍然是 5，不是 6。

//
// 2) UPDATE SET [col_id 3] = 33  (assuming INT32 column)
//   0x01    0x03      0x05              0x21 0x00 0x00 0x00
//   UPDATE  col_id=3  sizeof(int32)+1   <raw little-endian value>
//
// 3) UPDATE SET [col id 3] = NULL
//   0x01    0x03      0x00
//   UPDATE  col_id=3  NULL
//
// 4) UPDATE SET [col_id 2] = ""     ---- 空字符串
//   0x01    0x02      0x01          ""
//   UPDATE  col_id=2  len(hello)+1  <raw value>

注意：上面例子中： 新值为 null  和 “空字符串” 时的区别:   
区别是：一个长度为0； 一个长度为1;
相同点是： 都没有真正存储任何数据。

```

### `enum RowChangeList::ChangeType`

```
  enum ChangeType {
    ChangeType_min = 0,
    kUninitialized = 0,
    kUpdate = 1,
    kDelete = 2,
    kReinsert = 3,
    ChangeType_max = 3
  };
```

### 成员

只有一个成员。

注意：该类本质上，就是为了封装这个`Slice`对象。

#### `encoded_data_`成员

`Slice`类型。对一个`Row`的所有修改，都会被编码到这个`encoded_data_`中。

```
  Slice encoded_data_;
```

### 接口列表

#### `static CreateDelete()`
静态方法，创建一个`RowChangeList`对象，表示一个`DELETE`操作。

注意：这个方法返回的`RowChangeList`对象中的`Slice`成员，指向一个 静态的常量的地址。所以不应该修改它返回的对象。

```
  static RowChangeList CreateDelete() {
    return RowChangeList(Slice("\x02"));
  }
```

注意： 这里的常量值是`"\x02"`。   
原因是：`kDelete`的值就是`2`（参见上面的`enum RowChangeList::ChangeType`）。

在编码时，第一个字段是“操作类型”（这里的操作类型是`kDelete`，即就应该编码为`"\x02"`）。 并且，在为`DELETE`类型是，不需要其它数据了（表示将当前行删除）。

#### `is_reinsert()`

```
  bool is_reinsert() const {
    DCHECK_GT(encoded_data_.size(), 0);
    return encoded_data_[0] == kReinsert;
  }
```

说明：在`Slice`对象中，重载了`operator[]`方法，返回的是对应下标位置的值，返回值是`uint8_t`类型。 所以在这里，可以直接用`operator[]`返回的值，直接与 枚举值`kReinsert`进行比较判断。

#### `is_delete()`

```
  bool is_delete() const {
    DCHECK_GT(encoded_data_.size(), 0);
    return encoded_data_[0] == kDelete;
  }
```

#### `is_null()`

```
  bool is_null() const {
    return encoded_data_.size() == 0;
  }
```

注意：  
这个方法是判断“当前`RowChangeList`对象是否为`null`”，不是判断“其中的值是否为`null`”。

所以：这里判断的方法使用的是：`encoded_data_`的长度 是否为0。也就是说，其中连“类型”都没有被编码进去。

而如果是“其中的值是否为`null`”，判断方法是，对应的值的长度为0。
这时，其中一定是已经编码了“类型”(`ChangeType`)的，所以`encoded_data_`的长度一定不是0。

#### `ChangeType_Name()`

返回将`ChangeType`字段对应的字符串。

```
const char* RowChangeList::ChangeType_Name(RowChangeList::ChangeType t) {
  switch (t) {
    case kUninitialized:
      return "UNINITIALIZED";
    case kUpdate:
      return "UPDATE";
    case kDelete:
      return "DELETE";
    case kReinsert:
      return "REINSERT";
    default:
      return "UNKNOWN";
  }
}
```


## `class RowChangeListEncoder`

编码一个`RowChangeList`对象。

### 成员

#### `type_`属性

修改类型。

#### `dst_`属性

用来保存被编码后的值。

```
RowChangeList::ChangeType type_;
faststring *dst_;
```

### 接口列表

#### 构造函数

在构造函数中，传入一个`faststring`类型的指针，最终的编码结果就保存在这个`faststring`中。  

因为传入的是`faststring`指针，所以用户要保证该`faststring`对象的有效性。

#### 多个修改类型的`setter`方法

直接将`type_`成员修改为对应的 类型。

注意：`SetToReinsert()`提供了两个重载方法。
1. 一个是直接设置`type_`属性；
2. 一个是模板函数: 会传入一个`Row`，即除了设置`type_`以外，还会对数据进行编码。

因为插入数据时，除了可以为空的列，都需要提供‘具体的值’。所以，这里会逐个遍历`Schema`中的每个列，排除掉 属于“主键”的列，逐个进行插入。

```
  void SetToDelete() {
    SetType(RowChangeList::kDelete);
  }

  void SetToUpdate() {
    SetType(RowChangeList::kUpdate);
  }
  
  void SetToReinsert() {
    SetType(RowChangeList::kReinsert);
  }

  // Encodes a REINSERT for 'src_row'.
  // Both direct and indirect data from 'src_row' will be copied and encoded.
template<class RowType>
void RowChangeListEncoder::SetToReinsert(const RowType& src_row) {
  DCHECK_EQ(RowChangeList::kUninitialized, type_);
  SetType(RowChangeList::kReinsert);
  const Schema* schema = src_row.schema();
  for (int i = 0; i < schema->num_columns(); ++i) {
    // Reinserts don't need to store the keys.
    if (schema->is_key_column(i)) continue;
    ColumnId col_id = schema->column_id(i);
    const ColumnSchema& col_schema = schema->column(i);
    if (col_schema.is_nullable() && src_row.is_null(i)) {
      EncodeColumnMutationRaw(col_id, true /* null */, Slice());
      continue;
    }

    const void* cell_ptr = DCHECK_NOTNULL(src_row.cell_ptr(i));
    EncodeColumnMutation(col_schema, col_id, cell_ptr);
  }
}
```

注意： 对于`Reinsert`，不需要对`key`列进行编码。  
原因是：  
1. Kudu中是不允许更新`key`列的；
2. 而且在`RowChangeList`中，不保存`key`的信息（要修改哪一行，在使用的地方由其它变量标识）。

#### `as_changelist()`

将当前已经编码的数据，转化为一个`RowChangeList`对象。 

说明：  
在软件工程的“设计模式”，当前类就相当于“构造者模式”中的`Builder`。`as_changelist()`方法，就相当于`build()`方法。

#### `is_empty()`

当前`encoder`对象，是否对应为一个空的`RowChangeList`对象。

判断条件是：
1. “类型(`ChangeType`)”都还没有编码： 判断方法是`dst_->size() == 0`;
2. （如果类型已经编码）那么`type_`为`kReinsert`或`kUpdate`，并且 尚未编码任何的数据内容（`dst_size() == 1`即表示只编码了“类型”）；

**该函数的使用场景：**  
如果该函数返回`true`，那么它表达的真正含义是：当前`RowChangeList`是否相当于 “没有任何操作”。

#### `AddColumnUpdate()`

添加对一个`column`的update操作。

该方法会接受3个参数：
1. 列的schema；
2. column_id。 （注意：在`ColumnSchema`中，并没有`column_id`属性）
3. 列的值。是一个`const void* cell_ptr`，表示在内存中存储的类型。

如果`cell_ptr`参数为`null`，那么该列必须是“可为空的”，即`column_schema.is_nullable() == true`。

否则，`cell_ptr`参数指向一个 它的类型所对应的内存格式。  
比如，对于`STRING`类型，`cell_ptr`指向的是一个`Slice`对象； 如果是`INT32`类型，那么`cell_ptr`指向的是一个`int32_t`的内存。

```
void RowChangeListEncoder::AddColumnUpdate(const ColumnSchema& col_schema,
                                           int col_id,
                                           const void* cell_ptr) {
  SetToUpdate();
  EncodeColumnMutation(col_schema, col_id, cell_ptr);
}
```

#### `EncodeColumnMutation()`

只编码具体的数据，不修改`type_`属性。

**注意：在调用该方法之前，一定要先设置好`type_`属性。**

也正是因为不修改`type_`，而在`REINSERT`和`UPDATE`时的编码方法是相同的，所以该方法在`REINSERT`和`UPDATE`时都会被使用。

```
void RowChangeListEncoder::EncodeColumnMutation(const ColumnSchema& col_schema,
                                                int col_id,
                                                const void* cell_ptr) {
  DCHECK_NE(RowChangeList::kUninitialized, type_);
  DCHECK(type_ == RowChangeList::kUpdate || type_ == RowChangeList::kReinsert);

  Slice val_slice;
  if (cell_ptr != nullptr) {
    if (col_schema.type_info()->physical_type() == BINARY) {
      memcpy(&val_slice, cell_ptr, sizeof(val_slice));
    } else {
      val_slice = Slice(reinterpret_cast<const uint8_t*>(cell_ptr),
                        col_schema.type_info()->size());
    }
  } else {
    // NULL value.
    DCHECK(col_schema.is_nullable());
  }

  EncodeColumnMutationRaw(col_id, cell_ptr == nullptr, val_slice);
}
```

注意1：  
这里的实现是，对于任意的类型，都会构造一个`Slice`对象（对于`physical_type == BINARY`的列，因为它的value已经是一个`Slice`对象，所以直接 复制这个`Slice`即可。其它类型，会新建一个`Slice`对象）。

如果新值为`null`，那么构造的`Slice`对象是一个空对象。但是实际上这个对象和“空字符串”对应的`Slice`对象是一样的，所以在调用`EncodeColumnMutationRaw()`时，还需要另外一个变量来标识“是否为`null`”。

注意2：这里对于`physical_type == BINARY`的列，拷贝`Slice`对象时，使用的是`memcpy`，而不是`Slice`对象的“赋值操作符”。

>> 问题：为什么不使用`赋值操作符`？ 有没有深意？

注意3： 在最终调用的`EncodeColumnMutationRaw()`方法中，会将这个`Slice`对象的内容 拷贝到`dst_`中。所以最终在`dst_`中保存的是，各个类型的具体值。  
比如：对于`INT32`类型，那么里面存储的就是长度为4的`int32_t`；对于`STRING`类型，里面存储的是一个真实的字符串。


#### `private EncodeColumnMutationRaw()`

内部工具函数。 编码具体的数据内容。

注意：调用该方法前，必须已经设置好了`type_`。

参数中包含了 编码数据 所需要的所有信息（1. `column_id`; 2. `is_null`; 3. `raw value`），所以不再需要传入`ColumnSchema`对象。

参数`new_val`是`Slice`类型，它内部包含的值是 经过编码的格式。  
对于`STRING`和`BINARY`类型，它的值是用户提供的“原始字符串值”。否则，它是各个类型所对应的固定长度的值（比如：对于`INT32`类型，它的值就是一个`int32_t`类型的内存形式）。

```
void RowChangeListEncoder::EncodeColumnMutationRaw(int col_id, bool is_null, Slice new_val) {
  // The type must have been set beforehand.
  DCHECK_NE(RowChangeList::kUninitialized, type_);
  DCHECK(type_ == RowChangeList::kUpdate || type_ == RowChangeList::kReinsert);

  InlinePutVarint32(dst_, col_id);
  if (is_null) {
    dst_->push_back(0);
  } else {
    InlinePutVarint32(dst_, new_val.size() + 1);
    dst_->append(new_val.data(), new_val.size());
  }
}
```

## `class RowChangeListDecoder`

用来已经编码的`RowChangeList`对象，得到具体的 修改内容。


### `struct RowChangeListDecoder::DecodedUpdate`

用来表示对一个column的`update`内容。

**`col_id`**  
该更新所对应的列。

**`null`**  
该类的新值是否为`null`。

**`raw_value`**  
该列的新值。

其中的值的格式：
1. 对于定长的类型，比如`INT32`，内容就是一个内存中`int32_t`格式。
2. 对于变长的类型，比如`STRING`，内容就是 字符串的值。（注意：不是`Slice`）。

只有在`null == false`时，才会用到当前`raw_value`属性。

**`Validate()`方法**  
检查当前修改是否合法，如果合法，那么会返回一些信息（当前column在`Schema`中的下标，以及经过检查的有效的新值）。

函数参数：
1. `col_idx`: 当前列在 `Schema`中的 下标号。如果不存在，那么返回`-1`；
2. `valid_value`: 当前列在`Schema`中存在，并且它的`raw_value`是合法的，那么在`valid_value`会指向具体的值。
    + 如果是字符串，那么就直接指向字符串的首地址；
    + 如果是其他定长类型，那么就直接指向 该类型的内存地址；
    + 如果是null，那么该值为`nullptr`;

注意:  
参数`valid_value`是一个 指针的指针。

函数的行为：
1. 如果当前列在`Schema`中不存在，那么将`*col_idx = -1`，然后返回`Status:OK`。（注意：这时返回是的`OK`，不是错误）；
2. 如果当前列在`Schema`中存在，但是数据不合法，这时返回`Status::Corruption`；
3. 如果当前列在`Schema`中存在，并且数据是合法的，那么就设置`*col_idx`和`valid_value`:  
    说明：如果该列新值为null，那么`*value == nullptr`;

```
  struct DecodedUpdate {
    ColumnId col_id;
    bool null = false;
    Slice raw_value;

    Status Validate(const Schema& s,
                    int* col_idx,
                    const void** valid_value) const;
  };
```

### 使用方法举例
```
  // "rcl" is a RowChangeList object

  RowChangeListDecoder decoder(rcl);

  Status s = decoder.Init();
  if (!s.ok()) {
    return "[invalid: " + s.ToString() + "]";
  }

  if (decoder.is_delete()) {
    return string("DELETE");
  }

```

### 成员

#### `remaining_`

当前剩余的“未解码”的数据。

在构建`RowChangeListDecoder`对象的时候，会传入一个`RowChangeList`对象，这个属性就是对应它内部的`编码数据`。

所以每次解码一列的时候，都会向后移动位置，直到没有剩余的数据。

说明：因为是一个`Slice`对象，所以在其指针移动的时候，是很轻量的。

#### `type_`

修改类型。

```
  Slice remaining_;
  RowChangeList::ChangeType type_;
```

### 接口列表

#### `GetIncludeColumnIds()`


#### `DecodeNext()`

解析下一个修改。

注意： 必须是`kUpdate`或者`kReinsert`类型。

```
Status RowChangeListDecoder::DecodeNext(DecodedUpdate* dec) {
  DCHECK_NE(type_, RowChangeList::kUninitialized) << "Must call Init()";
  // Decode the column id.
  uint32_t id;
  if (PREDICT_FALSE(!GetVarint32(&remaining_, &id))) {
    return Status::Corruption("Invalid column ID varint in delta");
  }
  dec->col_id = id;

  uint32_t size;
  if (PREDICT_FALSE(!GetVarint32(&remaining_, &size))) {
    return Status::Corruption("Invalid size varint in delta");
  }

  dec->null = size == 0;
  if (dec->null) {
    return Status::OK();
  }

  size--;

  if (PREDICT_FALSE(remaining_.size() < size)) {
    return Status::Corruption(
        Substitute("truncated value for column id $0, expected $1 bytes, only $2 remaining",
                   id, size, remaining_.size()));
  }

  dec->raw_value = Slice(remaining_.data(), size);
  remaining_.remove_prefix(size);
  return Status::OK();
}
```























