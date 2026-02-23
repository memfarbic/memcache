# mmc_meta_container_lru.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_container_lru.cpp`
- **文件用途**: MmcMetaContainer 的 LRU 实现类，支持按介质类型的 LRU 淘汰
- **依赖项**: `mmc_mem_obj_meta.h`, `mf_rwlock.h`, `mmc_meta_container.h`

---

## 数据结构

### ValueLruItem

```cpp
struct ValueLruItem {
    Value value_;
    MediaType mediaType_;
    typename std::list<Key>::iterator lruIter_;
};
```

**声明位置**: 行 33-37

**说明**: LRU 值项
- `value_`: 实际值
- `mediaType_`: 介质类型
- `lruIter_`: LRU 链表中的迭代器

---

## 类定义

### MmcMetaContainerLRU

元数据容器的 LRU 实现，支持多介质类型的独立 LRU 链表。

---

#### 构造函数

```cpp
explicit MmcMetaContainerLRU(std::function<MediaType(const Value &)> getTypeFunc) : getTypeFunc_(getTypeFunc) {}
```

**声明位置**: 行 30

**功能描述**: 构造 LRU 容器

**参数**:
- `getTypeFunc`: 获取值介质类型的函数

---

### 成员变量

```cpp
private:
    std::unordered_map<Key, ValueLruItem> metaMap_;
    std::list<Key> lruLists_[MEDIA_NONE];
    ock::mf::ReadWriteLock lruLock_;
    ock::mf::ReadWriteLock metaLock_;
    std::function<MediaType(const Value &)> getTypeFunc_;
```

**说明**:
- `metaMap_`: 元数据映射表
- `lruLists_`: 每种介质类型独立的 LRU 链表
- `lruLock_`: LRU 链表读写锁
- `metaLock_`: 元数据映射表读写锁
- `getTypeFunc_`: 获取介质类型的函数

---

### Insert

```cpp
Result Insert(const Key &key, const Value &value) override
```

**功能描述**: 插入键值对

**代码逻辑**:
1. 获取写锁
2. 检查键是否已存在
3. 获取值的介质类型
4. 将键插入到对应介质类型的 LRU 链表头部
5. 创建 LRU 项并插入映射表
6. 释放锁

---

### Get

```cpp
Result Get(const Key &key, Value &value) override
```

**功能描述**: 获取键对应的值

**代码逻辑**:
1. 获取读锁
2. 在映射表中查找键
3. 如果找到则返回值
4. 释放锁

**注意**: 此方法不更新 LRU 位置

---

### Erase (key only)

```cpp
Result Erase(const Key &key) override
```

**功能描述**: 删除键

**代码逻辑**: 调用双参数版本的 Erase

---

### Erase (with value)

```cpp
Result Erase(const Key &key, Value &value) override
```

**功能描述**: 删除键并返回值

**代码逻辑**:
1. 获取写锁
2. 在映射表中查找键
3. 如果找到，从 LRU 链表和映射表中移除
4. 释放锁

---

### EraseAll

```cpp
Result EraseAll(std::function<void(const Key &, const Value &)> removeFunc) override
```

**功能描述**: 删除所有键值对

**代码逻辑**:
1. 获取写锁
2. 遍历所有键值对，调用 removeFunc
3. 清空映射表
4. 清空所有 LRU 链表
5. 释放锁

---

### EraseIf

```cpp
void EraseIf(std::function<bool(const Key &, const Value &)> matchFunc) override
```

**功能描述**: 删除满足条件的键值对

**代码逻辑**:
1. 获取写锁
2. 遍历映射表
3. 如果匹配条件则删除
4. 释放锁

---

### IterateIf

```cpp
void IterateIf(std::function<bool(const Key &, const Value &)> matchFunc,
               std::map<Key, Value> &matchedValues) override
```

**功能描述**: 遍历并收集满足条件的键值对

**代码逻辑**:
1. 获取读锁
2. 遍历映射表
3. 如果匹配条件则添加到输出
4. 释放锁

---

### GetAllKeys

```cpp
void GetAllKeys(std::vector<Key> &keys) override
```

**功能描述**: 获取所有键

**代码逻辑**:
1. 获取读锁
2. 遍历映射表，收集所有键
3. 释放锁

---

### Promote

```cpp
Result Promote(const Key &key) override
```

**功能描述**: 提升键在 LRU 中的位置（移到链表头部）

**代码逻辑**:
1. 获取读锁
2. 在映射表中查找键
3. 如果找到，更新 LRU 位置
4. 释放锁

---

### InsertLru

```cpp
Result InsertLru(const Key &key, MediaType type) override
```

**功能描述**: 将键插入到指定介质类型的 LRU 链表

**代码逻辑**:
1. 获取读锁
2. 在映射表中查找键
3. 如果找到，更新介质类型和 LRU 位置
4. 释放锁

---

### EvictOneLeastRecentlyUsed

```cpp
bool EvictOneLeastRecentlyUsed(std::function<EvictResult(const Key &, const Value &)> moveFunc,
                               const MediaType mediaType)
```

**声明位置**: 行 195-238

**功能描述**: 淘汰一个最近最少使用的键

**参数**:
- `moveFunc`: 移动/删除回调函数
- `mediaType`: 介质类型

**返回值**: 成功返回 true

**代码逻辑**:
1. 验证介质类型有效
2. 获取写锁
3. 从 LRU 链表尾部获取一个键
4. 在映射表中查找键
5. 调用回调函数处理
6. 根据回调结果删除或标记
7. 释放锁

---

### MultiLevelElimination

```cpp
void MultiLevelElimination(const uint16_t evictThresholdHigh, const uint16_t evictThresholdLow,
                           const std::vector<MediaType> &needEvictList,
                           const std::vector<uint16_t> &nowMemoryThresholds,
                           std::function<EvictResult(const Key &, const Value &)> moveFunc) override
```

**声明位置**: 行 240-268

**功能描述**: 多层级内存淘汰

**参数**:
- `evictThresholdHigh`: 高淘汰阈值
- `evictThresholdLow`: 低淘汰阈值
- `needEvictList`: 需要淘汰的介质类型列表
- `nowMemoryThresholds`: 当前内存阈值列表
- `moveFunc`: 移动/删除回调函数

**代码逻辑**:
1. 遍历需要淘汰的介质类型
2. 计算需要淘汰的数量
3. 循环调用 EvictOneLeastRecentlyUsed
4. 记录日志

**淘汰数量计算**:
```cpp
numEvictObjs = max(min(oriNum * (nowThreshold - evictThresholdLow) / evictThresholdHigh, oriNum), 1)
```

---

### 私有方法

#### UpdateLRU

```cpp
void UpdateLRU(const Key &key, ValueLruItem &lruItem)
```

**声明位置**: 行 271-280

**功能描述**: 更新键在 LRU 链表中的位置

**代码逻辑**:
1. 从当前 LRU 链表中移除
2. 插入到链表头部
3. 更新迭代器

---

#### GetBlobType

```cpp
MediaType GetBlobType(const Value &value)
```

**声明位置**: 行 282-288

**功能描述**: 获取值的介质类型

**代码逻辑**: 调用 getTypeFunc_ 函数

---

### 工厂函数实现

```cpp
template<typename Key, typename Value>
MmcRef<MmcMetaContainer<Key, Value>>
MmcMetaContainer<Key, Value>::Create(std::function<MediaType(const Value &)> GetTypeFunc)
{
    return MmcMakeRef<MmcMetaContainerLRU<Key, Value>>(GetTypeFunc).Get();
}
```

**声明位置**: 行 295-300

**功能描述**: 创建 LRU 容器实例

---

## 显式实例化

```cpp
template class MmcMetaContainer<std::string, MmcMemObjMetaPtr>;
```

**声明位置**: 行 292

**说明**: 显式实例化常用类型，提高编译效率

---

## 总结

此文件实现了元数据容器的 LRU 功能：

1. **多介质类型支持**: 每种介质类型有独立的 LRU 链表
2. **读写锁**: 使用读写锁提高并发性能
3. **多层级淘汰**: 支持按介质类型进行分层淘汰
4. **灵活回调**: 通过回调函数支持删除或向下移动

**LRU 策略**:
- 新插入的键在链表头部
- 访问时移动到链表头部
- 淘汰时从链表尾部开始
