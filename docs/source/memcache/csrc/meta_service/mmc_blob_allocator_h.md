# mmc_blob_allocator.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_blob_allocator.h`
- **文件用途**: 定义 Blob 内存分配器类，管理单一内存空间的分配和释放，使用空闲块管理算法
- **依赖项**: `mmc_mem_blob.h`, `mmc_spinlock.h`

---

## 数据结构

### SpaceRange

```cpp
struct SpaceRange {
    const uint64_t offset_ = 0;
    const uint64_t size_ = 0;

    SpaceRange(const uint64_t offset, const uint64_t size) : offset_(offset), size_(size) {}
};
```

**说明**: 内存空间范围描述符
- `offset_`: 空间起始偏移量
- `size_`: 空间大小

---

### RangeSizeFirst

```cpp
struct RangeSizeFirst {
    bool operator()(const SpaceRange &sr1, const SpaceRange &sr2) const
    {
        if (sr1.size_ != sr2.size_) {
            return sr1.size_ < sr2.size_;
        }
        return sr1.offset_ < sr2.offset_;
    }
};
```

**说明**: 按大小优先比较的仿函数，用于 sizeTree_ 的排序
- 首先按大小升序排序
- 大小相同时按偏移量升序排序

---

## 类定义

### MmcBlobAllocator

内存分配器类，负责管理单个 rank 上的特定介质类型的内存分配。

#### 构造函数

```cpp
MmcBlobAllocator(const uint32_t rank, const MediaType mediaType, const uint64_t bmAddr, const uint64_t capacity)
    : rank_(rank), mediaType_(mediaType), bmAddr_(bmAddr), capacity_(capacity)
{
    started_ = false;
    addressTree_[0] = capacity;
    sizeTree_.insert({0, capacity});
}
```

**声明位置**: 行 44-50

**功能描述**: 初始化内存分配器

**参数**:
- `rank`: Rank ID
- `mediaType`: 介质类型（HBM/DRAM）
- `bmAddr`: Blob Manager 地址
- `capacity`: 内存容量

**代码逻辑**:
1. 初始化成员变量
2. 将 `started_` 设为 false
3. 在 `addressTree_` 中创建初始空闲块 `[0, capacity)`
4. 在 `sizeTree_` 中插入初始空闲块

---

#### CanAlloc

```cpp
bool CanAlloc(uint64_t blobSize);
```

**声明位置**: 行 53

**功能描述**: 检查是否可以分配指定大小的内存

**参数**:
- `blobSize`: 需要分配的大小

**返回值**:
- `true`: 可以分配
- `false`: 不可以分配

**实现** (`mmc_blob_allocator.cpp`):
```cpp
bool MmcBlobAllocator::CanAlloc(uint64_t blobSize)
{
    if (!started_) {
        MMC_LOG_WARN("Allocator rank: " << rank_ << " mediaType: " << mediaType_ << " is stopped");
        return false;
    }
    SpaceRange anchor{0, AllocSizeAlignUp(blobSize)};

    spinlock_.lock();
    if (!started_) {
        spinlock_.unlock();
        MMC_LOG_WARN("Allocator rank: " << rank_ << " mediaType: " << mediaType_ << " is stopped");
        return false;
    }
    bool exists = (sizeTree_.lower_bound(anchor) != sizeTree_.end());
    spinlock_.unlock();

    return exists;
}
```

**代码逻辑**:
1. 检查分配器是否已启动
2. 对请求大小进行对齐
3. 在 sizeTree_ 中查找是否存在足够大的空闲块
4. 使用 `lower_bound` 查找第一个大于等于请求大小的块

---

#### Alloc

```cpp
MmcMemBlobPtr Alloc(uint64_t blobSize);
```

**声明位置**: 行 54

**功能描述**: 分配指定大小的内存块

**参数**:
- `blobSize`: 需要分配的大小

**返回值**: 成功返回 MmcMemBlobPtr，失败返回 nullptr

**实现** (`mmc_blob_allocator.cpp`):
```cpp
MmcMemBlobPtr MmcBlobAllocator::Alloc(uint64_t blobSize)
{
    if (!started_) {
        MMC_LOG_WARN("Allocator rank: " << rank_ << " mediaType: " << mediaType_ << " is stopped");
        return nullptr;
    }
    auto alignedSize = AllocSizeAlignUp(blobSize);

    SpaceRange anchor{0, alignedSize};

    spinlock_.lock();
    if (!started_) {
        spinlock_.unlock();
        MMC_LOG_WARN("Allocator rank: " << rank_ << " mediaType: " << mediaType_ << " is stopped");
        return nullptr;
    }
    auto sizePos = sizeTree_.lower_bound(anchor);
    if (sizePos == sizeTree_.end()) {
        spinlock_.unlock();
        MMC_LOG_WARN("Allocator rank: " << rank_ << " mediaType: " << mediaType_ << ", cap:" << allocatedSize_ << "/"
                                        << capacity_ << " cannot allocate with size: " << blobSize);
        return nullptr;
    }

    auto targetOffset = sizePos->offset_;
    auto targetSize = sizePos->size_;
    auto addrPos = addressTree_.find(targetOffset);
    if (addrPos == addressTree_.end()) {
        spinlock_.unlock();
        MMC_LOG_ERROR("Allocator rank: " << rank_ << " mediaType: " << mediaType_
                                         << " offset in size tree, not in address tree");
        return nullptr;
    }

    sizeTree_.erase(sizePos);
    addressTree_.erase(addrPos);
    if (targetSize > alignedSize) {
        SpaceRange left{targetOffset + alignedSize, targetSize - alignedSize};
        addressTree_.emplace(left.offset_, left.size_);
        sizeTree_.emplace(left);
    }
    allocatedSize_ += alignedSize;
    spinlock_.unlock();

    return MmcMakeRef<MmcMemBlob>(rank_, bmAddr_ + targetOffset, blobSize, mediaType_, ALLOCATED);
}
```

**代码逻辑**:
1. 检查分配器状态
2. 对齐请求大小
3. 使用首次适应算法（通过 sizeTree_ 的 lower_bound）查找合适的空闲块
4. 从两棵树中移除找到的空闲块
5. 如果空闲块大于请求大小，将剩余空间重新插入空闲树
6. 更新已分配大小
7. 返回新创建的 Blob 对象

---

#### Release

```cpp
Result Release(const MmcMemBlobPtr &blob);
```

**声明位置**: 行 55

**功能描述**: 释放已分配的内存块

**参数**:
- `blob`: 要释放的 Blob 指针

**返回值**: 成功返回 MMC_OK，失败返回错误码

**实现** (`mmc_blob_allocator.cpp`):
```cpp
Result MmcBlobAllocator::Release(const MmcMemBlobPtr &blob)
{
    if (blob == nullptr) {
        MMC_LOG_ERROR("blob is null");
        return MMC_ERROR;
    }
    auto alignedSize = AllocSizeAlignUp(blob->Size());
    MMC_ASSERT_RETURN(allocatedSize_ >= alignedSize, MMC_ERROR);
    auto blobAddr = blob->Gva();
    if (blobAddr < bmAddr_ || blobAddr + alignedSize > bmAddr_ + capacity_) {
        MMC_LOG_ERROR("blob address not in allocator");
        return MMC_ERROR;
    }

    auto offset = blobAddr - bmAddr_;
    uint64_t finalOffset = offset;
    uint64_t finalSize = alignedSize;

    spinlock_.lock();
    auto prevAddrPos = addressTree_.lower_bound(offset);
    if (prevAddrPos != addressTree_.end() && prevAddrPos->first == offset) {
        spinlock_.unlock();
        MMC_LOG_ERROR("blob already released");
        return MMC_ERROR;
    }
    if (prevAddrPos != addressTree_.begin()) {
        --prevAddrPos;
        if (prevAddrPos->first + prevAddrPos->second > offset) {
            spinlock_.unlock();
            MMC_LOG_ERROR("blob already released");
            return MMC_ERROR;
        }
        if (prevAddrPos != addressTree_.end() &&
            prevAddrPos->first + prevAddrPos->second == offset) { // 合并前一个range
            finalOffset = prevAddrPos->first;
            finalSize += prevAddrPos->second;
            sizeTree_.erase(SpaceRange{prevAddrPos->first, prevAddrPos->second});
            addressTree_.erase(prevAddrPos);
        }
    }

    auto nextAddrPos = addressTree_.find(offset + alignedSize);
    if (nextAddrPos != addressTree_.end()) {
        finalSize += nextAddrPos->second;
        sizeTree_.erase(SpaceRange{nextAddrPos->first, nextAddrPos->second});
        addressTree_.erase(nextAddrPos);
    }

    addressTree_.emplace(finalOffset, finalSize);
    sizeTree_.emplace(SpaceRange{finalOffset, finalSize});

    allocatedSize_ -= alignedSize;

    spinlock_.unlock();
    return MMC_OK;
}
```

**代码逻辑**:
1. 验证 Blob 参数
2. 验证地址范围
3. 计算偏移量
4. 尝试与前一个空闲块合并
5. 尝试与后一个空闲块合并
6. 将合并后的空闲块插入空闲树
7. 更新已分配大小

---

#### BuildFromBlobs

```cpp
Result BuildFromBlobs(std::map<std::string, MmcMemBlobDesc> &blobMap);
```

**声明位置**: 行 56

**功能描述**: 从已有的 Blob 描述重建分配器状态（用于恢复）

**参数**:
- `blobMap`: Blob 描述映射表

**返回值**: 成功返回 MMC_OK

**实现** (`mmc_blob_allocator.cpp`):
```cpp
Result MmcBlobAllocator::BuildFromBlobs(std::map<std::string, MmcMemBlobDesc> &blobMap)
{
    spinlock_.lock();
    if (started_) {
        spinlock_.unlock();
        MMC_LOG_ERROR("rebuild allocator failed, rank: " << rank_ << " mediaType: " << mediaType_
                                                         << ", allocator must not started and empty");
        return MMC_ERROR;
    }

    // 处理每个已分配的blob
    for (auto it = blobMap.begin(); it != blobMap.end();) {
        if (it->second.rank_ != rank_) {
            MMC_LOG_WARN("rebuild blob not match, allocator rank: " << rank_ << ", blob rank: " << it->second.rank_);
            it = blobMap.erase(it);
            continue;
        }

        if (it->second.mediaType_ != mediaType_) {
            MMC_LOG_WARN("rebuild blob not match, allocator mediaType: " << mediaType_ << ", blob mediaType: "
                                                                         << it->second.mediaType_);
            ++it;
            continue;
        }
        uint64_t gva = it->second.gva_;
        uint64_t size = it->second.size_;
        uint64_t offset = gva - bmAddr_; // 转换为分配器内部偏移

        Result res = ValidateAndAddAllocation(offset, size);
        if (res != MMC_OK) {
            MMC_LOG_ERROR("rebuild allocator failed, rank: " << rank_ << " mediaType: " << mediaType_
                                                             << ", blob off:" << offset << ", size: " << size);
            it = blobMap.erase(it);
            continue;
        }
        MMC_LOG_INFO("rebuild block successful, rank: " << it->second);
        ++it;
    }
    spinlock_.unlock();
    return MMC_OK;
}
```

**代码逻辑**:
1. 验证分配器未启动
2. 遍历 blobMap，处理每个已分配的 Blob
3. 验证 Blob 是否属于当前分配器（rank 和 mediaType 匹配）
4. 调用 ValidateAndAddAllocation 从空闲空间中扣除已分配部分

---

#### Start

```cpp
void Start()
{
    spinlock_.lock();
    started_ = true;
    spinlock_.unlock();
}
```

**声明位置**: 行 57-62

**功能描述**: 启动分配器，允许进行内存分配操作

---

#### Stop

```cpp
void Stop()
{
    spinlock_.lock();
    started_ = false;
    spinlock_.unlock();
}
```

**声明位置**: 行 63-68

**功能描述**: 停止分配器，暂停内存分配操作

---

#### CanUnmount

```cpp
bool CanUnmount()
{
    return allocatedSize_ == 0;
}
```

**声明位置**: 行 69-72

**功能描述**: 检查分配器是否可以卸载

**返回值**: 如果已分配大小为 0 则可卸载

---

#### GetUsageInfo

```cpp
std::pair<uint64_t, uint64_t> GetUsageInfo()
{
    return std::make_pair(capacity_, allocatedSize_);
}
```

**声明位置**: 行 74-77

**功能描述**: 获取内存使用信息

**返回值**: pair<总容量, 已使用大小>

---

#### GetInfo

```cpp
nlohmann::json GetInfo() const
{
    nlohmann::json info;
    info["rank"] = rank_;
    info["medium"] = MediumTypeToString(mediaType_);
    info["bmAddr"] = bmAddr_;
    info["capacity"] = capacity_;
    info["allocatedSize"] = allocatedSize_;
    return info;
}
```

**声明位置**: 行 79-88

**功能描述**: 获取分配器信息的 JSON 格式

---

### 私有方法

#### AllocSizeAlignUp (静态方法)

```cpp
static uint64_t AllocSizeAlignUp(uint64_t size);
```

**声明位置**: 行 91

**功能描述**: 将大小向上对齐到 4KB

**实现** (`mmc_blob_allocator.cpp`):
```cpp
uint64_t MmcBlobAllocator::AllocSizeAlignUp(uint64_t size)
{
    constexpr uint64_t alignSize = 4096UL;
    if (size > UINT64_MAX - alignSize + 1UL) {
        MMC_LOG_ERROR("Invalid size: " << size << " will occur overflow");
        return UINT64_MAX;
    }
    constexpr uint64_t alignSizeMask = ~(alignSize - 1UL);
    return (size + alignSize - 1UL) & alignSizeMask;
}
```

---

#### ValidateAndAddAllocation

```cpp
Result ValidateAndAddAllocation(uint64_t offset, uint64_t size);
```

**声明位置**: 行 92

**功能描述**: 验证并添加一个已分配的块到分配器（用于重建）

**实现** (`mmc_blob_allocator.cpp`):
```cpp
Result MmcBlobAllocator::ValidateAndAddAllocation(uint64_t offset, uint64_t size)
{
    // 验证地址范围有效性
    if (offset > std::numeric_limits<uint64_t>::max() - size) {
        MMC_LOG_ERROR("blob range overflow: offset:" << offset << ", size:" << size);
        return MMC_ERROR;
    }
    if (offset >= capacity_ || (offset + size) > capacity_) {
        MMC_LOG_ERROR("Blob out of range: offset: " << offset << ", size: " << size << ", capacity: " << capacity_);
        return MMC_ERROR;
    }

    // 计算对齐后的大小
    uint64_t alignedSize = AllocSizeAlignUp(size);
    if (offset > std::numeric_limits<uint64_t>::max() - alignedSize) {
        MMC_LOG_ERROR("offset plus size overflow: offset=" << offset << ", size=" << size);
        return MMC_ERROR;
    }

    // 查找包含该分配的空闲块
    auto it = addressTree_.upper_bound(offset);
    if (it != addressTree_.begin()) {
        --it; // 回退到可能包含offset的块
    }

    if (it == addressTree_.end() || it->first > offset || (it->first + it->second) < (offset + alignedSize)) {
        MMC_LOG_ERROR("No matching free block for blob: offset=" << offset << ", size=" << size);
        return MMC_ERROR;
    }

    // 从空闲树中移除该块
    uint64_t blockOffset = it->first;
    uint64_t blockSize = it->second;
    sizeTree_.erase(SpaceRange{blockOffset, blockSize});
    addressTree_.erase(it);

    // 分割剩余空间（前部）
    if (blockOffset < offset) {
        uint64_t frontSize = offset - blockOffset;
        SpaceRange left{blockOffset, frontSize};
        addressTree_.emplace(left.offset_, left.size_);
        sizeTree_.emplace(left);
    }

    // 分割剩余空间（后部）
    uint64_t endOffset = offset + alignedSize;
    uint64_t remainingSize = (blockOffset + blockSize) - endOffset;
    if (remainingSize > 0) {
        SpaceRange left{endOffset, remainingSize};
        addressTree_.emplace(left.offset_, left.size_);
        sizeTree_.emplace(left);
    }

    // 更新已分配大小
    allocatedSize_ += alignedSize;
    return MMC_OK;
}
```

---

## 成员变量

```cpp
private:
    const uint32_t rank_;       /* rank id of the space */
    const MediaType mediaType_; /* media type of the space */
    const uint64_t bmAddr_;     /* bm address */
    const uint64_t capacity_;   /* capacity of the space */

    std::map<uint64_t, uint64_t> addressTree_;
    std::set<SpaceRange, RangeSizeFirst> sizeTree_;

    volatile bool started_ = false;
    uint64_t allocatedSize_ = 0;

    Spinlock spinlock_;
```

**说明**:
- `addressTree_`: 按地址索引的空闲块树，key=offset, value=size
- `sizeTree_`: 按大小优先索引的空闲块集合
- 双树结构支持高效的分配和释放操作
- `started_`: 分配器启动标志
- `allocatedSize_`: 已分配的总大小
- `spinlock_`: 保护内部状态的自旋锁

---

## 文件级别的关系图

```
mmc_blob_allocator.h (内存分配器)
    |
    +-- 依赖 mmc_mem_blob.h (Blob 对象)
    |
    +-- 依赖 mmc_spinlock.h (自旋锁)
    |
    +-- 被 mmc_global_allocator.h 使用 (全局分配器)
    |
    +-- 首次适应分配算法
    +-- 空闲块合并策略
```

---

## 类型别名

```cpp
using MmcBlobAllocatorPtr = MmcRef<MmcBlobAllocator>;
```
