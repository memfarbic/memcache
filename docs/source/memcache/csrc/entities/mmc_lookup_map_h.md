# mmc_lookup_map.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/entities/mmc_lookup_map.h`
- **文件用途**: 定义分桶哈希表模板类，支持并发访问和迭代器遍历
- **依赖项**:
  - `unordered_map` - 哈希表
  - `mmc_spinlock.h` - 自旋锁
  - `mmc_types.h` - 类型定义

---

## 模板参数

**声明位置**: 行 22-23
**完整签名**:
```cpp
template<typename Key, typename Value, uint32_t numBuckets>
class MmcLookupMap
```
**模板参数**:
| 参数 | 类型 | 说明 |
|------|------|------|
| `Key` | 模板类型参数 | 键类型 |
| `Value` | 模板类型参数 | 值类型 |
| `numBuckets` | `uint32_t` | 桶数量（编译时常量） |

**约束条件**:
```cpp
static_assert(numBuckets > 0, "numBuckets must be positive");
```

---

## 内部类型定义

### BucketPtr

**声明位置**: 行 25
**完整签名**:
```cpp
using BucketPtr = std::unordered_map<Key, Value> *;
```
**功能描述**: 桶指针类型别名

### MapIterator

**声明位置**: 行 26
**完整签名**:
```cpp
using MapIterator = typename std::unordered_map<Key, Value>::iterator;
```
**功能描述**: 哈希表迭代器类型别名

---

## 内部类：Iterator

**声明位置**: 行 29-83
**完整签名**:
```cpp
class Iterator
```
**功能描述**: 跨桶迭代器，支持遍历所有桶中的键值对

---

#### Iterator 构造函数

**声明位置**: 行 31-37
**完整签名**:
```cpp
Iterator(BucketPtr begin, BucketPtr end) : curBucket_(begin), endBucket_(end)
{
    if (curBucket_ != endBucket_) {
        mapIter_ = curBucket_->begin();
        SkipEmptyBuckets();
    }
}
```
**功能描述**: 构造迭代器
**参数**:
- `begin` - 起始桶指针
- `end` - 结束桶指针
**代码逻辑**:
1. 初始化当前桶和结束桶指针
2. 如果不是结束迭代器，初始化第一个桶的迭代器
3. 跳过空桶

---

#### operator*

**声明位置**: 行 39-42
**完整签名**:
```cpp
std::pair<const Key, Value> &operator*() const
{
    return *mapIter_;
}
```
**功能描述**: 解引用迭代器
**返回值**: 当前键值对的引用

---

#### operator++

**声明位置**: 行 44-49
**完整签名**:
```cpp
Iterator &operator++()
```
**功能描述**: 前置递增运算符
**代码逻辑**:
1. 递增当前桶内的迭代器
2. 跳过空桶
3. 返回自身引用

---

#### operator==

**声明位置**: 行 51-60
**完整签名**:
```cpp
bool operator==(const Iterator &other) const
```
**功能描述**: 相等比较运算符
**代码逻辑**:
1. 比较当前桶指针
2. 如果都是结束迭代器，返回 true
3. 否则比较桶内迭代器

---

#### operator!=

**声明位置**: 行 62-65
**完整签名**:
```cpp
bool operator!=(const Iterator &other) const
```
**功能描述**: 不等比较运算符
**代码逻辑**: 比较当前桶指针是否不同

---

#### SkipEmptyBuckets

**声明位置**: 行 73-82
**完整签名**:
```cpp
void SkipEmptyBuckets()
```
**功能描述**: 跳过空桶，定位到下一个非空桶
**访问级别**: private
**代码逻辑**:
1. 循环检查当前桶
2. 如果当前桶迭代器已到末尾，移动到下一个桶
3. 重复直到找到非空桶或到达结束位置

---

## 主类方法实现

### begin

**声明位置**: 行 85-88
**完整签名**:
```cpp
Iterator begin()
```
**功能描述**: 返回起始迭代器
**返回值**: 指向第一个元素的迭代器

---

### end

**声明位置**: 行 90-93
**完整签名**:
```cpp
Iterator end()
```
**功能描述**: 返回结束迭代器
**返回值**: 指向末尾的迭代器

---

### Insert

**声明位置**: 行 102-111
**完整签名**:
```cpp
Result Insert(const Key &key, const Value &value)
```
**功能描述**: 向哈希表插入键值对
**参数**:
- `key` - 键
- `value` - 值
**返回值**: `Result` - 操作结果
**可能返回的错误**:
- `MMC_ERROR` - 键已存在

**代码逻辑**:
1. **计算桶索引**:
   ```cpp
   std::size_t index = GetIndex(key);
   ```

2. **加锁**:
   ```cpp
   std::lock_guard<std::mutex> guard(locks_[index]);
   ```

3. **插入键值对**:
   ```cpp
   auto ret = buckets_[index].emplace(key, value);
   if (ret.second) {
       return MMC_OK;
   }
   return MMC_ERROR;
   ```

**注意事项**:
- 每个桶有独立的锁，减少锁竞争
- 如果键已存在，返回错误

---

### Find

**声明位置**: 行 120-130
**完整签名**:
```cpp
Result Find(const Key &key, Value &value)
```
**功能描述**: 根据键查找值
**参数**:
- `key` - 要查找的键
- `value` - 输出参数，找到的值
**返回值**: `Result` - 操作结果
**可能返回的错误**:
- `MMC_ERROR` - 键不存在

**代码逻辑**:
1. **计算桶索引**:
   ```cpp
   std::size_t index = GetIndex(key);
   ```

2. **加锁**:
   ```cpp
   std::lock_guard<std::mutex> guard(locks_[index]);
   ```

3. **查找键**:
   ```cpp
   auto iter = buckets_[index].find(key);
   if (iter != buckets_[index].end()) {
       value = iter->second;
       return MMC_OK;
   }
   return MMC_ERROR;
   ```

---

### Erase

**声明位置**: 行 138-146
**完整签名**:
```cpp
Result Erase(const Key &key)
```
**功能描述**: 根据键删除键值对
**参数**:
- `key` - 要删除的键
**返回值**: `Result` - 操作结果
**可能返回的错误**:
- `MMC_ERROR` - 键不存在

**代码逻辑**:
1. **计算桶索引**:
   ```cpp
   std::size_t index = GetIndex(key);
   ```

2. **加锁**:
   ```cpp
   std::lock_guard<std::mutex> guard(locks_[index]);
   ```

3. **删除键**:
   ```cpp
   if (buckets_[index].erase(key) > 0) {
       return MMC_OK;
   }
   return MMC_ERROR;
   ```

---

### GetIndex

**声明位置**: 行 151-154
**完整签名**:
```cpp
std::size_t GetIndex(const Key &key) const
{
    return keyHasher_(key) % numBuckets;
}
```
**功能描述**: 计算键对应的桶索引
**参数**:
- `key` - 键
**返回值**: 桶索引
**访问级别**: private
**代码逻辑**:
1. 使用哈希函数计算键的哈希值
2. 对桶数量取模得到索引

---

## 成员变量

**声明位置**: 行 157-158
```cpp
private:
    std::hash<Key> keyHasher_;                        // 哈希函数
    std::unordered_map<Key, Value> buckets_[numBuckets]; // 桶数组
    std::mutex locks_[numBuckets];                    // 锁数组
```

**成员变量说明**:
| 变量名 | 类型 | 说明 |
|--------|------|------|
| `keyHasher_` | `std::hash<Key>` | 哈希函数对象 |
| `buckets_` | `std::unordered_map<Key, Value>[]` | 哈希桶数组 |
| `locks_` | `std::mutex[]` | 每个桶对应的互斥锁 |

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MmcLookupMap<Key, Value, numBuckets>                 │
├─────────────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ 成员变量                                                             │ │
│ │ + keyHasher_: hash<Key>              (哈希函数)                       │ │
│ │ + buckets_: unordered_map<K,V>[]     (桶数组)                        │ │
│ │ + locks_: mutex[]                    (锁数组)                        │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│ 公共方法                                                                │
│ + Insert(key, value): Result       (插入键值对)                         │
│ + Find(key, value): Result         (查找值)                             │
│ + Erase(key): Result               (删除键值对)                         │
│ + begin(): Iterator                (获取起始迭代器)                     │
│ + end(): Iterator                  (获取结束迭代器)                     │
└─────────────────────────────────────────────────────────────────────────┘
                          │
                          │ 包含
                          ▼
         ┌──────────────────────────────────────────────────────────────┐
         │                      Iterator (内部类)                        │
         ├──────────────────────────────────────────────────────────────┤
         │ + curBucket_: BucketPtr         (当前桶指针)                  │
         │ + endBucket_: BucketPtr         (结束桶指针)                  │
         │ + mapIter_: MapIterator         (桶内迭代器)                  │
         ├──────────────────────────────────────────────────────────────┤
         │ + operator*(): reference        (解引用)                      │
         │ + operator++(): Iterator&       (递增)                        │
         │ + operator==(): bool           (相等比较)                    │
         │ + operator!=(): bool           (不等比较)                    │
         │ + SkipEmptyBuckets(): void     (跳过空桶，private)            │
         └──────────────────────────────────────────────────────────────┘
```

---

## 分桶哈希表结构示意图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        MmcLookupMap 结构                                │
└─────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────┐
                    │    MmcLookupMap     │
                    │   <Key, Value, N>   │
                    └──────────┬──────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │  Bucket[0]  │     │  Bucket[1]  │     │ Bucket[N-1] │
    │  + Lock[0]  │     │  + Lock[1]  │     │  + Lock[N-1]│
    ├─────────────┤     ├─────────────┤     ├─────────────┤
    │ unordered   │     │ unordered   │     │ unordered   │
    │ _map        │     │ _map        │     │ _map        │
    │             │     │             │     │             │
    │ k1 → v1     │     │ k3 → v3     │     │ ...         │
    │ k2 → v2     │     │             │     │             │
    └─────────────┘     └─────────────┘     └─────────────┘
           │                   │                   │
           └───────────────────┴───────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     Iterator        │
                    │  (跨桶遍历)          │
                    └─────────────────────┘
```

---

## 使用示例

### 示例 1: 基本使用

```cpp
#include "mmc_lookup_map.h"

using namespace ock::mmc;

// 创建 16 桶的哈希表，键为 string，值为 int
MmcLookupMap<std::string, int, 16> lookupMap;

// 插入键值对
Result ret = lookupMap.Insert("key1", 100);
if (ret == MMC_OK) {
    std::cout << "Insert successful" << std::endl;
}

// 查找值
int value;
ret = lookupMap.Find("key1", value);
if (ret == MMC_OK) {
    std::cout << "Found: " << value << std::endl;
}

// 删除键值对
ret = lookupMap.Erase("key1");
```

### 示例 2: 迭代器遍历

```cpp
MmcLookupMap<std::string, int, 16> lookupMap;

// 插入多个键值对
lookupMap.Insert("a", 1);
lookupMap.Insert("b", 2);
lookupMap.Insert("c", 3);

// 使用迭代器遍历
for (auto it = lookupMap.begin(); it != lookupMap.end(); ++it) {
    const auto &key = it->first;
    const auto &value = it->second;
    std::cout << key << " = " << value << std::endl;
}

// 范围 for 循环
for (const auto &pair : lookupMap) {
    std::cout << pair.first << " = " << pair.second << std::endl;
}
```

### 示例 3: 使用智能指针作为值

```cpp
// 定义类型别名
using MmcMemObjMetaPtr = MmcRef<MmcMemObjMeta>;

// 创建键为 string，值为智能指针的哈希表
MmcLookupMap<std::string, MmcMemObjMetaPtr, 32> metaMap;

// 插入元数据
MmcMemObjMetaPtr meta = MmcMakeRef<MmcMemObjMeta>();
Result ret = metaMap.Insert("object_key", meta);

// 查找元数据
MmcMemObjMetaPtr foundMeta;
ret = metaMap.Find("object_key", foundMeta);
if (ret == MMC_OK) {
    uint64_t size = foundMeta->Size();
    std::cout << "Object size: " << size << std::endl;
}
```

### 示例 4: 错误处理

```cpp
MmcLookupMap<std::string, int, 16> lookupMap;

// 尝试插入重复键
Result ret = lookupMap.Insert("key1", 100);
ret = lookupMap.Insert("key1", 200);  // 返回 MMC_ERROR

// 查找不存在的键
int value;
ret = lookupMap.Find("nonexistent", value);
if (ret != MMC_OK) {
    std::cout << "Key not found" << std::endl;
}

// 删除不存在的键
ret = lookupMap.Erase("nonexistent");
if (ret != MMC_OK) {
    std::cout << "Delete failed" << std::endl;
}
```

### 示例 5: 实际应用场景

```cpp
// 场景：内存对象元数据存储
class MetaService {
private:
    MmcLookupMap<std::string, MmcMemObjMetaPtr, 256> metaMap_;

public:
    Result RegisterObject(const std::string &key, MmcMemObjMetaPtr meta) {
        return metaMap_.Insert(key, meta);
    }

    Result GetObject(const std::string &key, MmcMemObjMetaPtr &meta) {
        return metaMap_.Find(key, meta);
    }

    Result UnregisterObject(const std::string &key) {
        return metaMap_.Erase(key);
    }

    void ListAllObjects() {
        for (const auto &pair : metaMap_) {
            std::cout << "Key: " << pair.first
                      << ", Size: " << pair.second->Size()
                      << std::endl;
        }
    }
};
```

### 示例 6: 并发安全

```cpp
// MmcLookupMap 使用分桶锁，支持并发访问不同桶
MmcLookupMap<std::string, int, 16> lookupMap;

// 线程 1 插入 key1
std::thread t1([&]() {
    lookupMap.Insert("key1", 100);
});

// 线程 2 插入 key2 (可能在不同的桶中)
std::thread t2([&]() {
    lookupMap.Insert("key2", 200);
});

t1.join();
t2.join();

// 注意：如果两个键在同一个桶中，仍会串行执行
```
