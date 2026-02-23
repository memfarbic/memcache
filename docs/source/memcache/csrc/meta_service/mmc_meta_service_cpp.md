# mmc_meta_service.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_service.cpp`
- **文件用途**: MmcMetaService 类的实现文件
- **依赖项**: `mmc_meta_service.h`, `mmc_ref.h`, �mmc_smem_bm_helper.h`, `smem_store_factory.h`

---

## 函数实现

### Start

```cpp
Result MmcMetaService::Start(const mmc_meta_service_config_t &options)
{
    const int threadCountBase = 4;
    std::lock_guard<std::mutex> guard(mutex_);
    if (started_) {
        MMC_LOG_INFO("MetaService " << name_ << " already started");
        return MMC_OK;
    }
    options_ = options;
    MMC_VALIDATE_RETURN(options.evictThresholdHigh > options.evictThresholdLow,
                        "invalid param, evictThresholdHigh must large than evictThresholdLow", MMC_INVALID_PARAM);

    metaNetServer_ = MmcMakeRef<MetaNetServer>(this, name_ + "_MetaServer").Get();
    MMC_ASSERT_RETURN(metaNetServer_.Get() != nullptr, MMC_NEW_OBJECT_FAILED);
    /* init engine */
    NetEngineOptions netOptions;
    std::string url{options_.discoveryURL};
    NetEngineOptions::ExtractIpPortFromUrl(url, netOptions);
    netOptions.name = name_;
    netOptions.threadCount = threadCountBase;
    netOptions.rankId = 0;
    netOptions.startListener = true;
    netOptions.tlsOption = options_.accTlsConfig;
    netOptions.logFunc = SPDLOG_LogMessage;
    netOptions.logLevel = options_.logLevel;
    MMC_RETURN_ERROR(metaNetServer_->Start(netOptions), "Failed to start net server of meta service " << name_);

    metaBackUpMgrPtr_ = MMCMetaBackUpMgrFactory::GetInstance("DefaultMetaBackup");
    MMCMetaBackUpConfPtr defaultPtr = MmcMakeRef<MMCMetaBackUpConfDefault>(metaNetServer_).Get();
    MMC_ASSERT_RETURN(metaBackUpMgrPtr_ != nullptr, MMC_MALLOC_FAILED);
    if (options.haEnable) {
        MMC_RETURN_ERROR(metaBackUpMgrPtr_->Start(defaultPtr), "metaBackUpMgr start failed");
    }

    metaMgrProxy_ = MmcMakeRef<MmcMetaMgrProxy>(metaNetServer_).Get();
    MMC_RETURN_ERROR(metaMgrProxy_->Start(MMC_DATA_TTL_MS, options.evictThresholdHigh, options.evictThresholdLow),
                     "Failed to start meta mgr proxy of meta service " << name_);

    NetEngineOptions configStoreOpt{};
    NetEngineOptions::ExtractIpPortFromUrl(options_.configStoreURL, configStoreOpt);
    smem::StoreFactory::SetTlsInfo(MmcSmemBmHelper::TransSmemTlsConfig(options_.configStoreTlsConfig));
    confStore_ = smem::StoreFactory::CreateStoreServer(configStoreOpt.ip, configStoreOpt.port,
                                                       std::numeric_limits<uint32_t>::max());
    MMC_VALIDATE_RETURN(confStore_ != nullptr, "Failed to start config store server", MMC_ERROR);

    started_ = true;
    MMC_LOG_INFO("Started MetaService (" << name_ << ") at " << options_.discoveryURL);
    return MMC_OK;
}
```

**声明位置**: 行 24-72

**功能描述**: 启动元服务

**代码逻辑**:
1. 获取互斥锁，检查是否已启动
2. 保存配置选项
3. 验证淘汰阈值参数
4. 创建并启动网络服务器
5. 创建备份管理器（如果启用了 HA）
6. 创建并启动元管理器代理
7. 创建配置存储服务器
8. 设置启动标志

---

### BmRegister

```cpp
Result MmcMetaService::BmRegister(uint32_t rank, std::vector<uint16_t> mediaType, std::vector<uint64_t> bm,
                                  std::vector<uint64_t> capacity, std::map<std::string, MmcMemBlobDesc> &blobMap)
{
    std::lock_guard<std::mutex> guard(mutex_);
    if (!started_) {
        MMC_LOG_ERROR("MetaService (" << name_ << ") is not started");
        return MMC_NOT_STARTED;
    }

    if (mediaType.size() != bm.size() || bm.size() != capacity.size()) {
        MMC_LOG_ERROR("size invalid, media size:" << mediaType.size() << ", bm size:" << bm.size()
                                                  << ", capacity size:" << capacity.size());
        return MMC_INVALID_PARAM;
    }

    std::vector<MmcLocation> locs;
    std::vector<MmcLocalMemlInitInfo> infos;
    size_t typeNum = mediaType.size();
    for (size_t i = 0; i < typeNum; i++) {
        locs.emplace_back(rank, static_cast<MediaType>(mediaType[i]));
        MmcLocalMemlInitInfo locInfo{bm[i], capacity[i]};
        infos.emplace_back(locInfo);
    }
    MMC_ASSERT_RETURN(metaBackUpMgrPtr_ != nullptr, MMC_MALLOC_FAILED);
    MMC_ASSERT_RETURN(metaMgrProxy_ != nullptr, MMC_MALLOC_FAILED);
    MMC_RETURN_ERROR(metaBackUpMgrPtr_->Load(blobMap), "Mount loc { " << rank << " } load backup failed");
    MMC_RETURN_ERROR(metaMgrProxy_->Mount(locs, infos, blobMap), "Mount loc { " << rank << " } failed");
    MMC_LOG_INFO("Mount loc {rank:" << rank << ", rebuild size:" << blobMap.size() << ", mediaNum:" << typeNum
                                    << "} finish");
    if (blobMap.size() == 0) {
        if (rankMediaTypeMap_.find(rank) == rankMediaTypeMap_.end()) {
            rankMediaTypeMap_.insert({rank, {}});
        }

        for (size_t i = 0; i < typeNum; i++) {
            rankMediaTypeMap_[rank].insert(mediaType[i]);
        }
    }
    return MMC_OK;
}
```

**声明位置**: 行 74-113

**功能描述**: 注册 Blob Manager

**代码逻辑**:
1. 获取互斥锁，检查服务状态
2. 验证参数大小一致
3. 构造位置和初始化信息列表
4. 加载备份数据
5. 挂载到元管理器
6. 更新 Rank 介质类型映射

---

### BmUnregister

```cpp
Result MmcMetaService::BmUnregister(uint32_t rank, uint16_t mediaType)
{
    std::lock_guard<std::mutex> guard(mutex_);
    if (!started_) {
        MMC_LOG_ERROR("MetaService (" << name_ << ") is not started");
        return MMC_NOT_STARTED;
    }

    MmcLocation loc{rank, static_cast<MediaType>(mediaType)};
    MMC_RETURN_ERROR(metaMgrProxy_->Unmount(loc), "Unmount loc { " << rank << ", " << mediaType << " } failed");
    MMC_LOG_DEBUG("Unmount loc: " << loc << " finish");
    if (rankMediaTypeMap_.find(rank) != rankMediaTypeMap_.end() &&
        rankMediaTypeMap_[rank].find(mediaType) != rankMediaTypeMap_[rank].end()) {
        rankMediaTypeMap_[rank].erase(mediaType);
    }
    if (rankMediaTypeMap_.find(rank) != rankMediaTypeMap_.end() && rankMediaTypeMap_[rank].empty()) {
        rankMediaTypeMap_.erase(rank);
    }
    return MMC_OK;
}
```

**声明位置**: 行 115-134

**功能描述**: 注销 Blob Manager

**代码逻辑**:
1. 获取互斥锁，检查服务状态
2. 卸载内存池
3. 更新 Rank 介质类型映射

---

### ClearResource

```cpp
Result MmcMetaService::ClearResource(uint32_t rank)
{
    if (!started_) {
        MMC_LOG_ERROR("MetaService (" << name_ << ") is not started.");
        return MMC_NOT_STARTED;
    }
    std::unordered_set<uint16_t> mediaTypes;
    {
        std::lock_guard<std::mutex> guard(mutex_);
        if (rankMediaTypeMap_.find(rank) == rankMediaTypeMap_.end()) {
            MMC_LOG_DEBUG("Rank " << rank << " has no resources.");
            return MMC_OK;
        }
        mediaTypes = rankMediaTypeMap_[rank];
    }

    for (const auto &mediaType : mediaTypes) {
        MMC_LOG_INFO("Clear resource {rank, mediaType} -> { " << rank << ", " << mediaType << " }");
        BmUnregister(rank, mediaType);
    }
    return MMC_OK;
}
```

**声明位置**: 行 136-157

**功能描述**: 清理指定 Rank 的所有资源

**代码逻辑**:
1. 检查服务状态
2. 获取该 Rank 的所有介质类型
3. 逐个注销

---

### Stop

```cpp
void MmcMetaService::Stop()
{
    std::lock_guard<std::mutex> guard(mutex_);
    if (!started_) {
        MMC_LOG_WARN("MmcClientDefault has not been started");
        return;
    }
    metaBackUpMgrPtr_->Stop();
    metaMgrProxy_->Stop();
    metaNetServer_->Stop();
    MMC_LOG_INFO("Stop MmcMetaServiceDefault (" << name_ << ") at " << options_.discoveryURL);
    started_ = false;
}
```

**声明位置**: 行 159-171

**功能描述**: 停止元服务

**代码逻辑**:
1. 获取互斥锁
2. 检查服务状态
3. 停止备份管理器
4. 停止元管理器代理
5. 停止网络服务器
6. 清除启动标志

---

## 总结

此文件实现了元服务的核心功能：

1. **服务启动**: 初始化网络服务器、备份管理器、元管理器代理、配置存储
2. **BM 注册/注销**: 管理 Blob Manager 的生命周期
3. **资源清理**: 清理指定 Rank 的所有资源
4. **服务停止**: 按顺序停止所有子组件

**启动顺序**:
1. 网络服务器
2. 备份管理器（可选）
3. 元管理器代理
4. 配置存储

**停止顺序**:
1. 备份管理器
2. 元管理器代理
3. 网络服务器
