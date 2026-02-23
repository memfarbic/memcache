# mmc_net_engine_acc.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/acc_links_impl/mmc_net_engine_acc.h`
- **文件用途**: 定义 ACC 网络引擎实现类
- **依赖项**:
  - `mmc_net_engine.h` - 网络引擎接口
  - `mmc_net_common_acc.h` - ACC 网络通用定义
  - `mmc_thread_pool.h` - 线程池
  - `mmc_net_ctx_store.h` - 上下文存储

---

## 类定义

### NetEngineAcc 类

ACC 网络引擎实现，基于华为 acc_links 框架

```cpp
class NetEngineAcc final : public NetEngine {
public:
    ~NetEngineAcc() override;

    Result Start(const NetEngineOptions &options) override;
    void Stop() override;

    Result ConnectToPeer(uint32_t peerId, const std::string &peerIp, uint16_t port, NetLinkPtr &newLink,
                         bool isForce) override;

    Result Call(uint32_t targetId, int16_t opCode, const char *reqData, uint32_t reqDataLen, char **respData,
                uint32_t &respDataLen, int32_t timeoutInSecond) override;

    Result Send(uint32_t peerId, const char *reqData, uint32_t reqDataLen, int32_t timeoutInSecond) override;

private:
    static Result VerifyOptions(const NetEngineOptions &options);
    Result Initialize(const NetEngineOptions &options);
    void UnInitialize();
    Result StartInner();
    Result StopInner();

    /* callback function of tcp server */
    Result RegisterTcpServerHandler();
    Result HandleNewLink(const TcpConnReq &req, const TcpLinkPtr &link) const;
    Result HandleNeqRequest(const TcpReqContext &context);
    Result HandleMsgSent(TcpMsgSentResult result, const TcpMsgHeader &header, const TcpDataBufPtr &cbCtx);
    Result HandleLinkBroken(const TcpLinkPtr &link) const;
    Result HandleAllRequests4Response(const TcpReqContext &context);

    Result RegisterDecryptHandler(const std::string &decryptLibPath) const;

private:
    /* hot used variables */
    TcpServerPtr server_;
    NetLinkMapAccPtr peerLinkMap_;
    NetContextStorePtr ctxStore_;

    /* not hot used variables */
    NetEngineOptions options_{};
    bool started_ = false;
    bool inited_ = false;
    std::mutex connectMutex_;
    int32_t timeoutInSecond_ = 0;
    MmcThreadPoolPtr threadPool_;
};
```

**设计说明**:
- `final` 关键字表示此类不能被继承
- 成员变量按使用热度排列，热变量放在前面
- 实现了 NetEngine 接口的所有纯虚函数

---

## 公共方法

### NetEngineAcc::~NetEngineAcc()

```cpp
~NetEngineAcc() override;
```

**声明位置**: 行 24

**功能描述**: 析构函数，自动调用 Stop 清理资源

---

### NetEngineAcc::Start()

```cpp
Result Start(const NetEngineOptions &options) override;
```

**声明位置**: 行 26

**功能描述**: 启动网络引擎

**参数**: `options` - 网络引擎配置选项

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 30-71

---

### NetEngineAcc::Stop()

```cpp
void Stop() override;
```

**声明位置**: 行 27

**功能描述**: 停止网络引擎

**实现位置**: `mmc_net_engine_acc.cpp` 行 73-88

---

### NetEngineAcc::ConnectToPeer()

```cpp
Result ConnectToPeer(uint32_t peerId, const std::string &peerIp, uint16_t port, NetLinkPtr &newLink,
                     bool isForce) override;
```

**声明位置**: 行 29-30

**功能描述**: 连接到对端节点

**参数**:
- `peerId`: 对端节点 ID
- `peerIp`: 对端 IP 地址
- `port`: 对端端口
- `newLink`: 输出参数，新建的链接
- `isForce`: 是否强制重新连接

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 507-554

---

### NetEngineAcc::Call()

```cpp
Result Call(uint32_t targetId, int16_t opCode, const char *reqData, uint32_t reqDataLen, char **respData,
            uint32_t &respDataLen, int32_t timeoutInSecond) override;
```

**声明位置**: 行 32-33

**功能描述**: 同步 RPC 调用

**参数**:
- `targetId`: 目标节点 ID
- `opCode`: 操作码
- `reqData`: 请求数据
- `reqDataLen`: 请求数据长度
- `respData`: 输出参数，响应数据（需要调用者释放）
- `respDataLen`: 输出参数，响应数据长度
- `timeoutInSecond`: 超时时间（秒）

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 257-332

---

### NetEngineAcc::Send()

```cpp
Result Send(uint32_t peerId, const char *reqData, uint32_t reqDataLen, int32_t timeoutInSecond) override;
```

**声明位置**: 行 35

**功能描述**: 单向发送（不需要响应）

**参数**:
- `peerId`: 对端节点 ID
- `reqData`: 请求数据
- `reqDataLen`: 请求数据长度
- `timeoutInSecond`: 超时时间（秒）

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 334-341

---

## 私有方法

### NetEngineAcc::VerifyOptions()

```cpp
static Result VerifyOptions(const NetEngineOptions &options);
```

**声明位置**: 行 38

**功能描述**: 验证配置选项的有效性

**参数**: `options` - 配置选项

**返回值**: `Result` - MMC_OK 表示有效

**实现位置**: `mmc_net_engine_acc.cpp` 行 90-116

---

### NetEngineAcc::Initialize()

```cpp
Result Initialize(const NetEngineOptions &options);
```

**声明位置**: 行 39

**功能描述**: 初始化网络引擎

**参数**: `options` - 配置选项

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 348-382

---

### NetEngineAcc::UnInitialize()

```cpp
void UnInitialize();
```

**声明位置**: 行 40

**功能描述**: 清理网络引擎资源

**实现位置**: `mmc_net_engine_acc.cpp` 行 384-408

---

### NetEngineAcc::StartInner()

```cpp
Result StartInner();
```

**声明位置**: 行 41

**功能描述**: 内部启动逻辑

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 118-159

---

### NetEngineAcc::StopInner()

```cpp
Result StopInner();
```

**声明位置**: 行 42

**功能描述**: 内部停止逻辑

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 161-170

---

### NetEngineAcc::RegisterTcpServerHandler()

```cpp
Result RegisterTcpServerHandler();
```

**声明位置**: 行 45

**功能描述**: 注册 TCP 服务器回调处理器

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 410-424

---

### NetEngineAcc::HandleNewLink()

```cpp
Result HandleNewLink(const TcpConnReq &req, const TcpLinkPtr &link) const;
```

**声明位置**: 行 46

**功能描述**: 处理新链接建立事件

**参数**:
- `req`: 连接请求
- `link`: 新建立的链接

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 426-445

---

### NetEngineAcc::HandleNeqRequest()

```cpp
Result HandleNeqRequest(const TcpReqContext &context);
```

**声明位置**: 行 47

**功能描述**: 处理接收到的请求

**参数**: `context` - 请求上下文

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 447-479

---

### NetEngineAcc::HandleMsgSent()

```cpp
Result HandleMsgSent(TcpMsgSentResult result, const TcpMsgHeader &header, const TcpDataBufPtr &cbCtx);
```

**声明位置**: 行 48

**功能描述**: 处理消息发送完成事件

**参数**:
- `result`: 发送结果
- `header`: 消息头
- `cbCtx`: 回调上下文

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 481-484

---

### NetEngineAcc::HandleLinkBroken()

```cpp
Result HandleLinkBroken(const TcpLinkPtr &link) const;
```

**声明位置**: 行 49

**功能描述**: 处理链接断开事件

**参数**: `link` - 断开的链接

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 486-505

---

### NetEngineAcc::HandleAllRequests4Response()

```cpp
Result HandleAllRequests4Response(const TcpReqContext &context);
```

**声明位置**: 行 50

**功能描述**: 处理响应消息（客户端）

**参数**: `context` - 请求上下文

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 556-590

---

### NetEngineAcc::RegisterDecryptHandler()

```cpp
Result RegisterDecryptHandler(const std::string &decryptLibPath) const;
```

**声明位置**: 行 52

**功能描述**: 注册 TLS 解密处理器

**参数**: `decryptLibPath` - 解密库路径

**返回值**: `Result` - MMC_OK 表示成功

**实现位置**: `mmc_net_engine_acc.cpp` 行 592-608

---

## 成员变量

```cpp
private:
    /* hot used variables */
    TcpServerPtr server_;
    NetLinkMapAccPtr peerLinkMap_;
    NetContextStorePtr ctxStore_;

    /* not hot used variables */
    NetEngineOptions options_{};
    bool started_ = false;
    bool inited_ = false;
    std::mutex connectMutex_;
    int32_t timeoutInSecond_ = 0;
    MmcThreadPoolPtr threadPool_;
```

**热变量**:
- `server_`: TCP 服务器指针
- `peerLinkMap_`: 对端链接映射表
- `ctxStore_`: 上下文存储

**冷变量**:
- `options_`: 配置选项
- `started_`: 是否已启动
- `inited_`: 是否已初始化
- `connectMutex_`: 连接互斥锁
- `timeoutInSecond_`: 超时时间
- `threadPool_`: 线程池

---

## 文件级别的关系图

```
mmc_net_engine_acc.h (ACC 网络引擎)
    |
    +-- NetEngineAcc -> NetEngine 的 ACC 实现
            |
            +-- Start() -> 启动引擎
            +-- Stop() -> 停止引擎
            +-- ConnectToPeer() -> 连接对端
            +-- Call() -> 同步 RPC 调用
            +-- Send() -> 单向发送
            |
            +-- VerifyOptions() -> 验证配置
            +-- Initialize() -> 初始化
            +-- UnInitialize() -> 清理
            +-- StartInner() -> 内部启动
            +-- StopInner() -> 内部停止
            |
            +-- RegisterTcpServerHandler() -> 注册回调
            +-- HandleNewLink() -> 新链接处理
            +-- HandleNeqRequest() -> 请求处理
            +-- HandleMsgSent() -> 发送完成处理
            +-- HandleLinkBroken() -> 链接断开处理
            +-- HandleAllRequests4Response() -> 响应处理
            +-- RegisterDecryptHandler() -> 注册解密器
```

---

## 依赖关系

**依赖以下文件**:
- `mmc_net_engine.h` - 网络引擎接口
- `mmc_net_common_acc.h` - ACC 网络通用定义
- `mmc_thread_pool.h` - 线程池
- `mmc_net_ctx_store.h` - 上下文存储

**被以下文件依赖**:
- `mmc_net_engine.cpp` - 网络引擎工厂

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有 ACC 网络引擎定义都在 ock::mmc 命名空间内
}
}
```

---

## 生命周期

```
     Create()
         |
         v
    Initialize()
         |
         v
    StartInner()
         |
         v
    [运行中] <----> ConnectToPeer() / Call() / Send()
         |
         v
    StopInner()
         |
         v
    UnInitialize()
         |
         v
    Destroy
```
