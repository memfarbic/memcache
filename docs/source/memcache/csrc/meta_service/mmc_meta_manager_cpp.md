# mmc_meta_manager.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_manager.cpp`
- **文件用途**: MmcMetaManager 类的实现文件
- **依赖项**: `mmc_meta_manager.h`, `mmc_logger.h`, `mmc_meta_metric_manager.h`

---

## 常量定义

```cpp
constexpr int TIMEOUT_SECOND = 60;
```

**声明位置**: 行 22

**说明**: 操作超时时间（秒）

---

## 函数实现

### Get

```cpp
Result MmcMetaManager::Get(const std::string &key, uint64_t operateId, MmcBlobFilterPtr filterPtr,
                           MmcMemMetaDesc &objMeta)
{
    MmcMemObjMetaPtr memObj;
    auto ret = metaContainer_->Get(key, memObj);
    if (ret != MMC_OK) {
        MMC_LOG_ERROR("Get key: " << key << " failed. ErrCode: " << ret);
        return ret;
    }

    metaContainer_->Promote(key);

    std::unique_lock<std::mutex> guard(memObj->Mutex());
    objMeta.prot_ = memObj->Prot();
    objMeta.priority_ = memObj->Priority();
    objMeta.size_ = memObj->Size();

    std::vector<MmcMemBlobPtr> blobs = memObj->GetBlobs(filterPtr);
    for (auto blob : blobs) {
        uint32_t opRankId = GetRankIdByOperateId(operateId);
        uint32_t opSeq = GetSequenceByOperateId(operateId);
        auto ret = blob->UpdateState(key, opRankId, opSeq, MMC_READ_START);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("update key " << key << " blob state failed with error: " << ret);
            continue;
        }
        objMeta.blobs_.push_back(blob->GetDesc());
        break; // 只需返回一个
    }
    objMeta.numBlobs_ = objMeta.blobs_.size();

    MmcMetaMetricManager::GetInstance().IncrementGetCounter();
    return MMC_OK;
}
```

**声明位置**: 行 24-57

**功能描述**: 获取元数据并更新 LRU 位置

**代码逻辑**:
1. 从容器中获取元数据对象
2. 更新 LRU 位置
3. 获取元数据基本信息
4. 根据过滤器获取 Blob
5. 更新 Blob 状态为读取中
6. 更新指标计数器

---

### ExistKey

```cpp
Result MmcMetaManager::ExistKey(const std::string &key)
{
    MmcMemObjMetaPtr memObj;
    auto ret = metaContainer_->Get(key, memObj);
    if (ret != MMC_OK) {
        if (ret == MMC_UNMATCHED_KEY) {
            MMC_LOG_DEBUG("Not exist, key:" << key << " ret:" << ret);
        } else {
            MMC_LOG_ERROR("Failed to get key:" << key << " ret:" << ret);
        }
        return ret;
    }
    {
        std::unique_lock<std::mutex> guard(memObj->Mutex());
        MmcBlobFilterPtr filterPtr = MmcMakeRef<MmcBlobFilter>(UINT32_MAX, MEDIA_NONE, READABLE);
        if (filterPtr == nullptr) {
            MMC_LOG_ERROR("Failed to alloc filter");
            return MMC_MALLOC_FAILED;
        }
        std::vector<MmcMemBlobPtr> blobs = memObj->GetBlobs(filterPtr);
        if (blobs.empty()) {
            MMC_LOG_ERROR("Key is exist but do not have readable blob key:" << key);
            return MMC_OBJECT_NOT_EXISTS;
        }
    }
    metaContainer_->Promote(key);
    return MMC_OK;
}
```

**声明位置**: 行 59-86

**功能描述**: 检查键是否存在且可读

**代码逻辑**:
1. 尝试获取元数据对象
2. 检查是否有可读的 Blob
3. 更新 LRU 位置

---

### CheckAndEvict

```cpp
void MmcMetaManager::CheckAndEvict()
{
    std::vector<uint16_t> nowMemoryThresholds;
    const auto needEvictList = globalAllocator_->GetNeedEvictList(evictThresholdHigh_, nowMemoryThresholds);
    if (needEvictList.empty()) {
        return;
    }
    bool expected = false;
    if (!evictCheck_.compare_exchange_strong(expected, true)) {
        return;
    }
    auto moveFunc = [this](const std::string &key, const MmcMemObjMetaPtr &objMeta) -> EvictResult {
        return this->EvictCallBackFunction(key, objMeta);
    };

    auto evictFuture = threadPool_->Enqueue(
        [&](const std::vector<MediaType> &needEvictListL, const std::vector<uint16_t> &nowMemoryThresholds,
            const std::function<EvictResult(const std::string &key, const MmcMemObjMetaPtr &objMeta)> &moveFuncL) {
            metaContainer_->MultiLevelElimination(evictThresholdHigh_, evictThresholdLow_, needEvictListL,
                                                  nowMemoryThresholds, moveFuncL);
            bool expected = true;
            evictCheck_.compare_exchange_strong(expected, false);
        },
        needEvictList, nowMemoryThresholds, moveFunc);
    if (!evictFuture.valid()) {
        MMC_LOG_ERROR("submit evict task failed");
    }
}
```

**声明位置**: 行 88-115

**功能描述**: 检查内存使用情况并执行淘汰

**代码逻辑**:
1. 获取需要淘汰的介质类型列表
2. 如果没有需要淘汰的则直接返回
3. 使用原子操作确保只有一个淘汰任务在运行
4. 创建淘汰回调函数
5. 将淘汰任务提交到线程池
6. 任务完成后清除淘汰检查标志

---

### Alloc

```cpp
Result MmcMetaManager::Alloc(const std::string &key, const AllocOptions &allocOpt, uint64_t operateId,
                             MmcMemMetaDesc &objMeta)
{
    MmcMemObjMetaPtr tempMetaObj = MmcMakeRef<MmcMemObjMeta>();
    if (tempMetaObj == nullptr) {
        MMC_LOG_ERROR("Fail to malloc tempMetaObj");
        return MMC_MALLOC_FAILED;
    }
    std::vector<MmcMemBlobPtr> blobs;

    Result ret = globalAllocator_->Alloc(allocOpt, blobs);
    if (ret != MMC_OK) {
        if (!blobs.empty()) {
            for (auto &blob : blobs) {
                globalAllocator_->Free(blob);
            }
            blobs.clear();
        }
        MMC_LOG_ERROR("Alloc " << allocOpt.blobSize_ << " failed, ret:" << ret);
        return ret;
    }

    uint32_t opRankId = GetRankIdByOperateId(operateId);
    uint32_t opSeq = GetSequenceByOperateId(operateId);
    for (auto &blob : blobs) {
        MMC_LOG_DEBUG("Blob allocated, key=" << key << ", size=" << blob->Size() << ", rank=" << blob->Rank());
        blob->UpdateState(key, opRankId, opSeq, MMC_ALLOCATED_OK);
        tempMetaObj->AddBlob(blob);
    }

    ret = metaContainer_->Insert(key, tempMetaObj);
    if (ret != MMC_OK) {
        tempMetaObj->FreeBlobs(key, globalAllocator_, nullptr, false);
        if (ret != MMC_DUPLICATED_OBJECT) {
            MMC_LOG_ERROR("Fail to insert " << key << " into MmcMetaContainer. ret:" << ret);
        }
    } else {
        std::unique_lock<std::mutex> guard(tempMetaObj->Mutex());
        objMeta.prot_ = tempMetaObj->Prot();
        objMeta.priority_ = tempMetaObj->Priority();
        objMeta.size_ = tempMetaObj->Size();
        tempMetaObj->GetBlobsDesc(objMeta.blobs_);
        objMeta.numBlobs_ = objMeta.blobs_.size();

        MmcMetaMetricManager::GetInstance().IncrementAllocCounter();
    }
    return ret;
}
```

**声明位置**: 行 117-164

**功能描述**: 分配内存并创建元数据对象

**代码逻辑**:
1. 创建元数据对象
2. 调用全局分配器分配 Blob
3. 更新 Blob 状态
4. 将 Blob 添加到元数据对象
5. 将元数据对象插入容器
6. 如果失败则释放已分配的 Blob
7. 更新指标计数器

---

### UpdateState

```cpp
Result MmcMetaManager::UpdateState(const std::string &key, const MmcLocation &loc, const BlobActionResult &actRet,
                                   uint64_t operateId)
{
    MmcMemObjMetaPtr metaObj;
    // when update state, do not update the lru
    Result ret = metaContainer_->Get(key, metaObj);
    if (ret != MMC_OK || metaObj == nullptr) {
        MMC_LOG_ERROR("UpdateState: Cannot find " << key << " memObjMeta! ret:" << ret
                                                  << ", action:" << static_cast<uint32_t>(actRet));
        return MMC_UNMATCHED_KEY;
    }
    MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(loc.rank_, loc.mediaType_, NONE);
    {
        std::unique_lock<std::mutex> guard(metaObj->Mutex());
        ret = metaObj->UpdateBlobsState(key, filter, operateId, actRet);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("UpdateState: Failed to update blob state, ret: " << ret);
            return ret;
        }
    }
    if (actRet == MMC_WRITE_FAIL) {
        ret = Remove(key);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("UpdateState: Failed remove key " << key << ", ret: " << ret);
            return ret;
        }
    }
    return MMC_OK;
}
```

**声明位置**: 行 166-194

**功能描述**: 更新 Blob 状态

**代码逻辑**:
1. 获取元数据对象
2. 创建过滤器
3. 更新 Blob 状态
4. 如果写入失败则移除元数据对象

---

### PushRemoveList

```cpp
void MmcMetaManager::PushRemoveList(const std::string &key, const MmcMemObjMetaPtr &meta)
{
    auto future = threadPool_->Enqueue(
        [&](const std::string keyL, const MmcMemObjMetaPtr metaL, MmcGlobalAllocatorPtr allocator) {
            std::unique_lock<std::mutex> guard(metaL->Mutex());
            return metaL->FreeBlobs(keyL, allocator);
        },
        key, meta, globalAllocator_);
    if (!future.valid()) {
        // already locked when call, no need lock again
        meta->FreeBlobs(key, globalAllocator_);
    }
}
```

**声明位置**: 行 196-208

**功能描述**: 异步释放 Blob 内存

**代码逻辑**:
1. 提交释放任务到线程池
2. 如果提交失败则同步释放

---

### Remove

```cpp
Result MmcMetaManager::Remove(const std::string &key)
{
    MmcMemObjMetaPtr objMeta;
    MMC_RETURN_ERROR(metaContainer_->Erase(key, objMeta), "remove: Fail to erase from container!");
    std::unique_lock<std::mutex> guard(objMeta->Mutex());
    PushRemoveList(key, objMeta);

    MmcMetaMetricManager::GetInstance().IncrementRemoveCounter();
    return MMC_OK;
}
```

**声明位置**: 行 210-219

**功能描述**: 移除元数据对象

**代码逻辑**:
1. 从容器中删除元数据对象
2. 异步释放 Blob 内存
3. 更新指标计数器

---

### RemoveAll

```cpp
Result MmcMetaManager::RemoveAll()
{
    auto removeFunc = [this](const std::string &key, const MmcMemObjMetaPtr &objMeta) -> void {
        std::unique_lock<std::mutex> guard(objMeta->Mutex());
        this->PushRemoveList(key, objMeta);
    };

    MMC_RETURN_ERROR(metaContainer_->EraseAll(removeFunc), "RemoveAll: Fail to erase all from container!");

    MMC_LOG_INFO("All keys removed");
    return MMC_OK;
}
```

**声明位置**: 行 221-232

**功能描述**: 移除所有元数据对象

**代码逻辑**:
1. 创建移除回调函数
2. 调用容器的 EraseAll 方法

---

### Mount (single)

```cpp
Result MmcMetaManager::Mount(const MmcLocation &loc, const MmcLocalMemlInitInfo &localMemInitInfo,
                             std::map<std::string, MmcMemBlobDesc> &blobMap)
{
    Result ret = globalAllocator_->Mount(loc, localMemInitInfo);
    if (ret != MMC_OK) {
        MMC_LOG_ERROR("allocator mount failed, loc rank: " << loc.rank_ << " mediaType_: " << loc.mediaType_);
        return ret;
    }
    if (blobMap.empty()) {
        return globalAllocator_->Start(loc);
    }
    ret = globalAllocator_->BuildFromBlobs(loc, blobMap);
    if (ret != MMC_OK) {
        MMC_LOG_ERROR("build from blobs failed, loc rank: " << loc.rank_ << " mediaType_: " << loc.mediaType_);
        return ret;
    }

    if (!blobMap.empty()) {
        ret = RebuildMeta(blobMap);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("rebuild meta failed, loc rank: " << loc.rank_ << " mediaType_: " << loc.mediaType_);
            return ret;
        }
    }
    return MMC_OK;
}
```

**声明位置**: 行 234-259

**功能描述**: 挂载单个内存池

**代码逻辑**:
1. 挂载到全局分配器
2. 如果有 Blob 映射表则重建分配器状态
3. 启动分配器
4. 重建元数据

---

### Mount (vector)

```cpp
Result MmcMetaManager::Mount(const std::vector<MmcLocation> &locs,
                             const std::vector<MmcLocalMemlInitInfo> &localMemInitInfos,
                             std::map<std::string, MmcMemBlobDesc> &blobMap)
{
    if (locs.size() != localMemInitInfos.size()) {
        MMC_LOG_ERROR("Mount: loc size:" << locs.size() << " != localMemInitInfo size:" << localMemInitInfos.size());
        return MMC_INVALID_PARAM;
    }
    Result ret = MMC_OK;
    uint32_t i = 0;
    for (; i < locs.size(); i++) {
        ret = Mount(locs[i], localMemInitInfos[i], blobMap);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("Mount failed ret:" << ret << " loc rank:" << locs[i].rank_
                                              << " mediaType_: " << locs[i].mediaType_);
            break;
        }
    }
    if (ret != MMC_OK) {
        MMC_LOG_INFO("Mount locs partially failed, unmounting mounted locs...");
        for (; i > 0; i--) {
            auto unmountRet = Unmount(locs[i - 1]);
            if (unmountRet != MMC_OK) {
                MMC_LOG_ERROR("Unmount failed ret:" << unmountRet << " loc rank:" << locs[i - 1].rank_
                                                    << " mediaType_: " << locs[i - 1].mediaType_);
            }
        }
    }
    return MMC_OK;
}
```

**声明位置**: 行 261-290

**功能描述**: 批量挂载内存池（带事务性）

**代码逻辑**:
1. 验证参数
2. 逐个挂载
3. 如果失败则回滚已挂载的

---

### RebuildMeta

```cpp
Result MmcMetaManager::RebuildMeta(std::map<std::string, MmcMemBlobDesc> &blobMap)
{
    Result ret;
    for (auto &blob : blobMap) {
        std::string key = blob.first;
        MmcMemBlobDesc desc = blob.second;
        MmcMemBlobPtr blobPtr = MmcMakeRef<MmcMemBlob>(desc.rank_, desc.gva_, desc.size_,
                                                       static_cast<MediaType>(desc.mediaType_), READABLE);
        MmcMemObjMetaPtr objMeta;

        if (metaContainer_->Get(key, objMeta) == MMC_OK) {
            std::unique_lock<std::mutex> guard(objMeta->Mutex());
            if (objMeta->AddBlob(blobPtr) != MMC_OK) {
                globalAllocator_->Free(blobPtr);
            }
            continue;
        }

        objMeta = MmcMakeRef<MmcMemObjMeta>();
        if (objMeta != nullptr) {
            objMeta->AddBlob(blobPtr);
            ret = metaContainer_->Insert(key, objMeta);
            if (ret == MMC_OK) {
                continue;
            }
        }

        if (metaContainer_->Get(key, objMeta) == MMC_OK) {
            std::unique_lock<std::mutex> guard(objMeta->Mutex());
            if (objMeta->AddBlob(blobPtr) != MMC_OK) {
                globalAllocator_->Free(blobPtr);
            }
        } else {
            globalAllocator_->Free(blobPtr);
        }
    }
    return MMC_OK;
}
```

**声明位置**: 行 292-329

**功能描述**: 从 Blob 映射表重建元数据

**代码逻辑**:
1. 遍历 Blob 映射表
2. 为每个 Blob 创建 Blob 对象
3. 尝试添加到现有元数据对象
4. 如果不存在则创建新的元数据对象
5. 处理失败情况

---

### Unmount

```cpp
Result MmcMetaManager::Unmount(const MmcLocation &loc)
{
    Result ret = globalAllocator_->Stop(loc);
    if (ret != MMC_OK) {
        return ret;
    }
    // Force delete the blobs
    MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(loc.rank_, loc.mediaType_, NONE);

    auto matchFunc = [this, &filter](const std::string &key, const MmcMemObjMetaPtr &objMeta) -> bool {
        if (objMeta == nullptr) {
            MMC_LOG_ERROR("objMeta is null");
            return false;
        }
        std::unique_lock<std::mutex> guard(objMeta->Mutex());
        auto ret = objMeta->FreeBlobs(key, globalAllocator_, filter, false);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("Fail to force remove key:" << key << " blobs in when unmount! ret:" << ret);
            return false;
        }
        if (objMeta->NumBlobs() == 0) {
            return true;
        }
        return false;
    };

    metaContainer_->EraseIf(matchFunc);

    ret = globalAllocator_->Unmount(loc);
    return ret;
}
```

**声明位置**: 行 331-361

**功能描述**: 卸载内存池

**代码逻辑**:
1. 停止分配器
2. 创建过滤器
3. 删除属于该位置的所有 Blob
4. 如果元数据对象没有 Blob 了则删除
5. 卸载分配器

---

### GetAllSegmentInfo

```cpp
nlohmann::json MmcMetaManager::GetAllSegmentInfo() const
{
    return globalAllocator_->GetAllSegmentInfo();
}
```

**声明位置**: 行 363-366

**功能描述**: 获取所有分段信息

---

### Query

```cpp
Result MmcMetaManager::Query(const std::string &key, MemObjQueryInfo &queryInfo)
{
    MmcMemObjMetaPtr objMeta;
    if (metaContainer_->Get(key, objMeta) != MMC_OK) {
        MMC_LOG_WARN("Cannot find MmcMemObjMeta with key : " << key);
        return MMC_UNMATCHED_KEY;
    }
    std::unique_lock<std::mutex> guard(objMeta->Mutex());
    std::vector<MmcMemBlobDesc> blobs;
    objMeta->GetBlobsDesc(blobs);
    uint32_t i = 0;
    for (auto blob : blobs) {
        if (i >= MAX_BLOB_COPIES) {
            break;
        }
        queryInfo.blobRanks_[i] = blob.rank_;
        queryInfo.blobTypes_[i] = blob.mediaType_;
        i++;
    }
    queryInfo.numBlobs_ = i;
    queryInfo.size_ = objMeta->Size();
    queryInfo.prot_ = objMeta->Prot();
    queryInfo.valid_ = true;
    return MMC_OK;
}
```

**声明位置**: 行 368-392

**功能描述**: 查询键的信息

---

### GetAllKeys

```cpp
Result MmcMetaManager::GetAllKeys(std::vector<std::string> &keys)
{
    MMC_VALIDATE_RETURN(metaContainer_ != nullptr, "meta container not initialized! ", MMC_NOT_INITIALIZED);

    metaContainer_->GetAllKeys(keys);
    return MMC_OK;
}
```

**声明位置**: 行 394-400

**功能描述**: 获取所有键

---

### CopyBlob

```cpp
Result MmcMetaManager::CopyBlob(const MmcMemObjMetaPtr &objMeta, const MmcMemBlobDesc &srcBlob,
                                const MmcLocation &dstLoc)
{
    if (objMeta == nullptr) {
        MMC_LOG_ERROR("objMeta is null");
        return MMC_INVALID_PARAM;
    }
    AllocOptions allocOpt{};
    allocOpt.blobSize_ = srcBlob.size_;
    allocOpt.numBlobs_ = 1;
    allocOpt.mediaType_ = dstLoc.mediaType_;
    allocOpt.preferredRank_.clear();
    allocOpt.preferredRank_.push_back(dstLoc.rank_);
    allocOpt.flags_ = dstLoc.rank_ == UINT32_MAX ? 0 : ALLOC_FORCE_BY_RANK;

    std::vector<MmcMemBlobPtr> blobs;
    Result ret = MMC_OK;
    do {
        ret = globalAllocator_->Alloc(allocOpt, blobs);
        if (ret != MMC_OK || blobs.empty()) {
            MMC_LOG_ERROR("alloc failed, ret " << ret);
            ret = MMC_MALLOC_FAILED;
            break;
        }

        BlobCopyRequest request{srcBlob, blobs[0]->GetDesc()};
        Response response;
        // rpc 到目标节点复制
        ret = metaNetServer_->SyncCall(request.dstBlob_.rank_, request, response, TIMEOUT_SECOND);
        if (ret != MMC_OK || response.ret_ != MMC_OK) {
            MMC_LOG_ERROR("copy blob from rank " << request.srcBlob_.rank_ << " to rank " << request.dstBlob_.rank_
                                                 << " failed:" << ret << "," << response.ret_);
            ret = MMC_ERROR;
            break;
        }

        ret = blobs[0]->UpdateState(MMC_WRITE_OK);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("Failed to Update blob state, ret: " << ret);
            break;
        }
        // 挂载
        ret = objMeta->AddBlob(blobs[0]);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("AddBlob failed, ret " << ret);
            break;
        }
    } while (0);

    if (ret != MMC_OK && !blobs.empty()) {
        for (auto &blob : blobs) {
            globalAllocator_->Free(blob);
        }
    }
    return ret;
}
```

**声明位置**: 行 402-457

**功能描述**: 复制 Blob 到目标位置

**代码逻辑**:
1. 验证参数
2. 分配目标 Blob
3. 通过 RPC 复制数据
4. 更新 Blob 状态
5. 挂载到元数据对象
6. 失败则释放分配的 Blob

---

### MoveBlob

```cpp
Result MmcMetaManager::MoveBlob(const std::string &key, const MmcLocation &src, const MmcLocation &dst)
{
    MmcMemObjMetaPtr objMeta;
    if (metaContainer_->Get(key, objMeta) != MMC_OK) {
        MMC_LOG_ERROR("Cannot find MmcMemObjMeta with key : " << key);
        return MMC_UNMATCHED_KEY;
    }
    {
        std::unique_lock<std::mutex> guard(objMeta->Mutex());
        std::vector<MmcMemBlobDesc> blobsDesc;
        MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(src.rank_, src.mediaType_, READABLE);
        if (filter == nullptr) {
            MMC_LOG_ERROR("Fail to malloc filter");
            return MMC_MALLOC_FAILED;
        }
        objMeta->GetBlobsDesc(blobsDesc, filter);
        if (blobsDesc.empty()) {
            MMC_LOG_ERROR("blob for " << src << " to " << dst << " is empty with key : " << key << "," << objMeta);
            return MMC_UNMATCHED_KEY;
        }

        auto ret = CopyBlob(objMeta, blobsDesc[0], dst);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("key: " << key << " copy blob failed, ret " << ret);
            return ret;
        }

        filter = MmcMakeRef<MmcBlobFilter>(src.rank_, src.mediaType_, NONE);
        if (filter == nullptr) {
            MMC_LOG_ERROR("Fail to malloc filter");
            return MMC_MALLOC_FAILED;
        }
        ret = objMeta->FreeBlobs(key, globalAllocator_, filter);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("key: " << key << " free blob failed, ret " << ret);
            return ret;
        }
        MMC_LOG_INFO("move " << key << " from " << src << " to " << dst << " " << blobsDesc[0] << ", " << objMeta);
    }
    // insert to lru with metaItemMtx
    metaContainer_->InsertLru(key, dst.mediaType_);
    return MMC_OK;
}
```

**声明位置**: 行 459-501

**功能描述**: 移动 Blob 从源位置到目标位置

**代码逻辑**:
1. 获取元数据对象
2. 获取源 Blob
3. 复制到目标位置
4. 释放源 Blob
5. 更新 LRU 位置

---

### ReplicateBlob

```cpp
Result MmcMetaManager::ReplicateBlob(const std::string &key, const MmcLocation &loc)
{
    MmcMemObjMetaPtr objMeta;
    if (metaContainer_->Get(key, objMeta) != MMC_OK || objMeta.Get() == nullptr) {
        MMC_LOG_ERROR("Cannot find MmcMemObjMeta with key : " << key);
        return MMC_UNMATCHED_KEY;
    }
    std::unique_lock<std::mutex> guard(objMeta->Mutex());
    std::vector<MmcMemBlobDesc> blobsDesc;
    MmcBlobFilterPtr filter = MmcMakeRef<MmcBlobFilter>(UINT32_MAX, MEDIA_NONE, READABLE);
    objMeta->GetBlobsDesc(blobsDesc, filter);
    if (blobsDesc.empty()) {
        MMC_LOG_ERROR("blob is empty with key : " << key);
        return MMC_UNMATCHED_KEY;
    }

    return CopyBlob(objMeta, blobsDesc[0], loc);
}
```

**声明位置**: 行 503-520

**功能描述**: 复制 Blob 到指定位置

---

### EvictCallBackFunction

```cpp
EvictResult MmcMetaManager::EvictCallBackFunction(const std::string &key, const MmcMemObjMetaPtr &objMeta)
{
    std::unique_lock<std::mutex> guard(objMeta->Mutex());

    if (objMeta == nullptr) {
        MMC_LOG_ERROR("objMeta is null");
        return EvictResult::FAIL;
    }
    MediaType dstMedium = objMeta->MoveTo(true);
    MediaType srcMedium = MoveUp(dstMedium);
    if (dstMedium == MEDIA_NONE || srcMedium == MEDIA_NONE) {
        PushRemoveList(key, objMeta);
        return EvictResult::REMOVE; // 向下淘汰已无可能，直接删除
    }
    MmcLocation src{UINT32_MAX, srcMedium};
    MmcLocation dst{UINT32_MAX, dstMedium};

    uint64_t freeSize = globalAllocator_->GetFreeSpace(dstMedium);
    if (freeSize < objMeta->Size()) {
        MMC_LOG_WARN("key: " << key << " move to " << dst << " no space:" << freeSize << ", need:" << objMeta->Size());
        PushRemoveList(key, objMeta);
        return EvictResult::REMOVE; // 向下淘汰已无可能，直接删除
    }

    auto future = threadPool_->Enqueue(
        [&](const std::string keyL, const MmcLocation srcL, const MmcLocation dstL) {
            auto ret = MoveBlob(keyL, srcL, dstL);
            if (ret != MMC_OK) {
                Remove(keyL);
                MMC_LOG_WARN("key: " << keyL << " move blob from " << srcL << " to " << dstL << " failed: " << ret
                                     << ", remove it");
            }
            return ret;
        },
        key, src, dst);
    if (!future.valid()) {
        MMC_LOG_WARN("key: " << key << " move blob from " << src << " to " << dst << " failed");
        PushRemoveList(key, objMeta);
        return EvictResult::REMOVE; // 向下淘汰失败，直接删除
    }
    return EvictResult::MOVE_DOWN;
}
```

**声明位置**: 行 522-563

**功能描述**: 淘汰回调函数，尝试向下移动或删除

**代码逻辑**:
1. 获取目标介质类型
2. 如果无法向下移动则删除
3. 检查目标介质是否有足够空间
4. 提交异步移动任务
5. 返回操作结果

---

## 总结

此文件实现了元数据管理器的核心功能：

1. **元数据 CRUD**: 创建、读取、更新、删除元数据对象
2. **内存管理**: 分配、释放 Blob 内存
3. **淘汰策略**: 多层级内存淘汰，向下移动或删除
4. **异步处理**: 使用线程池异步释放内存和执行淘汰
5. **指标统计**: 记录分配、获取、删除等操作计数
