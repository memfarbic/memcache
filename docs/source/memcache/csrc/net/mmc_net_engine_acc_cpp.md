# mmc_net_engine_acc.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/acc_links_impl/mmc_net_engine_acc.cpp`
- **文件用途**: 实现 ACC 网络引擎
- **依赖项**:
  - `mf_tls_util.h` - TLS 工具
  - `acc_def.h` - ACC 框架定义
  - `acc_tcp_server.h` - ACC TCP 服务器
  - 其他网络相关头文件

---

## 常量

```cpp
constexpr const int16_t NET_SERVER_MAGIC = 3867;
constexpr const int NET_POOL_BASE = 16;
```

- `NET_SERVER_MAGIC`: 服务器魔数，用于验证连接
- `NET_POOL_BASE`: 线程池基础大小

---

## 函数

### NetEngineAcc::Start()

```cpp
Result NetEngineAcc::Start(const NetEngineOptions &options)
{
    std::lock_guard<std::mutex> guard(mutex_);
    if (started_) {
        MMC_LOG_INFO("NetEngineAcc " << options.name << " already started");
        return MMC_OK;
    }

    /* verify */
    MMC_RETURN_ERROR(VerifyOptions(options), "NetEngineAcc " << options.name << " option set error");

    /* initialize */
    MMC_RETURN_ERROR(Initialize(options), "NetEngineAcc " << options.name << " initialize error");

    /* check handler */
    if (options_.startListener && newLinkHandler_ == nullptr) {
        MMC_LOG_ERROR("No new link handler function registered, call 'RegisterNewLinkHandler' to register");
        return MMC_INVALID_PARAM;
    } else if (!options_.startListener && newLinkHandler_ != nullptr) {
        MMC_LOG_ERROR("No need to register new link handler function for the engine without listener");
        return MMC_INVALID_PARAM;
    }

    /* call start inner */
    auto ret = StartInner();
    if (ret != MMC_OK) {
        UnInitialize();
        MMC_LOG_ERROR("NetEngineAcc " << options.name << " start error");
        return ret;
    }

    threadPool_ = MmcMakeRef<MmcThreadPool>("net_pool", NET_POOL_BASE);
    if (threadPool_ == nullptr || threadPool_->Start() != MMC_OK) {
        StopInner();
        UnInitialize();
        MMC_LOG_ERROR("Failed to start thread pool");
        return ret;
    }

    started_ = true;
    return MMC_OK;
}
```

**声明位置**: 行 30-71

**功能描述**: 启动网络引擎

**参数**: `options` - 配置选项

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 检查是否已启动
2. 验证配置选项
3. 初始化引擎
4. 检查回调处理器
5. 调用内部启动
6. 启动线程池

---

### NetEngineAcc::Stop()

```cpp
void NetEngineAcc::Stop()
{
    std::lock_guard<std::mutex> guard(mutex_);
    if (!started_) {
        MMC_LOG_WARN("NetEngineAcc has not been started");
        return;
    }
    if (threadPool_ == nullptr) {
        threadPool_->Destroy();
    }
    MMC_ASSERT(StopInner() == MMC_OK);

    UnInitialize();

    started_ = false;
}
```

**声明位置**: 行 73-88

**功能描述**: 停止网络引擎

**代码逻辑**:
1. 检查是否已启动
2. 销毁线程池
3. 停止内部服务
4. 清理资源

---

### NetEngineAcc::VerifyOptions()

```cpp
Result NetEngineAcc::VerifyOptions(const NetEngineOptions &options)
{
    if (options.rankId == UINT16_MAX) {
        MMC_LOG_ERROR("verify NetEngineOptions failed, invalid rank id " << options.rankId);
        return MMC_INVALID_PARAM;
    } else if (options.name.empty()) {
        MMC_LOG_ERROR("verify NetEngineOptions failed, empty name");
        return MMC_INVALID_PARAM;
    }

    if (options.startListener) {
        if (options.ip.empty()) {
            MMC_LOG_ERROR("verify NetEngineOptions failed, ip is empty");
            return MMC_INVALID_PARAM;
        } else if (options.port <= N1024) {
            MMC_LOG_ERROR("verify NetEngineOptions failed, invalid port " << options.port);
            return MMC_INVALID_PARAM;
        }
    }

    if (options.threadCount > N1024 || options.threadCount == 0) {
        MMC_LOG_ERROR("verify NetEngineOptions failed, threadCount is too large, which should between 1~" << N1024);
        return MMC_INVALID_PARAM;
    }

    return MMC_OK;
}
```

**声明位置**: 行 90-116

**功能描述**: 验证配置选项的有效性

**参数**: `options` - 配置选项

**返回值**: `Result` - MMC_OK 表示有效

**验证项**:
- rankId 必须有效（非 UINT16_MAX）
- name 不能为空
- 如果启动监听器，ip 和 port 必须有效
- threadCount 必须在 1~1024 范围内

---

### NetEngineAcc::StartInner()

```cpp
Result NetEngineAcc::StartInner()
{
    /* construct acc tcp server */
    TcpServerOptions serverOptions;
    serverOptions.version = static_cast<int16_t>(NetProtoVersion::VERSION_1);
    serverOptions.magic = NET_SERVER_MAGIC;
    if (options_.startListener) {
        serverOptions.enableListener = true;
        serverOptions.listenIp = options_.ip;
        serverOptions.listenPort = options_.port;
    }
    serverOptions.workerCount = options_.threadCount;
    serverOptions.linkSendQueueSize = UN32;

    TcpTlsOption tlsOpt{};
    tlsOpt.enableTls = options_.tlsOption.tlsEnable;
    if (tlsOpt.enableTls) {
        tlsOpt.tlsTopPath = "/";
        tlsOpt.tlsCaPath = "/";
        tlsOpt.tlsCaFile.insert(options_.tlsOption.caPath);
        tlsOpt.tlsCrlPath = "/";
        std::string crlFile = options_.tlsOption.crlPath;
        if (!crlFile.empty()) {
            tlsOpt.tlsCrlFile.insert(crlFile);
        }
        tlsOpt.tlsCert = options_.tlsOption.certPath;
        tlsOpt.tlsPk = options_.tlsOption.keyPath;
        tlsOpt.tlsPkPwd = options_.tlsOption.keyPassPath;
        MMC_RETURN_ERROR(server_->LoadDynamicLib(options_.tlsOption.packagePath),
                         "Failed to load openssl dynamic library");
        if (!tlsOpt.tlsPkPwd.empty()) {
            MMC_RETURN_ERROR(RegisterDecryptHandler(options_.tlsOption.decrypterLibPath),
                             "Failed to register decrypt handler");
        }
    }

    /* start server, listen and thread will be started */
    MMC_RETURN_ERROR(server_->Start(serverOptions, tlsOpt),
                     "Failed to start tcp server for NetEngine " << options_.name);

    return MMC_OK;
}
```

**声明位置**: 行 118-159

**功能描述**: 内部启动逻辑

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 构造 TCP 服务器选项
2. 配置 TLS 选项（如果启用）
3. 启动 TCP 服务器

---

### NetEngineAcc::StopInner()

```cpp
Result NetEngineAcc::StopInner()
{
    /* stop server */
    if (server_ != nullptr) {
        server_->Stop();
        server_ = nullptr;
    }
    mf::MfTlsUtil::CloseTlsLib();
    return MMC_OK;
}
```

**声明位置**: 行 161-170

**功能描述**: 内部停止逻辑

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 停止 TCP 服务器
2. 关闭 TLS 库

---

### TraceSendRecord() / TraceSendWaitRecord()

```cpp
static void TraceSendRecord(int16_t opCode, uint64_t diff)
{
    switch (opCode) {
        case ML_ALLOC_REQ:
            TP_TRACE_RECORD(TP_ACC_SEND_ALLOC, diff, 0);
            break;
        // ... 更多操作码
        default:
            break;
    };
}

static void TraceSendWaitRecord(int16_t opCode, uint64_t diff)
{
    switch (opCode) {
        case ML_ALLOC_REQ:
            TP_TRACE_RECORD(TP_ACC_SEND_WAIT_ALLOC, diff, 0);
            break;
        // ... 更多操作码
        default:
            break;
    };
}
```

**声明位置**: 行 172-255

**功能描述**: 性能追踪辅助函数

**参数**:
- `opCode`: 操作码
- `diff`: 时间差（纳秒）

**用途**: 记录不同操作的性能指标

---

### NetEngineAcc::Call()

```cpp
Result NetEngineAcc::Call(uint32_t targetId, int16_t opCode, const char *reqData, uint32_t reqDataLen, char **respData,
                          uint32_t &respDataLen, int32_t timeoutInSecond)
{
    uint64_t startTime = TP_CURRENT_TIME_NS;
    MMC_ASSERT_RETURN(started_, MMC_NOT_STARTED);
    MMC_ASSERT_RETURN(reqData != nullptr, MMC_INVALID_PARAM);
    MMC_ASSERT_RETURN(reqDataLen != 0, MMC_INVALID_PARAM);
    MMC_ASSERT_RETURN(respData != nullptr, MMC_INVALID_PARAM);

    MMC_ASSERT_RETURN(opCode != -1, MMC_INVALID_PARAM);

    /* step1: do serialization */

    /* step2: get the link to send */
    NetLinkAccPtr link;
    auto result = peerLinkMap_->Find(targetId, link);
    if (!result || link == nullptr) {
        return MMC_LINK_NOT_FOUND; /* need to connect */
    }
    /* step3: copy data */
    auto dataBuf = MmcMakeRef<ock::acc::AccDataBuffer>(reqDataLen);
    MMC_ASSERT_RETURN(dataBuf.Get() != nullptr, MMC_NEW_OBJECT_FAILED);
    MMC_ASSERT_RETURN(dataBuf->AllocIfNeed(), MMC_NEW_OBJECT_FAILED);
    std::copy_n(reqData, reqDataLen, static_cast<char *>(dataBuf->DataPtrVoid()));
    dataBuf->SetDataSize(reqDataLen);

    /* step4: create wait handler and initialize */
    auto waiter = MmcMakeRef<NetWaitHandler>(ctxStore_);
    MMC_ASSERT_RETURN(waiter.Get() != nullptr, MMC_NEW_OBJECT_FAILED);
    MMC_ASSERT_RETURN(waiter->Initialize() == MMC_OK, MMC_ERROR);

    /* step5: put into ctx store before sent the data to peer in case of the peer responses very fast  */
    uint32_t seqNo = 0;
    result = ctxStore_->PutAndGetSeqNo<NetWaitHandler>(waiter.Get(), seqNo);
    MMC_ASSERT_RETURN(result == MMC_OK, result);
    MMC_ASSERT_RETURN(link->RealLink() != nullptr, MMC_ERROR);
    /* step6: send message to peer */

    result = link->RealLink()->NonBlockSend(MSG_TYPE_DATA, opCode, seqNo, dataBuf.Get(), nullptr);
    if (result != MMC_OK) {
        /* remove wait handler from context store */
        ctxStore_->RemoveSeqNo<NetWaitHandler>(seqNo);
        MMC_LOG_ERROR("Failed to send data to service " << targetId);
        return result;
    }
    uint64_t waitStartTime = TP_CURRENT_TIME_NS;
    /* step7: wait for response */
    result = waiter->TimedWait(timeoutInSecond);
    TraceSendWaitRecord(opCode, TP_CURRENT_TIME_NS - waitStartTime);
    /* if timeout */
    if (result == MMC_TIMEOUT) {
        /* remove waiter in context store */
        ctxStore_->RemoveSeqNo<NetWaitHandler>(seqNo);
        MMC_LOG_WARN("Peer " << targetId << " doesn't response within " << timeoutInSecond << " seconds");
        return result;
    }

    /* got response data and deserialize */
    auto &data = waiter->Data();
    MMC_ASSERT_RETURN(data.Get() != nullptr, MMC_ERROR);
    /* set response code */

    /* deserialize */

    if (*respData == nullptr) {
        *respData = (char *)malloc(data->DataLen());
        if (*respData == nullptr) {
            MMC_LOG_WARN("Failed to malloc resp date length:" << data->DataLen());
            return MMC_MALLOC_FAILED;
        }
    }
    std::copy_n(reinterpret_cast<char *>(data->DataIntPtr()), data->DataLen(), *respData);
    respDataLen = data->DataLen();
    TraceSendRecord(opCode, TP_CURRENT_TIME_NS - startTime);
    return result;
}
```

**声明位置**: 行 257-332

**功能描述**: 同步 RPC 调用

**参数**:
- `targetId`: 目标节点 ID
- `opCode`: 操作码
- `reqData`: 请求数据
- `reqDataLen`: 请求数据长度
- `respData`: 输出参数，响应数据
- `respDataLen`: 输出参数，响应数据长度
- `timeoutInSecond`: 超时时间（秒）

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 获取到对端的链接
2. 拷贝请求数据到缓冲区
3. 创建等待处理器
4. 存储到上下文存储并获取序列号
5. 发送请求
6. 等待响应（带超时）
7. 处理超时或正常响应

---

### NetEngineAcc::Send()

```cpp
Result NetEngineAcc::Send(uint32_t peerId, const char *reqData, uint32_t reqDataLen, int32_t timeoutInSecond)
{
    MMC_ASSERT_RETURN(started_, MMC_NOT_STARTED);
    MMC_ASSERT_RETURN(reqData != nullptr, MMC_INVALID_PARAM);
    MMC_ASSERT_RETURN(reqDataLen != 0, MMC_INVALID_PARAM);

    return MMC_OK;
}
```

**声明位置**: 行 334-341

**功能描述**: 单向发送（当前为占位实现）

**参数**:
- `peerId`: 对端节点 ID
- `reqData`: 请求数据
- `reqDataLen`: 请求数据长度
- `timeoutInSecond`: 超时时间（秒）

**返回值**: `Result` - MMC_OK

---

### NetEngineAcc::~NetEngineAcc()

```cpp
NetEngineAcc::~NetEngineAcc()
{
    Stop();
}
```

**声明位置**: 行 343-346

**功能描述**: 析构函数，自动停止引擎

---

### NetEngineAcc::Initialize()

```cpp
Result NetEngineAcc::Initialize(const NetEngineOptions &options)
{
    if (inited_) {
        MMC_LOG_INFO("NetEngine [" << options.name << "] already initialized");
        return MMC_OK;
    }

    MMC_LOG_DEBUG("Start to init NetEngine " << options.name);

    /* create concurrent link map */
    NetLinkMapAccPtr tmpLinkMap = MmcMakeRef<NetLinkMapAcc>();
    MMC_ASSERT_RETURN(tmpLinkMap != nullptr, MMC_NEW_OBJECT_FAILED);

    /* create ctx store */
    NetContextStorePtr tmpCtxStore = MmcMakeRef<NetContextStore>(UN65536);
    MMC_ASSERT_RETURN(tmpCtxStore != nullptr, MMC_NEW_OBJECT_FAILED);
    auto result = tmpCtxStore->Initialize();
    MMC_RETURN_ERROR(result, "Failed to initialize ctx store for communication seq number");

    /* create tcp server */
    auto tmpServer = TcpServer::Create();
    MMC_ASSERT_RETURN(tmpServer != nullptr, MMC_NEW_OBJECT_FAILED);
    server_ = tmpServer.Get();

    /* register callbacks */
    options_ = options;
    MMC_RETURN_ERROR(RegisterTcpServerHandler(), "NetEngineAcc " << options_.name << " register handler error");
    /* assign to member variables */
    peerLinkMap_ = tmpLinkMap;
    ctxStore_ = tmpCtxStore;

    inited_ = true;

    return MMC_OK;
}
```

**声明位置**: 行 348-382

**功能描述**: 初始化网络引擎

**参数**: `options` - 配置选项

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 创建链接映射表
2. 创建并初始化上下文存储
3. 创建 TCP 服务器
4. 注册回调处理器

---

### NetEngineAcc::UnInitialize()

```cpp
void NetEngineAcc::UnInitialize()
{
    if (!inited_) {
        MMC_LOG_DEBUG("NetEngine [" << options_.name << "] has not been initialized");
        return;
    }

    /* un-initialize ctx store */
    if (ctxStore_ != nullptr) {
        ctxStore_->UnInitialize();
        ctxStore_ = nullptr;
    }

    if (server_ != nullptr) {
        server_->Stop();
        server_ = nullptr;
    }

    /* clear link map */
    if (peerLinkMap_ != nullptr) {
        peerLinkMap_->Clear();
    }

    inited_ = false;
}
```

**声明位置**: 行 384-408

**功能描述**: 清理网络引擎资源

**代码逻辑**:
1. 清理上下文存储
2. 停止 TCP 服务器
3. 清空链接映射表

---

### NetEngineAcc::RegisterTcpServerHandler()

```cpp
Result NetEngineAcc::RegisterTcpServerHandler()
{
    MMC_ASSERT_RETURN(server_ != nullptr, MMC_NOT_INITIALIZED);

    using namespace std::placeholders;
    if (options_.startListener) {
        server_->RegisterNewLinkHandler(std::bind(&NetEngineAcc::HandleNewLink, this, _1, _2));
    }

    server_->RegisterNewRequestHandler(MSG_TYPE_DATA, std::bind(&NetEngineAcc::HandleNeqRequest, this, _1));
    server_->RegisterRequestSentHandler(MSG_TYPE_DATA, std::bind(&NetEngineAcc::HandleMsgSent, this, _1, _2, _3));
    server_->RegisterLinkBrokenHandler(std::bind(&NetEngineAcc::HandleLinkBroken, this, _1));

    return MMC_OK;
}
```

**声明位置**: 行 410-424

**功能描述**: 注册 TCP 服务器回调处理器

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 注册新链接处理器（如果启用监听）
2. 注册请求处理器
3. 注册发送完成处理器
4. 注册链接断开处理器

---

### NetEngineAcc::HandleNewLink()

```cpp
Result NetEngineAcc::HandleNewLink(const TcpConnReq &req, const TcpLinkPtr &link) const
{
    MMC_ASSERT_RETURN(link.Get() != nullptr, MMC_INVALID_PARAM);

    auto peerId = static_cast<uint32_t>(req.rankId);
    link->UpCtx(peerId);

    auto newLinkAcc = MmcMakeRef<NetLinkAcc>(peerId, link);
    MMC_ASSERT_RETURN(newLinkAcc != nullptr, MMC_NEW_OBJECT_FAILED);
    MMC_LOG_DEBUG("NEW Link");

    /* add into peer link map */
    peerLinkMap_->Add(peerId, newLinkAcc);
    MMC_LOG_INFO("HandleNewLink with peer rankId: " << req.rankId);
    Result ret = MMC_OK;
    if (newLinkHandler_ != nullptr) {
        ret = newLinkHandler_(newLinkAcc.Get());
    }
    return MMC_OK;
}
```

**声明位置**: 行 426-445

**功能描述**: 处理新链接建立事件

**参数**:
- `req`: 连接请求
- `link`: 新建立的链接

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 提取 peerId
2. 创建 NetLinkAcc 包装
3. 添加到链接映射表
4. 调用用户注册的回调

---

### NetEngineAcc::HandleNeqRequest()

```cpp
Result NetEngineAcc::HandleNeqRequest(const TcpReqContext &context)
{
    /* use result variable for real opcode */
    MMC_LOG_DEBUG("HandleNeqRequest Header " << context.Header().ToString());
    int16_t opCode = context.Header().result;
    MMC_ASSERT_RETURN(opCode < gHandlerSize, MMC_NET_REQ_HANDLE_NO_FOUND);
    if (reqReceivedHandlers_[opCode] == nullptr) {
        /*  client do reply response */
        MMC_ASSERT_RETURN(HandleAllRequests4Response(context) == MMC_OK, MMC_ERROR);
    } else {
        /* server do function */
        // context buf是link缓冲区，切线程需要将数据copy出来
        ock::acc::AccDataBufferPtr bufPtr = ock::acc::AccDataBuffer::Create(context.DataPtr(), context.DataLen());
        if (bufPtr.Get() == nullptr) {
            MMC_LOG_ERROR("req: " << context.SeqNo() << " alloc failed. op:" << opCode);
            return MMC_ERROR;
        }
        ock::acc::AccTcpRequestContext reqContext(context.Header(), bufPtr, context.Link());
        NetContextPtr asynCtxPtr = MmcMakeRef<NetContextAcc>(reqContext).Get();
        if (asynCtxPtr.Get() == nullptr) {
            MMC_LOG_ERROR("req: " << context.SeqNo() << " alloc ctx failed. op:" << opCode);
            return MMC_ERROR;
        }
        auto future = threadPool_->Enqueue(
            [&](int16_t opCode, NetContextPtr contextPtrL) { return reqReceivedHandlers_[opCode](contextPtrL); },
            opCode, asynCtxPtr);
        if (!future.valid()) {
            MMC_LOG_ERROR("req: " << context.SeqNo() << " add thread pool failed. op:" << opCode);
            return MMC_ERROR;
        }
    }
    return MMC_OK;
}
```

**声明位置**: 行 447-479

**功能描述**: 处理接收到的请求

**参数**: `context` - 请求上下文

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 提取操作码
2. 如果没有注册处理器，作为响应处理（客户端模式）
3. 如果有处理器，拷贝数据并提交到线程池执行（服务器模式）

---

### NetEngineAcc::HandleMsgSent()

```cpp
Result NetEngineAcc::HandleMsgSent(TcpMsgSentResult result, const TcpMsgHeader &header, const TcpDataBufPtr &cbCtx)
{
    return MMC_OK;
}
```

**声明位置**: 行 481-484

**功能描述**: 处理消息发送完成事件（占位实现）

**返回值**: `Result` - MMC_OK

---

### NetEngineAcc::HandleLinkBroken()

```cpp
Result NetEngineAcc::HandleLinkBroken(const TcpLinkPtr &link) const
{
    MMC_ASSERT_RETURN(link.Get() != nullptr, MMC_INVALID_PARAM);

    const auto peerId = static_cast<uint32_t>(link->UpCtx());

    MmcRef<NetLinkAcc> linkAcc = nullptr;
    peerLinkMap_->Find(peerId, linkAcc);

    Result ret = MMC_OK;
    if (linkBrokenHandler_ != nullptr && linkAcc.Get() != nullptr && linkAcc->RealLink() != nullptr) {
        if (linkAcc->RealLink()->Id() == link->Id()) {
            peerLinkMap_->Remove(peerId);
            MMC_RETURN_ERROR(linkBrokenHandler_(linkAcc.Get()), "Failed to remove link with id " << peerId);
        } else {
            MMC_LOG_WARN("Old linkId: " << linkAcc->RealLink()->Id() << ", rankId: " << peerId);
        }
    }
    return ret;
}
```

**声明位置**: 行 486-505

**功能描述**: 处理链接断开事件

**参数**: `link` - 断开的链接

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 提取 peerId
2. 查找链接
3. 验证链接 ID
4. 从映射表移除
5. 调用用户注册的回调

---

### NetEngineAcc::ConnectToPeer()

```cpp
Result NetEngineAcc::ConnectToPeer(uint32_t peerId, const std::string &peerIp, uint16_t port, NetLinkPtr &newLink,
                                   bool isForce)
{
    MMC_ASSERT_RETURN(started_, MMC_NOT_STARTED);
    MMC_ASSERT_RETURN(!peerIp.empty(), MMC_INVALID_PARAM);
    MMC_ASSERT_RETURN(port != 0, MMC_INVALID_PARAM);

    TcpConnReq connReq;
    connReq.rankId = peerId;
    connReq.version = static_cast<int16_t>(NetProtoVersion::VERSION_1);
    connReq.magic = NET_SERVER_MAGIC;

    NetLinkAccPtr linkAcc;
    /* connect to peer with mutex held, in case of multiple threads connect to peer at the same time */
    std::lock_guard<std::mutex> guard(connectMutex_);
    /* double check, in case of other thread already connected */
    bool result = peerLinkMap_->Find(peerId, linkAcc);
    if (result) {
        if (!isForce) {
            MMC_LOG_INFO("The link to peer " << peerId << " already exists");
            return MMC_OK;
        } else {
            peerLinkMap_->Remove(peerId);
        }
    }

    MMC_LOG_DEBUG("Connecting to " << peerIp << ":" << port << "");

    /* connect */
    TcpLinkPtr realLink;
    Result ret = server_->ConnectToPeerServer(peerIp, port, connReq, realLink);
    if (ret != MMC_OK || realLink == nullptr) {
        MMC_LOG_ERROR("Failed to connection to peerId: " << peerId << ", peerIpPort: " << peerIp << ":" << port);
        return ret;
    }
    realLink->UpCtx(peerId);

    /* add into peer link map */
    linkAcc = MmcMakeRef<NetLinkAcc>(peerId, realLink);
    MMC_ASSERT_RETURN(linkAcc != nullptr, MMC_NEW_OBJECT_FAILED);
    peerLinkMap_->Add(peerId, linkAcc);

    /* set peer id */
    newLink = linkAcc.Get();
    newLink->UpCtx(peerId);

    return MMC_OK;
}
```

**声明位置**: 行 507-554

**功能描述**: 连接到对端节点

**参数**:
- `peerId`: 对端节点 ID
- `peerIp`: 对端 IP 地址
- `port`: 对端端口
- `newLink`: 输出参数，新建的链接
- `isForce`: 是否强制重新连接

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 构造连接请求
2. 双重检查是否已存在连接
3. 如果存在且非强制模式，直接返回
4. 建立新连接
5. 添加到链接映射表

---

### NetEngineAcc::HandleAllRequests4Response()

```cpp
Result NetEngineAcc::HandleAllRequests4Response(const TcpReqContext &context)
{
    MMC_LOG_DEBUG("Get request with seqNo " << context.SeqNo());
    NetWaitHandler *out = nullptr;
    auto result = ctxStore_->GetSeqNoAndRemove<NetWaitHandler>(context.SeqNo(), out, false);
    /* check if out is nullptr */
    MMC_ASSERT_RETURN(out != nullptr, MMC_ERROR);
    if (result != MMC_OK) {
        if (out != nullptr) { /* decrease ref */
            out->DecreaseRef();
        }

        MMC_LOG_WARN("Failed to get waiter from ctx store with seqNo " << context.SeqNo() << ", probably timeout");
        return MMC_OK;
    }

    NetWaitHandlerPtr waiter = out;
    out->DecreaseRef(); /* decrease ref */

    auto dataBuf = MmcMakeRef<ock::acc::AccDataBuffer>(context.DataLen());
    MMC_ASSERT_RETURN(result == MMC_OK, MMC_NEW_OBJECT_FAILED);
    MMC_ASSERT_RETURN(dataBuf->AllocIfNeed(), MMC_NEW_OBJECT_FAILED);
    MMC_ASSERT_RETURN(context.DataPtr() != nullptr, MMC_ERROR);
    std::copy_n(static_cast<char *>(context.DataPtr()), context.DataLen(),
                reinterpret_cast<char *>(dataBuf->DataIntPtr()));
    dataBuf->SetDataSize(context.DataLen());

    result = waiter->Notify(context.Header().result, dataBuf.Get());
    if (result == MMC_ALREADY_NOTIFIED) {
        MMC_LOG_WARN("Already notify wait handler for seqNo " << context.SeqNo());
        return MMC_OK;
    }

    return MMC_OK;
}
```

**声明位置**: 行 556-590

**功能描述**: 处理响应消息（客户端模式）

**参数**: `context` - 请求上下文

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 根据序列号查找等待处理器
2. 拷贝响应数据
3. 通知等待的线程

---

### NetEngineAcc::RegisterDecryptHandler()

```cpp
Result NetEngineAcc::RegisterDecryptHandler(const std::string &decryptLibPath) const
{
    if (decryptLibPath.empty()) {
        MMC_LOG_WARN("No decrypter provided, using default decrypter handler");
        server_->RegisterDecryptHandler(mf::MfTlsUtil::DefaultDecrypter);
        return MMC_OK;
    }

    const auto decrypter = mf::MfTlsUtil::LoadDecryptFunction(decryptLibPath.c_str());
    if (decrypter == nullptr) {
        MMC_LOG_ERROR("failed to load customized decrypt function");
        return MMC_ERROR;
    }
    server_->RegisterDecryptHandler(decrypter);

    return MMC_OK;
}
```

**声明位置**: 行 592-608

**功能描述**: 注册 TLS 解密处理器

**参数**: `decryptLibPath` - 解密库路径

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 如果路径为空，使用默认解密器
2. 否则加载自定义解密函数
3. 注册到服务器

---

## 文件级别的关系图

```
mmc_net_engine_acc.cpp (ACC 网络引擎实现)
    |
    +-- NetEngineAcc::Start() -> 启动引擎
    +-- NetEngineAcc::Stop() -> 停止引擎
    +-- NetEngineAcc::VerifyOptions() -> 验证配置
    +-- NetEngineAcc::Initialize() -> 初始化
    +-- NetEngineAcc::UnInitialize() -> 清理
    +-- NetEngineAcc::StartInner() -> 内部启动
    +-- NetEngineAcc::StopInner() -> 内部停止
    +-- NetEngineAcc::Call() -> RPC 调用
    +-- NetEngineAcc::Send() -> 单向发送
    +-- NetEngineAcc::ConnectToPeer() -> 连接对端
    +-- NetEngineAcc::RegisterTcpServerHandler() -> 注册回调
    +-- NetEngineAcc::HandleNewLink() -> 新链接处理
    +-- NetEngineAcc::HandleNeqRequest() -> 请求处理
    +-- NetEngineAcc::HandleMsgSent() -> 发送完成处理
    +-- NetEngineAcc::HandleLinkBroken() -> 链接断开处理
    +-- NetEngineAcc::HandleAllRequests4Response() -> 响应处理
    +-- NetEngineAcc::RegisterDecryptHandler() -> 注册解密器
    +-- TraceSendRecord() -> 性能追踪
    +-- TraceSendWaitRecord() -> 性能追踪
```

---

## 依赖关系

**依赖以下文件**:
- `mf_tls_util.h` - TLS 工具
- `acc_def.h` - ACC 框架定义
- `acc_tcp_server.h` - ACC TCP 服务器
- `mmc_net_link_acc.h` - ACC 链接
- `mmc_net_ctx_store.h` - 上下文存储
- `mmc_msg_base.h` - 消息基础
- `mmc_net_wait_handle.h` - 等待处理器
- `mmc_net_common_acc.h` - ACC 网络通用定义
- `mmc_ptracer.h` - 性能追踪
- `mmc_net_ctx_acc.h` - ACC 上下文

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有 ACC 网络引擎实现都在 ock::mmc 命名空间内
}
}
```
