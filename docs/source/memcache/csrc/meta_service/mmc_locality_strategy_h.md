# mmc_locality_strategy.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_locality_strategy.h`
- **文件用途**: 定义内存分配策略类，提供亲和性分配、强制分配和随机分配策略
- **依赖项**: `mmc_mem_blob.h`, `mmc_types.h`, `mmc_blob_allocator.h`, `mmc_msg_base.h`

---

## 枚举定义

### AllocFlags

```cpp
enum AllocFlags {
    ALLOC_ARRANGE = 0,                 // 亲和性分配
    ALLOC_FORCE_BY_RANK = 1 << 0,      // 按rank强制分配
    ALLOC_RANDOM = 1 << 1,             // 随机分配
};
```

**声明位置**: 行 30-34

**说明**: 定义内存分配策略标志

---

## 数据结构

### MmcLocalMemCurInfo

```cpp
struct MmcLocalMemCurInfo {
    uint64_t capacity_;
};
```

**声明位置**: 行 36-38

**说明**: 本地内存当前信息（预留结构）

---

### MmcMemPoolCurInfo

```cpp
using MmcMemPoolCurInfo = std::map<MmcLocation, MmcLocalMemCurInfo>;
```

**声明位置**: 行 40

**说明**: 内存池当前信息映射表

---

### MmcAllocators

```cpp
using MmcAllocators = std::map<MmcLocation, MmcBlobAllocatorPtr>;
```

**声明位置**: 行 42

**说明**: 分配器映射表，key 为位置，value 为分配器指针

---

## 类定义

### MmcLocalityStrategy

内存分配策略类，提供静态方法实现不同的分配策略。

所有方法都是静态方法，无需实例化。

---

### ArrangeLocality

```cpp
static Result ArrangeLocality(const MmcAllocators &allocators, const AllocOptions &allocReq,
                              std::vector<MmcMemBlobPtr> &blobs, std::unordered_set<uint32_t> &excludeRanks)
```

**声明位置**: 行 46-83

**功能描述**: 亲和性分配策略，优先在 preferredRank 上分配，剩余部分随机分配

**参数**:
- `allocators`: 分配器映射表
- `allocReq`: 分配请求选项
- `blobs`: 输出分配的 Blob 列表
- `excludeRanks`: 排除的 Rank 集合

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 验证分配器非空且数量足够
2. 如果有 preferredRank 且不在排除列表中，尝试在该 Rank 上分配
3. 如果已分配数量达到请求数量，返回成功
4. 剩余数量使用 RandomAssign 策略分配

**分配策略**:
- 优先在 preferredRank 上分配一个 Blob
- 其余使用随机分配策略
- 自动排除已分配的 Rank

---

### ForceAssign

```cpp
static Result ForceAssign(const MmcAllocators &allocators, const AllocOptions &allocReq,
                          std::vector<MmcMemBlobPtr> &blobs)
```

**声明位置**: 行 85-123

**功能描述**: 强制按 Rank 分配策略

**参数**:
- `allocators`: 分配器映射表
- `allocReq`: 分配请求选项（必须指定 numBlobs=1）
- `blobs`: 输出分配的 Blob 列表

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 验证分配器非空
2. 验证 numBlobs 必须为 1
3. 构造目标位置
4. 查找对应的分配器
5. 调用分配器进行分配
6. 返回结果

**使用场景**:
- 需要强制在指定 Rank 上分配时
- 通常用于副本分配场景

---

### RandomAssign

```cpp
static Result RandomAssign(const MmcAllocators &allocators, const AllocOptions &allocReq,
                           std::vector<MmcMemBlobPtr> &blobs, std::unordered_set<uint32_t> &excludeRanks)
```

**声明位置**: 行 125-180

**功能描述**: 随机分配策略，从候选分配器中随机选择

**参数**:
- `allocators`: 分配器映射表
- `allocReq`: 分配请求选项
- `blobs`: 输出分配的 Blob 列表
- `excludeRanks`: 排除的 Rank 集合

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 验证分配器非空
2. 构造查找范围（按介质类型过滤）
3. 使用 lower_bound/upper_bound 获取候选分配器范围
4. 验证候选数量足够
5. 创建索引数组并随机打乱
6. 遍历打乱后的索引，跳过排除的 Rank
7. 调用分配器进行分配
8. 验证分配数量

**随机化策略**:
- 使用 std::random_device 生成随机种子
- 使用 std::mt19937 作为随机数引擎
- 使用 std::shuffle 打乱分配器顺序

---

## 分配策略选择

| 策略 | 标志 | 描述 | 使用场景 |
|------|------|------|----------|
| 亲和性分配 | ALLOC_ARRANGE | 优先在 preferredRank 分配，其余随机 | 默认策略 |
| 强制分配 | ALLOC_FORCE_BY_RANK | 强制在指定 Rank 分配 | 副本分配 |
| 随机分配 | ALLOC_RANDOM | 随机选择可用 Rank | 负载均衡 |

---

## 文件级别的关系图

```
mmc_locality_strategy.h (分配策略)
    |
    +-- 被 MmcGlobalAllocator 使用
    |
    +-- 操作 MmcBlobAllocator (单点分配器)
    |
    +-- 提供三种分配策略:
    |   +-- ArrangeLocality (亲和性)
    |   +-- ForceAssign (强制)
    |   +-- RandomAssign (随机)
    |
    +-- 支持按介质类型过滤
    +-- 支持 Rank 排除
```

---

## 使用示例

```cpp
// 亲和性分配：优先在 Rank 0 分配，其余随机
AllocOptions options;
options.numBlobs_ = 3;
options.preferredRank_ = {0};
options.flags_ = ALLOC_ARRANGE;
MmcLocalityStrategy::ArrangeLocality(allocators, options, blobs, excludeRanks);

// 强制分配：必须在 Rank 1 上分配
AllocOptions options;
options.numBlobs_ = 1;
options.preferredRank_ = {1};
options.flags_ = ALLOC_FORCE_BY_RANK;
MmcLocalityStrategy::ForceAssign(allocators, options, blobs);

// 随机分配：在所有可用 Rank 上随机分配
AllocOptions options;
options.numBlobs_ = 2;
options.flags_ = ALLOC_RANDOM;
MmcLocalityStrategy::RandomAssign(allocators, options, blobs, excludeRanks);
```
