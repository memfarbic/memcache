# mmc_meta_mgr_proxy.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_mgr_proxy.cpp`
- **文件用途**: MmcMetaMgrProxy 类的实现文件
- **依赖项**: `mmc_meta_mgr_proxy.h`, `mmc_ptracer.h`

---

## 函数实现

### Alloc

```cpp
Result MmcMetaMgrProxy::Alloc(const AllocRequest &req, AllocResponse &resp)
{
    metaMangerPtr_->CheckAndEvict();
    MmcMemMetaDesc objMeta;
    auto ret = metaMangerPtr_->Alloc(req.key_, req.options_, req.operateId_, objMeta);
    if (ret != MMC_OK) {
        if (ret != MMC_DUPLICATED_OBJECT) {
            MMC_RETURN_ERROR(ret, "Meta Alloc Fail, key  " << req.key_);
        } else {
            return ret;
        }
    }
    resp.numBlobs_ = objMeta.numBlobs_;
    resp.prot_ = objMeta.prot_;
    resp.priority_ = objMeta.priority_;
    resp.blobs_ = objMeta.blobs_;
    return MMC_OK;
}
```

**声明位置**: 行 17-34

**功能描述**: 处理分配请求

**代码逻辑**:
1. 触发淘汰检查
2. 调用元管理器分配内存
3. 处理结果（忽略重复对象错误）
4. 填充响应数据

---

### BatchAlloc

```cpp
Result MmcMetaMgrProxy::BatchAlloc(const BatchAllocRequest &req, BatchAllocResponse &resp)
{
    metaMangerPtr_->CheckAndEvict();
    resp.results_.resize(req.keys_.size());
    resp.blobs_.resize(req.keys_.size());
    MMC_ASSERT_RETURN(req.keys_.size() == req.options_.size(), MMC_ERROR);
    for (size_t i = 0; i < req.keys_.size(); ++i) {
        MmcMemMetaDesc objMeta{};
        TP_TRACE_BEGIN(TP_MMC_META_MGR_ALLOC);
        Result ret = metaMangerPtr_->Alloc(req.keys_[i], req.options_[i], req.operateId_, objMeta);
        TP_TRACE_END(TP_MMC_META_MGR_ALLOC, ret);
        if (ret != MMC_OK) {
            if (ret != MMC_DUPLICATED_OBJECT) {
                MMC_LOG_ERROR("Allocation failed for key: " << req.keys_[i] << ", error: " << ret);
            }
            resp.numBlobs_.push_back(0);
            resp.prots_.push_back(0);
            resp.priorities_.push_back(0);
            resp.leases_.push_back(0);
            resp.blobs_[i] = {};
        } else {
            resp.numBlobs_.push_back(objMeta.numBlobs_);
            resp.prots_.push_back(objMeta.prot_);
            resp.priorities_.push_back(objMeta.priority_);
            resp.leases_.push_back(0);
            resp.blobs_[i] = objMeta.blobs_;
        }
        resp.results_[i] = ret;
    }
    return MMC_OK;
}
```

**声明位置**: 行 36-66

**功能描述**: 处理批量分配请求

**代码逻辑**:
1. 触发淘汰检查
2. 调整响应数组大小
3. 验证参数
4. 逐个处理分配请求
5. 使用性能追踪记录耗时
6. 填充响应数据

**部分失败处理**: 即使部分分配失败，也返回成功，但结果数组中包含每个请求的返回码

---

### UpdateState

```cpp
Result MmcMetaMgrProxy::UpdateState(const UpdateRequest &req, Response &resp)
{
    MmcLocation loc{req.rank_, static_cast<MediaType>(req.mediaType_)};
    Result ret = metaMangerPtr_->UpdateState(req.key_, loc, req.actionResult_, req.operateId_);
    resp.ret_ = ret;
    return MMC_OK;
}
```

**声明位置**: 行 68-74

**功能描述**: 处理状态更新请求

**代码逻辑**:
1. 构造位置对象
2. 调用元管理器更新状态
3. 填充响应

---

### BatchUpdateState

```cpp
Result MmcMetaMgrProxy::BatchUpdateState(const BatchUpdateRequest &req, BatchUpdateResponse &resp)
{
    const size_t keyCount = req.keys_.size();
    if (keyCount != req.ranks_.size() || keyCount != req.mediaTypes_.size()) {
        MMC_LOG_ERROR("BatchUpdateState: Input vectors size mismatch {keyNum:"
                      << req.keys_.size() << ", rankNum:" << req.ranks_.size()
                      << ", mediaNum:" << req.mediaTypes_.size() << "}");
        return MMC_ERROR;
    }

    for (size_t i = 0; i < keyCount; ++i) {
        MmcLocation loc{req.ranks_[i], static_cast<MediaType>(req.mediaTypes_[i])};
        Result ret = metaMangerPtr_->UpdateState(req.keys_[i], loc, req.actionResults_[i], req.operateId_);
        if (ret != MMC_OK) {
            MMC_LOG_ERROR("update for key: " << req.keys_[i] << " failed, error: " << ret);
        }
        resp.results_.push_back(ret);
    }
    return MMC_OK;
}
```

**声明位置**: 行 76-95

**功能描述**: 处理批量状态更新请求

**代码逻辑**:
1. 验证参数大小一致
2. 逐个处理更新请求
3. 收集结果

---

### Get

```cpp
Result MmcMetaMgrProxy::Get(const GetRequest &req, AllocResponse &resp)
{
    MmcMemMetaDesc objMeta;
    MmcBlobFilterPtr filterPtr = MmcMakeRef<MmcBlobFilter>(UINT32_MAX, MEDIA_NONE, READABLE);
    MMC_RETURN_ERROR(metaMangerPtr_->Get(req.key_, req.operateId_, filterPtr, objMeta),
                     "failed to get objMeta for key " << req.key_);
    if (objMeta.numBlobs_ == 0 || objMeta.blobs_.empty()) {
        MMC_LOG_ERROR("key " << req.key_ << " already released ");
        return MMC_ERROR;
    }

    resp.blobs_ = objMeta.blobs_;
    resp.numBlobs_ = objMeta.blobs_.size();
    resp.prot_ = objMeta.prot_;
    resp.priority_ = objMeta.priority_;
    return MMC_OK;
}
```

**声明位置**: 行 97-113

**功能描述**: 处理获取请求

**代码逻辑**:
1. 创建可读 Blob 过滤器
2. 调用元管理器获取元数据
3. 验证结果
4. 填充响应

---

### BatchGet

```cpp
Result MmcMetaMgrProxy::BatchGet(const BatchGetRequest &req, BatchAllocResponse &resp)
{
    resp.numBlobs_.resize(req.keys_.size(), 0);
    resp.prots_.resize(req.keys_.size(), 0);
    resp.priorities_.resize(req.keys_.size(), 0);
    resp.leases_.resize(req.keys_.size(), 0);
    resp.blobs_.resize(req.keys_.size());

    MmcBlobFilterPtr filterPtr = MmcMakeRef<MmcBlobFilter>(UINT32_MAX, MEDIA_NONE, READABLE);
    for (size_t i = 0; i < req.keys_.size(); ++i) {
        MmcMemMetaDesc objMeta{};
        TP_TRACE_BEGIN(TP_MMC_META_MGR_GET);
        auto ret = metaMangerPtr_->Get(req.keys_[i], req.operateId_, filterPtr, objMeta);
        TP_TRACE_END(TP_MMC_META_MGR_GET, ret);
        if (ret != MMC_OK || objMeta.blobs_.empty() || objMeta.numBlobs_ != objMeta.blobs_.size()) {
            resp.numBlobs_[i] = 0;
            resp.blobs_[i] = {};
            resp.prots_[i] = 0;
            resp.priorities_[i] = 0;
            MMC_LOG_ERROR("Key " << req.keys_[i] << " not found");
        } else {
            resp.numBlobs_[i] = objMeta.numBlobs_;
            resp.blobs_[i] = objMeta.blobs_;
            resp.prots_[i] = objMeta.prot_;
            resp.priorities_[i] = objMeta.priority_;
        }
        resp.results_.push_back(ret);
    }
    return MMC_OK;
}
```

**声明位置**: 行 115-144

**功能描述**: 处理批量获取请求

**代码逻辑**:
1. 调整响应数组大小
2. 创建可读 Blob 过滤器
3. 逐个处理获取请求
4. 使用性能追踪记录耗时
5. 填充响应数据

---

### BatchExistKey

```cpp
Result MmcMetaMgrProxy::BatchExistKey(const BatchIsExistRequest &req, BatchIsExistResponse &resp)
{
    resp.results_.reserve(req.keys_.size());
    for (size_t i = 0; i < req.keys_.size(); ++i) {
        auto ret = metaMangerPtr_->ExistKey(req.keys_[i]);
        if (ret != MMC_OK && ret != MMC_UNMATCHED_KEY) {
            MMC_LOG_ERROR("get key: " << req.keys_[i] << " unexpected result: " << ret);
        }
        resp.results_.emplace_back(ret);
    }
    return MMC_OK;
}
```

**声明位置**: 行 146-157

**功能描述**: 批量检查键是否存在

**代码逻辑**:
1. 预留结果数组空间
2. 逐个检查键是否存在
3. 收集结果

---

## 总结

此文件实现了元管理器代理的核心功能：

1. **请求转换**: 将网络请求转换为元管理器调用
2. **批量操作**: 支持批量分配、获取、更新、检查等操作
3. **部分失败处理**: 批量操作中部分失败不影响整体返回
4. **性能追踪**: 使用 ptracer 记录关键操作的耗时
5. **淘汰触发**: 在分配操作前触发淘汰检查

**代理模式**:
- 隔离网络层和业务逻辑层
- 统一处理请求和响应
- 提供批量操作优化
