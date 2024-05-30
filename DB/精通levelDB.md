# 第一章 概述

Level DB优点：

* k,v采用字符串形式，无长度限制
* 持久化和内存存储
* 按key有序存储，可定制比较函数
* 批量写入
* 内存快照
* 支持迭代器

缺点：

* 不支持SQL
* 单进程，不支持多进程
* 不支持CS访问模式

# 第二章 基本数据结构

## 2.1 Slice

创建时需提供外部字节数组（Slice不把'\0'当作终止符，当作数据）

```C++
class Slice {
	...
private:
  const char* data_;
  size_t size_;
};
```

## 2.2 错误处理Status

包含了状态码

```C++
class Status {
	...
	enum Code {
	 kOk = 0, // 成员变量state_为空
	 kNotFound = 1,
	 kCorruption = 2,
	 kNotSupported = 3,
	 kInvalidArgument = 4,
	 kIOError = 5
	};

  // OK status has a null state_.  Otherwise, state_ is a new[] array
  // of the following form:
  //    state_[0..3] == length of message
  //    state_[4]    == code
  //    state_[5..]  == message
  const char* state_;
};
```

## 2.3 key比较函数接口Comparator

虚基类：

```C++
class Comparator{
public:
 virtual ~Comparator();
 virtual int Compare(const Slice &a, const Slice& b) const = 0;
 virtual const char* Name() const = 0;
// *start = "abcd"，limit="abzf"， 最终*start = "abd"
 virtual void FindShortestSeparator(std::string *start, const Slice &limit) const = 0;
// *key = "abcd" 最终 *key = "b"
 virtual void FindShortSuccessor(std::string *key) const = 0;
}
```

## 2.4 迭代器接口

```C++
class LEVELDB_EXPORT Iterator {
 public:
  Iterator();
  Iterator(const Iterator&) = delete;
  Iterator& operator=(const Iterator&) = delete;
  virtual ~Iterator();

  // An iterator is either positioned at a key/value pair, or
  // not valid.  This method returns true if the iterator is valid.
  virtual bool Valid() const = 0;

  // Position at the first key in the source.  The iterator is Valid()
  // after this call iff the source is not empty.
  virtual void SeekToFirst() = 0;

  // Position at the last key in the source.  The iterator is
  // Valid() after this call iff the source is not empty.
  virtual void SeekToLast() = 0;

  // Position at the first key in the source that is at or past target.
  // The iterator is Valid() after this call iff the source contains
  // an entry that comes at or past target.
  virtual void Seek(const Slice& target) = 0;

  // Moves to the next entry in the source.  After this call, Valid() is
  // true iff the iterator was not positioned at the last entry in the source.
  // REQUIRES: Valid()
  virtual void Next() = 0;

  // Moves to the previous entry in the source.  After this call, Valid() is
  // true iff the iterator was not positioned at the first entry in source.
  // REQUIRES: Valid()
  virtual void Prev() = 0;

  // Return the key for the current entry.  The underlying storage for
  // the returned slice is valid only until the next modification of
  // the iterator.
  // REQUIRES: Valid()
  virtual Slice key() const = 0;

  // Return the value for the current entry.  The underlying storage for
  // the returned slice is valid only until the next modification of
  // the iterator.
  // REQUIRES: Valid()
  virtual Slice value() const = 0;

  // If an error has occurred, return it.  Else return an ok status.
  virtual Status status() const = 0;

  // Clients are allowed to register function/arg1/arg2 triples that
  // will be invoked when this iterator is destroyed.
  //
  // Note that unlike all of the preceding methods, this method is
  // not abstract and therefore clients should not override it.
  using CleanupFunction = void (*)(void* arg1, void* arg2);
  void RegisterCleanup(CleanupFunction function, void* arg1, void* arg2);

 private:
  // Cleanup functions are stored in a single-linked list.
  // The list's head node is inlined in the iterator.
  struct CleanupNode {
    // True if the node is not used. Only head nodes might be unused.
    bool IsEmpty() const { return function == nullptr; }
    // Invokes the cleanup function.
    void Run() {
      assert(function != nullptr);
      (*function)(arg1, arg2);
    }

    // The head node is used if the function pointer is not null.
    CleanupFunction function;
    void* arg1;
    void* arg2;
    CleanupNode* next;
  };
  CleanupNode cleanup_head_; // Cleanup链表
};
```

Iterator析构函数会遍历cleanup链表中所有的节点，并调用相对应的函数，实现迭代器相关资源的释放与清除

## 2.5 系统参数

### 2.5.1 Options

用于`DB::open`，该结构分为**行为相关参数**和**性能相关参数**

#### 行为参数

Comparator比较器：默认字典序，等等

#### 性能参数

* write_buffer_size：内存中即将写入到硬盘的数据量（写缓存）
* max_open_files
* block_cache: block是从硬盘读数据的单位，设置缓存大小
* compression：压缩算法

### 2.5.2 读操作参数ReadOptions

用于`DB::Get`

可设置读取时进行数据校验；迭代器读取时将数据缓存到内存

### 2.5.3 写操作WriteOptions

用于`DB::Put`

可设置是否将OS缓存区的内容同步写入硬盘

# 第三章 使用

## 3.1 源码结构

### 

![image](assets/image-20240529184023-xlrj2st.png)

## 3.2 基本操作

在CMakeLists添加

```Cmake
add_executable(lyf_main
  "db/lyf_main.cc"
)
target_link_libraries(lyf_main leveldb)
```

```C++
// db/lyf_main.cc
#include "leveldb/db.h"
#include "leveldb/options.h"
#include "leveldb/slice.h"
#include "leveldb/status.h"
#include <iostream>

int main()
{
    leveldb::DB* db;
    leveldb::Options op;
    op.create_if_missing = true;
    leveldb::Status status = leveldb::DB::Open(op, "/tmp/testdb", &db);
    assert(status.ok());

    leveldb::Slice key("k1");
    std::string value;
    status = db->Get(leveldb::ReadOptions(), key, &value);
    if(!status.ok())
    {
        std::cout << key.data() << ": " << status.ToString() << std::endl;
        status = db->Put(leveldb::WriteOptions(), key, key);
    }

    if(status.ok())
    {
        std::cout << "Write successfully" << '\n';
        status = db->Get(leveldb::ReadOptions(), key, &value);
        if(status.ok())
            std::cout<<"the value of "<<key.data()<<": "<<value<<std::endl;
    }
    if(db->Delete(leveldb::WriteOptions(), key).ok())
        std::cout<<key.data()<<"-"<<value<<" is deleted!" <<std::endl;
    delete db;
}
```

## 3.3 批量操作

```C++
void batch_op()
{
    leveldb::DB* db;
    leveldb::Options op;
    op.create_if_missing = true;
    leveldb::Status status = leveldb::DB::Open(op, "/tmp/testdb", &db);
    assert(status.ok());

    std::string value;
    std::string key = "k1";
    std::stringstream ss;
    leveldb::WriteBatch batch;
    char index[10];

    for(int i = 0; i < 10; i++)
    {
        std::string pre = "k";
        sprintf(index, "%d", i);
        key = pre + index;
        batch.Put(key, index);
    }
    status = db->Write(leveldb::WriteOptions(), &batch);
    if(status.ok())
        std::cout << "batch write is finished!" << '\n';
  
    status = db->Get(leveldb::ReadOptions(), "k3", &value);
    if(status.ok())
        std::cout<<"read the value of k3: "<< value <<std::endl;
  
    // 将数据批量删除
    for(int i = 0; i<10; i++) 
    {
        std::string pre = "k";
        sprintf(index, "%d", i);
        key = pre + index;
        batch.Delete(key);
    }
    status = db->Write(leveldb::WriteOptions(), &batch);
    if(status.ok())
        std::cout<<"batch delete is finished!"<<std::endl;
    delete db;
}
```

## 3.4 迭代器

### 3.4.1 遍历

前向和后向遍历

```C++
void iter()
{
    leveldb::DB* db;
    leveldb::Options op;
    op.create_if_missing = true;
    leveldb::Status status = leveldb::DB::Open(op, "/tmp/testdb", &db);
    assert(status.ok());

    db->Put(leveldb::WriteOptions(), "key1", "value1");
    db->Put(leveldb::WriteOptions(), "key2", "value2");
    db->Put(leveldb::WriteOptions(), "key3", "value3");
    auto it = db->NewIterator(leveldb::ReadOptions());
    // q
    for(it->SeekToFirst(); it->Valid(); it->Next())
    {
        std::cout << it->key().ToString() << ": " << it->value().ToString() << '\n';
    }
    //后向遍历
    for(it->SeekToLast(); it->Valid(); it->Prev())
    {
        std::cout << it->key().ToString() << ": " << it->value().ToString() << '\n';
    }
    // 基于key范围
    for(it->Seek("key1"); it->Valid() && it->key().ToString() <= "key3"; it->Next())
    {
        std::cout << it->key().ToString() << ": " << it->value().ToString() << '\n';
    }
    delete it;
    delete db;
}
```

## 3.5 性能优化

### 3.5.1 压缩

DB的每个文件由压缩块组成，读写均已块为单位，写入时可设置压缩参数。

### 3.5.2 Cache

默认采用LRU：
```C++
    leveldb::DB* db;
    leveldb::Options op;
    // 10 MB
    op.block_cache = leveldb::NewLRUCache(10 * 1024 * 1024); 
    leveldb::Status status = leveldb::DB::Open(op, "/tmp/testdb", &db);
```

### 3.5.3 FilterPolicy

这是用来减少Get操作读磁盘时的I/O操作次数的参数，如创建布隆过滤器：

```C++
    leveldb::DB* db;
    leveldb::Options op;
    op.filter_policy = leveldb::NewBloomFilterPolicy(10);
    leveldb::Status status = leveldb::DB::Open(op, "/tmp/testdb", &db);
```
