[TOC]

# type_encodings.h

文件：`src/kudu/cfile/type_encodings.h`

## `class TypeEncodingInfo`

有以下几个作用：
1. 包含一些 类型“编码”和“解码”相关的信息；
2. 对于每种支持的编码类型，提供用来提供创建`BlockDecoder`和`BlockBuilder`对象的接口（也就是说，`TypeEncodingInfo`类，实际上是创建`BlockDecoder`和`BlockBuilder`对象的“**工厂类**”）。

说明：
1. 不允许显式的构造该对象。(构造函数是`private`的)，使用“静态方法”来获取相应的对象；
2. 对于任何一个<`DataType`, `EncodingType`>映射关系，全局只有一个`TypeEncodingInfo`对象即可。  
    即，如果确定了`DataType`和`EncodingType`，在进程中只会对应 唯一的`TypeEncodingInfo`对象。  

说明2：本类的“起名”来源：其实对于任何表的任一列，只要确定了它的`DataType`和`EncodingType`，就会有唯一的实现。也就是说，这里的对象是通用的。   
即`TypeEncodingInfo`，可以拆解为`type-encoding-info`，即：针对“数据类型”和“编码方法”的“通用信息”。

然后在这个信息中，提供了“读写操作”相关的结构，即创建`BlockBuilder`和`BlockDecoder`的“工厂方法”。

### 成员

#### `encoding_type_`
当前对象的“编码类型”；

#### `create_builder_func_`
创建`BlockBuilder`对象的“函数指针”。

#### `create_decoder_func_`
创建`BlockDecoder`对象的“函数指针”。

```
template<typename TypeEncodingTraitsClass>
  explicit TypeEncodingInfo(TypeEncodingTraitsClass t);

  EncodingType encoding_type_;

  typedef Status (*CreateBlockBuilderFunc)(BlockBuilder **, const WriterOptions *);
  const CreateBlockBuilderFunc create_builder_func_;

  typedef Status (*CreateBlockDecoderFunc)(BlockDecoder **, const Slice &,
                                           CFileIterator *);
  const CreateBlockDecoderFunc create_decoder_func_;
```

### 接口列表

#### ‘模板化’的“构造函数”

**该类的声明中并没有模板，但是构造函数中有模板参数**。

```
class TypeEncodingInfo {
public:
  ...

private:

  template<typename TypeEncodingTraitsClass>
  explicit TypeEncodingInfo(TypeEncodingTraitsClass t);
}
```

这个意思是说：
1. 因为“类声明”中没有“模板参数”，所以这个类不是“模板类”；
2. 因为“构造函数”中有“模板参数”，所以构造函数是“模板函数”。 

因为对于每种类型进行特化，实际上就会生成一个新的函数。所以，该类实际上会有很多个重载的‘构造函数’。

```
template<typename TypeEncodingTraitsClass>
TypeEncodingInfo::TypeEncodingInfo(TypeEncodingTraitsClass t)
    : encoding_type_(TypeEncodingTraitsClass::kEncodingType),
      create_builder_func_(TypeEncodingTraitsClass::CreateBlockBuilder),
      create_decoder_func_(TypeEncodingTraitsClass::CreateBlockDecoder) {
}
```

从具体的实现看，该函数对“模板参数”的要求是，**要有`3`个 静态的 “类成员”或“类函数”**：
1. `kEncodingType`：表示编码类型；
2. `CreateBlockBuilder`：创建`BlockBuilder`的工厂函数；
3. `CreateBlockDecoder`：创建`BlockDecoder`的工厂函数；

#### `static Get()`
给定一个`DataType`和`EncodingType`，返回针对这两个参数的“编解码工厂对象”。

```
Status TypeEncodingInfo::Get(const TypeInfo* typeinfo,
                             EncodingType encoding,
                             const TypeEncodingInfo** out) {
  return Singleton<TypeEncodingResolver>::get()->GetTypeEncodingInfo(typeinfo->physical_type(),
                                                                     encoding,
                                                                     out);
}
```

说明：`out`参数，是一个“指针的指针”，即最终获取的“编解码工厂对象”，是通过这个参数 返回给 调用者的。

#### `static GetDefaultEncoding()`
给定一个`DataType`(`TypeInfo`)，返回该类型的“默认编码类型”。

```
const EncodingType TypeEncodingInfo::GetDefaultEncoding(const TypeInfo* typeinfo) {
  return Singleton<TypeEncodingResolver>::get()->GetDefaultEncoding(typeinfo->physical_type());
}
```

#### `CreateBlockBuilder()`
工厂方法，创建一个`BlockBuilder`对象；

#### `CreateBlockDecoder()`
工厂方法，创建一个`BlockDecoder`对象；

## `DataTypeEncodingTraits`模板类 及其 多个特化类 和 子类

首先声明了 有`2`个模板参数的`DataTypeEncodingTraits`类。

注意：这里默认的内容是空的，具体的内容是由“特化类”中进行填充的。

**总结：**  
上面说过，在`TypeEncodingInfo`类的构造函数中，对“模板参数”的要求是有`3`个相应的成员。  
实际上，这里的`TypeEncodingTraits`类，就是最终传递给`TypeEncodingInfo`类的模板参数。  

如下文将说的，`TypeEncodingTraits`是子类，`DataTypeEncodingTraits`是父类。

在实现时：
1. 父类`DataTypeEncodingTraits`提供了`2`个：即那2个函数指针。
2. 子类`TypeEncodingInfo`提供了`1`个：即`kEncodingType`；

### 子类 -- `TypeEncodingTraits`

```
  template<DataType Type, EncodingType Encoding> struct TypeEncodingTraits
```

注意1：因为 `TypeEncodingTraits类`是 `DataTypeEncodingTraits类` 的子类，
所以对于上面所有的“`DataTypeEncodingTraits`的特化类”的子类，都有相应的`TypeEncodingTraits`子类。

在使用模板类 `TypeEncodingTraits` 的时候，只需要提供适当的模板参数，
那么就决定了哪个特化类是它的父类，从而也就继承了对应的方法。  

比如：在`DataTypeEncodingTraits`的每个特化类中，都提供了两个`static`方法（：`CreateBlockBuilder()`和`CreateBlockDecoder()`），所以，作为子类的`TypeEncodingTraits`，自然也就有了这两个方法。

### 特化类
 
有多个特化类：  
1. 特化一个参数（`偏特化`）：  
  - `template<DataType Type> struct DataTypeEncodingTraits<Type, PLAIN_ENCODING>`
  - `template<DataType Type> struct DataTypeEncodingTraits<Type, BIT_SHUFFLE>`
  - `template<DataType IntType> struct DataTypeEncodingTraits<IntType, RLE>`

2. 特化两个参数：
  - `template<> struct DataTypeEncodingTraits<BINARY, PLAIN_ENCODING>`
  - `template<> struct DataTypeEncodingTraits<BOOL, PLAIN_ENCODING>`
  - `template<> struct DataTypeEncodingTraits<BOOL, RLE>`
  - `template<> struct DataTypeEncodingTraits<BINARY, PREFIX_ENCODING>`
  - `template<> struct DataTypeEncodingTraits<BINARY, DICT_ENCODING>`

在所有的特化类中，都提供了两个静态方法(`static`方法)：`CreateBlockBuilder()`和`CreateBlockDecoder()`。 
  


#### “偏特化”的`DataTypeEncodingTraits`类

“偏特化”的参数是`EncodingType Encoding`。

##### 当`DataType`可以为任意值时
针对`EncodingType`的如下两个值进行了特化：
1. `PLAIN_ENCODING`
2. `BIT_SHUFFLE`

也就是说，对于所有的`DataType`，都有对应的`DataTypeEncodingTraits`类。  
即，对于任何的数据类型(`DataType`)，都可以进行这两种（`EncodingType`）编码。

注意：当`EncodingType == PLAIN_ENCODING`的时候，对于`DataType`为`BINARY`和`BOOL`的场景， 后面还进行了“全特化”。   
按照“模板函数”的匹配原则，对于`<BINARY, PLAIN_ENCODING>`和`<BOOL, PLAIN_ENCODING>`，会去匹配后面的“全特化”版本。  

##### 当`DataType`为整型(`IntType`)时
针对`EncodingType == RLE`进行了特化。

这里的意思是说，所有的`整型`都支持`RLE`编码。

#### “全特化”的`DataTypeEncodingTraits`类。

两个模板参数都进行指定；

说明：如果存在某个特化类型，就说明对于给定的`DataType`和`EncodingType`，就有对应的`DataTypeEncodingTraits`类的对象，进行“编解码”工作。

也就是说，如果给定的`DataType`和`EncodingType`，这里没有进行显式特化，那么就使用默认的`DataTypeEncodingTraits类`。 而默认的`DataTypeEncodingTraits`版本，并没有声明`2`个相应的工厂方法，所以是不能用来去初始化`TypeEncodingInfo`类的（所以，也就是不支持对相应的`<DataType, EncodingType>`对进行编码）。

注意1：针对`BINARY`和`BOOL`类型，在`EncodingType == PLAIN_ENCODING`时，会覆盖上面偏特化的“类版本”。

## `struct EncodingHash`
函数对象，用来对`pair<DataType, EncodingType>`计算`hash`值。

仅在`.cpp`文件中定义，所以只能在本文件中使用（实际上，只在`TypeEncodingResolver`类中使用，作为`std::unordered_map`的模板参数）。

```
struct EncodingMapHash {
  size_t operator()(pair<DataType, EncodingType> pair) const {
    return (pair.first + 31) ^ pair.second;
  }
};
```

说明：这里计算`hash`的方式，造成冲突的概率还是比较大的。

>> 问题：提交一个`PR`.

## `class TypeEncodingResolver`

仅在`.cpp`文件中定义，所以只能在本文件中使用

是`TypeEncodingInfo`的 友元类。

注意：**这是一个单例类**。在`TypeEncodingInfo`中的两个`static`方法(`Get()`和`GetDefaultEncoding()`)，会使用这个类。

1. 维护了所有支持的“`DataType` -> `EncodingType`”的映射，
2. 其中 对于每个“数据类型”，第一个被添加进去的“编码类型”，就是针对该数据类型的“默认编码类型”。

>> 备注：之前的版本中，`AddMapping()`方法的实现有点问题。之前的实现中，不能让每个`DataType`第一个添加进去`EncodingType`成为默认的，而是 最后一个。  
>> 这里给`Kudu`提了一个PR: http://gerrit.cloudera.org:8080/14147  
>> `commit_id： e253bb88c667c8337decda188bef85898661aadb`

### 成员

#### `mapping_`
从`pair<DataType, EncodingType>`到“`TypeEncodingInfo`对象”的映射。

#### `default_mapping_`
保存每种`DataType`的“默认编码”。

```
  unordered_map<pair<DataType, EncodingType>,
      unique_ptr<const TypeEncodingInfo>,
      EncodingMapHash > mapping_;

  unordered_map<DataType, EncodingType, std::hash<size_t> > default_mapping_;

```

### 接口方法

#### 构造函数

会针对所有支持的“映射”，都会调用一次`AddMapping`，从而构建映射关系。  
即从`pair<DataType, EncodingType>`到“`TypeEncodingInfo`对象”的映射。

#### `GetTypeEncodingInfo()`
工具方法，给定`DataType`和`EncodingType`，获取对应的`TypeEncodingInfo`对象。

```
  Status GetTypeEncodingInfo(DataType t, EncodingType e,
                             const TypeEncodingInfo** out) {
    if (e == AUTO_ENCODING) {
      e = GetDefaultEncoding(t);
    }
    const TypeEncodingInfo *type_info = mapping_[make_pair(t, e)].get();
    if (PREDICT_FALSE(type_info == nullptr)) {
      return Status::NotSupported(
          strings::Substitute("encoding $1 not supported for type $0",
                              DataType_Name(t),
                              EncodingType_Name(e)));
    }
    *out = type_info;
    return Status::OK();
  }
```

#### `GetDefaultEncoding()`
工具方法。获取每种`DataType`的“默认编码类型”。

```
  const EncodingType GetDefaultEncoding(DataType t) {
    return default_mapping_[t];
  }
```

#### `private AddMapping()`

构建 映射关系。

在构建本对象时（在“构造函数”中），会调用本函数，来构建所有支持的映射关系。

```
template<DataType type, EncodingType encoding> void AddMapping() {
    TypeEncodingTraits<type, encoding> traits;
    // The first call to AddMapping() for a given data-type is always the default one.
    // emplace() will no-op if the data-type is already present, so we can blindly
    // call emplace() here(i.e. no need to call find() to check before inserting)
    default_mapping_.emplace(type, encoding);
    mapping_.emplace(make_pair(type, encoding),
                     unique_ptr<TypeEncodingInfo>(new TypeEncodingInfo(traits)));
  }
```

注意：这里在向`default_mapping_`中添加映射关系的时候，没有检查是否已经指定的`DataType`已经存在，而是直接添加的。

原因是，对于一个`std::unordered_map`，如果其中的元素已经存在，那么它什么都不做。  
参见：http://www.cplusplus.com/reference/unordered_map/unordered_map/emplace/  
该方法的返回值是`std::pair<iterator, bool>`类型。其中的第二个`bool`类型，表示是否成功进行了插入。  
在要插入的元素已经存在时，`bool`的值为`false`。  

