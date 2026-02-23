# mmc_blob_allocator.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_blob_allocator.cpp`
- **文件用途**: MmcBlobAllocator 类的实现文件
- **依赖项**: `mmc_blob_allocator.h`, `mmc_logger.h`

---

## 函数实现

### CanAlloc

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

**声明位置**: 行 20-38

**功能描述**: 检查是否可以分配指定大小的内存块

**代码逻辑**:
1. 首次检查分配器是否已启动
2. 创建一个锚点 SpaceRange 用于查找，大小为对齐后的请求大小
3. 获取自旋锁
4. 二次检查分配器状态（防止检查和加锁之间状态改变）
5. 使用 `lower_bound` 在 sizeTree_ 中查找第一个大于等于请求大小的块
6. 释放锁并返回结果

**注意事项**:
- 双重检查模式确保线程安全
- 使用 lower_bound 实现首次适应算法

---

### Alloc

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

**声明位置**: 行 40-85

**功能描述**: 分配指定大小的内存块

**代码逻辑**:
1. 检查分配器状态
2. 对齐请求大小到 4KB 边界
3. 创建查找锚点
4. 获取自旋锁并二次检查状态
5. 在 sizeTree_ 中查找合适的空闲块
6. 如果找不到，释放锁并返回 nullptr
7. 找到后在 addressTree_ 中定位同一块
8. 从两棵树中移除该空闲块
9. 如果空闲块大于请求大小，将剩余部分重新插入空闲树
10. 更新已分配大小
11. 释放锁并返回新创建的 MmcMemBlob 对象

**内存分配策略**:
- 首次适应算法（First Fit）
- 最小满足大小分配（通过 sizeTree_ 按 size 排序实现）
- 自动分割剩余空间

---

### Release

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

**声明位置**: 行 87-142

**功能描述**: 释放已分配的内存块，并尝试与相邻空闲块合并

**代码逻辑**:
1. 验证输入参数（blob 非空）
2. 验证地址范围是否在分配器管理范围内
3. 计算相对偏移量
4. 获取自旋锁
5. 检查是否已经释放（地址冲突）
6. 尝试与前一个空闲块合并（物理地址连续）
7. 尝试与后一个空闲块合并
8. 将合并后的空闲块插入双树结构
9. 更新已分配大小
10. 释放锁

**空闲块合并策略**:
- 向前合并：检查前一个块的结束地址是否等于当前块的起始地址
- 向后合并：检查后一个块的起始地址是否等于当前块的结束地址
- 三块合并可能：前块 + 当前块 + 后块

---

### BuildFromBlobs

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

**声明位置**: 行 144-184

**功能描述**: 从已有的 Blob 描述重建分配器状态

**使用场景**: 元服务重启后从持久化数据恢复内存分配状态

**代码逻辑**:
1. 获取自旋锁
2. 验证分配器未启动（只能在启动前重建）
3. 遍历 blobMap 中的所有 Blob 描述
4. 验证每个 Blob 是否属于当前分配器（rank 和 mediaType 匹配）
5. 计算内部偏移量
6. 调用 ValidateAndAddAllocation 从空闲空间中扣除已分配部分
7. 处理失败的重建（从 map 中移除）
8. 释放锁

---

### ValidateAndAddAllocation

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

**声明位置**: 行 186-242

**功能描述**: 验证并添加一个已分配的块到分配器

**代码逻辑**:
1. 验证 offset + size 不会溢出
2. 验证地址范围在容量内
3. 计算对齐后的大小
4. 查找包含该分配区域的空闲块
5. 如果找不到包含该区域的空闲块，返回错误
6. 从双树中移除找到的空闲块
7. 如果空闲块前部有剩余，创建前部剩余空闲块
8. 如果空闲块后部有剩余，创建后部剩余空闲块
9. 更新已分配大小

**注意事项**:
- 此函数假设已持有 spinlock_
- 用于重建时从初始的完整空闲块中扣除已分配部分

---

### AllocSizeAlignUp

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

**声明位置**: 行 244-253

**功能描述**: 将大小向上对齐到 4KB 边界

**对齐算法**:
- 对齐大小: 4096 (4KB)
- 使用位运算实现高效对齐
- 公式: `(size + 4095) & ~4095` 等价于 `((size - 1) / 4096 + 1) * 4096`

**溢出保护**:
- 检查 size + alignSize 是否溢出

---

## 总结

此文件实现了 MmcBlobAllocator 类的核心功能：

1. **内存分配**: 首次适应算法，通过双树结构优化查找
2. **内存释放**: 自动合并相邻空闲块，减少碎片
3. **状态重建**: 支持从持久化数据恢复分配器状态
4. **线程安全**: 使用自旋锁保护内部状态

**双树设计**:
- `addressTree_`: 按地址索引，便于查找相邻块进行合并
- `sizeTree_`: 按大小优先索引，便于快速找到合适的空闲块
