# mmc_net_link_map.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/mmc_net_link_map.h`
- **文件用途**: 定义并发安全的网络链接映射容器
- **依赖项**:
  - `mmc_common_includes.h` - 公共头文件

---

## 类定义

### NetLinkMap 类

并发安全的链接映射容器，使用分片锁提高并发性能

```cpp
template<typename LINK_PTR>
class NetLinkMap final : public MmcReferable {
public:
    bool Find(const uint32_t id, LINK_PTR &link);
    bool Add(const uint32_t id, const LINK_PTR &link);
    bool Remove(const uint32_t id);
    void Clear();

private:
    static constexpr uint32_t gSubMapCount = 7L;

private:
    std::mutex mapMutex_[gSubMapCount];
    std::map<uint32_t, LINK_PTR> linkMaps_[gSubMapCount];
};
```

**设计说明**:
- 使用模板支持不同类型的链接指针
- 采用分片锁设计，将数据分为 7 个子桶，每个桶有独立的互斥锁
- 减少锁竞争，提高并发性能

---

## 方法

### NetLinkMap::Find()

```cpp
bool Find(const uint32_t id, LINK_PTR &link)
{
    auto bucket = id % gSubMapCount;
    {
        std::lock_guard<std::mutex> guard(mapMutex_[bucket]);
        auto iter = linkMaps_[bucket].find(id);
        if (iter != linkMaps_[bucket].end()) {
            link = iter->second;
            return true;
        }
    }
    return false;
}
```

**声明位置**: 行 32-44

**功能描述**: 根据 ID 查找链接

**参数**:
- `id`: 链接 ID（通常是 peerId）
- `link`: 输出参数，找到的链接智能指针

**返回值**: `bool` - true 表示找到，false 表示未找到

**代码逻辑**:
1. 根据 ID 计算桶索引（模运算）
2. 加锁对应桶的互斥锁
3. 在对应桶的 map 中查找
4. 如果找到，设置输出参数并返回 true
5. 否则返回 false

**使用示例**:
```cpp
NetLinkAccPtr link;
bool found = linkMap->Find(peerId, link);
if (found) {
    // 使用 link
}
```

---

### NetLinkMap::Add()

```cpp
bool Add(const uint32_t id, const LINK_PTR &link)
{
    auto bucket = id % gSubMapCount;
    {
        std::lock_guard<std::mutex> guard(mapMutex_[bucket]);
        return linkMaps_[bucket].emplace(id, link).second;
    }
}
```

**声明位置**: 行 53-60

**功能描述**: 添加链接到映射表

**参数**:
- `id`: 链接 ID
- `link`: 要添加的链接智能指针

**返回值**: `bool` - true 表示添加成功，false 表示 ID 已存在

**代码逻辑**:
1. 根据 ID 计算桶索引
2. 加锁对应桶的互斥锁
3. 使用 emplace 尝试插入
4. 返回 emplace 的 second 值（是否插入成功）

**注意**: 如果 ID 已存在，不会覆盖原有链接

---

### NetLinkMap::Remove()

```cpp
bool Remove(const uint32_t id)
{
    auto bucket = id % gSubMapCount;
    {
        std::lock_guard<std::mutex> guard(mapMutex_[bucket]);
        return linkMaps_[bucket].erase(id) != 0;
    }
}
```

**声明位置**: 行 68-75

**功能描述**: 从映射表中移除链接

**参数**: `id` - 要移除的链接 ID

**返回值**: `bool` - true 表示移除成功，false 表示 ID 不存在

**代码逻辑**:
1. 根据 ID 计算桶索引
2. 加锁对应桶的互斥锁
3. 调用 erase 移除
4. 返回是否实际移除了元素

---

### NetLinkMap::Clear()

```cpp
void Clear()
{
    for (uint32_t bucket = 0; bucket < gSubMapCount; bucket++) {
        std::lock_guard<std::mutex> guard(mapMutex_[bucket]);
        linkMaps_[bucket].clear();
    }
}
```

**声明位置**: 行 80-86

**功能描述**: 清空所有链接

**代码逻辑**:
1. 遍历所有桶
2. 依次获取每个桶的锁
3. 清空对应桶的 map

**注意**: 此操作会依次获取所有桶的锁，可能导致短暂的阻塞

---

## 常量

### gSubMapCount

```cpp
static constexpr uint32_t gSubMapCount = 7L;
```

**说明**: 子映射表的数量

**设计考虑**:
- 使用质数 7 可以使哈希分布更均匀
- 数量适中，既能减少锁竞争，又不会占用过多内存

---

## 成员变量

```cpp
std::mutex mapMutex_[gSubMapCount];
std::map<uint32_t, LINK_PTR> linkMaps_[gSubMapCount];
```

- `mapMutex_`: 每个子桶的互斥锁数组
- `linkMaps_`: 存储链接的 map 数组

---

## 文件级别的关系图

```
mmc_net_link_map.h (并发链接映射)
    |
    +-- NetLinkMap<LINK_PTR> -> 模板类
            |
            +-- Find(id, link) -> 查找链接
            +-- Add(id, link) -> 添加链接
            +-- Remove(id) -> 移除链接
            +-- Clear() -> 清空所有链接
            |
            +-> 分片锁设计 (7个桶)
```

---

## 依赖关系

**依赖以下文件**:
- `mmc_common_includes.h` - 公共头文件

**被以下文件依赖**:
- `mmc_net_common_acc.h` - ACC 网络通用定义
- `mmc_net_engine_acc.cpp` - ACC 网络引擎实现

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有链接映射定义都在 ock::mmc 命名空间内
}
}
```

---

## 性能特点

1. **分片锁**: 使用 7 个独立的锁，减少锁竞争
2. **O(log n)**: 使用 std::map，查找/插入/删除时间复杂度为 O(log n)
3. **缓存友好**: 每个桶独立，减少缓存行争用
4. **线程安全**: 所有操作都有锁保护
