# mmc_meta_net_server.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_net_server.cpp`
- **文件用途**: MetaNetServer 类的实现文件
- **依赖项**: `mmc_meta_net_server.h`, `mmc_msg_base.h`, `mmc_msg_client_meta.h`

---

## 辅助函数

### Join

```cpp
std::string Join(const std::vector<std::string> &vec)
{
    std::string result = "[";
    for (const std::string &str : vec) {
        result += "\"" + str + "\", ";
    }
    if (!vec.empty()) {
        result.pop_back(); // space
        result.pop_back(); // comma
    }
    result += "]";
    return result;
}
```

**声明位置**: 行 20-32

**功能描述**: 将字符串数组连接为 JSON 数组格式

---

## 函数实现

### 构造函数

```cpp
MetaNetServer::MetaNetServer(const MmcMetaServicePtr &metaService, const std::string inputName)
    : metaService_(metaService), name_(inputName)
{}
```

**声明位置**: 行 34-36

**功能描述**: 构造网络服务器

---

### 析构函数

```cpp
MetaNetServer::~MetaNetServer() {}
```

**声明位置**: 行 37

**功能描述**: 默认析构函数

---

### Start

```cpp
Result ock::mmc::MetaNetServer::Start(NetEngineOptions &options)
{
    std::lock_guard<std::mutex> guard(mutex_);
    if (started_) {
        MMC_LOG_INFO("MetaNetServer [" << name_ << "] already started");
        return MMC_OK;
    }

    MMC_ASSERT_RETURN(metaService_.Get() != nullptr, MMC_INVALID_PARAM);

    NetEnginePtr server = NetEngine::Create();
    MMC_ASSERT_RETURN(server != nullptr, MMC_MALLOC_FAILED);
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_ALLOC_REQ,
                                      std::bind(&MetaNetServer::HandleAlloc, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_BM_REGISTER_REQ,
                                      std::bind(&MetaNetServer::HandleBmRegister, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_PING_REQ,
                                      std::bind(&MetaNetServer::HandlePing, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_UPDATE_REQ,
                                      std::bind(&MetaNetServer::HandleUpdate, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_BATCH_UPDATE_REQ,
                                      std::bind(&MetaNetServer::HandleBatchUpdate, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_GET_REQ,
                                      std::bind(&MetaNetServer::HandleGet, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_BATCH_GET_REQ,
                                      std::bind(&MetaNetServer::HandleBatchGet, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_REMOVE_REQ,
                                      std::bind(&MetaNetServer::HandleRemove, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_BATCH_REMOVE_REQ,
                                      std::bind(&MetaNetServer::HandleBatchRemove, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::LM_REMOVE_ALL_REQ,
                                      std::bind(&MetaNetServer::HandleRemoveAll, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_IS_EXIST_REQ,
                                      std::bind(&MetaNetServer::HandleIsExist, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_BATCH_IS_EXIST_REQ,
                                      std::bind(&MetaNetServer::HandleBatchIsExist, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_BM_UNREGISTER_REQ,
                                      std::bind(&MetaNetServer::HandleBmUnregister, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_QUERY_REQ,
                                      std::bind(&MetaNetServer::HandleQuery, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_BATCH_QUERY_REQ,
                                      std::bind(&MetaNetServer::HandleBatchQuery, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::ML_BATCH_ALLOC_REQ,
                                      std::bind(&MetaNetServer::HandleBatchAlloc, this, std::placeholders::_1));
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::LM_PING_REQ, nullptr);
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::LM_META_REPLICATE_REQ, nullptr);
    server->RegRequestReceivedHandler(LOCAL_META_OPCODE_REQ::LM_BLOB_COPY_REQ, nullptr);
    server->RegNewLinkHandler(std::bind(&MetaNetServer::HandleNewLink, this, std::placeholders::_1));
    server->RegLinkBrokenHandler(std::bind(&MetaNetServer::HandleLinkBroken, this, std::placeholders::_1));

    /* start engine */
    MMC_ASSERT_RETURN(server->Start(options) == MMC_OK, MMC_NOT_STARTED);

    engine_ = server;
    started_ = true;
    MMC_LOG_INFO("initialize meta net server success [" << name_ << "]");
    return MMC_OK;
}
```

**声明位置**: 行 38-95

**功能描述**: 启动网络服务器

**代码逻辑**:
1. 获取互斥锁，检查是否已启动
2. 创建网络引擎
3. 注册所有消息处理函数
4. 注册连接事件处理函数
5. 启动网络引擎
6. 保存引擎指针并设置启动标志

---

### HandleBmRegister

```cpp
Result MetaNetServer::HandleBmRegister(const NetContextPtr &context)
{
    MMC_ASSERT_RETURN(metaService_ != nullptr, MMC_ERROR);
    MMC_ASSERT_RETURN(context != nullptr, MMC_ERROR);
    BmRegisterRequest req;
    context->GetRequest<BmRegisterRequest>(req);
    TP_TRACE_BEGIN(TP_MMC_META_BM_REGISTER);
    auto result = metaService_->BmRegister(req.rank_, req.mediaType_, req.addr_, req.capacity_, req.blobMap_);
    TP_TRACE_END(TP_MMC_META_BM_REGISTER, result);
    MMC_LOG_INFO("HandleBmRegister rank: " << req.rank_ << ", rebuild blob size: " << req.blobMap_.size()
                                           << ", ret: " << result);
    Response resp;
    resp.ret_ = result;
    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 97-111

**功能描述**: 处理 BM 注册请求

**代码逻辑**:
1. 反序列化请求
2. 调用元服务注册 BM
3. 记录性能追踪
4. 回复响应

---

### HandleBmUnregister

```cpp
Result MetaNetServer::HandleBmUnregister(const NetContextPtr &context)
{
    MMC_ASSERT_RETURN(metaService_ != nullptr, MMC_ERROR);
    BmUnregisterRequest req;
    Response resp;
    resp.ret_ = MMC_OK;
    context->GetRequest<BmUnregisterRequest>(req);
    for (auto type : req.mediaType_) {
        TP_TRACE_BEGIN(TP_MMC_META_BM_UNREGISTER);
        auto result = metaService_->BmUnregister(req.rank_, type);
        TP_TRACE_END(TP_MMC_META_BM_UNREGISTER, result);
        MMC_LOG_INFO("HandleBmUnregister: " << MmcLocation(req.rank_, static_cast<MediaType>(type))
                                            << ", ret:" << result);
        if (result != MMC_OK) {
            MMC_LOG_ERROR("HandleBmUnregister rank:" << req.rank_ << ", media:" << type << ", ret:" << result);
            resp.ret_ = result;
        }
    }
    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 113-132

**功能描述**: 处理 BM 注销请求

**代码逻辑**:
1. 反序列化请求
2. 遍历所有介质类型并注销
3. 收集结果
4. 回复响应

---

### HandlePing

```cpp
Result MetaNetServer::HandlePing(const NetContextPtr &context)
{
    std::string str{static_cast<char *>(context->Data()), context->DataLen()};
    NetMsgUnpacker unpacker(str);
    PingMsg req;
    req.Deserialize(unpacker);
    MMC_LOG_INFO("HandlePing num " << req.num);

    NetMsgPacker packer;
    PingMsg recv;
    recv.Serialize(packer);
    std::string serializedData = packer.String();
    uint32_t retSize = serializedData.length();
    return context->Reply(req.msgId, serializedData.c_str(), retSize);
}
```

**声明位置**: 行 134-148

**功能描述**: 处理 Ping 请求

---

### HandleNewLink

```cpp
Result MetaNetServer::HandleNewLink(const NetLinkPtr &link)
{
    MMC_LOG_INFO(name_ << " new link, id: " << link->Id());
    return MMC_OK;
}
```

**声明位置**: 行 150-154

**功能描述**: 处理新连接建立事件

---

### HandleLinkBroken

```cpp
Result MetaNetServer::HandleLinkBroken(const NetLinkPtr &link)
{
    MMC_LOG_DEBUG(name_ << " link broken");
    MMC_ASSERT_RETURN(metaService_ != nullptr, MMC_ERROR);
    int32_t rankId = link->Id();
    TP_TRACE_BEGIN(TP_MMC_META_CLEAR_RESOURCE);
    auto ret = metaService_->ClearResource(rankId);
    TP_TRACE_END(TP_MMC_META_CLEAR_RESOURCE, ret);
    return ret;
}
```

**声明位置**: 行 156-165

**功能描述**: 处理连接断开事件

**代码逻辑**: 清理该 Rank 对应的资源

---

### HandleAlloc

```cpp
Result MetaNetServer::HandleAlloc(const NetContextPtr &context)
{
    MMC_ASSERT_RETURN(context != nullptr, MMC_ERROR);
    AllocRequest req;
    AllocResponse resp;
    context->GetRequest<AllocRequest>(req);
    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_PUT);
    const auto result = metaMgrProxy->Alloc(req, resp);
    TP_TRACE_END(TP_MMC_META_PUT, result);
    if (result != MMC_OK) {
        if (result != MMC_DUPLICATED_OBJECT) {
            MMC_LOG_ERROR("HandleAlloc key " << req.key_ << " failed, error code=" << result);
        }
    } else {
        MMC_LOG_DEBUG("HandleAlloc key " << req.key_ << " success.");
    }
    resp.result_ = result;

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 167-187

**功能描述**: 处理分配请求

**代码逻辑**:
1. 反序列化请求
2. 调用元管理器代理处理分配
3. 记录性能追踪
4. 回复响应

---

### HandleBatchAlloc

```cpp
Result MetaNetServer::HandleBatchAlloc(const NetContextPtr &context)
{
    BatchAllocRequest req;
    BatchAllocResponse resp;

    Result getResult = context->GetRequest<BatchAllocRequest>(req);
    if (getResult != MMC_OK) {
        MMC_LOG_ERROR("Failed to get BatchAllocRequest: " << getResult);
        return getResult;
    }

    MMC_LOG_DEBUG("HandleBatchAlloc start. Keys count: " << req.keys_.size() << ", OperateId: " << req.operateId_
                                                         << ", Flags: " << req.flags_);
    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_BATCH_PUT);
    Result batchResult = metaMgrProxy->BatchAlloc(req, resp);
    TP_TRACE_END(TP_MMC_META_BATCH_PUT, batchResult);
    if (batchResult != MMC_OK) {
        MMC_LOG_ERROR("BatchAlloc failed. Keys count: " << req.keys_.size() << ", Error: " << batchResult);
    }
    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 189-210

**功能描述**: 处理批量分配请求

---

### HandleUpdate

```cpp
Result MetaNetServer::HandleUpdate(const NetContextPtr &context)
{
    UpdateRequest req;
    Response resp;
    context->GetRequest<UpdateRequest>(req);

    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_UPDATE);
    metaMgrProxy->UpdateState(req, resp);
    TP_TRACE_END(TP_MMC_META_UPDATE, resp.ret_);
    MMC_LOG_DEBUG("HandleUpdate key " << req.key_ << " finish.");

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 212-225

**功能描述**: 处理状态更新请求

---

### HandleBatchUpdate

```cpp
Result MetaNetServer::HandleBatchUpdate(const NetContextPtr &context)
{
    BatchUpdateRequest req;
    BatchUpdateResponse resp;
    context->GetRequest<BatchUpdateRequest>(req);

    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_BATCH_UPDATE);
    auto ret = metaMgrProxy->BatchUpdateState(req, resp);
    TP_TRACE_END(TP_MMC_META_BATCH_UPDATE, ret);
    (void)ret;
    MMC_LOG_DEBUG("HandleBatchUpdate keys (size " << req.keys_.size() << ") finish: " << Join(req.keys_));

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 227-241

**功能描述**: 处理批量状态更新请求

---

### HandleGet

```cpp
Result MetaNetServer::HandleGet(const NetContextPtr &context)
{
    GetRequest req;
    AllocResponse resp;
    context->GetRequest<GetRequest>(req);

    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_GET);
    metaMgrProxy->Get(req, resp);
    TP_TRACE_END(TP_MMC_META_GET, resp.result_);
    MMC_LOG_DEBUG("HandleGet key " << req.key_ << " finish.");

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 243-256

**功能描述**: 处理获取请求

---

### HandleBatchGet

```cpp
Result MetaNetServer::HandleBatchGet(const NetContextPtr &context)
{
    BatchGetRequest req;
    BatchAllocResponse resp;
    context->GetRequest<BatchGetRequest>(req);

    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_BATCH_GET);
    auto ret = metaMgrProxy->BatchGet(req, resp);
    TP_TRACE_END(TP_MMC_META_BATCH_GET, ret);
    (void)ret;
    MMC_LOG_DEBUG("HandleBatchGet keys (size  " << req.keys_.size() << ") finish: " << Join(req.keys_));

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 258-272

**功能描述**: 处理批量获取请求

---

### HandleRemove

```cpp
Result MetaNetServer::HandleRemove(const NetContextPtr &context)
{
    RemoveRequest req;
    Response resp;
    context->GetRequest<RemoveRequest>(req);

    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_REMOVE);
    metaMgrProxy->Remove(req, resp);
    TP_TRACE_END(TP_MMC_META_REMOVE, resp.ret_);
    MMC_LOG_DEBUG("HandleRemove key " << req.key_ << " finish.");

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 274-287

**功能描述**: 处理删除请求

---

### HandleBatchRemove

```cpp
Result MetaNetServer::HandleBatchRemove(const NetContextPtr &context)
{
    BatchRemoveRequest req;
    BatchRemoveResponse resp;
    context->GetRequest<BatchRemoveRequest>(req);

    MmcMetaMgrProxyPtr metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_BATCH_REMOVE);
    auto ret = metaMgrProxy->BatchRemove(req, resp);
    TP_TRACE_END(TP_MMC_META_BATCH_REMOVE, ret);
    (void)ret;
    MMC_LOG_DEBUG("HandleBatchRemove keys (size  " << req.keys_.size() << ") finish: " << Join(req.keys_));

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 289-303

**功能描述**: 处理批量删除请求

---

### HandleRemoveAll

```cpp
Result MetaNetServer::HandleRemoveAll(const NetContextPtr &context)
{
    RemoveAllRequest req;
    Response resp;
    context->GetRequest<RemoveAllRequest>(req);

    MmcMetaMgrProxyPtr metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_REMOVE_ALL);
    auto ret = metaMgrProxy->RemoveAll(req, resp);
    TP_TRACE_END(TP_MMC_META_REMOVE_ALL, ret);
    (void)ret;
    MMC_LOG_DEBUG("HandleRemoveAll finished");

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 305-319

**功能描述**: 处理删除所有请求

---

### HandleIsExist

```cpp
Result MetaNetServer::HandleIsExist(const NetContextPtr &context)
{
    IsExistRequest req;
    IsExistResponse resp;
    context->GetRequest<IsExistRequest>(req);

    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_EXIST);
    metaMgrProxy->ExistKey(req, resp);
    TP_TRACE_END(TP_MMC_META_EXIST, resp.ret_);
    MMC_LOG_DEBUG("HandleIsExist key " << req.key_ << " finish.");

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 321-334

**功能描述**: 处理检查存在请求

---

### HandleBatchIsExist

```cpp
Result MetaNetServer::HandleBatchIsExist(const NetContextPtr &context)
{
    BatchIsExistRequest req;
    BatchIsExistResponse resp;
    context->GetRequest<BatchIsExistRequest>(req);

    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_BATCH_EXIST);
    auto ret = metaMgrProxy->BatchExistKey(req, resp);
    TP_TRACE_END(TP_MMC_META_BATCH_EXIST, ret);
    (void)ret;
    MMC_LOG_DEBUG("HandleBatchIsExist keys (size " << req.keys_.size() << ") finish: " << Join(req.keys_));

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 336-350

**功能描述**: 处理批量检查存在请求

---

### HandleQuery

```cpp
Result MetaNetServer::HandleQuery(const NetContextPtr &context)
{
    QueryRequest req;
    QueryResponse resp;
    context->GetRequest<QueryRequest>(req);

    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_QUERY);
    auto ret = metaMgrProxy->Query(req, resp);
    TP_TRACE_END(TP_MMC_META_QUERY, ret);
    (void)ret;
    MMC_LOG_DEBUG("HandleQuery key " << req.key_ << " finish.");

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 352-366

**功能描述**: 处理查询请求

---

### HandleBatchQuery

```cpp
Result MetaNetServer::HandleBatchQuery(const NetContextPtr &context)
{
    BatchQueryRequest req;
    BatchQueryResponse resp;
    context->GetRequest<BatchQueryRequest>(req);

    auto &metaMgrProxy = metaService_->GetMetaMgrProxy();
    TP_TRACE_BEGIN(TP_MMC_META_BATCH_QUERY);
    auto ret = metaMgrProxy->BatchQuery(req, resp);
    TP_TRACE_END(TP_MMC_META_BATCH_QUERY, ret);
    (void)ret;
    MMC_LOG_DEBUG("HandleBatchQuery keys (size " << req.keys_.size() << ") finish: " << Join(req.keys_));

    return context->Reply(req.msgId, resp);
}
```

**声明位置**: 行 368-382

**功能描述**: 处理批量查询请求

---

### Stop

```cpp
void MetaNetServer::Stop()
{
    std::lock_guard<std::mutex> guard(mutex_);
    if (!started_) {
        MMC_LOG_WARN("MetaNetServer has not been started");
        return;
    }
    engine_->Stop();
    started_ = false;
}
```

**声明位置**: 行 384-393

**功能描述**: 停止网络服务器

**代码逻辑**:
1. 获取互斥锁
2. 检查是否已启动
3. 停止网络引擎
4. 清除启动标志

---

## 总结

此文件实现了元数据网络服务器的功能：

1. **消息路由**: 将不同类型的消息路由到对应的处理函数
2. **请求/响应**: 处理网络请求并生成响应
3. **连接管理**: 处理新连接和断开连接事件
4. **性能追踪**: 使用 ptracer 记录关键操作的耗时
5. **日志记录**: 记录操作日志

**支持的操作**:
- BM 注册/注销
- 分配/批量分配
- 获取/批量获取
- 更新/批量更新
- 删除/批量删除
- 检查存在/批量检查存在
- 查询/批量查询
- Ping
