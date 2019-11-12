[TOC]

文件： `src/kudu/tablet/mutation.h`

## `class Mutation`

用来描述“在一行数据”上的单个修改。

注意：该对象，也作为“修改链表”(`redo记录`)中的 节点(`Node`)。（所以，其中有`next_`成员，指向下一条修改）。

### 成员

#### `timestamp_`
当前修改的时间戳。

#### `next_`
指向在“redo链表”中的下一个节点。

#### `changelist_size_`
“修改内容”的大小。即在`changelist_data_`数组中的真实长度。

因为“修改内容”是一个`char*`指针，所以要用一个独立的变量，来表示它的长度。

#### `changelist_data_`
具体修改的内容。

注意1：（数据格式）  
其中存储数据的格式，就是`RowChangeList`的数据格式。 是可以直接用`RowChangeList(Slice(changelist_data_, changelist_size_))`来构造出`RowChangeList`对象的。

说明：因为对于`Mutation`来说，都是在进行“修改”，而kudu是不允许修改`key`的，所以在`Mutation`链表中，每一项的修改内容（即`changelist_data_`的内容），都是只包含对`value`列的修改内容（不包含`key`列的内容）。

```
  Timestamp timestamp_;
  Mutation *next_;

  uint32_t changelist_size_;
  char changelist_data_[0];
```

注意2：这里在声明`changelist_data_`的时候，不是直接声明为`char*`，而是用的“长度为0的数组”。

##### 使用“长度为0的char数组” VS 直接使用`char*`
参见： https://my.oschina.net/u/176416/blog/33054  浅析长度为0的数组  

注意：使用长度为`0`的数组时，经常会有以下特点：
1. 该数组 是当前`struct`的最后一个元素；
2. 有一个额外的元素，用来标识 该数组的真实长度；

**注意：申请内存时 长度的指定**  
“使用长度为`0`的数组”，在申请内存时，其长度并不只是`sizeof(x)`，还要加上“该数组实际的长度”。

比如：参见`Mutation::CreateInArena()`，在申请内存的时候，指定的长度是`sizeof(Mutation) + rcl.slice().size()`（这里`rcl.slice().size()`就是`changelist_data_`中要保存的数据的长度）。

**注意：长度为0的数组，是最后的一个元素**  
之所以要放在最后，是为了方便获取它的首地址。(因为是结构体的最后一个元素，所以不需要任何计算，在申请之后，数组自然就指向“新申请内存的最后位置”，也就是该数组应该在的位置)

比如：
```
struct Type {
    xxxx;
    T arr[0];
}
假定：
1. 整个`struct`的大小为len1， 即size(struct) = len1
2. 数组元素类型是`T`，元素个数为 len2

那么申请的内存大小为：
    size(struct) + sizeof(T) * len2

    |<----------- 申请的内存 --------------->|

    |----------------------------------------|
    
    |<--- struct的内存 --->|<--数组的内存 -->|
    
数组的首地址为：即 `object->arr`自然就指向了“数组的内存”的首地址，不需要任何额外的计算。
```

**注意：元素的访问方式**  
虽然申请的时候长度为0，但是因为本质上它仍然就是一个指针，所以可以向普通数组那样，使用`operator[]`去访问。

**总结：**  
“使用长度为`0`的数组”，相比于“使用指针”，区别是：
1. “长度为0的数组”并不占有内存空间，即计算`sizeof(struct)`中，不会计算该数组，而“指针方式”需要占用内存空间，在`sizeof(*)`时，会占用一个指针的大小。
2.  对于长度`len`的数组，在申请内存空间时，采用一次性分配的原则进行（即“结构体的内存”和“数组的内存"是连续的，这会有更好的“cpu cache命中率”）；对于包含指针的结构体，才申请空间时需分别进行，释放时也需分别释放，而且内存是不连续的。

### 接口列表

#### 静态方法
##### `static CreateInArena()`
传入一个“内存池”(`Arena`)，在其中创建一个`Mutation`对象。

**注意：不需要手动释放返回的`Mutation`对象。**  
虽然返回的`Mutation`对象是`new`出来的，但是它的地址是在“内存池”中的，所以调用者并不需要手动的对它进行析构。 因为在内存池被释放的时候，该对象占用的内存就会被自动释放。

```
template<class ArenaType>
inline Mutation *Mutation::CreateInArena(
  ArenaType *arena, Timestamp timestamp, const RowChangeList &rcl) {
  DCHECK(!rcl.is_null());

  size_t size = sizeof(Mutation) + rcl.slice().size();
  void *storage = arena->AllocateBytesAligned(size, BASE_PORT_H_ALIGN_OF(Mutation));
  CHECK(storage) << "failed to allocate storage from arena";
  auto ret = new (storage) Mutation();
  ret->timestamp_ = timestamp;
  ret->next_ = NULL;
  ret->changelist_size_ = rcl.slice().size();
  memcpy(ret->changelist_data_, rcl.slice().data(), rcl.slice().size());
  return ret;
}
```

说明：这里先从“内存池”中申请内存，然后利用`new (addr) Mutation`的方式，在之前已经申请的内存上，初始化`Mutation`对象（不是在“堆”上直接申请内存并初始化）。

##### `static StringifyMutationList()`

将“修改链表”，从当前节点开始，打印成字符串来描述。

只在`debug`时使用。

```
std::string Mutation::StringifyMutationList(const Schema &schema, const Mutation *head) {
  std::string ret;

  ret.append("[");

  bool first = true;
  while (head != nullptr) {
    if (!first) {
      ret.append(", ");
    }
    first = false;

    StrAppend(&ret, "@", head->timestamp().ToString(), "(");
    ret.append(head->changelist().ToString(schema));
    ret.append(")");

    head = head->acquire_next();
  }

  ret.append("]");
  return ret;
}
```

##### `static ReverseMutationList()`
翻转链表。

注意这里的算法，时间复杂度为`O(N)`。

翻转以后的头指针，通过参数`list`返回。

```
inline void Mutation::ReverseMutationList(Mutation** list) {
  Mutation* prev = nullptr;
  Mutation* cur = *list;
  while (cur != nullptr) {
    Mutation* next = cur->next_;
    cur->next_ = prev;
    prev = cur;
    cur = next;
  }
  *list = prev;
}
```


#### `changelist()`
将当前“修改”的内容，转化为`RowChangeList`对象。

```
  RowChangeList changelist() const {
    return RowChangeList(Slice(changelist_data_, changelist_size_));
  }
```

#### `next()`和`acquire_next()`
获取当前节点的下一个节点（即下一个“修改”）。

**注意：在针对一个“在内存中的store” 进行遍历它的所有修改时，一定要用`acquire_next()`方法。**

>> 问题：为什么？直接使用`next()`为什么会有问题？

```
  Mutation *next() { return next_; }
  const Mutation *next() const { return next_; }

  const Mutation* acquire_next() const {
    return reinterpret_cast<const Mutation*>(base::subtle::Acquire_Load(
        reinterpret_cast<const AtomicWord*>(&next_)));
  }
```

#### `set_next()`
设置当前节点的下一个节点。

```
  void set_next(Mutation *next) {
    next_ = next;
  }
```

#### `AppendToListAtomic()`

将修改原子的添加到“修改链表”的末尾。

这个方法的实现具有`Release`的内存语义。 所有的指针以及在链表中的所有`Mutation`对象，都必须是“按照`word`进行对齐”的。

注意：这个方法的原子语义来源。  
需要用户在外层进行 并发控制。   
比如说：用户已经控制了，不会在在同一个`Row`上进行并发操作。  

```
void Mutation::AppendToListAtomic(Mutation** redo_head, Mutation** redo_tail) {
  next_ = nullptr;
  if (*redo_tail == nullptr) {
    Release_Store(reinterpret_cast<AtomicWord*>(redo_head),
                  reinterpret_cast<AtomicWord>(this));
    Release_Store(reinterpret_cast<AtomicWord*>(redo_tail),
                  reinterpret_cast<AtomicWord>(this));
  } else {
    Release_Store(reinterpret_cast<AtomicWord*>(&(*redo_tail)->next_),
                  reinterpret_cast<AtomicWord>(this));
    Release_Store(reinterpret_cast<AtomicWord*>(redo_tail),
                  reinterpret_cast<AtomicWord>((*redo_tail)->next_));
  }
}
```

>> 问题：这里为什么能够保证“原子性”？ 连续两个`Release_Store()`操作之间，如果有并发，会不会违背原子性？是不是需要由用户在函数的外部进行控制？

#### `PrependToList()`

添加到“修改链表”的头部。

```
  void PrependToList(Mutation** list) {
    this->next_ = *list;
    *list = this;
  }
```
