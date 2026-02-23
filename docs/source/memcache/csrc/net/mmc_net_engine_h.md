# mmc_net_engine.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/mmc_net_engine.h`
- **文件用途**: 定义网络通信的核心抽象接口，包括 NetContext、NetLink 和 NetEngine 类
- **依赖项**:
  - `mmc_common_includes.h` - 公共头文件
  - `mmc_net_common.h` - 网络通用定义
  - `mmc_msg_packer.h` - 消息序列化/反序列化

---

## 回调函数类型定义

### NetReqReceivedHandler

```cpp
using NetReqReceivedHandler = std::function<int32_t(NetContextPtr &ctx)>;
```

**说明**: 请求接收回调函数类型

**参数**: `NetContextPtr &ctx` - 网络上下文智能指针引用

**返回值**: `int32_t` - 处理结果状态码

**用途**: 当接收到网络请求时调用此回调

---

### NetReqSentHandler

```cpp
using NetReqSentHandler = std::function<int32_t(NetContextPtr &ctx)>;
```

**说明**: 请求发送完成回调函数类型

**参数**: `NetContextPtr &ctx` - 网络上下文智能指针引用

**返回值**: `int32_t` - 处理结果状态码

**用途**: 当请求发送完成后调用此回调

---

### NetNewLinkHandler

```cpp
using NetNewLinkHandler = std::function<int32_t(const NetLinkPtr &link)>;
```

**说明**: 新连接建立回调函数类型

**参数**: `const NetLinkPtr &link` - 新建立的链接智能指针引用

**返回值**: `int32_t` - 处理结果状态码

**用途**: 当新的网络连接建立时调用此回调

---

### NetLinkBrokenHandler

```cpp
using NetLinkBrokenHandler = std::function<int32_t(const NetLinkPtr &link)>;
```

**说明**: 连接断开回调函数类型

**参数**: `const NetLinkPtr &link` - 断开的链接智能指针引用

**返回值**: `int32_t` - 处理结果状态码

**用途**: 当网络连接断开时调用此回调

---

## 类定义

### NetContext 类

网络上下文类，表示一次 RPC 调用的上下文信息

```cpp
class NetContext : public MmcReferable {
public:
    ~NetContext() override = default;

    virtual int32_t Reply(int16_t responseCode, const char *respData, uint32_t &respDataLen) = 0;

    virtual uint32_t SeqNo() const = 0;
    virtual int16_t OpCode() const = 0;
    virtual int16_t SrcRankId() const = 0;
    virtual uint32_t DataLen() const = 0;
    virtual void *Data() const = 0;

    template<typename RESP>
    int32_t Reply(int16_t opCode, RESP &resp);

    template<typename REQ>
    int32_t GetRequest(REQ &req);
};
```

---

#### NetContext::Reply() (模板方法 - POD版本)

```cpp
template<typename RESP>
int32_t Reply(int16_t opCode, RESP &resp)
{
    if (std::is_pod<RESP>::value) {
        uint32_t retSize = sizeof(RESP);
        return Reply(opCode, (char *)(&resp), retSize);
    } else {
        NetMsgPacker packer;
        resp.Serialize(packer);
        std::string serializedData = packer.String();
        uint32_t retSize = serializedData.length();
        return Reply(opCode, serializedData.c_str(), retSize);
    }
}
```

**声明位置**: 行 85-98

**功能描述**: 类型安全的响应方法（模板版本）

**参数**:
- `opCode`: 响应操作码
- `resp`: 响应数据对象引用

**返回值**: `int32_t` - 发送结果状态码

**代码逻辑**:
1. 检查响应类型是否为 POD (Plain Old Data) 类型
2. 如果是 POD 类型，直接发送内存内容
3. 如果不是 POD 类型，使用 NetMsgPacker 序列化后发送

**使用示例**:
```cpp
// POD 类型
struct AllocResponse {
    int32_t result;
    uint64_t address;
};
AllocResponse resp{0, 0x1000};
ctx->Reply(ML_ALLOC_RESP, resp);

// 非 POD 类型（需要实现 Serialize 方法）
class ComplexResponse {
public:
    void Serialize(NetMsgPacker &packer) const {
        packer.PackInt32(result);
        packer.PackString(message);
    }
private:
    int32_t result;
    std::string message;
};
ComplexResponse resp;
ctx->Reply(ML_COMPLEX_RESP, resp);
```

---

#### NetContext::GetRequest()

```cpp
template<typename REQ>
int32_t GetRequest(REQ &req)
{
    if (std::is_pod<REQ>::value) {
        std::copy_n(reinterpret_cast<char *>(Data()), DataLen(), reinterpret_cast<char *>(&req));
    } else {
        std::string str{(char *)Data(), DataLen()};
        NetMsgUnpacker unpacker(str);
        req.Deserialize(unpacker);
    }
    return MMC_OK;
}
```

**声明位置**: 行 100-111

**功能描述**: 从网络上下文中获取请求数据并解析到指定类型

**参数**:
- `req`: 输出参数，解析后的请求对象

**返回值**: `int32_t` - MMC_OK 表示成功

**代码逻辑**:
1. 检查请求类型是否为 POD 类型
2. 如果是 POD 类型，直接内存拷贝
3. 如果不是 POD 类型，使用 NetMsgUnpacker 反序列化

**使用示例**:
```cpp
// POD 类型
struct AllocRequest {
    uint64_t size;
    uint32_t rankId;
};
AllocRequest req;
ctx->GetRequest(req);

// 非 POD 类型（需要实现 Deserialize 方法）
class ComplexRequest {
public:
    void Deserialize(NetMsgUnpacker &unpacker) {
        result = unpacker.UnpackInt32();
        message = unpacker.UnpackString();
    }
private:
    int32_t result;
    std::string message;
};
ComplexRequest req;
ctx->GetRequest(req);
```

---

### NetLink 类

网络链接抽象类

```cpp
class NetLink : public MmcReferable {
public:
    virtual int32_t Id() const = 0;

    uint64_t UpCtx() const;
    void UpCtx(const uint64_t c);

private:
    uint64_t upCtx_ = 0;
};
```

---

#### NetLink::UpCtx() const

```cpp
uint64_t UpCtx() const
{
    return upCtx_;
}
```

**声明位置**: 行 128-131

**功能描述**: 获取与链接关联的上下文值

**返回值**: `uint64_t` - 用户自定义的上下文值

**用途**: 允许用户在链接上存储自定义数据（如 peerId）

---

#### NetLink::UpCtx(const uint64_t c)

```cpp
void UpCtx(const uint64_t c)
{
    upCtx_ = c;
}
```

**声明位置**: 行 138-141

**功能描述**: 设置与链接关联的上下文值

**参数**: `c` - 要设置的上下文值

---

### NetEngine 类

网络引擎核心类，提供 RPC 通信功能

```cpp
class NetEngine : public MmcReferable {
public:
    static NetEnginePtr Create();

    ~NetEngine() override = default;

    virtual Result Start(const NetEngineOptions &options) = 0;
    virtual void Stop() = 0;

    virtual Result ConnectToPeer(uint32_t peerId, const std::string &peerIp, uint16_t port, NetLinkPtr &newLink,
                                 bool isForce) = 0;

    void RegRequestReceivedHandler(int16_t opCode, const NetReqReceivedHandler &h);
    void RegRequestSentHandler(int16_t opCode, const NetReqSentHandler &h);
    void RegNewLinkHandler(const NetNewLinkHandler &h);
    void RegLinkBrokenHandler(const NetLinkBrokenHandler &h);

    template<typename REQ, typename RESP>
    Result Call(uint32_t peerId, int16_t opCode, const REQ &req, RESP &resp, int32_t timeoutInSecond);

    template<typename REQ>
    Result Send(uint32_t peerId, const REQ &req, int32_t timeoutInSecond);

    virtual Result Call(uint32_t targetId, int16_t opCode, const char *reqData, uint32_t reqDataLen, char **respData,
                        uint32_t &respDataLen, int32_t timeoutInSecond) = 0;

    virtual Result Send(uint32_t peerId, const char *reqData, uint32_t reqDataLen, int32_t timeoutInSecond) = 0;

protected:
    constexpr static int16_t gHandlerMax = UN32;
    constexpr static int16_t gHandlerMin = 0;

protected:
    int16_t gHandlerSize = 0;
    NetReqReceivedHandler reqReceivedHandlers_[gHandlerMax];
    NetReqSentHandler reqSentHandlers_[gHandlerMax];
    NetNewLinkHandler newLinkHandler_ = nullptr;
    NetLinkBrokenHandler linkBrokenHandler_ = nullptr;

    std::mutex mutex_;
};
```

---

#### NetEngine::RegRequestReceivedHandler()

```cpp
inline void NetEngine::RegRequestReceivedHandler(int16_t opCode, const NetReqReceivedHandler &h)
{
    MMC_ASSERT_RET_VOID(opCode >= 0 && opCode < gHandlerMax);

    std::lock_guard<std::mutex> guard(mutex_);
    reqReceivedHandlers_[opCode] = h;
    gHandlerSize++;
}
```

**声明位置**: 行 352-359

**功能描述**: 注册请求接收回调处理器

**参数**:
- `opCode`: 操作码，用于标识请求类型
- `h`: 回调处理器函数

**代码逻辑**:
1. 验证操作码在有效范围内
2. 加锁保护
3. 将处理器存入数组
4. 增加处理器计数

**使用示例**:
```cpp
engine->RegRequestReceivedHandler(ML_ALLOC_REQ, [](NetContextPtr &ctx) {
    AllocRequest req;
    AllocResponse resp;
    ctx->GetRequest(req);
    // 处理请求...
    ctx->Reply(ML_ALLOC_RESP, resp);
    return MMC_OK;
});
```

---

#### NetEngine::RegRequestSentHandler()

```cpp
inline void NetEngine::RegRequestSentHandler(int16_t opCode, const NetReqSentHandler &h)
{
    MMC_ASSERT_RET_VOID(h != nullptr);
    MMC_ASSERT_RET_VOID(opCode >= 0 && opCode < gHandlerMax);

    std::lock_guard<std::mutex> guard(mutex_);
    reqSentHandlers_[opCode] = h;
}
```

**声明位置**: 行 361-368

**功能描述**: 注册请求发送完成回调处理器

**参数**:
- `opCode`: 操作码
- `h`: 回调处理器函数

**代码逻辑**:
1. 验证处理器非空
2. 验证操作码在有效范围内
3. 加锁保护
4. 将处理器存入数组

---

#### NetEngine::RegNewLinkHandler()

```cpp
inline void NetEngine::RegNewLinkHandler(const NetNewLinkHandler &h)
{
    MMC_ASSERT_RET_VOID(h != nullptr);

    std::lock_guard<std::mutex> guard(mutex_);
    newLinkHandler_ = h;
}
```

**声明位置**: 行 370-376

**功能描述**: 注册新连接建立回调处理器

**参数**: `h` - 回调处理器函数

**使用示例**:
```cpp
engine->RegNewLinkHandler([](const NetLinkPtr &link) {
    MMC_LOG_INFO("New link established, id: " << link->Id());
    return MMC_OK;
});
```

---

#### NetEngine::RegLinkBrokenHandler()

```cpp
inline void NetEngine::RegLinkBrokenHandler(const NetLinkBrokenHandler &h)
{
    MMC_ASSERT_RET_VOID(h != nullptr);

    std::lock_guard<std::mutex> guard(mutex_);
    linkBrokenHandler_ = h;
}
```

**声明位置**: 行 378-384

**功能描述**: 注册连接断开回调处理器

**参数**: `h` - 回调处理器函数

---

#### NetEngine::Call() (模板方法)

```cpp
template<typename REQ, typename RESP>
Result Call(uint32_t peerId, int16_t opCode, const REQ &req, RESP &resp, int32_t timeoutInSecond)
{
    char *respData = nullptr;
    uint32_t respLen = 0;
    if (std::is_pod<REQ>::value && std::is_pod<RESP>::value) {
        /* do call */
        respLen = sizeof(RESP);
        respData = reinterpret_cast<char *>(&resp);
        return Call(peerId, opCode, reinterpret_cast<char *>(const_cast<REQ *>(&req)), sizeof(REQ), &respData,
                    respLen, timeoutInSecond);
    } else if (std::is_pod<REQ>::value && !std::is_pod<RESP>::value) {
        /* do call */
        respLen = UINT32_MAX;
        respData = nullptr;
        auto result = Call(peerId, opCode, reinterpret_cast<char *>(const_cast<REQ *>(&req)), sizeof(REQ),
                           &respData, respLen, timeoutInSecond);
        MMC_RETURN_ERROR(result, "NetEngine call error, op " << opCode << ", peerId " << peerId);

        /* deserialize */
        std::string respStr(respData, respLen);
        NetMsgUnpacker unpacker(respStr);
        result = resp.Deserialize(unpacker);
        if (respData != nullptr) {
            free(respData);
        }
        MMC_RETURN_ERROR(result, "deserialize failed");

        return result;
    } else if (!std::is_pod<REQ>::value && std::is_pod<RESP>::value) {
        /* serialize request */
        NetMsgPacker packer;
        req.Serialize(packer);
        std::string serializedData = packer.String();

        /* do call */
        respLen = sizeof(RESP);
        respData = reinterpret_cast<char *>(&resp);
        return Call(peerId, opCode, serializedData.c_str(), serializedData.length(), &respData, respLen,
                    timeoutInSecond);
    } else {
        NetMsgPacker packer;
        req.Serialize(packer);
        std::string serializedData = packer.String();

        /* do call */
        respLen = sizeof(RESP);
        respData = nullptr;
        Result result = Call(peerId, opCode, serializedData.c_str(), serializedData.length(), &respData, respLen,
                             timeoutInSecond);
        MMC_RETURN_ERROR(result, "NetEngine call error, op " << opCode << ", peerId " << peerId);

        /* deserialize */
        std::string respStr(respData, respLen);
        if (respData != nullptr) {
            free(respData);
        }
        NetMsgUnpacker unpacker(respStr);
        result = resp.Deserialize(unpacker);
        MMC_RETURN_ERROR(result, "deserialize failed");

        return result;
    }
}
```

**声明位置**: 行 222-285

**功能描述**: 类型安全的同步 RPC 调用（模板方法）

**参数**:
- `peerId`: 对端节点 ID
- `opCode`: 操作码
- `req`: 请求数据
- `resp`: 输出参数，响应数据
- `timeoutInSecond`: 超时时间（秒）

**返回值**: `Result` - 调用结果

**代码逻辑**:

支持四种组合的处理方式：

1. **POD REQ + POD RESP**: 直接内存拷贝，零开销
2. **POD REQ + 非 POD RESP**: 请求直接拷贝，响应反序列化
3. **非 POD REQ + POD RESP**: 请求序列化，响应直接拷贝
4. **非 POD REQ + 非 POD RESP**: 请求和响应都序列化

**使用示例**:
```cpp
AllocRequest req{size, rankId};
AllocResponse resp;
Result ret = engine->Call<AllocRequest, AllocResponse>(
    peerId, ML_ALLOC_REQ, req, resp, 60);
```

---

#### NetEngine::Send() (模板方法)

```cpp
template<typename REQ>
Result Send(uint32_t peerId, const REQ &req, int32_t timeoutInSecond)
{
    if (std::is_pod<REQ>::value) {
        /* do send */
        return Send(peerId, static_cast<char *>(req), sizeof(REQ), timeoutInSecond);
    } else {
        /* serialize request */
        NetMsgPacker packer;
        auto result = packer.Serialize(req);
        const std::string serializedData = packer.String();

        /* do send */
        return Send(peerId, serializedData.c_str(), serializedData.length(), timeoutInSecond);
    }
}
```

**声明位置**: 行 296-311

**功能描述**: 类型安全的单向发送（不需要响应）

**参数**:
- `peerId`: 对端节点 ID
- `req`: 请求数据
- `timeoutInSecond`: 超时时间（秒）

**返回值**: `Result` - 发送结果

**使用示例**:
```cpp
NotificationRequest noti{event, data};
Result ret = engine->Send<NotificationRequest>(peerId, noti, 10);
```

---

## 文件级别的关系图

```
mmc_net_engine.h (网络引擎核心接口)
    |
    +-- NetReqReceivedHandler -> 请求接收回调类型
    +-- NetReqSentHandler -> 请求发送完成回调类型
    +-- NetNewLinkHandler -> 新连接回调类型
    +-- NetLinkBrokenHandler -> 连接断开回调类型
    |
    +-- NetContext -> RPC 上下文
    |       |
    |       +-- Reply() -> 响应请求
    |       +-- GetRequest() -> 获取请求数据
    |       +-- SeqNo/OpCode/SrcRankId/DataLen/Data -> 访问器
    |
    +-- NetLink -> 网络链接
    |       |
    |       +-- Id() -> 链接 ID
    |       +-- UpCtx() -> 上下文值
    |
    +-- NetEngine -> 网络引擎
            |
            +-- Start/Stop -> 生命周期
            +-- ConnectToPeer -> 连接管理
            +-- Call/Send -> RPC 调用
            +-- Reg*Handler -> 回调注册
```

---

## 依赖关系

**依赖以下文件**:
- `mmc_common_includes.h` - 公共头文件
- `mmc_net_common.h` - 网络通用定义
- `mmc_msg_packer.h` - 消息序列化

**被以下文件依赖**:
- `mmc_net_engine.cpp` - 网络引擎实现
- `mmc_net_engine_acc.h` - ACC 网络引擎实现
- `mmc_net_ctx_acc.h` - ACC 上下文实现
- `mmc_net_link_acc.h` - ACC 链接实现

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有网络引擎定义都在 ock::mmc 命名空间内
}
}
```
