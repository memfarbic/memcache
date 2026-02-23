# mmc_mem_obj_meta.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/entities/mmc_mem_obj_meta.cpp`
- **文件用途**: 实现 `MmcMemObjMeta` 类的方法，包括 Blob 管理、状态更新和介质层级移动
- **依赖项**:
  - `mmc_mem_obj_meta.h` - Meta 类定义
  - `chrono` - 时间库
  - `mmc_global_allocator.h` - 全局分配器

---

## 常量定义

### MAX_NUM_BLOB_CHAINS

**声明位置**: 行 20
**完整签名**:
```cpp
static const uint16_t MAX_NUM_BLOB_CHAINS = 5;
```
**功能描述**: 每个 MemObject 最多支持的 Blob 链数量
**说明**: 为确保 `MmcMemObjMeta` 类大小不超过 64 字节

---

## 类方法实现

### MmcMemObjMeta::AddBlob

**声明位置**: 行 22-42
**完整签名**:
```cpp
Result MmcMemObjMeta::AddBlob(const MmcMemBlobPtr &blob)
```
**功能描述**: 向内存对象添加一个 Blob
**参数**:
- `blob` - 要添加的 Blob 智能指针
**返回值**: `Result` - 操作结果
**可能返回的错误**:
- `MMC_ERROR` - Blob 大小不一致、Blob 为空或已存在

**代码逻辑**:
1. **检查大小一致性**:
   ```cpp
   if (numBlobs_ != 0 && size_ != blob->Size()) {
       MMC_LOG_ERROR("add blob size:" << blob->Size() << " != meta size:" << size_);
       return MMC_ERROR;
   }
   ```

2. **检查空指针**:
   ```cpp
   for (const auto &old : blobs_) {
       if (old == nullptr || blob == nullptr) {
           MMC_LOG_ERROR("null ptr find: " << (old == nullptr));
           return MMC_ERROR;
       }
   ```

3. **检查重复**:
   ```cpp
   if (old->GetDesc() == blob->GetDesc()) {
       MMC_LOG_INFO("find old block: " << blob->GetDesc());
       return MMC_OK;
   }
   ```

4. **添加 Blob**:
   ```cpp
   blobs_.emplace_back(blob);
   numBlobs_++;
   size_ = blob->Size();
   ```

**流程图**:
```
         ┌─────────────────┐
         │    AddBlob()    │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐     ┌──────────────────┐
         │ 大小一致?        │────▶│ 返回 ERROR       │
         │ (非首个)         │     └──────────────────┘
         └────────┬────────┘
                  │ 是
                  ▼
         ┌─────────────────┐     ┌──────────────────┐
         │ 非空检查         │────▶│ 返回 ERROR       │
         └────────┬────────┘     └──────────────────┘
                  │ 通过
                  ▼
         ┌─────────────────┐
         │ 是否已存在?      │────▶│ 返回 OK (已存在)  │
         └────────┬────────┘     └──────────────────┘
                  │ 否
                  ▼
         ┌─────────────────┐
         │ 添加到列表       │
         │ 更新计数和大小   │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 返回 MMC_OK     │
         └─────────────────┘
```

---

### MmcMemObjMeta::RemoveBlobs

**声明位置**: 行 44-59
**完整签名**:
```cpp
Result MmcMemObjMeta::RemoveBlobs(const MmcBlobFilterPtr &filter, bool revert)
```
**功能描述**: 从列表中移除匹配的 Blob（不释放内存）
**参数**:
- `filter` - Blob 过滤器（可选）
- `revert` - 是否反转过滤结果（默认 false）
**返回值**: `Result` - 操作结果
**访问级别**: private

**代码逻辑**:
1. **保存旧的数量**:
   ```cpp
   uint8_t oldNumBlobs = numBlobs_;
   ```

2. **遍历并删除**:
   ```cpp
   for (auto iter = blobs_.begin(); iter != blobs_.end();) {
       auto &blob = *iter;
       if ((blob != nullptr) && (blob->MatchFilter(filter) ^ revert)) {
           iter = blobs_.erase(iter);
           numBlobs_--;
       } else {
           iter++;
       }
   }
   ```

3. **返回结果**:
   ```cpp
   return numBlobs_ < oldNumBlobs ? MMC_OK : MMC_ERROR;
   ```

**注意事项**:
- 使用 XOR 运算符 `^` 实现反转逻辑
- 只从列表中移除，不释放内存
- 如果没有任何 Blob 被移除，返回 ERROR

---

### MmcMemObjMeta::FreeBlobs

**声明位置**: 行 61-86
**完整签名**:
```cpp
Result MmcMemObjMeta::FreeBlobs(const std::string &key, MmcGlobalAllocatorPtr &allocator,
                                const MmcBlobFilterPtr &filter, bool doBackupRemove)
```
**功能描述**: 释放匹配过滤条件的 Blob（包括内存释放）
**参数**:
- `key` - 内存对象的键
- `allocator` - 全局分配器指针
- `filter` - Blob 过滤器（可选）
- `doBackupRemove` - 是否移除备份（默认 true）
**返回值**: `Result` - 操作结果

**代码逻辑**:
1. **检查 Blob 数量**:
   ```cpp
   if (NumBlobs() == 0) {
       return MMC_OK;
   }
   ```

2. **获取 Blob 列表**:
   ```cpp
   std::vector<MmcMemBlobPtr> blobs = GetBlobs(filter);
   ```

3. **从列表移除**:
   ```cpp
   RemoveBlobs(filter);
   ```

4. **处理每个 Blob**:
   ```cpp
   Result result = MMC_OK;
   for (size_t i = 0; i < blobs.size(); i++) {
       // 可选地移除备份
       if (doBackupRemove) {
           MMC_RETURN_ERROR(blobs[i]->BackupRemove(key), "memBlob remove backup error");
       }
       // 更新状态为 REMOVING
       auto ret = blobs[i]->UpdateState(key, 0, 0, MMC_REMOVE_START);
       if (ret != MMC_OK) {
           MMC_LOG_ERROR("remove op, meta update failed:" << ret);
           result = MMC_ERROR;
       }
       // 释放内存
       ret = allocator->Free(blobs[i]);
       if (ret != MMC_OK) {
           MMC_LOG_ERROR("Error in free blobs! failed:" << ret);
           result = MMC_ERROR;
       }
   }
   ```

**流程图**:
```
         ┌─────────────────┐
         │    FreeBlobs()  │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ numBlobs_ == 0? │────▶│ 返回 MMC_OK       │
         └────────┬────────┘     └──────────────────┘
                  │ 否
                  ▼
         ┌─────────────────┐
         │ GetBlobs()      │
         │ 获取匹配列表     │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ RemoveBlobs()   │
         │ 从列表移除       │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 遍历每个 Blob   │◀────────┐
         └────────┬────────┘         │
                  │                  │
         ┌────────┴────────┐         │
         │                 │         │
         ▼                 ▼         │
   doBackupRemove?    UpdateState   │
      BackupRemove    (REMOVING)     │
         │                 │         │
         └────────┬────────┘         │
                  │                  │
                  ▼                  │
         ┌─────────────────┐         │
         │ allocator->Free │─────────┘
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 返回结果         │
         └─────────────────┘
```

---

### MmcMemObjMeta::GetBlobs

**声明位置**: 行 88-97
**完整签名**:
```cpp
std::vector<MmcMemBlobPtr> MmcMemObjMeta::GetBlobs(const MmcBlobFilterPtr &filter, bool revert)
```
**功能描述**: 获取匹配过滤条件的 Blob 列表
**参数**:
- `filter` - Blob 过滤器（可选）
- `revert` - 是否反转过滤结果（默认 false）
**返回值**: Blob 智能指针向量

**代码逻辑**:
1. 创建结果向量
2. 遍历 blobs_ 列表:
   ```cpp
   for (auto blob : blobs_) {
       if ((blob != nullptr) && (blob->MatchFilter(filter) ^ revert)) {
           blobs.emplace_back(blob);
       }
   }
   ```
3. 返回结果向量

**注意事项**:
- 使用 XOR 运算符 `^` 实现反转逻辑
- filter 为 nullptr 时，MatchFilter 返回 true（匹配所有）

---

### MmcMemObjMeta::GetBlobsDesc

**声明位置**: 行 99-105
**完整签名**:
```cpp
void MmcMemObjMeta::GetBlobsDesc(std::vector<MmcMemBlobDesc> &blobsDesc, const MmcBlobFilterPtr &filter, bool revert)
```
**功能描述**: 获取匹配过滤条件的 Blob 描述符列表
**参数**:
- `blobsDesc` - 输出参数，Blob 描述符列表
- `filter` - Blob 过滤器（可选）
- `revert` - 是否反转过滤结果（默认 false）

**代码逻辑**:
1. 调用 GetBlobs 获取 Blob 列表
2. 遍历列表，调用 GetDesc 获取描述符
3. 将描述符添加到输出列表

---

### MmcMemObjMeta::UpdateBlobsState

**声明位置**: 行 107-125
**完整签名**:
```cpp
Result MmcMemObjMeta::UpdateBlobsState(const std::string &key, const MmcBlobFilterPtr &filter, uint64_t operateId,
                                       BlobActionResult actRet)
```
**功能描述**: 批量更新匹配 Blob 的状态
**参数**:
- `key` - 内存对象的键
- `filter` - Blob 过滤器
- `operateId` - 操作 ID
- `actRet` - 操作结果
**返回值**: `Result` - 操作结果

**代码逻辑**:
1. **获取 Blob 列表**:
   ```cpp
   std::vector<MmcMemBlobPtr> blobs = GetBlobs(filter);
   ```

2. **解析操作 ID**:
   ```cpp
   uint32_t opRankId = GetRankIdByOperateId(operateId);
   uint32_t opSeq = GetSequenceByOperateId(operateId);
   ```

3. **更新每个 Blob 的状态**:
   ```cpp
   Result result = MMC_OK;
   for (auto blob : blobs) {
       auto ret = blob->UpdateState(key, opRankId, opSeq, actRet);
       if (ret != MMC_OK) {
           MMC_LOG_ERROR("Update rank:" << opRankId << ", seq:" << opSeq << " blob state by " << std::to_string(actRet)
                                        << " Fail!");
           result = MMC_ERROR;
       }
   }
   ```

4. **返回结果**: 即使部分失败，也会更新所有 Blob，但返回 ERROR

---

### MmcMemObjMeta::MoveTo

**声明位置**: 行 127-138
**完整签名**:
```cpp
MediaType MmcMemObjMeta::MoveTo(bool down)
```
**功能描述**: 移动内存对象到不同介质层级
**参数**:
- `down` - true 向下移动（HBM→DRAM），false 向上移动（DRAM→HBM）
**返回值**: 目标介质类型

**代码逻辑**:
1. **获取当前介质类型**:
   ```cpp
   MediaType mediaType = MediaType::MEDIA_NONE;
   for (auto blob : blobs_) {
       if (blob != nullptr) {
           mediaType = static_cast<MediaType>(blob->Type());
           break;
       }
   }
   ```

2. **执行移动**:
   ```cpp
   return down ? MoveDown(mediaType) : MoveUp(mediaType);
   ```

**介质层级**:
- `MEDIA_HBM` (高带宽内存，上层)
- `MEDIA_DRAM` (动态随机存取存储器，下层)
- `MEDIA_NONE` (无效)

**移动方向**:
- `down = true`: HBM → DRAM → NONE
- `down = false`: DRAM → HBM, HBM → NONE, NONE → NONE

---

### MmcMemObjMeta::GetBlobType

**声明位置**: 行 140-148
**完整签名**:
```cpp
MediaType MmcMemObjMeta::GetBlobType()
```
**功能描述**: 获取 Blob 的介质类型
**返回值**: 介质类型（MEDIA_HBM/MEDIA_DRAM/MEDIA_NONE）

**代码逻辑**:
1. 遍历 blobs_ 列表
2. 返回第一个非空 Blob 的介质类型
3. 如果没有 Blob，返回 MEDIA_NONE

**注意事项**:
- 假设所有 Blob 的介质类型相同
- 如果列表为空，返回 MEDIA_NONE

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       MmcMemObjMeta 方法实现                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────┐      ┌────────────────┐      ┌────────────────┐   │
│  │   AddBlob      │      │  RemoveBlobs   │      │  FreeBlobs     │   │
│  │  - 检查大小     │      │  - 从列表移除   │      │  - 获取列表     │   │
│  │  - 检查重复     │      │  - 更新计数     │      │  - 移除备份     │   │
│  │  - 添加到列表   │      │                │      │  - 更新状态     │   │
│  └────────────────┘      └────────────────┘      │  - 释放内存     │   │
│                                                   └────────────────┘   │
│                                                                          │
│  ┌────────────────┐      ┌────────────────┐      ┌────────────────┐   │
│  │   GetBlobs     │      │ GetBlobsDesc   │      │UpdateBlobsState│   │
│  │  - 过滤匹配     │      │  - 获取列表     │      │  - 获取列表     │   │
│  │  - 返回向量     │      │  - 获取描述符   │      │  - 解析操作ID   │   │
│  └────────────────┘      │  - 返回描述     │      │  - 批量更新     │   │
│                           └────────────────┘      └────────────────┘   │
│                                                                          │
│  ┌────────────────┐      ┌────────────────┐                            │
│  │    MoveTo      │      │  GetBlobType   │                            │
│  │  - 获取介质类型 │      │  - 遍历列表     │                            │
│  │  - MoveUp/Down │      │  - 返回类型     │                            │
│  └────────────────┘      └────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────┘
                    │                          │
                    │ 调用                     │ 调用
                    ▼                          ▼
         ┌─────────────────────┐    ┌────────────────────────────────────┐
         │   MmcMemBlob        │    │    MmcGlobalAllocator              │
         │  MatchFilter()      │    │    Free(blob)                      │
         │  UpdateState()      │    │                                     │
         │  GetDesc()          │    │                                     │
         └─────────────────────┘    └────────────────────────────────────┘
```

---

## 介质层级移动示意图

```
                    ┌─────────────────┐
                    │    MEDIA_HBM    │  (上层，高带宽)
                    └────────┬────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
         MoveTo(false)              MoveTo(true)
         (向上移动)                   (向下移动)
                │                         │
                ▼                         ▼
         ┌─────────────┐         ┌─────────────┐
         │   MEDIA_    │         │   MEDIA_    │
         │   DRAM      │         │   DRAM      │  (下层)
         └─────────────┘         └──────┬──────┘
                                       │
                              MoveTo(true)
                              (向下移动)
                                       │
                                       ▼
                                ┌─────────────┐
                                │  MEDIA_NONE │  (无效)
                                └─────────────┘
```

---

## 使用示例

### 示例 1: 添加和管理 Blob

```cpp
#include "mmc_mem_obj_meta.h"

using namespace ock::mmc;

MmcMemObjMetaPtr meta = MmcMakeRef<MmcMemObjMeta>();

// 创建并添加 Blob
MmcMemBlobPtr blob1 = MmcMakeRef<MmcMemBlob>(0, 0x100000, 4096, MEDIA_HBM, ALLOCATED);
Result ret = meta->AddBlob(blob1);

MmcMemBlobPtr blob2 = MmcMakeRef<MmcMemBlob>(1, 0x200000, 4096, MEDIA_DRAM, ALLOCATED);
ret = meta->AddBlob(blob2);

// 获取属性
uint16_t numBlobs = meta->NumBlobs();  // 2
uint64_t size = meta->Size();           // 4096
```

### 示例 2: 过滤和获取 Blob

```cpp
// 获取所有 Blob
std::vector<MmcMemBlobPtr> allBlobs = meta->GetBlobs();

// 获取 Rank 0 的 Blob
MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(0, MEDIA_NONE, NONE);
std::vector<MmcMemBlobPtr> rank0Blobs = meta->GetBlobs(filter);

// 获取非 Rank 0 的 Blob（反转过滤）
std::vector<MmcMemBlobPtr> otherBlobs = meta->GetBlobs(filter, true);

// 获取 Blob 描述符
std::vector<MmcMemBlobDesc> descs;
meta->GetBlobsDesc(descs, filter);
```

### 示例 3: 批量更新状态

```cpp
// 生成操作 ID
uint64_t operateId = GenerateOperateId(0);

// 更新所有 Blob 到 READABLE
Result ret = meta->UpdateBlobsState("my_key", nullptr, operateId, MMC_WRITE_OK);

// 只更新特定条件的 Blob
MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(0, MEDIA_HBM, ALLOCATED);
ret = meta->UpdateBlobsState("my_key", filter, operateId, MMC_WRITE_OK);
```

### 示例 4: 释放 Blob

```cpp
MmcGlobalAllocatorPtr allocator = ...;

// 释放所有 Blob
Result ret = meta->FreeBlobs("my_key", allocator);

// 释放特定 Blob（不移除备份）
MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(0, MEDIA_HBM, NONE);
ret = meta->FreeBlobs("my_key", allocator, filter, false);
```

### 示例 5: 介质层级移动

```cpp
// 当前在 HBM
MediaType currentType = meta->GetBlobType();  // MEDIA_HBM

// 向下移动到 DRAM
MediaType newType = meta->MoveTo(true);       // MEDIA_DRAM

// 向上移动回 HBM
newType = meta->MoveTo(false);                // MEDIA_HBM
```

### 示例 6: 错误处理

```cpp
Result ret = meta->AddBlob(blob);
if (ret != MMC_OK) {
    MMC_LOG_ERROR("Failed to add blob: " << ret);
}

ret = meta->UpdateBlobsState("key", nullptr, opId, MMC_WRITE_OK);
if (ret != MMC_OK) {
    MMC_LOG_ERROR("Failed to update blobs state");
}
```
