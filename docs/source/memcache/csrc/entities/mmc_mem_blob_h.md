# mmc_mem_blob.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/entities/mmc_mem_blob.h`
- **文件用途**: 定义内存块（Blob）的核心类，包括状态管理、租约管理和元数据备份功能
- **依赖项**:
  - `nlohmann/json.hpp` - JSON 库
  - `mmc_blob_state.h` - 状态机定义
  - `mmc_common_includes.h` - 公共头文件
  - `mmc_meta_lease_manager.h` - 租约管理器
  - `mmc_montotonic.h` - 单调时间
  - `mmc_def.h` - 宏定义
  - `mmc_meta_backup_mgr_factory.h` - 备份管理器工厂

---

## 常量定义

### MAX_BLOB_COPIES

**声明位置**: 通过其他头文件引入
**说明**: 每个 MemObject 最多支持的 Blob 副本数量

---

## 辅助结构体

### MemObjQueryInfo

**声明位置**: 行 26-58
**完整签名**:
```cpp
struct MemObjQueryInfo {
    uint64_t size_;
    uint16_t prot_;
    uint8_t numBlobs_;
    bool valid_;
    uint32_t blobRanks_[MAX_BLOB_COPIES];
    uint16_t blobTypes_[MAX_BLOB_COPIES];

    MemObjQueryInfo();
    MemObjQueryInfo(const uint64_t size, const uint16_t prot, const uint8_t numBlobs, const bool valid);

    nlohmann::json toJson(const std::string &key) const;
};
```
**功能描述**: 内存对象查询信息结构

**成员变量**:
| 变量名 | 类型 | 说明 |
|--------|------|------|
| `size_` | `uint64_t` | 对象大小 |
| `prot_` | `uint16_t` | 访问权限 |
| `numBlobs_` | `uint8_t` | Blob 数量 |
| `valid_` | `bool` | 是否有效 |
| `blobRanks_` | `uint32_t[]` | Blob 所在 Rank 列表 |
| `blobTypes_` | `uint16_t[]` | Blob 介质类型列表 |

#### toJson

**声明位置**: 行 38-57
**完整签名**:
```cpp
nlohmann::json toJson(const std::string &key) const
```
**功能描述**: 将查询信息转换为 JSON 格式
**返回值**: JSON 对象
**代码逻辑**:
1. 创建 JSON 对象，包含 key、size、prot、numBlobs、valid
2. 创建 blobs 数组，遍历每个 blob 添加 rank 和 medium 信息
3. 返回 JSON 对象

---

### MmcBlobFilter

**声明位置**: 行 63-70
**完整签名**:
```cpp
struct MmcBlobFilter : public MmcReferable {
    uint32_t rank_{UINT32_MAX};
    MediaType mediaType_{MEDIA_NONE};
    BlobState state_{NONE};
    MmcBlobFilter(const uint32_t &rank, const MediaType &mediaType, const BlobState &state);
};
```
**功能描述**: Blob 过滤器，用于筛选特定条件的 Blob

**继承关系**:
```
MmcReferable
    ▲
    │
MmcBlobFilter
```

**成员变量**:
| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `rank_` | `uint32_t` | `UINT32_MAX` | Rank ID 过滤条件 |
| `mediaType_` | `MediaType` | `MEDIA_NONE` | 介质类型过滤条件 |
| `state_` | `BlobState` | `NONE` | 状态过滤条件 |

**过滤规则**:
- 当字段为默认值时，该字段不参与过滤（匹配任意值）
- 例如 `rank_ = UINT32_MAX` 表示匹配所有 Rank

---

## 类逐个解读

### MmcMemBlob

**声明位置**: 行 72-182
**完整签名**:
```cpp
class MmcMemBlob final : public MmcReferable
```
**功能描述**: 内存块（Blob）核心类，表示一块具有特定位置和大小的内存区域
**继承关系**:
```
MmcReferable
    ▲
    │
MmcMemBlob (final)
```

#### 构造函数和析构函数

**MmcMemBlob() = delete** (行 74)
- 默认构造函数已删除，必须使用参数化构造

**MmcMemBlob 构造函数** (行 75-78)
```cpp
MmcMemBlob(const uint32_t &rank, const uint64_t &gva, const uint64_t &size, const MediaType &mediaType,
           const BlobState &state)
```
**功能描述**: 参数化构造函数
**参数**:
- `rank` - Rank ID
- `gva` - 全局虚拟地址
- `size` - 数据大小
- `mediaType` - 介质类型
- `state` - 初始状态

**~MmcMemBlob() override = default** (行 79)
- 默认析构函数

---

#### UpdateState (重载1)

**声明位置**: 行 86
**完整签名**:
```cpp
Result UpdateState(const std::string &key, uint32_t rankId, uint32_t operateId, BlobActionResult ret)
```
**功能描述**: 更新 Blob 状态并执行相应的租约操作
**参数**:
- `key` - 内存对象的键
- `rankId` - 操作发起的 Rank ID
- `operateId` - 操作 ID
- `ret` - 操作结果码
**返回值**: `Result` - 操作结果
**代码逻辑** (实现位于 cpp):
1. 查找当前状态在转换表中的记录
2. 查找操作结果对应的转换动作
3. 如果是从 `ALLOCATED` 转到 `READABLE` (写入成功)，执行备份操作
4. 更新状态
5. 执行租约操作函数（如果有）
**注意事项**: 实现在 `mmc_mem_blob.cpp` 中

---

#### UpdateState (重载2)

**声明位置**: 行 93
**完整签名**:
```cpp
Result UpdateState(BlobActionResult ret)
```
**功能描述**: 更新 Blob 状态（不执行租约操作）
**参数**:
- `ret` - 操作结果码
**返回值**: `Result` - 操作结果
**代码逻辑** (实现位于 cpp):
1. 查找当前状态在转换表中的记录
2. 查找操作结果对应的转换动作
3. 更新状态，不执行租约操作
**注意事项**: 实现在 `mmc_mem_blob.cpp` 中

---

#### Next (链表操作)

**声明位置**: 行 101
**完整签名**:
```cpp
Result Next(const MmcMemBlobPtr &nextBlob)
```
**功能描述**: 链接下一个 Blob（实现未在此头文件中）
**参数**:
- `nextBlob` - 要链接的下一个 Blob
**返回值**: `Result` - 如果 nextBlob 为 nullptr 返回 0

---

#### Next (获取)

**声明位置**: 行 107
**完整签名**:
```cpp
MmcMemBlobPtr Next()
```
**功能描述**: 获取链接的下一个 Blob
**返回值**: 下一个 Blob 的智能指针

---

#### Rank

**声明位置**: 行 184-187 (内联实现)
**完整签名**:
```cpp
inline uint32_t Rank() const { return rank_; }
```
**功能描述**: 获取 Blob 所在的 Rank ID
**返回值**: Rank ID

---

#### Gva

**声明位置**: 行 189-192 (内联实现)
**完整签名**:
```cpp
inline uint64_t Gva() const { return gva_; }
```
**功能描述**: 获取 Blob 的全局虚拟地址
**返回值**: 全局虚拟地址

---

#### Size

**声明位置**: 行 194-197 (内联实现)
**完整签名**:
```cpp
inline uint64_t Size() const { return size_; }
```
**功能描述**: 获取 Blob 的数据大小
**返回值**: 数据大小（字节）

---

#### Type

**声明位置**: 行 199-202 (内联实现)
**完整签名**:
```cpp
inline uint16_t Type() const { return mediaType_; }
```
**功能描述**: 获取 Blob 的介质类型
**返回值**: 介质类型（MEDIA_HBM/MEDIA_DRAM）

---

#### State

**声明位置**: 行 204-207 (内联实现)
**完整签名**:
```cpp
inline BlobState State() { return state_; }
```
**功能描述**: 获取 Blob 的当前状态
**返回值**: 当前状态

---

#### Prot

**声明位置**: 行 209-212 (内联实现)
**完整签名**:
```cpp
inline uint16_t Prot() { return prot_; }
```
**功能描述**: 获取 Blob 的访问权限
**返回值**: 访问权限值

---

#### MatchFilter

**声明位置**: 行 214-223 (内联实现)
**完整签名**:
```cpp
inline bool MatchFilter(const MmcBlobFilterPtr &filter) const
```
**功能描述**: 检查 Blob 是否匹配过滤条件
**参数**:
- `filter` - 过滤器指针
**返回值**: 匹配返回 true，否则返回 false
**代码逻辑**:
1. 如果 filter 为 nullptr，返回 true（匹配所有）
2. 检查 rank_、mediaType_、state_ 是否满足过滤条件
3. 每个字段在为默认值时不参与过滤

---

#### GetDesc

**声明位置**: 行 225-228 (内联实现)
**完整签名**:
```cpp
inline MmcMemBlobDesc GetDesc() const { return MmcMemBlobDesc{rank_, gva_, size_, mediaType_}; }
```
**功能描述**: 获取 Blob 的描述符
**返回值**: `MmcMemBlobDesc` 对象

---

#### ExtendLease

**声明位置**: 行 230-234 (内联实现)
**完整签名**:
```cpp
inline Result ExtendLease(const uint32_t id, const uint32_t requestId, uint64_t ttl)
{
    metaLeaseManager_.Add(id, requestId, ttl);
    return MMC_OK;
}
```
**功能描述**: 延长 Blob 的租约
**参数**:
- `id` - Rank ID
- `requestId` - 请求 ID
- `ttl` - 租约延长时长（毫秒）
**返回值**: 始终返回 `MMC_OK`

---

#### IsLeaseExpired

**声明位置**: 行 235-242 (内联实现)
**完整签名**:
```cpp
inline bool IsLeaseExpired()
{
    if (metaLeaseManager_.UseCount() == 0) {
        return true;
    }
    metaLeaseManager_.Wait();
    return true;
}
```
**功能描述**: 检查租约是否已过期
**返回值**: 租约过期返回 true
**代码逻辑**:
1. 如果没有活跃租约，直接返回 true
2. 否则等待租约过期

---

#### Backup

**声明位置**: 行 166
**完整签名**:
```cpp
Result Backup(const std::string &key)
```
**功能描述**: 备份 Blob 元数据
**参数**:
- `key` - 内存对象的键
**返回值**: 操作结果

---

#### BackupRemove

**声明位置**: 行 168
**完整签名**:
```cpp
Result BackupRemove(const std::string &key)
```
**功能描述**: 移除 Blob 元数据备份
**参数**:
- `key` - 内存对象的键
**返回值**: 操作结果

---

#### 流输出运算符

**声明位置**: 行 153-164
**完整签名**:
```cpp
friend std::ostream &operator<<(std::ostream &os, const MmcMemBlob &blob)
friend std::ostream &operator<<(std::ostream &os, const MmcRef<MmcMemBlob> &blob)
```
**功能描述**: 输出 Blob 信息到流
**输出格式**: `Blob{rank=xxx,gva=xxx,size=xxx,media=xxx,state=xxx,prot=xxx,lease=xxx}`

---

#### 成员变量

**声明位置**: 行 170-181
```cpp
private:
    const uint32_t rank_;              /* rank id of the blob located */
    const uint64_t gva_;               /* global virtual address */
    const uint64_t size_;              /* data size of the blob */
    const enum MediaType mediaType_;   /* media type where blob located */
    BlobState state_{BlobState::NONE}; /* state of the blob */
    uint16_t prot_{0};                 /* prot, i.e. access */
    MmcMetaLeaseManager metaLeaseManager_;
    static const StateTransTable stateTransTable_;
```

**成员变量说明**:
| 变量名 | 类型 | 说明 |
|--------|------|------|
| `rank_` | `const uint32_t` | Rank ID（不可变） |
| `gva_` | `const uint64_t` | 全局虚拟地址（不可变） |
| `size_` | `const uint64_t` | 数据大小（不可变） |
| `mediaType_` | `const MediaType` | 介质类型（不可变） |
| `state_` | `BlobState` | 当前状态（可变） |
| `prot_` | `uint16_t` | 访问权限（可变） |
| `metaLeaseManager_` | `MmcMetaLeaseManager` | 租约管理器 |
| `stateTransTable_` | `static const StateTransTable` | 全局状态转换表 |

---

## 类型别名

```cpp
using MmcMemBlobPtr = MmcRef<MmcMemBlob>;
using MmcBlobFilterPtr = MmcRef<MmcBlobFilter>;
```

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          MmcMemBlob                                 │
├─────────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ const 成员 (不可变)                                              │ │
│ │ + rank_: uint32_t         (Rank ID)                             │ │
│ │ + gva_: uint64_t          (全局虚拟地址)                         │ │
│ │ + size_: uint64_t         (数据大小)                             │ │
│ │ + mediaType_: MediaType   (介质类型)                             │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 可变成员                                                         │ │
│ │ + state_: BlobState         (当前状态)                           │ │
│ │ + prot_: uint16_t           (访问权限)                           │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 组件                                                             │ │
│ │ + metaLeaseManager_: MmcMetaLeaseManager                         │ │
│ │ + stateTransTable_: static const StateTransTable                 │ │
│ └─────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────┤
│ 公共方法                                                            │
│ + UpdateState(key, rankId, operateId, ret): Result                 │
│ + UpdateState(ret): Result                                         │
│ + Next(blob): Result               / Next(): MmcMemBlobPtr          │
│ + Rank(): uint32_t                 / Gva(): uint64_t                │
│ + Size(): uint64_t                 / Type(): uint16_t               │
│ + State(): BlobState               / Prot(): uint16_t               │
│ + MatchFilter(filter): bool        / GetDesc(): MmcMemBlobDesc      │
│ + ExtendLease(id, requestId, ttl): Result                          │
│ + IsLeaseExpired(): bool                                            │
│ + Backup(key): Result               / BackupRemove(key): Result     │
└─────────────────────────────────────────────────────────────────────┘
         │                              │
         │ 使用                          │
         ▼                              ▼
┌──────────────────────┐    ┌──────────────────────────────────────────┐
│ MmcMemBlobDesc       │    │      MmcMetaLeaseManager                │
└──────────────────────┘    └──────────────────────────────────────────┘
```

---

## 使用示例

### 示例 1: 创建 Blob

```cpp
#include "mmc_mem_blob.h"

using namespace ock::mmc;

// 创建一个位于 Rank 0, HBM 介质, 大小 4KB 的 Blob
MmcMemBlobPtr blob = MmcMakeRef<MmcMemBlob>(
    0,           // rank
    0x100000,    // gva
    4096,        // size
    MEDIA_HBM,   // mediaType
    ALLOCATED    // initialState
);

// 获取 Blob 属性
uint32_t rank = blob->Rank();
uint64_t gva = blob->Gva();
uint64_t size = blob->Size();
```

### 示例 2: 更新 Blob 状态

```cpp
// 分配完成后，准备写入
blob->UpdateState("my_key", 0, 1001, MMC_ALLOCATED_OK);

// 写入成功
blob->UpdateState("my_key", 0, 1001, MMC_WRITE_OK);
// 现在 blob->State() == READABLE

// 开始读取
blob->UpdateState("my_key", 1, 1002, MMC_READ_START);

// 读取完成
blob->UpdateState("my_key", 1, 1002, MMC_READ_FINISH);
```

### 示例 3: 使用过滤器

```cpp
// 创建过滤器：查找 Rank 0 上的所有 Blob
MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(0, MEDIA_NONE, NONE);

// 检查 Blob 是否匹配
if (blob->MatchFilter(filter)) {
    std::cout << "Blob matches filter" << std::endl;
}

// 创建过滤器：查找 HBM 上的 READABLE 状态的 Blob
MmcBlobFilterPtr filter2 = MmcMakeRef<MmcBlobFilter>(
    UINT32_MAX,  // 任意 Rank
    MEDIA_HBM,   // HBM 介质
    READABLE     // READABLE 状态
);
```

### 示例 4: 获取描述符

```cpp
MmcMemBlobDesc desc = blob->GetDesc();
std::cout << "Blob descriptor: " << desc << std::endl;
// 输出: blob{size=4096,gva=1048576,rank=0,media=0}
```

### 示例 5: 租约操作

```cpp
// 延长租约
blob->ExtendLease(0, 1001, 5000);  // 延长 5000ms

// 等待租约过期
if (blob->IsLeaseExpired()) {
    std::cout << "Blob lease expired" << std::endl;
}
```

### 示例 6: 查询信息转 JSON

```cpp
MemObjQueryInfo info(4096, 0x06, 2, true);
info.blobRanks_[0] = 0;
info.blobTypes_[0] = MEDIA_HBM;
info.blobRanks_[1] = 1;
info.blobTypes_[1] = MEDIA_DRAM;

nlohmann::json json = info.toJson("test_key");
std::cout << json.dump(2) << std::endl;
```
