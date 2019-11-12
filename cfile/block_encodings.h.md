[TOC]

文件: `src/kudu/cfile/block_encoding.h`

注意：`BlockBuilder`和`BlockDecoder`都是**抽象类**。用来定义构建一个`CFile Block`的接口。

参见`TypeEncodingInfo`类，根据`DataType`和`EncodingType`，区分了多种子类。

# `class BlockBuilder`

## `AppendExtraInfo()`
向当前`CFile`中添加一些“附加信息”。

比如说：在“字典编码模式”下，添加一个“`dictionary block`”。

注意1：对于大部分子类，该方法的内容都是空（即什么都不做）。

注意2：只有`BinaryDictBlockBuilder`类，重载了该方法，进行了一些工作。

## `IsBlockFull()`

在`CFileWriter`中，会使用该方法，来决定是否当前的`CFile Block`已经满了。

**说明：判断是否为满了的标志：**  
它的“预计的大小”，超过了配置的`WriterOptions::cfile_block_size`。

注意：如果这个`CFile Block`是“满”的，`CFileWriter`会去调用`FinishCurDataBlock()`方法。

## `Add()`
向当前`CFile Block`中，添加“一组值”。

返回值是：“实际被添加进去的值”的数量。

注意：在当前`Block`满的时候，“返回值”（实际被添加进去的“值”的数量），可能会小于“
传入的‘值’的数量”。

## `Finish()`
表示添加数据结束了（即不再添加新的数据）。

参数`ordinal_pos`: 当前`CFile Block`中，第一行数据，在整个`DiskRowSet`中对应的“行号”。

函数的返回值：`Slice`对象所“间接引用”的数据，在当前`BlockBuilder`对象的“缓存池”中。

如果发生如下两种情况，`Slice`中的指针会失效（注意：失效之后，不可以再用原来的`Slice`对象引用它）
1. 当前`BlockBuilder`对象被析构；
2. 再次调用了`Finish()`方法。

## `Reset()`
丢弃掉已经添加的数据。

注意：之前通过`Finish()`或`GetFirstKey()`返回的`Slice`所“间接引用”的内存，都会被失效。

说明：在执行该方法以后，`Count() == 0`等式成立。

## `Count()`
被添加到当前`BlockBuilder`中的“值”的数量。(因为1个“值”只来自于一行数据，所以`Count()`也是当前已经处理的“行”的数量)

## `GetFirstKey()`
在当前`index block`中的第一个“值”。

说明1：对于“基于指针”的类型（比如`STRING`），它所指向的内存，在调用`Reset()`之前是有效的。

说明2：如果当前block尚未添加任何数据，那么返回`Status::NotFound`.

## `GetLastKey()`
在当前`index block`中的最后一个“值”。

说明1：和`GetFirstKey()`一样，对于“基于指针”的类型（比如`STRING`），它所指向的内存，在调用`Reset()`之前是有效的。

说明2：和`GetFirstKey()`一样，如果当前block尚未添加任何数据，那么返回`Status::NotFound`.

```
  virtual Status AppendExtraInfo(CFileWriter *c_writer, CFileFooterPB* footer) {
    return Status::OK();
  }

  virtual bool IsBlockFull() const = 0;

  virtual int Add(const uint8_t *vals, size_t count) = 0;

  // Return a Slice which represents the encoded data.
  //
  virtual Slice Finish(rowid_t ordinal_pos) = 0;

  virtual void Reset() = 0;

  virtual size_t Count() const = 0;

  virtual Status GetFirstKey(void *key) const = 0;

  virtual Status GetLastKey(void *key) const = 0;

```

# `class BlockDecoder`

用来解析一个`cfile block`.

## 接口列表

### `ParseHeader()`
解析当前`cfile block`的`header`。

注意区分：这里要解析的是`cfile block`自己的`header`，并不是`cfile`的`header`）.

### `SeekToPositionInBlock()`
在当前`cfile block`内部，跳转到“指定的序号的行”。

比如说`SeekToPositionInBlock(0)`，就是要跳转到当前`cfile block`的第`0`行。

注意：传入的`pos`参数，不能大于当前`cfile block`的总行数（即`Count()`的返回值）。

### `SeekAtOrAfterValue()`

跳转到“指定的`value`值”。

**注意：如果调用该方法，必须保证当前`cfile block`中的数据是 “有序”的。** 

在执行完本函数以后，有两种可能的结果：
1. 指向“所给定的值”；
2. 指向 最小的、大于“所给定的值”的值。即大于“所给定值”后面的第一个值。

说明1：如果是能够“精确找到”的，那么参数`exact_match`的值会被赋值为`true`;

这样，上层使用者 就可以根据该参数的值，来判断在当前`cfile block`中是否存在指定的值。

说明2：和普通的在有序场景上，使用迭代器的方式类似。 
1. 如果“给定的值”，如果小于在`cfile block`中最小的值（即第一行数据），那么在函数返回后，指针会指向“第一条数据”；
2. 如果“给定的值”，大于该`cfile block`中的所有数据，那么在该函数返回后，那么会返回`Status::NotFound`错误。

### `SeekForward()`
将当前“解码器” 前进“指定数量”的值。

**这个方法的主要场景：就是为了“跳过一段值”。**

说明：参数`n`既是一个“传入参数”，也是一个“传出参数”。 在该函数成功执行后，参数`n`的值为“实际跳过的值个数”。

注意：如果“所给定需要跳过的数量”，超过了当前`cfile block`中剩余的数量，那么在该函数返回后，指向“文件的末尾”。这时参数`n`的值会被减小为“剩余的值的数量”。

### `CopyNextValues()`
将指定数量的值，拷贝到`dst`中。

注意：调用者要保证，在`dst`中，一定要有足够的空间（即至少要能容纳下`n`个）。

说明1：`dst`是一个`ColumnDataView`类型，其实就是封装了一个列的“一组值”。参见`src/kudu/common/columnblock.h`

说明2：参数`n`既是一个“传入参数”，也是一个“传出参数”。 在该函数成功执行后，参数`n`的值为“实际拷贝的值个数”。

说明3：对于`STRING`和`BINARY`这种，需要“间接引用”其它地址的数据的 数据类型，它们所间接引用的数据，会使用`dst`对象中的`arena`来进行重新分配。

### `CopyNextAndEval()`
从当前`cfile block`中获取一组值，并且检查它们是否符合“给定的过滤条件”。

说明1：其它都和`CopyNextValues()`类似，只是多了 要判断是否符合“过滤条件”。

注意：这里是“要判断是否符合‘过滤条件’”，但不是“根据‘过滤条件’进行过滤”。 也就是说，在`cfile block`中的行，无论是否符合“过滤条件”，最终都会被拷贝走。只不过，如果不符合过滤条件，那么会被"打上标记".

说明2：拷贝的目的地是`ColumnDataView`，用来打标记的对象是`SelectionVectorView`。

>> 问题：注释中的“后置条件”，是什么意思？

### `HasNext()`
是否已经达到末尾。

说明：如果该值为`true`，那么下一次调用`CopyNextValues()`，至少会拷贝`1`条数据。

### `Count()`
在当前`cfile block`中的，数据总行数。

### `GetCurrentIndex()`
获取当前数据行，在当前`cfile block`中的序号。

### `GetFirstRowId()`
获取当前数据行，在当前`Rowset`中的“行号”。

区分：`GetCurrentIndex()` VS `GetFirstRowId()`
1. 前者是返回在`cfile block`中的序号。
2. 后者是返回在`Rowset`中的序号。 如果当前`cfile block`中的第一行数据，在`rowset`中的序号为`m`，那么`GetFirstRow`

```
class BlockDecoder {
 public:
  BlockDecoder() { }

  virtual Status ParseHeader() = 0;

  virtual void SeekToPositionInBlock(uint pos) = 0;

  virtual Status SeekAtOrAfterValue(const void *value,
                                    bool *exact_match) = 0;

  virtual void SeekForward(int* n) {
    DCHECK(HasNext());
    *n = std::min(*n, static_cast<int>(Count() - GetCurrentIndex()));
    DCHECK_GE(*n, 0);
    SeekToPositionInBlock(GetCurrentIndex() + *n);
  }

  virtual Status CopyNextValues(size_t *n, ColumnDataView *dst) = 0;

  // Fetch the next values from the block and evaluate whether they satisfy
  // the predicate. Mark the row in the view into the selection vector. This
  // view denotes the current location in the CFile.
  //
  // Modifies *n to contain the number of values fetched.
  //
  // POSTCONDITION: ctx->decoder_eval_supported_ is not kNotSet. State must
  // be consistent throughout the entire column.
  virtual Status CopyNextAndEval(size_t* n,
                                 ColumnMaterializationContext* ctx,
                                 SelectionVectorView* sel,
                                 ColumnDataView* dst) {
    RETURN_NOT_OK(CopyNextValues(n, dst));
    ctx->SetDecoderEvalNotSupported();
    return Status::OK();
  }

  virtual bool HasNext() const = 0;

  virtual size_t Count() const = 0;

  // Return the position within the block of the currently seeked
  // entry (ie the entry that will next be returned by CopyNextValues())
  virtual size_t GetCurrentIndex() const = 0;

  // Return the first rowid stored in this block.
  // TODO: get rid of this from the block decoder, and put it in a generic
  // header which is shared by all data blocks.
  virtual rowid_t GetFirstRowId() const = 0;

  virtual ~BlockDecoder() {}
 private:
  DISALLOW_COPY_AND_ASSIGN(BlockDecoder);
};
```
