# mmc_global_allocator.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_global_allocator.h`
- **文件用途**: 定义全局内存分配器类，管理多个 Blob 分配器，支持跨 Rank 的内存分配策略
- **依赖项**: `mmc_blob_allocator.h`, `mmc_locality_strategy.h`, `mmc_read_write_lock.h`

---

## 常量定义

```cpp
constexpr int LEVEL_BASE = 100;
```

**说明**: 内存淘汰阈值计算基数，用于百分比计算

---

## 类型别名

```cpp
using MmcMemPoolInitInfo = std::map<MmcLocation, MmcLocalMemlInitInfo>;
```

**说明**: 内存池初始化信息映射表，key 为位置，value 为初始化信息

---

## 函数

### operator<< (std::vector<MmcMemBlobPtr>)

```cpp
inline std::ostream &operator<<(std::ostream &os, const std::vector<MmcMemBlobPtr> &vec)
{
    os << ", Rank[";
    for (size_t i = 0; i < vec.size(); ++i) {
        if (i > 0) {
            os << ", ";
        }
        os << vec[i]->Rank();
    }
    os << "]";
    return os;
}
```

**声明位置**: 行 28-39

**功能描述**: 输出 Blob 向量的 Rank 信息到流

---

## 类定义

### MmcGlobalAllocator

全局内存分配器类，管理所有 Rank 和介质类型的分配器。

#### 成员变量

```cpp
private:
    MmcAllocators allocators_;
    ReadWriteLock globalAllocLock_;
```

**说明**:
- `allocators_`: 分配器映射表，key 为位置，value 为分配器指针
- `globalAllocLock_`: 读写锁，保护分配器映射表

---

#### 构造函数

```cpp
MmcGlobalAllocator() = default;
```

**声明位置**: 行 43

**功能描述**: 默认构造函数

---

### Alloc

```cpp
Result Alloc(const AllocOptions &allocOpt, std::vector<MmcMemBlobPtr> &blobs)
```

**声明位置**: 行 56-115

**功能描述**: 根据分配选项分配 Blob

**参数**:
- `allocOpt`: 分配选项
  - `preferredRank_`: 首选 Rank 列表
  - `numBlobs_`: 需要分配的 Blob 数量
  - `mediaType_`: 介质类型
  - `flags_`: 分配标志
- `blobs`: 输出分配的 Blob 列表

**返回值**: 成功返回 MMC_OK

**分配策略说明**:
1. `preferredRank_` 为空时（实际上至少有一个 local rank），numBlobs 可以是任意值
2. `preferredRank_` 非空时，rank 不能重复，且 rank 个数 <= numBlobs
3. `preferredRank_` 大于 1 的时候，必定是强制分配场景

**代码逻辑**:
1. 验证参数（rank 唯一性、numBlobs >= preferredRank 数量）
2. 如果未指定介质类型，使用最高层介质类型
3. 如果 preferredRank 数量 <= 1，执行简单分配
4. 如果 preferredRank 数量 > 1，执行强制分配：
   - 对每个 preferred rank 强制分配一个 Blob
   - 对剩余数量按随机分配策略处理
5. 验证分配结果

---

### Free (vector)

```cpp
void Free(std::vector<MmcMemBlobPtr> &blobs) noexcept
{
    for (const auto &blob : blobs) {
        Free(blob);
    }
    blobs.clear();
}
```

**声明位置**: 行 117-123

**功能描述**: 释放 Blob 向量中的所有 Blob

**代码逻辑**:
1. 遍历 Blob 列表
2. 对每个 Blob 调用 Free 释放
3. 清空向量

---

### Free (single)

```cpp
Result Free(const MmcMemBlobPtr &blob)
```

**声明位置**: 行 125-153

**功能描述**: 释放单个 Blob

**参数**:
- `blob`: 要释放的 Blob 指针

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 验证 Blob 非空
2. 获取读锁
3. 根据 Blob 的位置查找对应的分配器
4. 调用分配器的 Release 方法
5. 释放读锁

---

### Mount

```cpp
Result Mount(const MmcLocation &loc, const MmcLocalMemlInitInfo &localMemInitInfo)
```

**声明位置**: 行 155-171

**功能描述**: 挂载新的内存池贡献者（Rank + 介质类型组合）

**参数**:
- `loc`: 挂载位置（Rank + 介质类型）
- `localMemInitInfo`: 本地内存初始化信息

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 获取写锁
2. 检查位置是否已存在
3. 如果不存在，创建新的 Blob 分配器并插入映射表
4. 释放写锁

---

### Start

```cpp
Result Start(const MmcLocation &loc)
```

**声明位置**: 行 173-192

**功能描述**: 启动指定位置的分配器

**参数**:
- `loc`: 位置

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 获取读锁
2. 查找分配器
3. 调用分配器的 Start 方法
4. 释放读锁

---

### Stop

```cpp
Result Stop(const MmcLocation &loc)
```

**声明位置**: 行 194-214

**功能描述**: 停止指定位置的分配器

**参数**:
- `loc`: 位置

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 获取读锁
2. 查找分配器
3. 调用分配器的 Stop 方法
4. 释放读锁

---

### Unmount

```cpp
Result Unmount(const MmcLocation &loc)
```

**声明位置**: 行 216-241

**功能描述**: 卸载指定位置的内存池

**参数**:
- `loc`: 位置

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 获取写锁
2. 查找分配器
3. 检查是否可以卸载（已分配大小为 0）
4. 从映射表中移除
5. 释放写锁

---

### BuildFromBlobs

```cpp
Result BuildFromBlobs(const MmcLocation &location, std::map<std::string, MmcMemBlobDesc> &blobMap)
```

**声明位置**: 行 243-263

**功能描述**: 从 Blob 描述重建分配器状态

**参数**:
- `location`: 位置
- `blobMap`: Blob 描述映射表

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 获取读锁
2. 查找分配器
3. 调用分配器的 BuildFromBlobs 方法
4. 释放读锁

---

### GetUsedInfo

```cpp
void GetUsedInfo(uint64_t (&totalSize)[MEDIA_NONE], uint64_t (&usedSize)[MEDIA_NONE])
```

**声明位置**: 行 265-274

**功能描述**: 获取各介质类型的内存使用信息

**参数**:
- `totalSize`: 输出各介质类型的总容量数组
- `usedSize`: 输出各介质类型的已使用大小数组

**代码逻辑**:
1. 获取读锁
2. 遍历所有分配器
3. 累加各介质类型的容量和使用量
4. 释放读锁

---

### GetFreeSpace

```cpp
uint64_t GetFreeSpace(MediaType type)
```

**声明位置**: 行 276-285

**功能描述**: 获取指定介质类型的剩余空间

**参数**:
- `type`: 介质类型

**返回值**: 剩余空间大小

**代码逻辑**:
1. 调用 GetUsedInfo 获取使用信息
2. 计算并返回剩余空间

---

### GetNeedEvictList

```cpp
std::vector<MediaType> GetNeedEvictList(const uint64_t level, std::vector<uint16_t> &nowMemoryThresholds)
```

**声明位置**: 行 287-310

**功能描述**: 获取需要淘汰的介质类型列表

**参数**:
- `level`: 当前内存使用水平（百分比 * LEVEL_BASE）
- `nowMemoryThresholds`: 输出当前的内存阈值

**返回值**: 需要淘汰的介质类型列表

**代码逻辑**:
1. 获取使用信息
2. 从下层向上层遍历（先淘汰下层，后淘汰上层）
3. 检查每种介质类型的使用率是否超过阈值
4. 如果超过，添加到淘汰列表
5. 返回淘汰列表

**淘汰策略**:
- 从低层介质（DRAM）向高层介质（HBM）遍历
- 使用率公式: `usedSize * LEVEL_BASE > totalSize * level`

---

### GetAllSegmentInfo

```cpp
nlohmann::json GetAllSegmentInfo()
```

**声明位置**: 行 312-326

**功能描述**: 获取所有分段信息的 JSON 格式

**返回值**: 包含所有分配器信息的 JSON 数组

**代码逻辑**:
1. 获取读锁
2. 遍历所有分配器
3. 收集每个分配器的信息
4. 释放读锁
5. 返回 JSON 数组

---

### 私有方法

#### InnerAlloc

```cpp
Result InnerAlloc(const AllocOptions &allocReq, std::vector<MmcMemBlobPtr> &blobs,
                  std::unordered_set<uint32_t> &excludeRanks) const
```

**声明位置**: 行 329-355

**功能描述**: 内部分配方法，根据标志选择分配策略

**参数**:
- `allocReq`: 分配请求
- `blobs`: 输出分配的 Blob 列表
- `excludeRanks`: 排除的 Rank 列表

**返回值**: 成功返回 MMC_OK

**分配策略**:
- `ALLOC_ARRANGE`: 亲和性分配（优先 preferredRank）
- `ALLOC_FORCE_BY_RANK`: 强制按 Rank 分配
- `ALLOC_RANDOM`: 随机分配

---

#### GetTopLayerMediumType

```cpp
MediaType GetTopLayerMediumType()
```

**声明位置**: 行 357-367

**功能描述**: 获取最高层介质类型

**返回值**: 第一个分配器的介质类型

**代码逻辑**:
1. 获取读锁
2. 返回第一个分配器的介质类型
3. 释放读锁

---

## 类型别名

```cpp
using MmcGlobalAllocatorPtr = MmcRef<MmcGlobalAllocator>;
```

**说明**: 全局分配器的智能指针类型

---

## 文件级别的关系图

```
mmc_global_allocator.h (全局内存分配器)
    |
    +-- 管理 MmcBlobAllocator (单点分配器)
    |
    +-- 使用 MmcLocalityStrategy (分配策略)
    |
    +-- 使用 ReadWriteLock (读写锁)
    |
    +-- 支持多 Rank、多介质类型
    +-- 跨 Rank 分配策略
```
