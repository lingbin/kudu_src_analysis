[TOC]

文件：`src/kudu/common/schema.h`

## `DCHECK_SCHEMA_EQ()`宏

只在`debug`模式下打开。如果不相等，会打印错误信息。

```
#define DCHECK_SCHEMA_EQ(s1, s2) \
  do { \
    DCHECK((s1).Equals((s2))) << "Schema " << (s1).ToString() \
                              << " does not match " << (s2).ToString(); \
  } while (0)

```

## `DCHECK_KEY_PROJECTION_SCHEMA_EQ`宏

只在`debug`模式下打开。如果不相等，会打印错误信息。

比较两个`Schema`中的 `key列`。

```
#define DCHECK_KEY_PROJECTION_SCHEMA_EQ(s1, s2) \
  do { \
    DCHECK((s1).KeyEquals((s2))) << "Key-Projection Schema " \
                                 << (s1).ToString() << " does not match " \
                                 << (s2).ToString(); \
  } while (0)
```

## `struct ColumnId`

内部仅仅封装一个`int32_t`类型的整型变量。

注意：`sizeof(ColumnId) == sizeof(int32_t) == 4`，即不会因为进行了封装，就增大了`ColumnId`的大小。封装只是增加了一些接口，方便使用。

**为什么不是使用`typedef`将`ColumnId`声明为`int32_t`，而是要单独定义一个类？**
答：
1. 首先，封装成一个类以后，因为想获取值的函数都是内敛的，所以对性能是没有影响的。
2. 封装成类以后，就可以提供更多的函数，比如这里提供了 自动转化为`int32_t`、`strings::internal::SubstituteArg`、 `AlphaNum`等的函数，这样在一个工具函数中，就可以很方便的使用。

比如，如果需要将`column_id`转化为字符串，在使用`typedef`时，使用它就是使用一个`int32_t`，所以在任何需要转化的地方，都需要自己再手写 “将`int32_t`转化为字符串”的代码，而封装成类以后，利用工具函数就可以直接搞定了。

```
// The ID of a column. Each column in a table has a unique ID.
struct ColumnId {
  explicit ColumnId(int32_t t) : t_(t) {}
  ColumnId() = default;
  ColumnId(const ColumnId& o) = default;

  ColumnId& operator=(const ColumnId& rhs) { t_ = rhs.t_; return *this; }
  ColumnId& operator=(const int32_t& rhs) { t_ = rhs; return *this; }
  operator const int32_t() const { return t_; } // NOLINT
  operator const strings::internal::SubstituteArg() const { return t_; } // NOLINT
  operator const AlphaNum() const { return t_; } // NOLINT
  bool operator==(const ColumnId & rhs) const { return t_ == rhs.t_; }
  bool operator<(const ColumnId & rhs) const { return t_ < rhs.t_; }
  friend std::ostream& operator<<(std::ostream& os, ColumnId column_id) {
    return os << column_id.t_;
  }
 private:
  int32_t t_;
};
```

## `class ColumnTypeAttributes`

对应的`PB`结构是 `ColumnTypeAttributesPB`（见文件`common.proto`）。

用来表述一些类型的属性，目前只有一种，就是针对`DECIMAL`类型，描述了它的“精度”和“有效数字”信息。

### 成员

注意：因为这里两个类型都是`int8_t`类型，所以它的精度最大也就是`256`位有效数字（在对应的`PB`结构`ColumnTypeAttributesPB`中，使用的是`int32`）。

>> 扩展：在阿里的`AnalyticDB`中，号称支持1000位有效数字。不太清楚在`ADB`中`DECIMAL`类型的性能如何

```
  int8_t precision;
  int8_t scale;
```

注意：对于`STRING`类型（在其它数据库中，常称为是`VARCHAR`类型），没有一个“长度”属性。

### 接口列表

#### `EqualsForType()`

比较两个属性是否相同。

**注意：当前的实现中，只是比较`precision`和`scale`成员。**  
但是`Kudu`中实际是支持3种个`DECIMAL`类型的（分别是`DECIMAL32`/`DECIMAL64`/`DECIMAL128`），但是这里在比较的时候，并没有比较`DataType`是否相同。

即一个`DECIMAL32`类型，和一个`DECIMAL64`类型，两者的属性是会可能会被判断相等的。

不过，目前该方法只在`ColumnSchema::EqualsType()`中用到了1次（用来判断两个类型是否相等），在`ColumnSchema::EqualsType()`中，会先判断`DataType`是否相同。

```
bool ColumnTypeAttributes::EqualsForType(ColumnTypeAttributes other,
                                         DataType type) const {
  switch (type) {
    case DECIMAL32:
    case DECIMAL64:
    case DECIMAL128:
      return precision == other.precision && scale == other.scale;
    default:
      return true; // true because unhandled types don't use ColumnTypeAttributes.
  }
}
```

注意：对于非`DECIMAL`类型，这里都是直接返回`true`了。 因为对于其它类型，目前都没有相应的`ColumnTypeAttributes`对象属性。 这里直接返回`true`，写代码比较方便。

如果返回`true`以后，对于任意类型的比较，都可以直接使用如下形式。

```
    // 两个TypeInfo对象， left和right
    // 对应的TypeAttribute对象为  left_attr 和 right_attr
    // 说明，通过两个`ColumnSchema`对象，很容易能够拿到它们的`TypeInfo`和`TypeAttribute`对象。
    // 
    if (left->type() == right->type() 
        && left_attr.EqualsForType(right_attr, left->type()) {
        ...
    }
```
而不需要先判断 `left->type()`是否为 `DECIMAL`的类型，如果是，再去调用`EqualsForType()`。 

#### `ToStringForType()`

将当前的精度信息，打印为字符串。格式为`"($0, $1)"`。

```
string ColumnTypeAttributes::ToStringForType(DataType type) const {
  switch (type) {
    case DECIMAL32:
    case DECIMAL64:
    case DECIMAL128:
      return Substitute("($0, $1)", precision, scale);
    default:
      return "";
  }
}
```

## `struct ColumnStorageAttributes`

保存一个列的物理存储信息，比如：“压缩格式”和“编码方式”。

注意与`ColumnSchema`的关系：
1. `ColumnStorageAttributes`：侧重于描述一列的 物理存储信息，以及字节的展示形式；
2. `ColumnSchema`: 是一个 纯粹的逻辑描述。

也就是说，一个非常完美的抽象中，`ColumnSchema`中是不应该包含`ColumnStorageAttributes`的。但是，在`Kudu`目前的实现中，`ColumnStorageAttributes`是`ColumnSchema`的一个成员。

这里独立出来（而不是将这些属性，直接写在`ColumnSchema`中），是为了代码上更好的封装，即将来可以将它们从`ColumnSchema`中移除。（参见`ColumnSchema::attributes()`的注释）

### 成员

```
  EncodingType encoding;
  CompressionType compression;

  int32_t cfile_block_size;
```

## `stuct ColumnSchemaDelta`

描述对一个列的修改内容。

当前的实现是一个`POD`的。 

注意：如果将来支持了更多的修改类型，那么该类可能就不再是`POD`的了。

对应的`PB`结构是`ColumnSchemaDeltaPB`。(参见`common.proto`)

`ColumnSchema`对象是“可拷贝”和“可赋值”的。

注意1：  
目前`Kudu`不支持修改列的“类型”和`nullable`属性。

注意2：  
这个结构是根据`kudu client`端的消息生成的，所以查找依据是`column name`，而不是`column_id`。

注意3：  
对于修改的内容，都是`boost::optional<>`类型的。也就是说，这些成员，是可以为空的（类似有`ProtoBuffer`结构中的`optional`修饰符）。

注意4：
这里的`default_value`成员，类型是`Slice`类型:
1. 如果是数值类型、`BOOL`类型、`UNIXTIME_MICROS`类型：那么其中保存的值，是该数值在内存中的存储形式。比如`INT32`类型，那么该`Slice`对象的大小就是4个字节。
2. 如果是`BINARY`和`STRING`类型：那么它们实际的内容，就是在该`Slice`对象所指向的内存中。

```
  const std::string name;

  boost::optional<std::string> new_name;

  // NB: these properties of a column cannot be changed yet,
  // ergo type and nullable should always be empty.
  // TODO(wdberkeley) allow changing of type and nullability
  boost::optional<DataType> type;

  boost::optional<bool> nullable;

  boost::optional<Slice> default_value;

  bool remove_default;

  boost::optional<EncodingType> encoding;
  boost::optional<CompressionType> compression;
  boost::optional<int32_t> cfile_block_size;

  boost::optional<std::string> new_comment;
```

## `class ColumnSchema`

描述一个列的schema信息，包含了列相关的所有元信息。

对应的`PB`结构是`ColumnSchemaPB`，参见文件`common.proto`。

注意：`ColumnSchema`和`ColumnSchemaPB`的属性，并不是完全对应的:
1. 在`ColumnSchema`中, 没有`column_id`的属性；（在`ColumnSchemaPB`中有）；
2. 在`ColumnSchema`中, 没有`is_key`的属性；（在`ColumnSchemaPB`中有）

这两个属性（`column_id`和`is_key`），都是通过`Schema`中的属性进行计算得到的。

在`Schema`中，其中会保存一个`ColumnSchema`的列表，用来表示一行数据中包含的各个列。其中还会包含一个`column_id`的列表(与`ColumnSchema`列表一一对应)，以及一个属性`num_key_columns_`，表示“前几列”是key。

### `enum ColumnSchema::ToStringMode`

指定不同的“转化为字符串”的模式。

主要区别是：是否包含一些属性，比如“编码格式”，“压缩类型”，“默认值”信息等。

>> 问题：什么时间需要包含？什么时间不需要包含？

```
  enum class ToStringMode {
    // Include encoding type, compression type, and default read/write value.
    WITH_ATTRIBUTES,
    // Do not include above attributes.
    WITHOUT_ATTRIBUTES,
  };
```

### `enum ColumnSchema::CompareFlags`

在`Equails()`方法中，如何对两个列进行比较。

```
  // compare types in Equals function
  enum CompareFlags {
    COMPARE_NAME = 1 << 0,
    COMPARE_TYPE = 1 << 1,
    COMPARE_OTHER = 1 << 2,

    COMPARE_NAME_AND_TYPE = COMPARE_NAME | COMPARE_TYPE,
    COMPARE_ALL = COMPARE_NAME | COMPARE_TYPE | COMPARE_OTHER
  };
```

问题：每个比较类型，各自用在什么场合？

### 成员

**默认值 是`Variant`类型**

包括`read_default_`和`write_default_`。

因为在`Variant`类型中，既封装了列的类型，也封装了列的值（对于`STRING`和`BINARY`类型，具体指向的值是在`VARIANT`中动态`new`出来的，`Variant`对象负责进行析构）。

**默认值 成员使用`std::shared_ptr<>`**  
注意：`ColumnSchema`在方法间传递时，总是会进行拷贝的。

所以`read_default_`和`write_default_`成员，都是使用`std::shared_ptr<>`的，这样避免在拷贝`ColumnSchema`的时候，只需要增加它们的引用计数，不需要也拷贝它们的内容。

>> 问题：**为什么“默认值”还分为“读默认值”和“写默认值”两种？**  

**注意：在`ColumnSchema`对象中，并没有`column_id`这个属性**

对于“类型”，这里是保存的是`TypeInfo`对象指针，并不是直接保存的`DataType`。

```
  std::string name_;
  const TypeInfo *type_info_;
  bool is_nullable_;
  // use shared_ptr since the ColumnSchema is always copied around.
  std::shared_ptr<Variant> read_default_;
  std::shared_ptr<Variant> write_default_;
  ColumnStorageAttributes attributes_;
  ColumnTypeAttributes type_attributes_;
  std::string comment_;
```

### 接口

#### 很多`getter`和`setter`方法

#### `ApplyDelta()`

这个方法是将`ColumnSchemaDelta`（表示的修改）应用到当前`ColumnSchema`对象上。

只有这个方法返回`OK`，schema才会真正的被修改。也就是说，如果这个方法返回`false`，那么schema就不会有任何修改。

>> 说明：目前只有一个安全检查，就是对于 `BINARY`和`STRING`之外的类型，对于默认值的长度进行了检查。

**为什么在检查“默认值的长度”时，忽略掉了`physical_type()`为`BINARY`类型的列？**  
这个检查的目的是：判断给定的值的长度，是否符合对应的列类型。如果长度不正确，那么说明这个修改是无效的，可以及早退出，保证原对象不会被修改。

比如，对于`INT32`类型，所给的默认值长度必须为4。

如果`physical_type`为`BINARY`类型，说明该列的类型为`STRING`或者`BINARY`，对应的`cpp_type`为`Slice`类型，那么他们的`size()`返回值为`sizeof(Slice)`，这是一个固定值。  
即无论`Slice`的数据内容是什么，`size()`的返回值都是一样的。

即对于这两种类型，`TypeInfo::size()`并不能得到它们的“值大小”（对于其它类型，是可以的）。

而`ColumnSchemaDelta::default_value`的类型是`Slice`类型，调用`Slice::size()`就是它的“值大小”。

所以对于这两种类型，将两者比较没有意义。

另一角度，对于`BINARY`和`STRING`类型，它们对 值的长度是“没有限制”的（而“默认值”也是它们的值），对于这两种类型的列，即`default_value`中具体的内容可能为任意长度，所以通过检查值的长度，是不能排除掉无效的修改的。

因此，对于`BINARY`和`STRING`类型，不需要检查它们的“默认值”的长度。

#### `Compare()`

比较当前`ColumnSchema`所对应的 两个值。 

实际上，因为每种类型，“比较函数”都是固定的，直接调用即可。

#### `memory_footprint_excluding_this()`

返回当前对象占用的内存大小。不包括`this`对象本身的内存。

这个方法，应该在当前对象（`ColumnSchema`）作为“其它对象”的成员时使用。

**对象占用内存的两个部分：**  

一个对象占用的内存，分为两个部分：
1. `sizeof()`的部分：也叫做“当前对象自身的内存占用”，或者称为“‘this部分’占用的内存”。 
2. 其中的指针成员，或者各个嵌套的成员内的指针成员，所动态分配的内存。  
    这部分内存，虽然也是本对象占用的，但是不属于“this部分”。

与内存相关的两个函数，对应关系为：
1. `memory_footprint_excluding_this()`: 对应上面第(2)条；
2. `memory_footprint_including_this()`: 同时包含上面的第(1)条和第(2)条；

比如说，本对象中有`std::string`对象，那么
1. `sizeof(std::string)`部分，是属于“this部分”的。
2. 在`std::string`中的“字符指针”所指向的内存地址，是动态`new`出来的，是“该类占用的，但是又不属于‘this’的”。 注意：对于`std::string`，这部分占用的内存大小应该是`std::string::capacity()`，而不是`std::string::size()`。

注意：在当前的`ColumnSchema`对象中，“被该对象占用，但是又不属于‘this’的”内存，目前只有一个。 就是字符串属性(`name_`)中动态分配的内存大小。

```
size_t ColumnSchema::memory_footprint_excluding_this() const {
  // Rough approximation.
  return name_.capacity();
}
```

>> 问题：这里应该也加上`comment_.capacity()`。

#### `memory_footprint_including_this()`

返回当前对象占用的内存大小。也包括`this`对象本身的内存。

这个方法，应该在当前对象直接在堆上分配时使用。

```
size_t ColumnSchema::memory_footprint_including_this() const {
  return kudu_malloc_usable_size(this) + memory_footprint_excluding_this();
}
```

#### `Stringify()`

将当前 column的值，打印成字符串。

注意，当前`ColumnSchema`对象，只有元信息，没有具体的数据信息。 所以，如果要打印一个“单元格”的值，必须将“单元格对象”作为参数传入进来。

注意：这个方法只会打印“值”。没有其它信息，比如列类型、列名称等。

```
  // Stringify the given cell. This just stringifies the cell contents,
  // and doesn't include the column name or type.
  std::string Stringify(const void *cell) const {
    std::string ret;
    type_info_->AppendDebugStringForValue(cell, &ret);
    return ret;
  }
```

#### `DebugCellAppend()`

打印这个`cell`的调试信息。

`Cell`是一个模板类，具体见文件`row.h`。这里使用了它的两个方法：`in_null()`和`ptr()`。

**DebugCellAppend()  VS  Stringify()**  
1. `Stringify()`只打印“单元格”的值，没有其它信息；
2. `DebugCellAppend()`不仅打印“单元格”的值，还包括 列的名字 和 列类型。

输入结果举例： `STRING foo=bar`

```
template<class CellType>
  void DebugCellAppend(const CellType& cell, std::string* ret) const {
    ret->append(type_info_->name());
    ret->append(" ");
    ret->append(name_);
    ret->append("=");
    if (is_nullable_ && cell.is_null()) {
      ret->append("NULL");
    } else {
      type_info_->AppendDebugStringForValue(cell.ptr(), ret);
    }
  }
```

## `class Schema`

用来描述一个行集合的schema信息。（也可以理解为“table中列schema的集合”）

一个`Schema`就是一个“列信息的集合”，其中包含了指定哪些列是“主键”的信息。  

**注意：“主键”必须在所有列的最前面。**

**注意：**  
`Schema`对象是“可拷贝”和“可赋值”的，但是这个对象的成员很多，所以“拷贝”的成本比较高。
所以，**应该尽量使用`move`语义，或者在传递时使用“指针”或者“引用”**。

对于创建`Schema`对象的函数，应该优先使用`Schema`类型的指针，并且使用`Schema::Reset()`，而不应该使用“按值返回”的方式("return by value")。

### `enum Schema::ToStringMode`

指定将一个`Schema`对象转化为字符串的方式。

区别在于：
1. 是否打印`column_id`;
2. 是否打印列的`存储属性`；

```
  enum ToStringMode {
    BASE_INFO = 0,
    // Include column ids if this instance has them.
    WITH_COLUMN_IDS = 1 << 0,
    // Include column attributes.
    WITH_COLUMN_ATTRIBUTES = 1 << 1,
  };
```

### `enum Schema::SchemaComparisonType`

决定比较两个`Schema`对象的方法。

目前的主要区别是：是否比较`column_id`相关的属性。

**这里要进行区分“比较方式”的原因：**  
是为了能够让“初始化`column_id`相关属性的`Schema`对象”，能够和“尚未初始化`column_id`相关属性的`Schema`对象”进行比较。

```
  enum SchemaComparisonType {
    COMPARE_COLUMNS = 1 << 0,
    COMPARE_COLUMN_IDS = 1 << 1,

    COMPARE_ALL = COMPARE_COLUMNS | COMPARE_COLUMN_IDS
  };
```

### 成员

**注意：**  
如果要添加新的成员，可能需要修改的地方有：
1. `CopyFrom()`;
2. `move constructor`;
3. `assignment operator`;

#### `cols_`

列schema的集合；

#### `num_key_columns_`

当前的列表中，前几列是可以。这个数值就说明了 前面几列 是key。

#### `col_ids_`

column_id的列表。

注意：有的时候，在构造Schema对象的时候，是没有提供colunn_id列表的，这个时候`col_ids_`是一个空列表。

#### `max_col_id_`

当前的`Schema`中的最大的column_id。

#### `col_offsets_`

`std::vector<size_t>`类型，描述当前每个列，在物理存储时（在一行数据中）的偏移量。

第一列的偏移量为`0`。

注意：`col_offsets_.size() == cols_.size() + 1`，因为在`col_offsets_`中，因为它的最后一个元素的值，表示总共所有列的总大小。

```
        |----------------|----------------|---------------|
            column_1            column_2       column_3
     offset_1         offset_2        offset_3        offset_4          

如上图，一共有3列，即`cols_.size() == 3`。
但是在`col_offsets_`的大小应该为4。
其中
1. offset_x（x= 1,2,3）分别表示前三列的偏移量；
2. offset_4 的值，就是所有列的总大小。
```

最后一个元素的作用： 在希望获取当前`schema`的一行数据（所有列）占用的总内存大小时（见`byte_size()`接口方法），可以直接返回。

#### `name_to_index_bytes_`

记录在`name_to_index_`中占用的内存大小。

实现方法： 在`std::unordered_map`中，传入了一个 "内存分配器"。

#### `name_to_index_`
是个`std::unordered_map`类型: 
1. key是column_name，即`ColumnSchema`的`name_`属性，注意这里使用的是`StringPiece`类型;(这里`StringPiece`对象所指向的内存地址，就是`cols_`中每个列的`name_`属性)
2. value是该列的顺序号。即在所有列中处于“第几列”，从`0`开始。

注意：之所以使用`StringPiece`类型，而不是`std::string`类型，是为了:
1. 减少不必要的拷贝；
2. 在查找的时候，也可以直接使用`StringPiece`类型的key去查找，因为很多场景下，上下文中已经有了`StringPiece`对象（即使上下文中只有`std::string`对象，用来构造出一个`StringPiece`对象的成本也是非常低的，因为不需要拷贝字符串）所以很多时候都可以避免拷贝（相比于用`std::string`作为key的方式）。

**注意：复制`ColumnSchema`对象时，不能直接拷贝`name_to_index_成员`**  
因为这个map的key是`StringPiece`对象，其引用的地址是当前`cols_`对象中的每个`ColumnSchema`对象的name属性。   
为了保证每个`key`都要引用 当前对象中`cols_`中的相应属性，在复制对象时，应该使用`cols_`来手动生成`name_to_index_`，而不是直接拷贝。 （参见`CopyFrom()`函数）

这个map定制了一个“可计数的内存分配器”("counting allocator")，是一个`STLCountingAllocator`对象（参见`gutil/stl_util.h`文件），从而可以精确地获取它所占用的内存大小（保存在`name_to_index_bytes`属性中）。

#### `id_to_index_`
是一个`IdMapping`类型，从`column_id`到“列的顺序号”的映射。

注意：有的时候，在构造`Schema`对象的时候，是没有提供`colunn_id`列表的，这个时候`id_to_index_.size()`为`0`。

#### `first_is_deleted_virtual_column_idx_`
用来缓存第一个`IS_DELETED`的列。 如果在当前`Schema`中，没有类型为`IS_DELETE`的列，该属性的值为`kColumnNotFound`。

注意：如果一个列是`IS_DELETE`类型，那么说这个列是“虚拟列”（"virtual column"）。(将来，“虚拟列”可能会有很多类型，`IS_DELETE`是其中的一种。 目前只有这一种)

#### `has_nullables_`

标识当前`Schema`中，是否有“可为空”("nullable")的列。

```
  std::vector<ColumnSchema> cols_;
  size_t num_key_columns_;
  std::vector<ColumnId> col_ids_;
  ColumnId max_col_id_;
  std::vector<size_t> col_offsets_;

  // The keys of this map are StringPiece references to the actual name members of the
  // ColumnSchema objects inside cols_. This avoids an extra copy of those strings,
  // and also allows us to do lookups on the map using StringPiece keys, sometimes
  // avoiding copies.
  //
  // The map is instrumented with a counting allocator so that we can accurately
  // measure its memory footprint.
  int64_t name_to_index_bytes_;
  typedef STLCountingAllocator<std::pair<const StringPiece, size_t> > NameToIndexMapAllocator;
  typedef std::unordered_map<
      StringPiece,
      size_t,
      std::hash<StringPiece>,
      std::equal_to<StringPiece>,
      NameToIndexMapAllocator> NameToIndexMap;
  NameToIndexMap name_to_index_;

  IdMapping id_to_index_;

  int first_is_deleted_virtual_column_idx_;

  bool has_nullables_;

  // NOTE: if you add more members, make sure to add the appropriate code to
  // CopyFrom() and the move constructor and assignment operator as well, to
  // prevent subtle bugs.
```

### 接口列表

方法很多，下面只说重要的，或者有注意事项的。

#### 一堆`getter`和`setter`方法

都是普通的方法，没有需要特别说明的，自行看代码。

#### `Reset()`

提供了两个重载的`Reset()`。两个的区别是：是否指定了`column_id`列表。

注意：在`Reset()`方法中，会对参数的有效性进行检查。

所以，如果是使用`ColumnSchema`数组直接去构造`Schema`对象时（见`Schama`相应的构造函数），**最好是先构造出一个空的`Schema`对象，然后再调用相应的`Reset()`方法来进行赋值**。这样的好处是：因为`Reset()`有安全检查，所以不会构造出一个无效的`Schema`对象。（如果发现`Schema`是无效的，`Reset()`方法会抛出一个`assert`错误）。

#### 构造函数

##### 移动构造函数

大部分“非基本类型”的成员，都是采用`std::move()`进行构造。

**`name_to_index_`成员：**  
因为它采用了 定制的内存分配器，在“内存分配器”中会拥有一个指向`name_to_index_bytes_`的指针。

参见注释：调用`std::unordered_map::swap()`，只会交换两个map中保存的内容，但是不会交换“内存分配器”。其中使用的`std::move`语义，也会“移动”分配器。也就是说，调用了`name_to_index_.swap(xxx)`以后，会导致当前`name_to_index_`中的内存分配器……

>> 问题：  
>> 在“移动构造函数”中，是不是不应该再对`name_to_index_bytes_`进行`std::swap()`？  




#### `byte_size()`

当前`Schema`的一行数据，所有列一共需要占用的字节数。

注意：  
1. 不包含`null bitmap`的部分。
2. 对于可变长类型（`STRING`和`BINARY`）,只包含它的基础部分，不包含它所指向的具体数据的长度。

该方法不需要再遍历`cols`去计算，而是直接使用`col_offsets_`的最后一个元素（在填充`col_offsets_`的时候，就已经计算好保存在那里了）。

```
  size_t byte_size() const {
    DCHECK(initialized());
    return col_offsets_.back();
  }
```

#### `key_byte_size()`

在当前`Schema`所对应的一行数据中，所有`key`列所占用的内存大小。

```
  // Return the number of bytes needed to represent
  // only the key portion of this schema.
  size_t key_byte_size() const {
    return col_offsets_[num_key_columns_];
  }

```

说明：因为 1) 每个列的偏移量都已经计算好了（在`col_offsets_`中保存）；2) “key列”一定在“value列”的前面; 所以“第1个value列”的偏移量，就是“所有key列”要占用的内存大小。

#### `num_columns()`

获取当前`Schema`中列的数量。

这里要说的是：在实现该方法时的注意事项。

其实，在本类中，有很多个成员都可以达到目的。比如`cols_.size()`，`name_to_index_.size()`。

**不可以使用`id_to_index_.size()`或`col_ids_.size()`**  
因为有的时候，在构造`Schema`对象的时候，是没有提供`colunn_id`列表的，这个时候`id_to_index_.size()`和`col_ids_.size()`都为`0`。

**使用`name_to_index_.size`，要好于`cols_.size()`**  

`cols_`是一个`std:vector`类型，它在内部并没有记录一个`size_`属性，它的`size()`方法的实现类似于下面的形式：（参见： https://blog.csdn.net/v_july_v/article/details/6681522）
```
size_type size() const { 
    return size_type(end() - begin()); 
}
```
也就是说，`std::vector::size()`的实现是：将 “尾指针”减去“首指针”来计算 长度。

但是，要注意的是：通过两个指针相减，来计算长度，是隐藏着一个除法操作的：要除以在`std::vector`中存储类型的大小，即`sizeof(T)`。

```

        |---------------|---------------|-------------|...
        ^   sizeof(T)       sizeof(T)      sizeof(T)  ^
        |                                             |
       begin                                         end
       
    数组的长度 size = (end - begin) / sizeof(T)
```

因为sizeof(T)是一个常量，而“除以一个常量”会被编译器优化为一个乘法，所以最终`std::vector::size()`方法的开销，是一个乘法的开销。

另一方面，`name_to_index_`是一个`unordered_map`类型，它在内部会记录一个`size_`属性，所以`size()`方法就是直接返回该成员的值。

因为“乘法指令的开销”，是远大于“普通的load指令”的开销的，所以这里应该使用`name_to_index_.size()`。

```
  size_t num_columns() const {
    return name_to_index_.size();
  }
```

#### `column_by_id()`

通过“ColumnId”获取对应的`ColumnSchema`对象。

注意：
因为在某些时候，`col_ids_`和`id_to_index_`都是为空的。
而这个函数的返回值是`ColumnSchema&`的引用，所以该必须是一个存在的`ColumnSchema`对象（在方法的实现中也有检查`DCHECK_GE(idx, 0)`）。

所以，调用这个方法的时候，当前`Schema`对象中每个列的column_id都是应该被赋值过的。 （如果没有被赋值过，那么不能调用这个方法）

```
  const ColumnSchema& column_by_id(ColumnId id) const {
    int idx = find_column_by_id(id);
    DCHECK_GE(idx, 0);
    return cols_[idx];
  }
```

#### `initialized()`

当前`Schema`对象，是否已经被初始化。

对于一个`Schema`对象，如果已经指定了它的column列表，那么它就被认为是已经初始化了。

这里选择的判断方法是：当前的`col_offsets_`不是empty的。

```
bool initialized() const {
    return !col_offsets_.empty();
  }
```

#### `Compare()`

将当前`Schema`的两行数据，进行比较。

比较的方法是：从前向后，逐列进行比较。

注意：因为kudu中有 “主键”的特性，所以只需要比较所有的主键列（不需要比较value列）。

```
  template<class RowTypeA, class RowTypeB>
  int Compare(const RowTypeA& lhs, const RowTypeB& rhs) const {
    DCHECK(KeyEquals(*lhs.schema()) && KeyEquals(*rhs.schema()));

    for (size_t col = 0; col < num_key_columns_; col++) {
      int col_compare = column(col).Compare(lhs.cell_ptr(col), rhs.cell_ptr(col));
      if (col_compare != 0) {
        return col_compare;
      }
    }
    return 0;
  }
```

#### `CreateKeyProjection()`

对当前`Schema`中的key列进行投影。

>> 说明：一个“投影”，选择列集合的一个“子集”。也是用`Schema`类型进行表示。

返回一个`Schema`对象，其中的所有列都是`key`.

**一个待做的优化点：**  因为一个`Schema`的key是不变的，所以key列对应的投影也是不变的。也就是说，实际上是可以将“key列的投影”缓存住。这样多次调用就不需要每次都重新构建。

当然，不是所有的读写流程中都会去获取“key列的投影”，所以不用在构建`Schema`的时候就去创建出来。可以采用"lazy"的方式：即在第一次获取时才进行缓存。这样，对于多次获取的时候，后续请求就不需要在临时创建。

为了简单，在拷贝`Schema`对象的时候，也不需要拷贝这个“key列的投影”，如果在新的schema上需要获取“key列的投影”，重新构造即可。

```
  // Return the projection of this schema which contains only
  // the key columns.
  // TODO: this should take a Schema* out-parameter to avoid an
  // extra copy of the ColumnSchemas.
  // TODO this should probably be cached since the key projection
  // is not supposed to change, for a single schema.
  Schema CreateKeyProjection() const {
    std::vector<ColumnSchema> key_cols(cols_.begin(),
                                  cols_.begin() + num_key_columns_);
    std::vector<ColumnId> col_ids;
    if (!col_ids_.empty()) {
      col_ids.assign(col_ids_.begin(), col_ids_.begin() + num_key_columns_);
    }

    return Schema(key_cols, col_ids, num_key_columns_);
  }
```

#### `CopyWithColumnId()`

返回一个新的Schema对象，并且将`column_id`属性设置好。

注意：调用该方法时，要求当前`Schema`对象，column_id相关的属性，尚未被赋值。

#### `CreateProjectionByNames()`
给定一组列名，创建这组列的投影。

**结果中的所有列，都不再认为是key列**  

返回的`Schema`对象中，column_id相关的属性是否被初始化，取决于当前`Schema`对象中是否已经进行了初始化。即如果当前`Schema`对象中已经初始化了`column_id`，那么结果中也是已经初始化了column_id的。

#### `EncodeComparableKey()`

是一个模板函数。给定某个类型的`Row`，把它的key列进行编码，保存到一个`buff`中。

在编码以后，`buff`是可以直接按照字典序进行“二进制比较”的，（即可以使用`memcpy`进行比较）。

传进来的`buff`参数类型是`faststring*`，用来保存数据。返回的是`Slice`类型，该`Slice`中的数据指针 指向传入的`faststring`参数。

注意，在该方法中，会将传入的`buff`参数 原来的值覆盖掉。（即不是在`buff`的末尾追加，而是覆盖掉）。

```
template <class RowType>
  Slice EncodeComparableKey(const RowType& row, faststring *dst) const {
    DCHECK_KEY_PROJECTION_SCHEMA_EQ(*this, *row.schema());

    dst->clear();
    for (size_t i = 0; i < num_key_columns_; i++) {
      DCHECK(!cols_[i].is_nullable());
      const TypeInfo* ti = cols_[i].type_info();
      bool is_last = i == num_key_columns_ - 1;
      GetKeyEncoder<faststring>(ti).Encode(row.cell_ptr(i), is_last, dst);
    }
    return Slice(*dst);
  }
```

#### `VerifyProjectionCompatibility()`

作用：检查一个“投影”是否与“当前`Schema`对象”是兼容的。

>> 备注：虽然当前是`public`方法，但是这个方法只被`GetMappedReadProjection()`使用，所以应该是`private`的(后面向kudu提交一个PR)。

使用场合：  
根据用户指定要查询的列（column name列表），构建出来一个`Schema`对象。  

这里用户所提供的“投影”，是来自于“用户的请求”。（因为用户并不直接感知column_id，所以在这个投影中，"column_id相关的属性"是一定没有被初始化的）。

但是在server上，是拿“用户的请求”和“表自身的`Schema`对象”进行比较。而在server上，这个“表自身的schema”中，一定是有"column_id"的。

**注意：**  
对于在“投影”中的虚拟列("virtual column")，是不需要进行检查的。

比较方法：
1. “投影”中的所有列，在当前`Schema`中都必须存在（除了“虚拟列”）；
2. “投影”中的所有列的类型，和当前`Schema`对象中也必须相同。

#### `GetMappedReadProjection()`

利用当前`Schema`对象，去转化“传入参数中的`Schema`对象”。

使用目的：  
主要就是将 用户client发过了的一个`Schema`(是没有`column id`的)，转化为一个有`column id`的。

这个方法在读取流程中，会被使用到。(该方法的名称中个，就有`read`字样)。

>> 问题： 为什么方法中会传入两个`Schema`对象，直接在第一个`Schema`对象上修改不行吗？

#### `GetProjectionMapping()`
是一个模板方法，在写流程中会用到。  

调用这个方法时，“当前`Schema`对象”是一个“投影”。会传入一个`base Schema`对象（tablet的schema）。

这个方法的作用：这个方法是在一个`Projector::Init()`中被调用。即根据当前的`projection schema`对象和“对应的`base schema`对象”，初始化当前`Projector`对象（其中的一些成员）。

遍历“当前`schema`对象”，对于其中的每一个列，根据是否在`base schema`中是否存在，分别取调用不同的函数。

1. `ProjectBaseColumn()`: 该列同时在当前`projection schema`和"base schema"中存在，并且两个列的类型 必须是相同的。（如果类型不相同，因为kudu目前不支持`adapter`，所以会报错） 
2. `ProjectDefaultColumn()`: 该列在"base schema"中不存在，但是该列在当前`Schema`中可以有“默认值”，或者可以为“nullable”。
3. `ProjectExtraColumn()`: 该列在"base schema"中不存在，并且该列在当前`Schema`中既没有“默认值”，也不可以为“nullable”。 (说明：这种场景，因为没有办法在“目标row”中进行填充它的值，**所以如果出现这种情况，就会报错**。)

注意：如果两个`Schema`对象都是有`column_id`的，那么映射关系通过`column_id`进行查找，否则，通过`column_name`进行查找。 

## `class SchemaBuilder`

用来帮助"create/alter"一个表的schema。

### 使用举例：

```
// Example:
  Status s;
  SchemaBuilder builder(base_schema);
  s = builder.RemoveColumn("value");
  s = builder.AddKeyColumn("key2", STRING);
  s = builder.AddColumn("new_c1", UINT32);
  ...
  Schema new_schema = builder.Build();
```

### 成员

**`col_names_`**成员  
主要用来判断是否“重复”。 因为一行中的多个列，不允许有重名的。

```
  ColumnId next_id_;
  std::vector<ColumnId> col_ids_;
  std::vector<ColumnSchema> cols_;
  std::unordered_set<std::string> col_names_;
  size_t num_key_columns_;
```

### 接口列表

提供的接口函数有：
1. `AddKeyColumn()`方法；
2. `AddColumn()`方法；
3. `AddNullableColumn()`方法；
4. `RemoveColumn()`方法；
5. `RenameColumn()`方法；
6. `ApplyColumnSchemaDelta()`方法；
7. `Build()`方法；
8. `BuildWithoutIds()`方法：当前只有单测中使用。

这几个接口方法，都只是简单的操作本类的几个属性，没有什么特别需要注意的地方。

有一个地方需要注意：现在有两个属性保存了column name：1) `col_names_`; 2) `col_`中的每个元素都有`name_`属性；

所以如果有修改column name的地方，这两个地方都要同时修改。

修改column name的事件有：
1. 对column进行重命名；
2. 应用一个`ColumnSchemaDelta`对象，并且该修改信息中包含了重命名（有`new_name`属性）。

另外，在删除列的时候，也需要同时将对应的名字，从`col_names_`中移除。

#### `is_valid()`方法

一个有效的`Schema`，必须是包含列的。

所以这里判断是否为valid的方法是：当前`cols.size()`是否非空。

#### `Reset()`方法

有两个重载的`Reset()`方法:
1. 没有参数的`Reset()`，会清空当前对象的所有成员；
2. 有参数的`Reset()`,参数是一个`Schema`对象，该方法会使用这个`Schema`来初始化当前的`SchemaBuilder`对象。

注意：在有参数的`Reset()`返回后，和`column_id`有关的属性（`newt_id`和`col_ids`）都会被赋值。

因为在`Schema`对象中，可能是不包含`column_id`信息的，所以在这个`Reset(Schema)`中，如果发现参数`Schema`对象没有`column_id`信息，那么就会为初始化这些列的column_id。

#### `RemoveColumn()`

注意：在移除一个列时，也不会修改`next_id_`。即使“被移除列”的id是最大的。






