---
title: live555 源码分析：基础设施
date: 2017-08-30 15:05:49
categories: live555
tags:
- 音视频开发
- 源码分析
- live555
---

live555 由多个模块组成，其中 UsageEnvironment 、 BasicUsageEnvironment 和 groupsock 分别提供了事件循环，输入输出，基本的数据结构，以及网络 IO 等功能，它们可以看作 live555 的基础设施。对于 live555 的源码分析，就从这些基础设施，基本的数据结构开始。
<!--more-->
# HashTable
首先来看 `HashTable`，这是 live555 定义的一个范型关联容器。

## UsageEnvironment 的 HashTable
UsageEnvironment 中的 `HashTable` 类定义了接口，该接口定义如下：

```
class HashTable {
public:
  virtual ~HashTable();
  
  // The following must be implemented by a particular
  // implementation (subclass):
  static HashTable* create(int keyType);
  
  virtual void* Add(char const* key, void* value) = 0;
  // Returns the old value if different, otherwise 0
  virtual Boolean Remove(char const* key) = 0;
  virtual void* Lookup(char const* key) const = 0;
  // Returns 0 if not found
  virtual unsigned numEntries() const = 0;
  Boolean IsEmpty() const { return numEntries() == 0; }
  
  // Used to iterate through the members of the table:
  class Iterator {
  public:
    // The following must be implemented by a particular
    // implementation (subclass):
    static Iterator* create(HashTable const& hashTable);
    
    virtual ~Iterator();
    
    virtual void* next(char const*& key) = 0; // returns 0 if none
    
  protected:
    Iterator(); // abstract base class
  };
  
  // A shortcut that can be used to successively remove each of
  // the entries in the table (e.g., so that their values can be
  // deleted, if they happen to be pointers to allocated memory).
  void* RemoveNext();
  
  // Returns the first entry in the table.
  // (This is useful for deleting each entry in the table, if the entry's destructor also removes itself from the table.)
  void* getFirst(); 
  
protected:
  HashTable(); // abstract base class
};

// Warning: The following are deliberately the same as in
// Tcl's hash table implementation
int const STRING_HASH_KEYS = 0;
int const ONE_WORD_HASH_KEYS = 1;
```

尽管 `HashTable` 要定义为范型关联容器，然而它没有使用 C++ 的模板。`HashTable` 要求键为一个 C 风格的字符串，即 `char const*`，而值为一个 `void*`，以此可以表示各种不同类型的值。就像 C++ 标准库中的许多容器一眼，`HashTable` 内部还定义了一个迭代器类，用来遍历这个容器。

其它添加、移除、查找元素等操作，与其它的普通容器设计并没有太大的不同。

`HashTable` 类实现了公共的与具体存储结构无关的两个模板函数 `RemoveNext()` 和 `getFirst()`，这两个函数分别用于移除容器中的下一个元素并返回元素的值，及得到容器中的第一个元素，它们的实现如下：
```
#include "HashTable.hh"

HashTable::HashTable() {
}

HashTable::~HashTable() {
}

HashTable::Iterator::Iterator() {
}

HashTable::Iterator::~Iterator() {}

void* HashTable::RemoveNext() {
  Iterator* iter = Iterator::create(*this);
  char const* key;
  void* removedValue = iter->next(key);
  if (removedValue != 0) Remove(key);

  delete iter;
  return removedValue;
}

void* HashTable::getFirst() {
  Iterator* iter = Iterator::create(*this);
  char const* key;
  void* firstValue = iter->next(key);

  delete iter;
  return firstValue;
}
```
这两个函数，都借助于容器的迭代器来实现。`Iterator` 类的 `next(char const*& key)` 接收一个传出参数，用于将键返回给调用者。

从接口定义的角度来看 live555 定义的 `HashTable`。它的 `Iterator` 类对象是由 `Iterator` 类的静态方法 `create(HashTable const& hashTable)` 创建的，但销毁创建的对象的职责却是在调用者这里，这大大破坏了 `HashTable` 接口的实现的灵活性，比如创建的对象因此而无法做缓存。

`HashTable` 类及其迭代器类 `Iterator` 各自定义了一个静态方法 `create()`。这里使用了桥接的方式，`HashTable` 的这个方法像一座桥一样，把 `HashTable` 接口与接口的实现联系了起来。这个类静态函数都在实现接口的类中定义。

`HashTable` 有两种类型，分别用两个整型值来标识，`STRING_HASH_KEYS` 和 `ONE_WORD_HASH_KEYS`。当然，依赖于在 `create()` 方法中是根据 `HashTable` 类型创建相同类的对象还是创建不同类的对象，可以将 `create()` 方法的设计理解为是桥接，或者工厂方法。

## BasicUsageEnvironment 的 BasicHashTable
BasicUsageEnvironment 中的 `BasicHashTable` 提供了一个 `HashTable` 的实现。`BasicHashTable` 的定义如下：

```
// A simple hash table implementation, inspired by the hash table
// implementation used in Tcl 7.6: <http://www.tcl.tk/>

#define SMALL_HASH_TABLE_SIZE 4

class BasicHashTable: public HashTable {
private:
	class TableEntry; // forward

public:
  BasicHashTable(int keyType);
  virtual ~BasicHashTable();

  // Used to iterate through the members of the table:
  class Iterator; friend class Iterator; // to make Sun's C++ compiler happy
  class Iterator: public HashTable::Iterator {
  public:
    Iterator(BasicHashTable const& table);

  private: // implementation of inherited pure virtual functions
    void* next(char const*& key); // returns 0 if none

  private:
    BasicHashTable const& fTable;
    unsigned fNextIndex; // index of next bucket to be enumerated after this
    TableEntry* fNextEntry; // next entry in the current bucket
  };

private: // implementation of inherited pure virtual functions
  virtual void* Add(char const* key, void* value);
  // Returns the old value if different, otherwise 0
  virtual Boolean Remove(char const* key);
  virtual void* Lookup(char const* key) const;
  // Returns 0 if not found
  virtual unsigned numEntries() const;

private:
  class TableEntry {
  public:
    TableEntry* fNext;
    char const* key;
    void* value;
  };

  TableEntry* lookupKey(char const* key, unsigned& index) const;
    // returns entry matching "key", or NULL if none
  Boolean keyMatches(char const* key1, char const* key2) const;
    // used to implement "lookupKey()"

  TableEntry* insertNewEntry(unsigned index, char const* key);
    // creates a new entry, and inserts it in the table
  void assignKey(TableEntry* entry, char const* key);
    // used to implement "insertNewEntry()"

  void deleteEntry(unsigned index, TableEntry* entry);
  void deleteKey(TableEntry* entry);
    // used to implement "deleteEntry()"

  void rebuild(); // rebuilds the table as its size increases

  unsigned hashIndexFromKey(char const* key) const;
    // used to implement many of the routines above

  unsigned randomIndex(uintptr_t i) const {
    return (unsigned)(((i*1103515245) >> fDownShift) & fMask);
  }

private:
  TableEntry** fBuckets; // pointer to bucket array
  TableEntry* fStaticBuckets[SMALL_HASH_TABLE_SIZE];// used for small tables
  unsigned fNumBuckets, fNumEntries, fRebuildSize, fDownShift, fMask;
  int fKeyType;
};
```

`BasicHashTable` 定义了一个类 `TableEntry`，用于表示键-值对。`fNext` 字段用于指向计算的键的哈希值冲突的不同的键-值对中的下一个。

来看 `BasicHashTable` 的创建：
```
BasicHashTable::BasicHashTable(int keyType)
  : fBuckets(fStaticBuckets), fNumBuckets(SMALL_HASH_TABLE_SIZE),
    fNumEntries(0), fRebuildSize(SMALL_HASH_TABLE_SIZE*REBUILD_MULTIPLIER),
    fDownShift(28), fMask(0x3), fKeyType(keyType) {
  for (unsigned i = 0; i < SMALL_HASH_TABLE_SIZE; ++i) {
    fStaticBuckets[i] = NULL;
  }
}
. . . . . .
HashTable* HashTable::create(int keyType) {
  return new BasicHashTable(keyType);
}
```

`BasicHashTable` 用 `TableEntry` 指针的数组保存所有的键-值对。在该类对象创建时，一个小的 `TableEntry` 指针数组 `fStaticBuckets` 会随着对象的创建而创建，在 `BasicHashTable` 中元素比较少时，直接在这个数组中保存键值对，以此来优化当元素比较少的性能，降低内存分配的开销。

`fBuckets` 指向保存键-值对的 `TableEntry` 指针数组，在对象创建初期，它指向 `fStaticBuckets`，而在哈希桶扩容时，它指向新分配的 `TableEntry` 指针数组。对于容器中元素的访问，都通过
 `fBuckets` 来完成。`fNumBuckets` 用于保存 `TableEntry` 指针数组的长度。`fNumEntries` 用于保存容器中键-值对的个数。`fRebuildSize` 为哈希桶扩容的阈值，即当 `BasicHashTable` 中保存的键值对超过该值时，哈希桶需要扩容。`fDownShift` 和 `fMask` 用于计算哈希值，并把哈希值映射到哈希桶容量范围内。

## 向 BasicHashTable 中插入元素
可以通过 `Add(char const* key, void* value)` 向 `BasicHashTable` 中插入元素，这个函数的定义如下：

```
void* BasicHashTable::Add(char const* key, void* value) {
  void* oldValue;
  unsigned index;
  TableEntry* entry = lookupKey(key, index);
  if (entry != NULL) {
    // There's already an item with this key
    oldValue = entry->value;
  } else {
    // There's no existing entry; create a new one:
    entry = insertNewEntry(index, key);
    oldValue = NULL;
  }
  entry->value = value;

  // If the table has become too large, rebuild it with more buckets:
  if (fNumEntries >= fRebuildSize) rebuild();

  return oldValue;
}
```

向 BasicHashTable 中插入元素的过车大致如下：
1. 查找 BasicHashTable 中与要插入的键-值对的键匹配的元素 `TableEntry`。
2. 若找到，把该元素的旧的值保存在 `oldValue` 中。
3. 若没有找到，则通过 `insertNewEntry(index, key)` 创建一个 `TableEntry` 并加入到哈希桶中，`oldValue` 被赋值为 NULL。
4. 把要插入的键-值对的值保存进新创建或找到的 `TableEntry` 中。
5. 如果 BasicHashTable 中的元素个数超出 `fRebuildSize` 的大小，则对哈希桶扩容。
6. 返回元素的旧的值。

查找 BasicHashTable 中与要插入的键-值对的键匹配的元素 `TableEntry` 的过程如下：
```
BasicHashTable::TableEntry* BasicHashTable
::lookupKey(char const* key, unsigned& index) const {
  TableEntry* entry;
  index = hashIndexFromKey(key);

  for (entry = fBuckets[index]; entry != NULL; entry = entry->fNext) {
    if (keyMatches(key, entry->key)) break;
  }

  return entry;
}

Boolean BasicHashTable
::keyMatches(char const* key1, char const* key2) const {
  // The way we check the keys for a match depends upon their type:
  if (fKeyType == STRING_HASH_KEYS) {
    return (strcmp(key1, key2) == 0);
  } else if (fKeyType == ONE_WORD_HASH_KEYS) {
    return (key1 == key2);
  } else {
    unsigned* k1 = (unsigned*)key1;
    unsigned* k2 = (unsigned*)key2;

    for (int i = 0; i < fKeyType; ++i) {
      if (k1[i] != k2[i]) return False; // keys differ
    }
    return True;
  }
}
. . . . . .
unsigned BasicHashTable::hashIndexFromKey(char const* key) const {
  unsigned result = 0;

  if (fKeyType == STRING_HASH_KEYS) {
    while (1) {
      char c = *key++;
      if (c == 0) break;
      result += (result<<3) + (unsigned)c;
    }
    result &= fMask;
  } else if (fKeyType == ONE_WORD_HASH_KEYS) {
    result = randomIndex((uintptr_t)key);
  } else {
    unsigned* k = (unsigned*)key;
    uintptr_t sum = 0;
    for (int i = 0; i < fKeyType; ++i) {
      sum += k[i];
    }
    result = randomIndex(sum);
  }

  return result;
}
```
`lookupKey()` 首先通过 `hashIndexFromKey(key)` 根据键值对的键计算哈希值，并把该值映射到哈希桶容量范围内，得到索引。然后根据得到的索引，查找与传入的键匹配的元素。

这里可以更加清晰地看到，不同类型的 `HashTable` 之间的区别主要在于对待键的方式不同。STRING_HASH_KEYS 型的 `HashTable`，其键为传入的字符串指针指向的字符串的内容，而
 ONE_WORD_HASH_KEYS 型的 `HashTable`，其键则为传入的字符串指针本身。

计算最终的哈希值，并把该值映射到哈希桶容量范围内，得到索引的过程如下：
```
  unsigned randomIndex(uintptr_t i) const {
    return (unsigned)(((i*1103515245) >> fDownShift) & fMask);
  }
```

`insertNewEntry(index, key)` 创建一个 `TableEntry` 并加入到哈希桶中的过程如下：
```
BasicHashTable::TableEntry* BasicHashTable
::insertNewEntry(unsigned index, char const* key) {
  TableEntry* entry = new TableEntry();
  entry->fNext = fBuckets[index];
  fBuckets[index] = entry;

  ++fNumEntries;
  assignKey(entry, key);

  return entry;
}

void BasicHashTable::assignKey(TableEntry* entry, char const* key) {
  // The way we assign the key depends upon its type:
  if (fKeyType == STRING_HASH_KEYS) {
    entry->key = strDup(key);
  } else if (fKeyType == ONE_WORD_HASH_KEYS) {
    entry->key = key;
  } else if (fKeyType > 0) {
    unsigned* keyFrom = (unsigned*)key;
    unsigned* keyTo = new unsigned[fKeyType];
    for (int i = 0; i < fKeyType; ++i) keyTo[i] = keyFrom[i];

    entry->key = (char const*)keyTo;
  }
}
```
可见插入的过程主要为，
1. 创建 TableEntry 对象，并把它插入到键所对应的 TableEntry 指针数组中的 TableEntry 元素链的头部。
2. 增加容器中元素个数的计数。
3. 为 TableEntry 分配传入的键。

依据 `HashTable` 类型的不同，分配键的方式也不同。
1. 对于 STRING_HASH_KEYS 型的 `HashTable`，需要将传入的字符串指针指向的字符串的内容复制一份，赋值给 TableEntry 的 `key`。
2. 对于 ONE_WORD_HASH_KEYS 型的 `HashTable`，需要将传入的字符串指针本身，赋值给 TableEntry 的 `key`。
3. 对于 `fKeyType` 大于 0 的情况，需要将传入的字符串指针指向的字符串的内容的前 (sizeof(unsigned) * fKeyType) 个字节复制一份，赋值给 TableEntry 的 `key`。这种代码真是看得人胆颤心惊，万一传入的 key 字符串长度小于 (sizeof(unsigned) * fKeyType) 个字节呢？。。。

对比 `keyMatches()` 和 `assignKey()` 函数的实现，不难发现，当 `HashTable` 类型 `fKeyType` 大于0，且不是 `ONE_WORD_HASH_KEYS` 时，要求作为哈希表中键值对的键的字符串的长度固定为 (sizeof(unsigned) * fKeyType) 个字节。

然后来看，BasicHashTable 中对哈希桶扩容的过程：
```
void BasicHashTable::rebuild() {
  // Remember the existing table size:
  unsigned oldSize = fNumBuckets;
  TableEntry** oldBuckets = fBuckets;

  // Create the new sized table:
  fNumBuckets *= 4;
  fBuckets = new TableEntry*[fNumBuckets];
  for (unsigned i = 0; i < fNumBuckets; ++i) {
    fBuckets[i] = NULL;
  }
  fRebuildSize *= 4;
  fDownShift -= 2;
  fMask = (fMask<<2)|0x3;

  // Rehash the existing entries into the new table:
  for (TableEntry** oldChainPtr = oldBuckets; oldSize > 0;
       --oldSize, ++oldChainPtr) {
    for (TableEntry* hPtr = *oldChainPtr; hPtr != NULL;
	 hPtr = *oldChainPtr) {
      *oldChainPtr = hPtr->fNext;

      unsigned index = hashIndexFromKey(hPtr->key);

      hPtr->fNext = fBuckets[index];
      fBuckets[index] = hPtr;
    }
  }

  // Free the old bucket array, if it was dynamically allocated:
  if (oldBuckets != fStaticBuckets) delete[] oldBuckets;
}
```
在这里就是，
1. 为 `fBuckets` 分配一块新的内存，容量为原来的4倍。
2. 适当更新 `fNumBuckets`，`fRebuildSize`，`fDownShift` 和 `fMask` 等。
3. 将老的 `fBuckets` 中的元素，依据元素的 `key` 和新的哈希桶的容量，搬到新的 `fBuckets` 中。
4. 根据需要释放老的 `fBuckets` 的内存。

## 在 BasicHashTable 中查找元素
然后来看在 BasicHashTable 中查找元素的过程：
```
void* BasicHashTable::Lookup(char const* key) const {
  unsigned index;
  TableEntry* entry = lookupKey(key, index);
  if (entry == NULL) return NULL; // no such entry

  return entry->value;
}
```
这个过程主要是根据 key，通过 `lookupKey()` 查找到对应的元素 TableEntry，然后返回其 value。

## 移除 BasicHashTable 中的元素
来看移除 BasicHashTable 中的元素的过程：
```
Boolean BasicHashTable::Remove(char const* key) {
  unsigned index;
  TableEntry* entry = lookupKey(key, index);
  if (entry == NULL) return False; // no such entry

  deleteEntry(index, entry);

  return True;
}
. . . . . .
void BasicHashTable::deleteEntry(unsigned index, TableEntry* entry) {
  TableEntry** ep = &fBuckets[index];

  Boolean foundIt = False;
  while (*ep != NULL) {
    if (*ep == entry) {
      foundIt = True;
      *ep = entry->fNext;
      break;
    }
    ep = &((*ep)->fNext);
  }

  if (!foundIt) { // shouldn't happen
#ifdef DEBUG
    fprintf(stderr, "BasicHashTable[%p]::deleteEntry(%d,%p): internal error - not found (first entry %p", this, index, entry, fBuckets[index]);
    if (fBuckets[index] != NULL) fprintf(stderr, ", next entry %p", fBuckets[index]->fNext);
    fprintf(stderr, ")\n");
#endif
  }

  --fNumEntries;
  deleteKey(entry);
  delete entry;
}

void BasicHashTable::deleteKey(TableEntry* entry) {
  // The way we delete the key depends upon its type:
  if (fKeyType == ONE_WORD_HASH_KEYS) {
    entry->key = NULL;
  } else {
    delete[] (char*)entry->key;
    entry->key = NULL;
  }
}
```
移除 BasicHashTable 中的元素的过程，也是免不了要先找到元素在 `BasicHashTable` 中的 TableEntry 的，找到之后，通过 `deleteEntry()` 移除元素。

在 `deleteEntry()` 中，先把元素从 `BasicHashTable` 中移除出去，然后通过 `deleteKey()` 释放 key 占用的内存，随后释放 TableEntry 本身占用的内存。这里把 TableEntry 从 `BasicHashTable` 中移除出去采用了下面这种方法：
```
  TableEntry** ep = &fBuckets[index];

  Boolean foundIt = False;
  while (*ep != NULL) {
    if (*ep == entry) {
      foundIt = True;
      *ep = entry->fNext;
      break;
    }
    ep = &((*ep)->fNext);
  }
```
通过二级指针，遍历链表一趟，就将元素移除出去了。记得这是这中场景下 Linus 大神鼓励的一种写法。从链表中删除一个元素，用好几个临时变量，或者加许多判断的方法，都弱爆了。

## 通过 Iterator 遍历 BasicHashTable
可以使用 `BasicHashTable` 中定义的 `Iterator` 来遍历它。过程如下：
```
BasicHashTable::Iterator::Iterator(BasicHashTable const& table)
  : fTable(table), fNextIndex(0), fNextEntry(NULL) {
}

void* BasicHashTable::Iterator::next(char const*& key) {
  while (fNextEntry == NULL) {
    if (fNextIndex >= fTable.fNumBuckets) return NULL;

    fNextEntry = fTable.fBuckets[fNextIndex++];
  }

  BasicHashTable::TableEntry* entry = fNextEntry;
  fNextEntry = entry->fNext;

  key = entry->key;
  return entry->value;
}
. . . . . .
HashTable::Iterator* HashTable::Iterator::create(HashTable const& hashTable) {
  // "hashTable" is assumed to be a BasicHashTable
  return new BasicHashTable::Iterator((BasicHashTable const&)hashTable);
}
```
`BasicHashTable` 的 `fBuckets` 中的每个元素都保存一个 `TableEntry` 的链表。在这里会逐个链表地遍历。

 live555 定义的 `HashTable` 的内容就是这些了。

# UsageEnvironment
live555 中，`UsageEnvironment` 类扮演一个简单的控制器的角色。UsageEnvironment 模块中，`UsageEnvironment` 类的定义如下：
```
class TaskScheduler; // forward

// An abstract base class, subclassed for each use of the library

class UsageEnvironment {
public:
  Boolean reclaim();
      // returns True iff we were actually able to delete our object

  // task scheduler:
  TaskScheduler& taskScheduler() const {return fScheduler;}

  // result message handling:
  typedef char const* MsgString;
  virtual MsgString getResultMsg() const = 0;

  virtual void setResultMsg(MsgString msg) = 0;
  virtual void setResultMsg(MsgString msg1, MsgString msg2) = 0;
  virtual void setResultMsg(MsgString msg1, MsgString msg2, MsgString msg3) = 0;
  virtual void setResultErrMsg(MsgString msg, int err = 0) = 0;
	// like setResultMsg(), except that an 'errno' message is appended.  (If "err == 0", the "getErrno()" code is used instead.)

  virtual void appendToResultMsg(MsgString msg) = 0;

  virtual void reportBackgroundError() = 0;
	// used to report a (previously set) error message within
	// a background event

  virtual void internalError(); // used to 'handle' a 'should not occur'-type error condition within the library.

  // 'errno'
  virtual int getErrno() const = 0;

  // 'console' output:
  virtual UsageEnvironment& operator<<(char const* str) = 0;
  virtual UsageEnvironment& operator<<(int i) = 0;
  virtual UsageEnvironment& operator<<(unsigned u) = 0;
  virtual UsageEnvironment& operator<<(double d) = 0;
  virtual UsageEnvironment& operator<<(void* p) = 0;

  // a pointer to additional, optional, client-specific state
  void* liveMediaPriv;
  void* groupsockPriv;

protected:
  UsageEnvironment(TaskScheduler& scheduler); // abstract base class
  virtual ~UsageEnvironment(); // we are deleted only by reclaim()

private:
  TaskScheduler& fScheduler;
};
```
`UsageEnvironment` 类持有 `TaskScheduler` 的引用，并提供文本的输出操作，用于输出信息，其它还提供了获取 errno 的操作，在发生内部错误时的处理程序 `internalError()`，以及销毁自身的操作。`UsageEnvironment` 类本身实现了如下这几个函数：
```
Boolean UsageEnvironment::reclaim() {
  // We delete ourselves only if we have no remainining state:
  if (liveMediaPriv == NULL && groupsockPriv == NULL) {
    delete this;
    return True;
  }

  return False;
}

UsageEnvironment::UsageEnvironment(TaskScheduler& scheduler)
  : liveMediaPriv(NULL), groupsockPriv(NULL), fScheduler(scheduler) {
}

UsageEnvironment::~UsageEnvironment() {
}

// By default, we handle 'should not occur'-type library errors by calling abort().  Subclasses can redefine this, if desired.
// (If your runtime library doesn't define the "abort()" function, then define your own (e.g., that does nothing).)
void UsageEnvironment::internalError() {
  abort();
}
```
这些函数都比较简单，这里不再赘述。

`UsageEnvironment` 是一个接口类，BasicUsageEnvironment 模块中，通过两个类 `BasicUsageEnvironment` 和 `BasicUsageEnvironment0`，提供了它的一个实现。`BasicUsageEnvironment0` 类提供了那组直接操作字符串的函数的实现，该类的定义如下：
```
class BasicUsageEnvironment0: public UsageEnvironment {
public:
  // redefined virtual functions:
  virtual MsgString getResultMsg() const;

  virtual void setResultMsg(MsgString msg);
  virtual void setResultMsg(MsgString msg1,
		    MsgString msg2);
  virtual void setResultMsg(MsgString msg1,
		    MsgString msg2,
		    MsgString msg3);
  virtual void setResultErrMsg(MsgString msg, int err = 0);

  virtual void appendToResultMsg(MsgString msg);

  virtual void reportBackgroundError();

protected:
  BasicUsageEnvironment0(TaskScheduler& taskScheduler);
  virtual ~BasicUsageEnvironment0();

private:
  void reset();

  char fResultMsgBuffer[RESULT_MSG_BUFFER_MAX];
  unsigned fCurBufferSize;
  unsigned fBufferMaxSize;
};
```

这个类定义了一个缓冲区，大小为 `RESULT_MSG_BUFFER_MAX`。类的实现如下：
```
BasicUsageEnvironment0::BasicUsageEnvironment0(TaskScheduler& taskScheduler)
  : UsageEnvironment(taskScheduler),
    fBufferMaxSize(RESULT_MSG_BUFFER_MAX) {
  reset();
}

BasicUsageEnvironment0::~BasicUsageEnvironment0() {
}

void BasicUsageEnvironment0::reset() {
  fCurBufferSize = 0;
  fResultMsgBuffer[fCurBufferSize] = '\0';
}


// Implementation of virtual functions:

char const* BasicUsageEnvironment0::getResultMsg() const {
  return fResultMsgBuffer;
}

void BasicUsageEnvironment0::setResultMsg(MsgString msg) {
  reset();
  appendToResultMsg(msg);
}

void BasicUsageEnvironment0::setResultMsg(MsgString msg1, MsgString msg2) {
  setResultMsg(msg1);
  appendToResultMsg(msg2);
}

void BasicUsageEnvironment0::setResultMsg(MsgString msg1, MsgString msg2,
				       MsgString msg3) {
  setResultMsg(msg1, msg2);
  appendToResultMsg(msg3);
}

void BasicUsageEnvironment0::setResultErrMsg(MsgString msg, int err) {
  setResultMsg(msg);

  if (err == 0) err = getErrno();
. . . . . .
  appendToResultMsg(strerror(err));
#endif
}

void BasicUsageEnvironment0::appendToResultMsg(MsgString msg) {
  char* curPtr = &fResultMsgBuffer[fCurBufferSize];
  unsigned spaceAvailable = fBufferMaxSize - fCurBufferSize;
  unsigned msgLength = strlen(msg);

  // Copy only enough of "msg" as will fit:
  if (msgLength > spaceAvailable-1) {
    msgLength = spaceAvailable-1;
  }

  memmove(curPtr, (char*)msg, msgLength);
  fCurBufferSize += msgLength;
  fResultMsgBuffer[fCurBufferSize] = '\0';
}

void BasicUsageEnvironment0::reportBackgroundError() {
  fputs(getResultMsg(), stderr);
}
```
这组函数提供了把传入的字符串，加进缓冲区，以及把缓冲区中的内容输出到标准输出的功能。

`BasicUsageEnvironment` 类则提供了那组用于输出基本数据类型的操作符。这个类的定义如下：
```
class BasicUsageEnvironment: public BasicUsageEnvironment0 {
public:
  static BasicUsageEnvironment* createNew(TaskScheduler& taskScheduler);

  // redefined virtual functions:
  virtual int getErrno() const;

  virtual UsageEnvironment& operator<<(char const* str);
  virtual UsageEnvironment& operator<<(int i);
  virtual UsageEnvironment& operator<<(unsigned u);
  virtual UsageEnvironment& operator<<(double d);
  virtual UsageEnvironment& operator<<(void* p);

protected:
  BasicUsageEnvironment(TaskScheduler& taskScheduler);
      // called only by "createNew()" (or subclass constructors)
  virtual ~BasicUsageEnvironment();
};
```

定义比较简单。然后来看它的实现：
```
BasicUsageEnvironment::BasicUsageEnvironment(TaskScheduler& taskScheduler)
: BasicUsageEnvironment0(taskScheduler) {
. . . . . .
}

BasicUsageEnvironment::~BasicUsageEnvironment() {
}

BasicUsageEnvironment*
BasicUsageEnvironment::createNew(TaskScheduler& taskScheduler) {
  return new BasicUsageEnvironment(taskScheduler);
}

int BasicUsageEnvironment::getErrno() const {
#if defined(__WIN32__) || defined(_WIN32) || defined(_WIN32_WCE)
  return WSAGetLastError();
#else
  return errno;
#endif
}

UsageEnvironment& BasicUsageEnvironment::operator<<(char const* str) {
  if (str == NULL) str = "(NULL)"; // sanity check
  fprintf(stderr, "%s", str);
  return *this;
}

UsageEnvironment& BasicUsageEnvironment::operator<<(int i) {
  fprintf(stderr, "%d", i);
  return *this;
}

UsageEnvironment& BasicUsageEnvironment::operator<<(unsigned u) {
  fprintf(stderr, "%u", u);
  return *this;
}

UsageEnvironment& BasicUsageEnvironment::operator<<(double d) {
  fprintf(stderr, "%f", d);
  return *this;
}

UsageEnvironment& BasicUsageEnvironment::operator<<(void* p) {
  fprintf(stderr, "%p", p);
  return *this;
}
```
`BasicUsageEnvironment` 类还提供了一个静态的创建对象的函数 `createNew()` 用来创建 `BasicUsageEnvironment` 的对象。对于那些输出操作符的实现，都比较直接。

感觉 live555 这组 I/O 函数的实现并不是太好，C++ 标准库对这些接口都有着良好的实现，但 live555 似乎并没有要引入 C++ 标准库的打算。

# TaskScheduler
TaskScheduler 是 live555 中的任务调度器，它实现了 live555 的事件循环。UsageEnvironment 模块中，`TaskScheduler` 类的定义如下：
```
typedef void TaskFunc(void* clientData);
typedef void* TaskToken;
typedef u_int32_t EventTriggerId;

class TaskScheduler {
public:
  virtual ~TaskScheduler();

  virtual TaskToken scheduleDelayedTask(int64_t microseconds, TaskFunc* proc,
					void* clientData) = 0;
	// Schedules a task to occur (after a delay) when we next
	// reach a scheduling point.
	// (Does not delay if "microseconds" <= 0)
	// Returns a token that can be used in a subsequent call to
	// unscheduleDelayedTask() or rescheduleDelayedTask()
        // (but only if the task has not yet occurred).

  virtual void unscheduleDelayedTask(TaskToken& prevTask) = 0;
	// (Has no effect if "prevTask" == NULL)
        // Sets "prevTask" to NULL afterwards.
        // Note: This MUST NOT be called if the scheduled task has already occurred.

  virtual void rescheduleDelayedTask(TaskToken& task,
				     int64_t microseconds, TaskFunc* proc,
				     void* clientData);
        // Combines "unscheduleDelayedTask()" with "scheduleDelayedTask()"
        // (setting "task" to the new task token).
        // Note: This MUST NOT be called if the scheduled task has already occurred.

  // For handling socket operations in the background (from the event loop):
  typedef void BackgroundHandlerProc(void* clientData, int mask);
    // Possible bits to set in "mask".  (These are deliberately defined
    // the same as those in Tcl, to make a Tcl-based subclass easy.)
    #define SOCKET_READABLE    (1<<1)
    #define SOCKET_WRITABLE    (1<<2)
    #define SOCKET_EXCEPTION   (1<<3)
  virtual void setBackgroundHandling(int socketNum, int conditionSet, BackgroundHandlerProc* handlerProc, void* clientData) = 0;
  void disableBackgroundHandling(int socketNum) { setBackgroundHandling(socketNum, 0, NULL, NULL); }
  virtual void moveSocketHandling(int oldSocketNum, int newSocketNum) = 0;
        // Changes any socket handling for "oldSocketNum" so that occurs with "newSocketNum" instead.

  virtual void doEventLoop(char volatile* watchVariable = NULL) = 0;
      // Causes further execution to take place within the event loop.
      // Delayed tasks, background I/O handling, and other events are handled, sequentially (as a single thread of control).
      // (If "watchVariable" is not NULL, then we return from this routine when *watchVariable != 0)

  virtual EventTriggerId createEventTrigger(TaskFunc* eventHandlerProc) = 0;
      // Creates a 'trigger' for an event, which - if it occurs - will be handled (from the event loop) using "eventHandlerProc".
      // (Returns 0 iff no such trigger can be created (e.g., because of implementation limits on the number of triggers).)
  virtual void deleteEventTrigger(EventTriggerId eventTriggerId) = 0;

  virtual void triggerEvent(EventTriggerId eventTriggerId, void* clientData = NULL) = 0;
      // Causes the (previously-registered) handler function for the specified event to be handled (from the event loop).
      // The handler function is called with "clientData" as parameter.
      // Note: This function (unlike other library functions) may be called from an external thread
      // - to signal an external event.  (However, "triggerEvent()" should not be called with the
      // same 'event trigger id' from different threads.)

  // The following two functions are deprecated, and are provided for backwards-compatibility only:
  void turnOnBackgroundReadHandling(int socketNum, BackgroundHandlerProc* handlerProc, void* clientData) {
    setBackgroundHandling(socketNum, SOCKET_READABLE, handlerProc, clientData);
  }
  void turnOffBackgroundReadHandling(int socketNum) { disableBackgroundHandling(socketNum); }

  virtual void internalError(); // used to 'handle' a 'should not occur'-type error condition within the library.

protected:
  TaskScheduler(); // abstract base class
};
```
`TaskScheduler` 的接口可以分为如下的几组：
1. 调度定时器任务。这主要包括 `scheduleDelayedTask()`、`unscheduleDelayedTask()` 和 `rescheduleDelayedTask()` 这几个函数，它们分别用于调度一个延迟任务，取消一个延迟任务，以及重新调度一个延迟任务。
2. 后台调度 Socket I/O 处理操作。这主要包括 `setBackgroundHandling()`、`disableBackgroundHandling()`、`moveSocketHandling()`、`turnOnBackgroundReadHandling()` 和 `turnOffBackgroundReadHandling()` 这样几个函数，它们用于设置、修改或取消 socket 上特定 I/O 事件的处理程序。
3. 用户事件调度。这主要包括 `createEventTrigger()`、`deleteEventTrigger()` 和 `triggerEvent()` 这样几个函数，它们用于创建、删除及触发一个用户自定义事件。
4. 执行事件循环。这个由 `doEventLoop()` 函数完成，这个函数通常也是应用程序的主循环。
5. 内部错误处理程序。这个指 `internalError()` 函数，用于在 `TaskScheduler` 发生内部错误时，执行一些处理。

`TaskScheduler` 类本身提供了几个简单函数的实现：
```
TaskScheduler::TaskScheduler() {
}

TaskScheduler::~TaskScheduler() {
}

void TaskScheduler::rescheduleDelayedTask(TaskToken& task,
					  int64_t microseconds, TaskFunc* proc,
					  void* clientData) {
  unscheduleDelayedTask(task);
  task = scheduleDelayedTask(microseconds, proc, clientData);
}

// By default, we handle 'should not occur'-type library errors by calling abort().  Subclasses can redefine this, if desired.
void TaskScheduler::internalError() {
  abort();
}
```
它们都简单而易于理解，这里不再赘述。

BasicUsageEnvironment 模块中同样提供了，`TaskScheduler` 接口的实现。与 `UsageEnvironment` 接口的情况类似，`TaskScheduler` 类的接口同样由两个类来实现，分别是 `BasicTaskScheduler` 和 `BasicTaskScheduler0`，其中
 `BasicTaskScheduler0` 类实现我们前面提到的第 1 组，第 3 组接口，以及 `doEventLoop()` 的框架，而 `BasicTaskScheduler` 则用于实现第 2 组接口，并实现 `doEventLoop()` 的事件循环的循环体。

`BasicTaskScheduler0` 类定义如下：
```
class HandlerSet; // forward

#define MAX_NUM_EVENT_TRIGGERS 32

// An abstract base class, useful for subclassing
// (e.g., to redefine the implementation of socket event handling)
class BasicTaskScheduler0: public TaskScheduler {
public:
  virtual ~BasicTaskScheduler0();

  virtual void SingleStep(unsigned maxDelayTime = 0) = 0;
      // "maxDelayTime" is in microseconds.  It allows a subclass to impose a limit
      // on how long "select()" can delay, in case it wants to also do polling.
      // 0 (the default value) means: There's no maximum; just look at the delay queue

public:
  // Redefined virtual functions:
  virtual TaskToken scheduleDelayedTask(int64_t microseconds, TaskFunc* proc,
				void* clientData);
  virtual void unscheduleDelayedTask(TaskToken& prevTask);

  virtual void doEventLoop(char volatile* watchVariable);

  virtual EventTriggerId createEventTrigger(TaskFunc* eventHandlerProc);
  virtual void deleteEventTrigger(EventTriggerId eventTriggerId);
  virtual void triggerEvent(EventTriggerId eventTriggerId, void* clientData = NULL);

protected:
  BasicTaskScheduler0();

protected:
  // To implement delayed operations:
  DelayQueue fDelayQueue;

  // To implement background reads:
  HandlerSet* fHandlers;
  int fLastHandledSocketNum;

  // To implement event triggers:
  EventTriggerId volatile fTriggersAwaitingHandling; // implemented as a 32-bit bitmap
  EventTriggerId fLastUsedTriggerMask; // implemented as a 32-bit bitmap
  TaskFunc* fTriggeredEventHandlers[MAX_NUM_EVENT_TRIGGERS];
  void* fTriggeredEventClientDatas[MAX_NUM_EVENT_TRIGGERS];
  unsigned fLastUsedTriggerNum; // in the range [0,MAX_NUM_EVENT_TRIGGERS)
};
```
`BasicTaskScheduler0` 类的成员函数，基本上就是继承自 `TaskScheduler` 类中，它要实现功能的那部分接口，但它新添加了一个虚函数 `SingleStep()` ，用于让其子类覆写，实现事件循环中的单次迭代。

`BasicTaskScheduler0` 类的成员变量则是清晰地分为三组：`fDelayQueue` 用于实现定时器操作；`fHandlers` 和 `fLastHandledSocketNum` 用于实现 Socket I/O 事件处理操作；`fTriggersAwaitingHandling`、`fLastUsedTriggerMask`、`fTriggeredEventHandlers`、`fTriggeredEventClientDatas` 和 `fLastUsedTriggerNum` 用于实现用户事件。

***对于 `fHandlers` 和 `fLastHandledSocketNum`，感觉实际上没有必要在
 `BasicTaskScheduler0` 类中定义。纵观 `BasicTaskScheduler0` 类的整个实现，除初始化这两个成员变量之外，不存在其它的访问操作。从这两个变量的职责来说，也不在 `BasicTaskScheduler0` 类的职责范围内。感觉这两个变量实际上放在 `BasicTaskScheduler` 类中更合适一点。***

`BasicTaskScheduler0` 类对象创建及销毁过程如下：
```
BasicTaskScheduler0::BasicTaskScheduler0()
  : fLastHandledSocketNum(-1), fTriggersAwaitingHandling(0), fLastUsedTriggerMask(1), fLastUsedTriggerNum(MAX_NUM_EVENT_TRIGGERS-1) {
  fHandlers = new HandlerSet;
  for (unsigned i = 0; i < MAX_NUM_EVENT_TRIGGERS; ++i) {
    fTriggeredEventHandlers[i] = NULL;
    fTriggeredEventClientDatas[i] = NULL;
  }
}

BasicTaskScheduler0::~BasicTaskScheduler0() {
  delete fHandlers;
}
```
在类对象创建的过程中，创建和/或初始化成员对象，对象销毁时，则销毁成员对象。

## 时间的表示
在 live555 的 BasicUsageEnvironment 模块中，用 `Timeval` 类来描述时间，并用 `DelayInterval` 来描述延迟时间。这两个类的定义如下：
```
class Timeval {
public:
  time_base_seconds seconds() const {
    return fTv.tv_sec;
  }
  time_base_seconds seconds() {
    return fTv.tv_sec;
  }
  time_base_seconds useconds() const {
    return fTv.tv_usec;
  }
  time_base_seconds useconds() {
    return fTv.tv_usec;
  }

  int operator>=(Timeval const& arg2) const;
  int operator<=(Timeval const& arg2) const {
    return arg2 >= *this;
  }
  int operator<(Timeval const& arg2) const {
    return !(*this >= arg2);
  }
  int operator>(Timeval const& arg2) const {
    return arg2 < *this;
  }
  int operator==(Timeval const& arg2) const {
    return *this >= arg2 && arg2 >= *this;
  }
  int operator!=(Timeval const& arg2) const {
    return !(*this == arg2);
  }

  void operator+=(class DelayInterval const& arg2);
  void operator-=(class DelayInterval const& arg2);
  // returns ZERO iff arg2 >= arg1

protected:
  Timeval(time_base_seconds seconds, time_base_seconds useconds) {
    fTv.tv_sec = seconds; fTv.tv_usec = useconds;
  }

private:
  time_base_seconds& secs() {
    return (time_base_seconds&)fTv.tv_sec;
  }
  time_base_seconds& usecs() {
    return (time_base_seconds&)fTv.tv_usec;
  }

  struct timeval fTv;
};
. . . . . .
class DelayInterval: public Timeval {
public:
  DelayInterval(time_base_seconds seconds, time_base_seconds useconds)
    : Timeval(seconds, useconds) {}
};
```
`DelayInterval` 类基本上就是 `Timeval` 类的别名，而之所以重新定义这样一个类，大概主要是为了便于阅读维护吧。`Timeval` 类用标准库中保存时间的 `struct timeval` 结构保存时间值，但通过操作符重载，提供了一些方便操作时间值的操作符函数。以成员函数的方式定义的操作符函数的实现如下：
```
int Timeval::operator>=(const Timeval& arg2) const {
  return seconds() > arg2.seconds()
    || (seconds() == arg2.seconds()
	&& useconds() >= arg2.useconds());
}

void Timeval::operator+=(const DelayInterval& arg2) {
  secs() += arg2.seconds(); usecs() += arg2.useconds();
  if (useconds() >= MILLION) {
    usecs() -= MILLION;
    ++secs();
  }
}

void Timeval::operator-=(const DelayInterval& arg2) {
  secs() -= arg2.seconds(); usecs() -= arg2.useconds();
  if ((int)useconds() < 0) {
    usecs() += MILLION;
    --secs();
  }
  if ((int)seconds() < 0)
    secs() = usecs() = 0;

}
```
这些操作符函数的实现，都比较直观。

除了以成员操作符函数定义的这些操作符之外，还包括如下这些非成员函数的操作符：
```
#ifndef max
inline Timeval max(Timeval const& arg1, Timeval const& arg2) {
  return arg1 >= arg2 ? arg1 : arg2;
}
#endif
#ifndef min
inline Timeval min(Timeval const& arg1, Timeval const& arg2) {
  return arg1 <= arg2 ? arg1 : arg2;
}
#endif
. . . . . .
DelayInterval operator-(const Timeval& arg1, const Timeval& arg2) {
  time_base_seconds secs = arg1.seconds() - arg2.seconds();
  time_base_seconds usecs = arg1.useconds() - arg2.useconds();

  if ((int)usecs < 0) {
    usecs += MILLION;
    --secs;
  }
  if ((int)secs < 0)
    return DELAY_ZERO;
  else
    return DelayInterval(secs, usecs);
}


///// DelayInterval /////

DelayInterval operator*(short arg1, const DelayInterval& arg2) {
  time_base_seconds result_seconds = arg1*arg2.seconds();
  time_base_seconds result_useconds = arg1*arg2.useconds();

  time_base_seconds carry = result_useconds/MILLION;
  result_useconds -= carry*MILLION;
  result_seconds += carry;

  return DelayInterval(result_seconds, result_useconds);
}

#ifndef INT_MAX
#define INT_MAX	0x7FFFFFFF
#endif
const DelayInterval DELAY_ZERO(0, 0);
const DelayInterval DELAY_SECOND(1, 0);
const DelayInterval DELAY_MINUTE = 60*DELAY_SECOND;
const DelayInterval DELAY_HOUR = 60*DELAY_MINUTE;
const DelayInterval DELAY_DAY = 24*DELAY_HOUR;
const DelayInterval ETERNITY(INT_MAX, MILLION-1);
// used internally to make the implementation work
```
它们的实现也都比较直观。

## 延迟任务的表示及组织
在 live555 的 BasicUsageEnvironment 模块中，用 `DelayQueueEntry` 类表示一个延迟任务。该类定义如下：
```
class DelayQueueEntry {
public:
  virtual ~DelayQueueEntry();

  intptr_t token() {
    return fToken;
  }

protected: // abstract base class
  DelayQueueEntry(DelayInterval delay);

  virtual void handleTimeout();

private:
  friend class DelayQueue;
  DelayQueueEntry* fNext;
  DelayQueueEntry* fPrev;
  DelayInterval fDeltaTimeRemaining;

  intptr_t fToken;
  static intptr_t tokenCounter;
};
```
延迟任务通过 token 来标识，token 在对象创建时，借助于全局的 `tokenCounter` 产生。`fDeltaTimeRemaining` 用于表示延迟任务需要被执行的时间距当前时间的间隔。由 `fNext` 和 `fPrev` 不难猜到，在 BasicUsageEnvironment 模块中是以双向链表来组织延迟任务的。`handleTimeout()` 函数是延迟任务的主体，需要由具体的子类提供实现。

 `DelayQueueEntry` 类的具体实现如下：
```
intptr_t DelayQueueEntry::tokenCounter = 0;

DelayQueueEntry::DelayQueueEntry(DelayInterval delay)
  : fDeltaTimeRemaining(delay) {
  fNext = fPrev = this;
  fToken = ++tokenCounter;
}

DelayQueueEntry::~DelayQueueEntry() {
}

void DelayQueueEntry::handleTimeout() {
  delete this;
}
```

BasicUsageEnvironment 模块中实际使用 `AlarmHandler` 来描述延迟任务的，其定义如下：
```
class AlarmHandler: public DelayQueueEntry {
public:
  AlarmHandler(TaskFunc* proc, void* clientData, DelayInterval timeToDelay)
    : DelayQueueEntry(timeToDelay), fProc(proc), fClientData(clientData) {
  }

private: // redefined virtual functions
  virtual void handleTimeout() {
    (*fProc)(fClientData);
    DelayQueueEntry::handleTimeout();
  }

private:
  TaskFunc* fProc;
  void* fClientData;
};
```
BasicUsageEnvironment 模块需要用 `DelayQueueEntry` 类表示和组织延迟任务，而在接口层，也就是 `TaskScheduler` 中则是通过 `TaskFunc` 和用户数据指针来表示延迟任务。`AlarmHandler` 协助完成接口的结构到实现的结构的转换。

BasicUsageEnvironment 模块使用 `DelayQueue` 把 `DelayQueueEntry` 组织为双向链表，该类定义如下：
```
class DelayQueue: public DelayQueueEntry {
public:
  DelayQueue();
  virtual ~DelayQueue();

  void addEntry(DelayQueueEntry* newEntry); // returns a token for the entry
  void updateEntry(DelayQueueEntry* entry, DelayInterval newDelay);
  void updateEntry(intptr_t tokenToFind, DelayInterval newDelay);
  void removeEntry(DelayQueueEntry* entry); // but doesn't delete it
  DelayQueueEntry* removeEntry(intptr_t tokenToFind); // but doesn't delete it

  DelayInterval const& timeToNextAlarm();
  void handleAlarm();

private:
  DelayQueueEntry* head() { return fNext; }
  DelayQueueEntry* findEntryByToken(intptr_t token);
  void synchronize(); // bring the 'time remaining' fields up-to-date

  _EventTime fLastSyncTime;
};
```
首先来看一下 `DelayQueue` 类对象构造和销毁的过程：
```
DelayQueue::DelayQueue()
  : DelayQueueEntry(ETERNITY) {
  fLastSyncTime = TimeNow();
}

DelayQueue::~DelayQueue() {
  while (fNext != this) {
    DelayQueueEntry* entryToRemove = fNext;
    removeEntry(entryToRemove);
    delete entryToRemove;
  }
}
```
然后来看一下向链表中添加元素的过程：
```
void DelayQueue::addEntry(DelayQueueEntry* newEntry) {
  synchronize();

  DelayQueueEntry* cur = head();
  while (newEntry->fDeltaTimeRemaining >= cur->fDeltaTimeRemaining) {
    newEntry->fDeltaTimeRemaining -= cur->fDeltaTimeRemaining;
    cur = cur->fNext;
  }

  cur->fDeltaTimeRemaining -= newEntry->fDeltaTimeRemaining;

  // Add "newEntry" to the queue, just before "cur":
  newEntry->fNext = cur;
  newEntry->fPrev = cur->fPrev;
  cur->fPrev = newEntry->fPrev->fNext = newEntry;
}
. . . . . .
void DelayQueue::synchronize() {
  // First, figure out how much time has elapsed since the last sync:
  _EventTime timeNow = TimeNow();
  if (timeNow < fLastSyncTime) {
    // The system clock has apparently gone back in time; reset our sync time and return:
    fLastSyncTime  = timeNow;
    return;
  }
  DelayInterval timeSinceLastSync = timeNow - fLastSyncTime;
  fLastSyncTime = timeNow;

  // Then, adjust the delay queue for any entries whose time is up:
  DelayQueueEntry* curEntry = head();
  while (timeSinceLastSync >= curEntry->fDeltaTimeRemaining) {
    timeSinceLastSync -= curEntry->fDeltaTimeRemaining;
    curEntry->fDeltaTimeRemaining = DELAY_ZERO;
    curEntry = curEntry->fNext;
  }
  curEntry->fDeltaTimeRemaining -= timeSinceLastSync;
}
```
通过这两个函数，可以更加清楚的看到，在 `DelayQueue` 中是怎么组织延迟任务的。`DelayQueue` 因为其本身是一个 `DelayQueueEntry`，实际上它是一个环形双向链表。它的 `fNext` 指向这个链表的逻辑上的头部元素，但它本身是这个链表的尾部元素。双向链表中每个元素的 `fDeltaTimeRemaining` 保存的是这个任务应该被调度执行的时间点，与它前面的那个任务应该被调度执行的时间点之间的差值，当该值为 0 时，也就表示这个任务需要被执行了。这样也就是说， `DelayQueue` 是一个双向环形的有序链表，顺序按照所需的执行时间排列。

`removeEntry()` 用于从双向链表中移除一个任务：
```
void DelayQueue::removeEntry(DelayQueueEntry* entry) {
  if (entry == NULL || entry->fNext == NULL) return;

  entry->fNext->fDeltaTimeRemaining += entry->fDeltaTimeRemaining;
  entry->fPrev->fNext = entry->fNext;
  entry->fNext->fPrev = entry->fPrev;
  entry->fNext = entry->fPrev = NULL;
  // in case we should try to remove it again
}

DelayQueueEntry* DelayQueue::removeEntry(intptr_t tokenToFind) {
  DelayQueueEntry* entry = findEntryByToken(tokenToFind);
  removeEntry(entry);
  return entry;
}
. . . . . .
DelayQueueEntry* DelayQueue::findEntryByToken(intptr_t tokenToFind) {
  DelayQueueEntry* cur = head();
  while (cur != this) {
    if (cur->token() == tokenToFind) return cur;
    cur = cur->fNext;
  }

  return NULL;
}
```
`removeEntry(DelayQueueEntry* entry)` 中，在 `entry->fNext == NULL` 成立时会直接返回，也是由于 `DelayQueue` 实际是一个双向环形链表的缘故。

 `DelayQueue` 还提供了用于更新延迟任务执行时间的接口：
```
void DelayQueue::updateEntry(DelayQueueEntry* entry, DelayInterval newDelay) {
  if (entry == NULL) return;

  removeEntry(entry);
  entry->fDeltaTimeRemaining = newDelay;
  addEntry(entry);
}

void DelayQueue::updateEntry(intptr_t tokenToFind, DelayInterval newDelay) {
  DelayQueueEntry* entry = findEntryByToken(tokenToFind);
  updateEntry(entry, newDelay);
}
```

此外，`timeToNextAlarm()` 用于计算最近的一个任务执行的时间，而 `handleAlarm()` 则用于执行该任务。
```
DelayInterval const& DelayQueue::timeToNextAlarm() {
  if (head()->fDeltaTimeRemaining == DELAY_ZERO) return DELAY_ZERO; // a common case

  synchronize();
  return head()->fDeltaTimeRemaining;
}

void DelayQueue::handleAlarm() {
  if (head()->fDeltaTimeRemaining != DELAY_ZERO) synchronize();

  if (head()->fDeltaTimeRemaining == DELAY_ZERO) {
    // This event is due to be handled:
    DelayQueueEntry* toRemove = head();
    removeEntry(toRemove); // do this first, in case handler accesses queue

    toRemove->handleTimeout();
  }
}
```

## 延迟任务调度
看过了 live555 的 BasicUsageEnvironment 模块中时间的表示，以及延迟任务的表示及组织之后，再来看延迟任务的调度。

`BasicTaskScheduler0` 类通过 `scheduleDelayedTask()`  和 `unscheduleDelayedTask()` 函数实现定时器任务调度，它们分别用于调度一个延迟任务及取消一个延迟任务，它们的实现如下：
```
TaskToken BasicTaskScheduler0::scheduleDelayedTask(int64_t microseconds,
						 TaskFunc* proc,
						 void* clientData) {
  if (microseconds < 0) microseconds = 0;
  DelayInterval timeToDelay((long)(microseconds/1000000), (long)(microseconds%1000000));
  AlarmHandler* alarmHandler = new AlarmHandler(proc, clientData, timeToDelay);
  fDelayQueue.addEntry(alarmHandler);

  return (void*)(alarmHandler->token());
}

void BasicTaskScheduler0::unscheduleDelayedTask(TaskToken& prevTask) {
  DelayQueueEntry* alarmHandler = fDelayQueue.removeEntry((intptr_t)prevTask);
  prevTask = NULL;
  delete alarmHandler;
}
```
延迟任务调度也就是把延迟任务放进 `DelayQueue` 中，而取消延迟任务则是，把任务从 `DelayQueue` 中移除。

后面再来看延迟任务被执行的过程。

## 用户事件任务调度
用户事件任务调度接口，让调用者可以创建任务，并触发该任务在事件循环中执行。这组接口主要包括这样几个：`createEventTrigger()`、`deleteEventTrigger()` 和 `triggerEvent()`，它们分别用于创建任务，删除任务，及触发任务执行。
 
这些接口的实现如下：
```
EventTriggerId BasicTaskScheduler0::createEventTrigger(TaskFunc* eventHandlerProc) {
  unsigned i = fLastUsedTriggerNum;
  EventTriggerId mask = fLastUsedTriggerMask;

  do {
    i = (i+1)%MAX_NUM_EVENT_TRIGGERS;
    mask >>= 1;
    if (mask == 0) mask = 0x80000000;

    if (fTriggeredEventHandlers[i] == NULL) {
      // This trigger number is free; use it:
      fTriggeredEventHandlers[i] = eventHandlerProc;
      fTriggeredEventClientDatas[i] = NULL; // sanity

      fLastUsedTriggerMask = mask;
      fLastUsedTriggerNum = i;

      return mask;
    }
  } while (i != fLastUsedTriggerNum);

  // All available event triggers are allocated; return 0 instead:
  return 0;
}

void BasicTaskScheduler0::deleteEventTrigger(EventTriggerId eventTriggerId) {
  fTriggersAwaitingHandling &=~ eventTriggerId;

  if (eventTriggerId == fLastUsedTriggerMask) { // common-case optimization:
    fTriggeredEventHandlers[fLastUsedTriggerNum] = NULL;
    fTriggeredEventClientDatas[fLastUsedTriggerNum] = NULL;
  } else {
    // "eventTriggerId" should have just one bit set.
    // However, we do the reasonable thing if the user happened to 'or' together two or more "EventTriggerId"s:
    EventTriggerId mask = 0x80000000;
    for (unsigned i = 0; i < MAX_NUM_EVENT_TRIGGERS; ++i) {
      if ((eventTriggerId&mask) != 0) {
	fTriggeredEventHandlers[i] = NULL;
	fTriggeredEventClientDatas[i] = NULL;
      }
      mask >>= 1;
    }
  }
}

void BasicTaskScheduler0::triggerEvent(EventTriggerId eventTriggerId, void* clientData) {
  // First, record the "clientData".  (Note that we allow "eventTriggerId" to be a combination of bits for multiple events.)
  EventTriggerId mask = 0x80000000;
  for (unsigned i = 0; i < MAX_NUM_EVENT_TRIGGERS; ++i) {
    if ((eventTriggerId&mask) != 0) {
      fTriggeredEventClientDatas[i] = clientData;
    }
    mask >>= 1;
  }

  // Then, note this event as being ready to be handled.
  // (Note that because this function (unlike others in the library) can be called from an external thread, we do this last, to
  //  reduce the risk of a race condition.)
  fTriggersAwaitingHandling |= eventTriggerId;
}
```
`BasicTaskScheduler0` 的 `fTriggeredEventHandlers` 和 `fTriggeredEventClientDatas` 用于保存任务本身，它们分别保存任务主体函数，以及执行任务时传入的用户数据。它们都是数组，每个任务占用一个元素，相同索引处的元素属于同一个任务。数组的长度为 `MAX_NUM_EVENT_TRIGGERS`，即 32，也就是说最多可以创建的任务的个数为 32。

`fTriggersAwaitingHandling` 用于记录当前触发了哪些任务。每个任务的触发状态都对应于其中的一个位，当对应的位置 1 时，表示任务被触发，需要执行；反之则不需要执行。比如 `fTriggeredEventHandlers` 和 `fTriggeredEventClientDatas` 中索引 0 处的任务的触发状态，对应于 `fTriggersAwaitingHandling` 的最高位，索引 1 处的任务的触发状态，对应于次高位，依次类推。

创建任务，即是在 `fTriggeredEventHandlers` 和 `fTriggeredEventClientDatas` 中为任务找到一个空闲的位置，把任务的主体函数的指针保存起来，返回任务的索引在 `fTriggersAwaitingHandling`  中的对应位的掩码，作为任务的标识。`fLastUsedTriggerNum` 用于防止遍历 `fTriggeredEventHandlers` 查找时的无限循环。

删除任务即是移除任务相关的所有数据，包括复位 `fTriggersAwaitingHandling` 中的触发状态，以及 `fTriggeredEventHandlers` 和 `fTriggeredEventClientDatas` 中任务的主体函数指针和用户数据。

触发事件任务则是置为任务在 `fTriggersAwaitingHandling` 中对应的位，并设置任务数据。任务的实际执行同样需要在事件循环中执行。

## 事件循环执行框架
`BasicTaskScheduler0` 中执行事件循环由 `doEventLoop()` 函数完成，具体实现如下：
```
void BasicTaskScheduler0::doEventLoop(char volatile* watchVariable) {
  // Repeatedly loop, handling readble sockets and timed events:
  while (1) {
    if (watchVariable != NULL && *watchVariable != 0) break;
    SingleStep();
  }
}
```
主要特别关注的是传入的参数 `watchVariable`：调用者可以通过这个参数，来在事件循环的外部控制，事件循环何时结束。

## Socket I/O 事件描述及其组织
BasicUsageEnvironment 模块中，用 `HandlerDescriptor` 描述要监听的 socket 上的事件及事件发生时的处理程序，该类定义如下：
```
class HandlerDescriptor {
  HandlerDescriptor(HandlerDescriptor* nextHandler);
  virtual ~HandlerDescriptor();

public:
  int socketNum;
  int conditionSet;
  TaskScheduler::BackgroundHandlerProc* handlerProc;
  void* clientData;

private:
  // Descriptors are linked together in a doubly-linked list:
  friend class HandlerSet;
  friend class HandlerIterator;
  HandlerDescriptor* fNextHandler;
  HandlerDescriptor* fPrevHandler;
};
```
`socketNum` 为要监听的 socket，`conditionSet` 描述要监听的 socket 上的事件，`handlerProc` 为事件发生时的处理程序，`clientData` 为传递给事件处理程序的用户数据。而 `fNextHandler` 和 `fPrevHandler` 则用于将
 `HandlerDescriptor` 组织起来。不难猜到，BasicUsageEnvironment 模块中 `HandlerDescriptor` 也是要被组织为双向链表的。

 `HandlerDescriptor` 类的实现如下：
```
HandlerDescriptor::HandlerDescriptor(HandlerDescriptor* nextHandler)
  : conditionSet(0), handlerProc(NULL) {
  // Link this descriptor into a doubly-linked list:
  if (nextHandler == this) { // initialization
    fNextHandler = fPrevHandler = this;
  } else {
    fNextHandler = nextHandler;
    fPrevHandler = nextHandler->fPrevHandler;
    nextHandler->fPrevHandler = this;
    fPrevHandler->fNextHandler = this;
  }
}

HandlerDescriptor::~HandlerDescriptor() {
  // Unlink this descriptor from a doubly-linked list:
  fNextHandler->fPrevHandler = fPrevHandler;
  fPrevHandler->fNextHandler = fNextHandler;
}
```

BasicUsageEnvironment 模块中，使用 `HandlerSet` 来维护所有的  `HandlerDescriptor`，这个类的定义如下：
```
class HandlerSet {
public:
  HandlerSet();
  virtual ~HandlerSet();

  void assignHandler(int socketNum, int conditionSet, TaskScheduler::BackgroundHandlerProc* handlerProc, void* clientData);
  void clearHandler(int socketNum);
  void moveHandler(int oldSocketNum, int newSocketNum);

private:
  HandlerDescriptor* lookupHandler(int socketNum);

private:
  friend class HandlerIterator;
  HandlerDescriptor fHandlers;
};
```

`HandlerSet`/`HandlerDescriptor` 的设计与 `DelayQueue`/`DelayQueueEntry` 的设计非常相似。`HandlerSet` 的实现如下：
```
HandlerSet::HandlerSet()
  : fHandlers(&fHandlers) {
  fHandlers.socketNum = -1; // shouldn't ever get looked at, but in case...
}

HandlerSet::~HandlerSet() {
  // Delete each handler descriptor:
  while (fHandlers.fNextHandler != &fHandlers) {
    delete fHandlers.fNextHandler; // changes fHandlers->fNextHandler
  }
}

void HandlerSet
::assignHandler(int socketNum, int conditionSet, TaskScheduler::BackgroundHandlerProc* handlerProc, void* clientData) {
  // First, see if there's already a handler for this socket:
  HandlerDescriptor* handler = lookupHandler(socketNum);
  if (handler == NULL) { // No existing handler, so create a new descr:
    handler = new HandlerDescriptor(fHandlers.fNextHandler);
    handler->socketNum = socketNum;
  }

  handler->conditionSet = conditionSet;
  handler->handlerProc = handlerProc;
  handler->clientData = clientData;
}

void HandlerSet::clearHandler(int socketNum) {
  HandlerDescriptor* handler = lookupHandler(socketNum);
  delete handler;
}

void HandlerSet::moveHandler(int oldSocketNum, int newSocketNum) {
  HandlerDescriptor* handler = lookupHandler(oldSocketNum);
  if (handler != NULL) {
    handler->socketNum = newSocketNum;
  }
}

HandlerDescriptor* HandlerSet::lookupHandler(int socketNum) {
  HandlerDescriptor* handler;
  HandlerIterator iter(*this);
  while ((handler = iter.next()) != NULL) {
    if (handler->socketNum == socketNum) break;
  }
  return handler;
}
```
`HandlerSet` 类似地，被设计为 `HandlerDescriptor` 的双向循环列表，只是其中的元素的顺序没有意义。

BasicUsageEnvironment 模块还提供了迭代器 `HandlerIterator`，用于遍历`HandlerSet` ，其定义及实现如下：
```
class HandlerIterator {
public:
  HandlerIterator(HandlerSet& handlerSet);
  virtual ~HandlerIterator();

  HandlerDescriptor* next(); // returns NULL if none
  void reset();

private:
  HandlerSet& fOurSet;
  HandlerDescriptor* fNextPtr;
};

///////////////////////////////////// Implementation
HandlerIterator::HandlerIterator(HandlerSet& handlerSet)
  : fOurSet(handlerSet) {
  reset();
}

HandlerIterator::~HandlerIterator() {
}

void HandlerIterator::reset() {
  fNextPtr = fOurSet.fHandlers.fNextHandler;
}

HandlerDescriptor* HandlerIterator::next() {
  HandlerDescriptor* result = fNextPtr;
  if (result == &fOurSet.fHandlers) { // no more
    result = NULL;
  } else {
    fNextPtr = fNextPtr->fNextHandler;
  }

  return result;
}
```
总结一下，可以监听每个 socket 上的事件，并在事件发生时执行处理程序，监听的 socket 上的事件及事件处理程序由 `HandlerDescriptor` 描述；所有的 `HandlerDescriptor` 由 `HandlerSet` 组织为一个双向的循环链表，元素之间的实际顺序没有意义，新加入的元素被放在逻辑上的链表头部。

## Socket I/O 事件处理任务调度
Socket I/O 事件处理任务调度都在 `BasicTaskScheduler` 类中完成，这个类的定义如下：
```
class BasicTaskScheduler: public BasicTaskScheduler0 {
public:
  static BasicTaskScheduler* createNew(unsigned maxSchedulerGranularity = 10000/*microseconds*/);
  virtual ~BasicTaskScheduler();

protected:
  BasicTaskScheduler(unsigned maxSchedulerGranularity);
      // called only by "createNew()"

  static void schedulerTickTask(void* clientData);
  void schedulerTickTask();

protected:
  // Redefined virtual functions:
  virtual void SingleStep(unsigned maxDelayTime);

  virtual void setBackgroundHandling(int socketNum, int conditionSet, BackgroundHandlerProc* handlerProc, void* clientData);
  virtual void moveSocketHandling(int oldSocketNum, int newSocketNum);

protected:
  // To implement background reads:
  HandlerSet* fHandlers;
  int fLastHandledSocketNum;

  unsigned fMaxSchedulerGranularity;

  // To implement background operations:
  int fMaxNumSockets;
  fd_set fReadSet;
  fd_set fWriteSet;
  fd_set fExceptionSet;
. . . . . .
};
```
`fHandlers` 用于组织 `HandlerDescriptor`，`fReadSet`、`fWriteSet`、`fExceptionSet` 和 `fMaxNumSockets` 主要是为了适配 `select()` 接口，分别用于描述要监听其可读事件、可写事件、异常事件的 socket 集合，以及要监听的 socket 中 socket number 最大的那个。

Socket I/O 事件处理任务调度，由 `setBackgroundHandling()` 和 `moveSocketHandling()` 这两个函数完成，它们的实现如下：
```
void BasicTaskScheduler
  ::setBackgroundHandling(int socketNum, int conditionSet, BackgroundHandlerProc* handlerProc, void* clientData) {
  if (socketNum < 0) return;
#if !defined(__WIN32__) && !defined(_WIN32) && defined(FD_SETSIZE)
  if (socketNum >= (int)(FD_SETSIZE)) return;
#endif
  FD_CLR((unsigned)socketNum, &fReadSet);
  FD_CLR((unsigned)socketNum, &fWriteSet);
  FD_CLR((unsigned)socketNum, &fExceptionSet);
  if (conditionSet == 0) {
    fHandlers->clearHandler(socketNum);
    if (socketNum+1 == fMaxNumSockets) {
      --fMaxNumSockets;
    }
  } else {
    fHandlers->assignHandler(socketNum, conditionSet, handlerProc, clientData);
    if (socketNum+1 > fMaxNumSockets) {
      fMaxNumSockets = socketNum+1;
    }
    if (conditionSet&SOCKET_READABLE) FD_SET((unsigned)socketNum, &fReadSet);
    if (conditionSet&SOCKET_WRITABLE) FD_SET((unsigned)socketNum, &fWriteSet);
    if (conditionSet&SOCKET_EXCEPTION) FD_SET((unsigned)socketNum, &fExceptionSet);
  }
}

void BasicTaskScheduler::moveSocketHandling(int oldSocketNum, int newSocketNum) {
  if (oldSocketNum < 0 || newSocketNum < 0) return; // sanity check
#if !defined(__WIN32__) && !defined(_WIN32) && defined(FD_SETSIZE)
  if (oldSocketNum >= (int)(FD_SETSIZE) || newSocketNum >= (int)(FD_SETSIZE)) return; // sanity check
#endif
  if (FD_ISSET(oldSocketNum, &fReadSet)) {FD_CLR((unsigned)oldSocketNum, &fReadSet); FD_SET((unsigned)newSocketNum, &fReadSet);}
  if (FD_ISSET(oldSocketNum, &fWriteSet)) {FD_CLR((unsigned)oldSocketNum, &fWriteSet); FD_SET((unsigned)newSocketNum, &fWriteSet);}
  if (FD_ISSET(oldSocketNum, &fExceptionSet)) {FD_CLR((unsigned)oldSocketNum, &fExceptionSet); FD_SET((unsigned)newSocketNum, &fExceptionSet);}
  fHandlers->moveHandler(oldSocketNum, newSocketNum);

  if (oldSocketNum+1 == fMaxNumSockets) {
    --fMaxNumSockets;
  }
  if (newSocketNum+1 > fMaxNumSockets) {
    fMaxNumSockets = newSocketNum+1;
  }
}
```
对于 `setBackgroundHandling()`，当 `conditionSet` 为非 0 值时，会更新或者新建对特定 socket 的监听；为 0 时，则将清除对该 socket 的监听。 `moveSocketHandling()` 更新对于 socket 事件的监听。

`BasicTaskScheduler` 的 `SingleStep()` 实现事件循环的单次迭代：
```
void BasicTaskScheduler::SingleStep(unsigned maxDelayTime) {
  fd_set readSet = fReadSet; // make a copy for this select() call
  fd_set writeSet = fWriteSet; // ditto
  fd_set exceptionSet = fExceptionSet; // ditto

  DelayInterval const& timeToDelay = fDelayQueue.timeToNextAlarm();
  struct timeval tv_timeToDelay;
  tv_timeToDelay.tv_sec = timeToDelay.seconds();
  tv_timeToDelay.tv_usec = timeToDelay.useconds();
  // Very large "tv_sec" values cause select() to fail.
  // Don't make it any larger than 1 million seconds (11.5 days)
  const long MAX_TV_SEC = MILLION;
  if (tv_timeToDelay.tv_sec > MAX_TV_SEC) {
    tv_timeToDelay.tv_sec = MAX_TV_SEC;
  }
  // Also check our "maxDelayTime" parameter (if it's > 0):
  if (maxDelayTime > 0 &&
      (tv_timeToDelay.tv_sec > (long)maxDelayTime/MILLION ||
       (tv_timeToDelay.tv_sec == (long)maxDelayTime/MILLION &&
	tv_timeToDelay.tv_usec > (long)maxDelayTime%MILLION))) {
    tv_timeToDelay.tv_sec = maxDelayTime/MILLION;
    tv_timeToDelay.tv_usec = maxDelayTime%MILLION;
  }

  int selectResult = select(fMaxNumSockets, &readSet, &writeSet, &exceptionSet, &tv_timeToDelay);
  if (selectResult < 0) {
#if defined(__WIN32__) || defined(_WIN32)
    int err = WSAGetLastError();
    // For some unknown reason, select() in Windoze sometimes fails with WSAEINVAL if
    // it was called with no entries set in "readSet".  If this happens, ignore it:
    if (err == WSAEINVAL && readSet.fd_count == 0) {
      err = EINTR;
      // To stop this from happening again, create a dummy socket:
      if (fDummySocketNum >= 0) closeSocket(fDummySocketNum);
      fDummySocketNum = socket(AF_INET, SOCK_DGRAM, 0);
      FD_SET((unsigned)fDummySocketNum, &fReadSet);
    }
    if (err != EINTR) {
#else
    if (errno != EINTR && errno != EAGAIN) {
#endif
	// Unexpected error - treat this as fatal:
#if !defined(_WIN32_WCE)
	perror("BasicTaskScheduler::SingleStep(): select() fails");
	// Because this failure is often "Bad file descriptor" - which is caused by an invalid socket number (i.e., a socket number
	// that had already been closed) being used in "select()" - we print out the sockets that were being used in "select()",
	// to assist in debugging:
	fprintf(stderr, "socket numbers used in the select() call:");
	for (int i = 0; i < 10000; ++i) {
	  if (FD_ISSET(i, &fReadSet) || FD_ISSET(i, &fWriteSet) || FD_ISSET(i, &fExceptionSet)) {
	    fprintf(stderr, " %d(", i);
	    if (FD_ISSET(i, &fReadSet)) fprintf(stderr, "r");
	    if (FD_ISSET(i, &fWriteSet)) fprintf(stderr, "w");
	    if (FD_ISSET(i, &fExceptionSet)) fprintf(stderr, "e");
	    fprintf(stderr, ")");
	  }
	}
	fprintf(stderr, "\n");
#endif
	internalError();
      }
  }

  // Call the handler function for one readable socket:
  HandlerIterator iter(*fHandlers);
  HandlerDescriptor* handler;
  // To ensure forward progress through the handlers, begin past the last
  // socket number that we handled:
  if (fLastHandledSocketNum >= 0) {
    while ((handler = iter.next()) != NULL) {
      if (handler->socketNum == fLastHandledSocketNum) break;
    }
    if (handler == NULL) {
      fLastHandledSocketNum = -1;
      iter.reset(); // start from the beginning instead
    }
  }
  while ((handler = iter.next()) != NULL) {
    int sock = handler->socketNum; // alias
    int resultConditionSet = 0;
    if (FD_ISSET(sock, &readSet) && FD_ISSET(sock, &fReadSet)/*sanity check*/) resultConditionSet |= SOCKET_READABLE;
    if (FD_ISSET(sock, &writeSet) && FD_ISSET(sock, &fWriteSet)/*sanity check*/) resultConditionSet |= SOCKET_WRITABLE;
    if (FD_ISSET(sock, &exceptionSet) && FD_ISSET(sock, &fExceptionSet)/*sanity check*/) resultConditionSet |= SOCKET_EXCEPTION;
    if ((resultConditionSet&handler->conditionSet) != 0 && handler->handlerProc != NULL) {
      fLastHandledSocketNum = sock;
          // Note: we set "fLastHandledSocketNum" before calling the handler,
          // in case the handler calls "doEventLoop()" reentrantly.
      (*handler->handlerProc)(handler->clientData, resultConditionSet);
      break;
    }
  }
  if (handler == NULL && fLastHandledSocketNum >= 0) {
    // We didn't call a handler, but we didn't get to check all of them,
    // so try again from the beginning:
    iter.reset();
    while ((handler = iter.next()) != NULL) {
      int sock = handler->socketNum; // alias
      int resultConditionSet = 0;
      if (FD_ISSET(sock, &readSet) && FD_ISSET(sock, &fReadSet)/*sanity check*/) resultConditionSet |= SOCKET_READABLE;
      if (FD_ISSET(sock, &writeSet) && FD_ISSET(sock, &fWriteSet)/*sanity check*/) resultConditionSet |= SOCKET_WRITABLE;
      if (FD_ISSET(sock, &exceptionSet) && FD_ISSET(sock, &fExceptionSet)/*sanity check*/) resultConditionSet |= SOCKET_EXCEPTION;
      if ((resultConditionSet&handler->conditionSet) != 0 && handler->handlerProc != NULL) {
	fLastHandledSocketNum = sock;
	    // Note: we set "fLastHandledSocketNum" before calling the handler,
            // in case the handler calls "doEventLoop()" reentrantly.
	(*handler->handlerProc)(handler->clientData, resultConditionSet);
	break;
      }
    }
    if (handler == NULL) fLastHandledSocketNum = -1;//because we didn't call a handler
  }

  // Also handle any newly-triggered event (Note that we do this *after* calling a socket handler,
  // in case the triggered event handler modifies The set of readable sockets.)
  if (fTriggersAwaitingHandling != 0) {
    if (fTriggersAwaitingHandling == fLastUsedTriggerMask) {
      // Common-case optimization for a single event trigger:
      fTriggersAwaitingHandling &=~ fLastUsedTriggerMask;
      if (fTriggeredEventHandlers[fLastUsedTriggerNum] != NULL) {
	(*fTriggeredEventHandlers[fLastUsedTriggerNum])(fTriggeredEventClientDatas[fLastUsedTriggerNum]);
      }
    } else {
      // Look for an event trigger that needs handling (making sure that we make forward progress through all possible triggers):
      unsigned i = fLastUsedTriggerNum;
      EventTriggerId mask = fLastUsedTriggerMask;

      do {
	i = (i+1)%MAX_NUM_EVENT_TRIGGERS;
	mask >>= 1;
	if (mask == 0) mask = 0x80000000;

	if ((fTriggersAwaitingHandling&mask) != 0) {
	  fTriggersAwaitingHandling &=~ mask;
	  if (fTriggeredEventHandlers[i] != NULL) {
	    (*fTriggeredEventHandlers[i])(fTriggeredEventClientDatas[i]);
	  }

	  fLastUsedTriggerMask = mask;
	  fLastUsedTriggerNum = i;
	  break;
	}
      } while (i != fLastUsedTriggerNum);
    }
  }

  // Also handle any delayed event that may have come due.
  fDelayQueue.handleAlarm();
}
```
这个函数有点长，但清晰地分为如下几个部分：
1. 根据定时器任务列表中，距当前时间最近的任务所需执行的时间点以及传入的最大延迟时间，计算 `select()` 所能够等待地最长时间。
2. 执行 `select()` 等待 socket 上的时间。
3. `select()` 超时或某个 socket 上的 I/O 事件到来，首先执行发生 I/O 事件的 socket 的 I/O 事件处理程序。这个函数一次最多执行一个 socket 上的 I/O 处理程序。
4. 执行用户事件处理程序。也是一次最多执行一个。
5. 执行定时器任务，同样是一次最多执行一个。

live555 的基础设施基本上就是这些了。

### [打赏](https://www.wolfcstech.com/about/donate.html)

Done.

# live555 源码分析系列文章
[live555 源码分析：简介](https://www.wolfcstech.com/2017/08/28/live555_src_analysis_introduction/)
[live555 源码分析：基础设施](https://www.wolfcstech.com/2017/08/30/live555_src_analysis_infrasture/)
[live555 源码分析：MediaSever](https://www.wolfcstech.com/2017/08/31/live555_src_analysis_mediaserver/)
[Wireshark 抓包分析 RTSP/RTP/RTCP 基本工作过程](https://www.wolfcstech.com/2017/09/01/live555_src_analysis_rtsp_rtp_rtcp_wireshark/)
[live555 源码分析：RTSPServer](https://www.wolfcstech.com/2017/09/03/live555_src_analysis_rtspserver/)
[live555 源码分析：DESCRIBE 的处理](https://www.wolfcstech.com/2017/09/04/live555_src_analysis_describe/)
[live555 源码分析：SETUP 的处理](https://www.wolfcstech.com/2017/09/05/live555_src_analysis_setup/)
[live555 源码分析：PLAY 的处理](https://www.wolfcstech.com/2017/09/05/live555_src_analysis_play/)
[live555 源码分析：RTSPServer 组件结构](https://www.wolfcstech.com/2017/09/06/live555_src_analysis_rtspserver_arch/)
[live555 源码分析：ServerMediaSession](https://www.wolfcstech.com/2017/09/07/live555_src_analysis_servermediasession/)
[live555 源码分析：子会话 SDP 行生成](https://www.wolfcstech.com/2017/09/07/live555_src_analysis_subsession_sdp/)
[live555 源码分析：子会话 SETUP](https://www.wolfcstech.com/2017/09/08/live555_src_analysis_subsession_setup/)
[live555 源码分析：播放启动](https://www.wolfcstech.com/2017/09/08/live555_src_analysis_start_streaming/)
