# mmc_meta_container.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_container.h`
- **文件用途**: 定义元数据容器抽象接口类，提供元数据的 CRUD 操作和 LRU 淘汰功能
- **依赖项**: `mmc_ref.h`

---

## 类定义

### MmcMetaContainer

元数据容器抽象模板类，定义元数据存储和管理的接口。

**模板参数**:
- `Key`: 键类型（通常是 std::string）
- `Value`: 值类型（通常是 MmcMemObjMetaPtr）

---

### Insert

```cpp
virtual Result Insert(const Key &key, const Value &value) = 0;
```

**声明位置**: 行 26

**功能描述**: 插入一个键值对

**参数**:
- `key`: 键
- `value`: 值

**返回值**: 成功返回 MMC_OK，已存在返回 MMC_DUPLICATED_OBJECT

---

### Get

```cpp
virtual Result Get(const Key &key, Value &value) = 0;
```

**声明位置**: 行 27

**功能描述**: 获取指定键的值

**参数**:
- `key`: 键
- `value`: 输出值

**返回值**: 成功返回 MMC_OK，不存在返回 MMC_UNMATCHED_KEY

---

### Erase (key only)

```cpp
virtual Result Erase(const Key &key) = 0;
```

**声明位置**: 行 28

**功能描述**: 删除指定键（不返回值）

**参数**:
- `key`: 键

**返回值**: 成功返回 MMC_OK

---

### Erase (with value)

```cpp
virtual Result Erase(const Key &key, Value &value) = 0;
```

**声明位置**: 行 29

**功能描述**: 删除指定键并返回值

**参数**:
- `key`: 键
- `value`: 输出被删除的值

**返回值**: 成功返回 MMC_OK

---

### EraseAll

```cpp
virtual Result EraseAll(std::function<void(const Key &, const Value &)> removeFunc) = 0;
```

**声明位置**: 行 30

**功能描述**: 删除所有键值对

**参数**:
- `removeFunc`: 删除回调函数，用于处理每个被删除的键值对

**返回值**: 成功返回 MMC_OK

---

### EraseIf

```cpp
virtual void EraseIf(std::function<bool(const Key &, const Value &)> matchFunc) = 0;
```

**声明位置**: 行 31

**功能描述**: 删除满足条件的键值对

**参数**:
- `matchFunc`: 匹配函数，返回 true 表示删除

---

### IterateIf

```cpp
virtual void IterateIf(std::function<bool(const Key &, const Value &)> matchFunc,
                       std::map<Key, Value> &matchedValues) = 0;
```

**声明位置**: 行 32-33

**功能描述**: 遍历并收集满足条件的键值对

**参数**:
- `matchFunc`: 匹配函数
- `matchedValues`: 输出匹配的键值对

---

### GetAllKeys

```cpp
virtual void GetAllKeys(std::vector<Key> &keys) = 0;
```

**声明位置**: 行 34

**功能描述**: 获取所有键

**参数**:
- `keys`: 输出键列表

---

### Promote

```cpp
virtual Result Promote(const Key &key) = 0;
```

**声明位置**: 行 35

**功能描述**: 提升键在 LRU 链表中的位置（访问时调用）

**参数**:
- `key`: 键

**返回值**: 成功返回 MMC_OK

---

### InsertLru

```cpp
virtual Result InsertLru(const Key &key, MediaType type) = 0;
```

**声明位置**: 行 36

**功能描述**: 将键插入到指定介质类型的 LRU 链表

**参数**:
- `key`: 键
- `type`: 介质类型

**返回值**: 成功返回 MMC_OK

---

### MultiLevelElimination

```cpp
virtual void MultiLevelElimination(const uint16_t evictThresholdHigh, const uint16_t evictThresholdLow,
                                   const std::vector<MediaType> &needEvictList,
                                   const std::vector<uint16_t> &nowMemoryThresholds,
                                   std::function<EvictResult(const Key &, const Value &)> moveFunc) = 0;
```

**声明位置**: 行 37-40

**功能描述**: 多层级内存淘汰

**参数**:
- `evictThresholdHigh`: 高淘汰阈值
- `evictThresholdLow`: 低淘汰阈值
- `needEvictList`: 需要淘汰的介质类型列表
- `nowMemoryThresholds`: 当前内存阈值列表
- `moveFunc`: 移动/删除回调函数

---

### Create

```cpp
static MmcRef<MmcMetaContainer<Key, Value>> Create(std::function<MediaType(const Value &)> GetTypeFunc);
```

**声明位置**: 行 42

**功能描述**: 工厂方法，创建元数据容器实例

**参数**:
- `GetTypeFunc`: 获取值介质类型的函数

**返回值**: 元数据容器智能指针

---

## 实现类

**MmcMetaContainerLRU**: LRU 实现类，定义在 `mmc_meta_container_lru.cpp` 中

---

## 文件级别的关系图

```
mmc_meta_container.h (元数据容器接口)
    |
    +-- 被 MmcMetaContainerLRU 实现
    |
    +-- 被 MmcMetaManager 使用
    |
    +-- 提供操作:
    |   +-- 基本 CRUD (Insert, Get, Erase)
    |   +-- 批量操作 (EraseAll, EraseIf, IterateIf)
    |   +-- LRU 管理 (Promote, InsertLru)
    |   +-- 多层级淘汰 (MultiLevelElimination)
```
