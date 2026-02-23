# mmc_mem_obj_meta.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/entities/mmc_mem_obj_meta.h`
- **文件用途**: 定义内存对象的元数据类，管理多个 Blob 副本、访问权限和优先级
- **依赖项**:
  - `vector` - 动态数组
  - `mutex` - 互斥锁
  - `list` - 链表
  - `mmc_mem_blob.h` - Blob 类
  - `mmc_meta_lease_manager.h` - 租约管理器
  - `mmc_montotonic.h` - 单调时间
  - `mmc_msg_packer.h` - 消息打包
  - `mmc_ref.h` - 引用计数智能指针
  - `mmc_spinlock.h` - 自旋锁
  - `mmc_global_allocator.h` - 全局分配器

---

## 常量定义

### MAX_NUM_BLOB_CHAINS

**声明位置**: cpp 文件中
**完整签名**:
```cpp
static const uint16_t MAX_NUM_BLOB_CHAINS = 5;
```
**功能描述**: 每个 MemObject 最多支持的 Blob 链数量
**说明**: 为了确保 `MmcMemObjMeta` 类大小不超过 64 字节

---

## 类逐个解读

### MmcMemObjMeta

**声明位置**: 行 30-136
**完整签名**:
```cpp
class MmcMemObjMeta : public MmcReferable
```
**功能描述**: 内存对象元数据类，管理一个内存对象的多个 Blob 副本
**继承关系**:
```
MmcReferable
    ▲
    │
MmcMemObjMeta
```

---

#### 构造函数和析构函数

**MmcMemObjMeta() = default** (行 32)
- 默认构造函数

**MmcMemObjMeta 参数化构造** (行 33-39)
```cpp
MmcMemObjMeta(uint16_t prot, uint8_t priority, std::vector<MmcMemBlobPtr> blobs, uint64_t size)
    : prot_(prot), priority_(priority)
{
    for (auto &blob : blobs) {
        AddBlob(blob);
    }
}
```
**功能描述**: 参数化构造函数
**参数**:
- `prot` - 访问权限
- `priority` - 优先级
- `blobs` - Blob 列表
- `size` - 大小（未直接使用，通过 AddBlob 获取）
**代码逻辑**:
1. 设置 prot_ 和 priority_
2. 遍历 blobs，调用 AddBlob 添加

**~MmcMemObjMeta() override = default** (行 40)
- 默认析构函数

---

#### AddBlob

**声明位置**: 行 48
**完整签名**:
```cpp
Result AddBlob(const MmcMemBlobPtr &blob)
```
**功能描述**: 向内存对象添加一个 Blob
**参数**:
- `blob` - 要添加的 Blob 智能指针
**返回值**: `Result` - 操作结果
**代码逻辑** (实现位于 cpp):
1. 检查 Blob 大小是否一致
2. 检查是否已存在相同的 Blob
3. 添加到 blobs_ 列表
4. 更新 numBlobs_ 和 size_

---

#### FreeBlobs

**声明位置**: 行 56
**完整签名**:
```cpp
Result FreeBlobs(const std::string &key, MmcGlobalAllocatorPtr &allocator, const MmcBlobFilterPtr &filter = nullptr,
                 bool doBackupRemove = true)
```
**功能描述**: 释放匹配过滤条件的 Blob
**参数**:
- `key` - 内存对象的键
- `allocator` - 全局分配器指针
- `filter` - Blob 过滤器（可选）
- `doBackupRemove` - 是否移除备份（默认 true）
**返回值**: `Result` - 操作结果
**代码逻辑** (实现位于 cpp):
1. 获取匹配的 Blob 列表
2. 移除这些 Blob
3. 对每个 Blob:
   - 可选地移除备份
   - 更新状态为 REMOVING
   - 调用分配器释放内存

---

#### Prot

**声明位置**: 行 140-143 (内联实现)
**完整签名**:
```cpp
inline uint16_t Prot() { return prot_; }
```
**功能描述**: 获取访问权限
**返回值**: 访问权限值

---

#### Priority

**声明位置**: 行 145-148 (内联实现)
**完整签名**:
```cpp
inline uint8_t Priority() { return priority_; }
```
**功能描述**: 获取优先级
**返回值**: 优先级值

---

#### NumBlobs

**声明位置**: 行 150-153 (内联实现)
**完整签名**:
```cpp
inline uint16_t NumBlobs() { return numBlobs_; }
```
**功能描述**: 获取 Blob 数量
**返回值**: Blob 数量

---

#### Size

**声明位置**: 行 155-158 (内联实现)
**完整签名**:
```cpp
inline uint64_t Size() { return size_; }
```
**功能描述**: 获取内存对象大小
**返回值**: 大小（字节）

---

#### GetBlobs

**声明位置**: 行 81
**完整签名**:
```cpp
std::vector<MmcMemBlobPtr> GetBlobs(const MmcBlobFilterPtr &filter = nullptr, bool revert = false)
```
**功能描述**: 获取匹配过滤条件的 Blob 列表
**参数**:
- `filter` - Blob 过滤器（可选）
- `revert` - 是否反转过滤结果（默认 false）
**返回值**: Blob 智能指针向量
**代码逻辑** (实现位于 cpp):
1. 遍历 blobs_ 列表
2. 对每个 Blob 调用 MatchFilter
3. 根据 revert 决定是否包含匹配/不匹配的 Blob
4. 返回结果列表

---

#### GetBlobsDesc

**声明位置**: 行 83-84
**完整签名**:
```cpp
void GetBlobsDesc(std::vector<MmcMemBlobDesc> &blobsDesc, const MmcBlobFilterPtr &filter = nullptr,
                  bool revert = false)
```
**功能描述**: 获取匹配过滤条件的 Blob 描述符列表
**参数**:
- `blobsDesc` - 输出参数，Blob 描述符列表
- `filter` - Blob 过滤器（可选）
- `revert` - 是否反转过滤结果（默认 false）
**代码逻辑** (实现位于 cpp):
1. 调用 GetBlobs 获取 Blob 列表
2. 对每个 Blob 调用 GetDesc 获取描述符
3. 将描述符添加到输出列表

---

#### UpdateBlobsState

**声明位置**: 行 86-87
**完整签名**:
```cpp
Result UpdateBlobsState(const std::string &key, const MmcBlobFilterPtr &filter, uint64_t operateId,
                        BlobActionResult actRet)
```
**功能描述**: 批量更新匹配 Blob 的状态
**参数**:
- `key` - 内存对象的键
- `filter` - Blob 过滤器
- `operateId` - 操作 ID
- `actRet` - 操作结果
**返回值**: `Result` - 操作结果
**代码逻辑** (实现位于 cpp):
1. 获取匹配的 Blob 列表
2. 从 operateId 解析出 rankId 和 sequence
3. 对每个 Blob 调用 UpdateState

---

#### MoveTo

**声明位置**: 行 99
**完整签名**:
```cpp
MediaType MoveTo(bool down)
```
**功能描述**: 移动内存对象到不同介质层级
**参数**:
- `down` - true 向下移动（HBM→DRAM），false 向上移动（DRAM→HBM）
**返回值**: 目标介质类型
**代码逻辑** (实现位于 cpp):
1. 获取第一个 Blob 的介质类型
2. 调用 MoveDown 或 MoveUp

---

#### GetBlobType

**声明位置**: 行 101
**完整签名**:
```cpp
MediaType GetBlobType()
```
**功能描述**: 获取 Blob 的介质类型
**返回值**: 介质类型（MEDIA_HBM/MEDIA_DRAM/MEDIA_NONE）
**代码逻辑** (实现位于 cpp):
1. 遍历 blobs_
2. 返回第一个非空 Blob 的介质类型
3. 如果没有 Blob，返回 MEDIA_NONE

---

#### Mutex

**声明位置**: 行 103-106
**完整签名**:
```cpp
inline std::mutex &Mutex() { return mutex_; }
```
**功能描述**: 获取内部互斥锁引用
**返回值**: 互斥锁引用
**注意事项**: 在访问/修改 Meta 数据前必须先加锁

---

#### RemoveBlobs

**声明位置**: 行 126
**完整签名**:
```cpp
Result RemoveBlobs(const MmcBlobFilterPtr &filter = nullptr, bool revert = false)
```
**功能描述**: 从列表中移除匹配的 Blob（不释放内存）
**参数**:
- `filter` - Blob 过滤器（可选）
- `revert` - 是否反转过滤结果（默认 false）
**访问级别**: private
**代码逻辑** (实现位于 cpp):
1. 遍历 blobs_ 列表
2. 使用 erase 删除匹配的 Blob
3. 更新 numBlobs_

---

#### 成员变量

**声明位置**: 行 128-135
```cpp
private:
    /* make sure the size of this class is 64 bytes */
    uint16_t prot_{0};               /* prot of the mem object, i.e. accessibility */
    uint8_t priority_{0};            /* priority of the memory object, used for eviction */
    uint8_t numBlobs_{0};            /* number of blob that the memory object, i.e. replica count */
    std::list<MmcMemBlobPtr> blobs_; /* 24 bytes */
    uint64_t size_{0};               /* byteSize of each blob */
    std::mutex mutex_;               /* must lock before read/write this meta */
```

**成员变量说明**:
| 变量名 | 类型 | 说明 |
|--------|------|------|
| `prot_` | `uint16_t` | 访问权限（类似 Unix 文件权限） |
| `priority_` | `uint8_t` | 优先级，用于淘汰策略 |
| `numBlobs_` | `uint8_t` | Blob 数量（副本数） |
| `blobs_` | `std::list<MmcMemBlobPtr>` | Blob 列表 |
| `size_` | `uint64_t` | 每个 Blob 的大小 |
| `mutex_` | `std::mutex` | 保护 Meta 数据的互斥锁 |

**内存布局说明**: 类设计目标是不超过 64 字节
- `prot_` (2) + `priority_` (1) + `numBlobs_` (1) = 4 字节
- `blobs_` ≈ 24 字节
- `size_` = 8 字节
- `mutex_` ≈ 40 字节（典型实现）
- 总计 ≈ 76 字节（可能因实现略有不同）

---

#### 流输出运算符

**声明位置**: 行 108-123
**完整签名**:
```cpp
friend std::ostream &operator<<(std::ostream &os, const MmcMemObjMeta &obj)
friend std::ostream &operator<<(std::ostream &os, const MmcRef<MmcMemObjMeta> &obj)
```
**功能描述**: 输出 Meta 信息到流
**输出格式**:
```
MmcMemObjMeta{numBlobs=x,size=xxx,priority=x,prot=xxx}Blob{...}...
```

---

## 类型别名

```cpp
using MmcMemObjMetaPtr = MmcRef<MmcMemObjMeta>;
```

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        MmcMemObjMeta                                    │
├─────────────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ 成员变量                                                             │ │
│ │ + prot_: uint16_t              (访问权限)                            │ │
│ │ + priority_: uint8_t           (优先级)                              │ │
│ │ + numBlobs_: uint8_t           (Blob 数量)                           │ │
│ │ + size_: uint64_t              (每个 Blob 大小)                      │ │
│ │ + blobs_: list<MmcMemBlobPtr>  (Blob 列表)                           │ │
│ │ + mutex_: mutex                (保护锁)                               │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│ 公共方法                                                                │
│ + AddBlob(blob): Result                                                │
│ + FreeBlobs(key, allocator, filter, doBackupRemove): Result            │
│ + GetBlobs(filter, revert): vector<MmcMemBlobPtr>                      │
│ + GetBlobsDesc(blobsDesc, filter, revert): void                        │
│ + UpdateBlobsState(key, filter, operateId, actRet): Result             │
│ + MoveTo(down): MediaType                                             │
│ + GetBlobType(): MediaType                                            │
│ + Mutex(): mutex&                                                      │
│ + Prot(): uint16_t                     / + Priority(): uint8_t          │
│ + NumBlobs(): uint16_t                 / + Size(): uint64_t             │
└─────────────────────────────────────────────────────────────────────────┘
                    │                              │
                    │ 包含                        │ 使用
                    ▼                              ▼
         ┌──────────────────────┐     ┌──────────────────────────────────┐
         │    MmcMemBlob        │     │    MmcGlobalAllocator            │
         │  (多个副本)          │     │    Free(blob)                    │
         └──────────────────────┘     └──────────────────────────────────┘
```

---

## 使用示例

### 示例 1: 创建内存对象元数据

```cpp
#include "mmc_mem_obj_meta.h"

using namespace ock::mmc;

// 创建 Blob
MmcMemBlobPtr blob1 = MmcMakeRef<MmcMemBlob>(0, 0x100000, 4096, MEDIA_HBM, READABLE);
MmcMemBlobPtr blob2 = MmcMakeRef<MmcMemBlob>(1, 0x200000, 4096, MEDIA_DRAM, READABLE);

// 创建内存对象元数据
std::vector<MmcMemBlobPtr> blobs = {blob1, blob2};
MmcMemObjMetaPtr meta = MmcMakeRef<MmcMemObjMeta>(
    0x06,      // prot (读写权限)
    5,         // priority
    blobs,     // blob 列表
    4096       // size
);

// 获取属性
uint16_t prot = meta->Prot();        // 0x06
uint8_t priority = meta->Priority(); // 5
uint16_t numBlobs = meta->NumBlobs(); // 2
uint64_t size = meta->Size();        // 4096
```

### 示例 2: 添加和获取 Blob

```cpp
MmcMemObjMetaPtr meta = MmcMakeRef<MmcMemObjMeta>();

// 添加 Blob
MmcMemBlobPtr blob = MmcMakeRef<MmcMemBlob>(0, 0x100000, 4096, MEDIA_HBM, ALLOCATED);
Result ret = meta->AddBlob(blob);

// 获取所有 Blob
std::vector<MmcMemBlobPtr> allBlobs = meta->GetBlobs();

// 使用过滤器获取特定 Blob
MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(0, MEDIA_HBM, READABLE);
std::vector<MmcMemBlobPtr> filteredBlobs = meta->GetBlobs(filter);

// 反向过滤（获取不匹配的 Blob）
std::vector<MmcMemBlobPtr> otherBlobs = meta->GetBlobs(filter, true);
```

### 示例 3: 更新 Blob 状态

```cpp
// 更新所有 Blob 的状态
uint64_t operateId = GenerateOperateId(0);
Result ret = meta->UpdateBlobsState("my_key", nullptr, operateId, MMC_READ_START);

// 只更新特定 Rank 的 Blob
MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(0, MEDIA_NONE, NONE);
ret = meta->UpdateBlobsState("my_key", filter, operateId, MMC_READ_FINISH);
```

### 示例 4: 释放 Blob

```cpp
MmcGlobalAllocatorPtr allocator = ...;

// 释放所有 Blob
Result ret = meta->FreeBlobs("my_key", allocator);

// 释放特定条件的 Blob
MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(0, MEDIA_HBM, READABLE);
ret = meta->FreeBlobs("my_key", allocator, filter, true);

// 释放但不移除备份
ret = meta->FreeBlobs("my_key", allocator, nullptr, false);
```

### 示例 5: 介质层级移动

```cpp
// 向下移动（HBM → DRAM）
MediaType newType = meta->MoveTo(true);
// newType = MEDIA_DRAM

// 向上移动（DRAM → HBM）
newType = meta->MoveTo(false);
// newType = MEDIA_HBM
```

### 示例 6: 线程安全访问

```cpp
MmcMemObjMetaPtr meta = ...;

// 加锁访问
{
    std::lock_guard<std::mutex> lock(meta->Mutex());

    // 安全地读取/修改 Meta
    uint16_t numBlobs = meta->NumBlobs();
    std::vector<MmcMemBlobPtr> blobs = meta->GetBlobs();
}
```

### 示例 7: 获取 Blob 描述符

```cpp
// 获取所有 Blob 的描述符
std::vector<MmcMemBlobDesc> descs;
meta->GetBlobsDesc(descs);

for (const auto &desc : descs) {
    std::cout << "Blob: " << desc << std::endl;
}

// 获取特定 Blob 的描述符
MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(0, MEDIA_HBM, READABLE);
meta->GetBlobsDesc(descs, filter);
```
