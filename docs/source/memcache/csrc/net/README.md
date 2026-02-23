# net 模块文档

## 模块概述

`net` 模块提供 MemCache 项目的网络通信基础设施，支持 TCP、RDMA 等多种传输协议。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/net/`

## 文件列表

### 核心头文件
- `mmc_net_common.h` - 网络通用定义和类型
- `mmc_net_engine.h` - 网络引擎核心接口
- `mmc_net_link_map.h` - 并发安全链接映射

### 实现文件
- `mmc_net_engine.cpp` - 网络引擎工厂实现

### ACC Links 实现子目录 (`acc_links_impl/`)
- `mmc_net_common_acc.h` - ACC 网络通用定义
- `mmc_net_engine_acc.h` / `mmc_net_engine_acc.cpp` - ACC 网络引擎实现
- `mmc_net_ctx_store.h` - 上下文存储（扁平数组+哈希表两级结构）
- `mmc_net_ctx_acc.h` - ACC 网络上下文
- `mmc_net_link_acc.h` - ACC 网络链接
- `mmc_net_wait_handle.h` - RPC 等待处理器

---

## 详细文档

### mmc_net_common.h

**功能**: 定义网络通信的通用类型、枚举和结构

**主要内容**:
- `PROTOCOL_TCP`: TCP 协议前缀常量
- `NetProtoVersion`: 网络协议版本枚举
- `NetProtocol`: 网络协议类型枚举（TCP/RDMA/URMA/UDS/SHM）
- `NetEngineOptions`: 网络引擎配置选项结构
- `ExternalLog`: 外部日志函数类型
- 智能指针类型别名：`NetContextPtr`, `NetLinkPtr`, `NetEnginePtr`

**详细文档**: [mmc_net_common_h.md](mmc_net_common_h.md)

---

### mmc_net_engine.h

**功能**: 定义网络通信的核心抽象接口

**主要内容**:
- 回调函数类型：
  - `NetReqReceivedHandler`: 请求接收回调
  - `NetReqSentHandler`: 请求发送完成回调
  - `NetNewLinkHandler`: 新连接建立回调
  - `NetLinkBrokenHandler`: 连接断开回调
- `NetContext`: RPC 上下文类
  - `Reply()`: 响应请求（支持 POD 和序列化类型）
  - `GetRequest()`: 获取请求数据
  - 访问器：`SeqNo()`, `OpCode()`, `SrcRankId()`, `DataLen()`, `Data()`
- `NetLink`: 网络链接抽象类
  - `Id()`: 获取链接 ID
  - `UpCtx()`: 上下文值访问器
- `NetEngine`: 网络引擎核心类
  - `Start()`/`Stop()`: 生命周期管理
  - `ConnectToPeer()`: 连接管理
  - `Call()`: 同步 RPC 调用（模板方法，支持 POD/序列化类型）
  - `Send()`: 单向发送
  - `Reg*Handler()`: 回调注册

**详细文档**: [mmc_net_engine_h.md](mmc_net_engine_h.md)

---

### mmc_net_engine.cpp

**功能**: 实现网络引擎工厂方法

**主要内容**:
- `NetEngine::Create()`: 创建 NetEngineAcc 实例

**详细文档**: [mmc_net_engine_cpp.md](mmc_net_engine_cpp.md)

---

### mmc_net_link_map.h

**功能**: 并发安全的网络链接映射容器

**主要内容**:
- `NetLinkMap<LINK_PTR>`: 模板类，支持不同类型的链接指针
  - 分片锁设计（7 个桶）
  - `Find()`: 查找链接
  - `Add()`: 添加链接
  - `Remove()`: 移除链接
  - `Clear()`: 清空所有链接

**详细文档**: [mmc_net_link_map_h.md](mmc_net_link_map_h.md)

---

### acc_links_impl/mmc_net_common_acc.h

**功能**: ACC 网络实现的通用类型定义

**主要内容**:
- TCP 相关类型别名
- ACC 类智能指针类型别名
- `NetSeqNo`: 序列号联合体（32 位编码）
  - 位域：realSeq(24) + version(6) + fromFlat(1) + isResp(1)
  - `SetValue()`: 设置字段值
  - `ToString()`: 调试输出
  - `IsResp()`: 判断是否为响应

**详细文档**: [mmc_net_common_acc_h.md](mmc_net_common_acc_h.md)

---

### acc_links_impl/mmc_net_ctx_store.h

**功能**: 网络上下文存储，两级存储结构

**主要内容**:
- `NetContextStore`: 上下文存储类
  - 扁平数组（快速访问，CAS 无锁操作）
  - 哈希表（溢出处理）
  - 版本号机制（6 位版本，防 ABA 问题）
  - `Initialize()`/`UnInitialize()`: 生命周期
  - `PutAndGetSeqNo()`: 存储并获取序列号
  - `GetSeqNoAndRemove()`: 获取并移除
  - `RemoveSeqNo()`: 仅移除

**详细文档**: [mmc_net_ctx_store_h.md](mmc_net_ctx_store_h.md)

---

### acc_links_impl/mmc_net_ctx_acc.h

**功能**: ACC 网络上下文实现

**主要内容**:
- `NetContextAcc`: NetContext 的 ACC 实现
  - 包装底层的 `TcpReqContext`
  - `Reply()`: 响应请求
  - 访问器：`SeqNo()`, `OpCode()`, `SrcRankId()`, `DataLen()`, `Data()`

**详细文档**: [mmc_net_ctx_acc_h.md](mmc_net_ctx_acc_h.md)

---

### acc_links_impl/mmc_net_link_acc.h

**功能**: ACC 网络链接实现

**主要内容**:
- `NetLinkAcc`: NetLink 的 ACC 实现
  - `Id()`: 获取链接 ID
  - `RealLink()`: 获取底层 TCP 链接

**详细文档**: [mmc_net_link_acc_h.md](mmc_net_link_acc_h.md)

---

### acc_links_impl/mmc_net_wait_handle.h

**功能**: RPC 等待处理器，同步等待响应

**主要内容**:
- `NetWaitHandler`: 等待处理器类
  - 使用 pthread 条件变量
  - `Initialize()`: 初始化（使用 CLOCK_MONOTONIC）
  - `TimedWait()`: 等待响应或超时
  - `Notify()`: 通知等待线程
  - `GetResult()`/`Data()`: 获取结果和数据

**详细文档**: [mmc_net_wait_handle_h.md](mmc_net_wait_handle_h.md)

---

### acc_links_impl/mmc_net_engine_acc.h

**功能**: ACC 网络引擎头文件

**主要内容**:
- `NetEngineAcc`: NetEngine 的 ACC 实现
  - `Start()`/`Stop()`: 启动/停止引擎
  - `ConnectToPeer()`: 连接对端
  - `Call()`: 同步 RPC 调用
  - `Send()`: 单向发送
  - 私有方法：
    - `VerifyOptions()`: 验证配置
    - `Initialize()`/`UnInitialize()`: 初始化/清理
    - `StartInner()`/`StopInner()`: 内部启动/停止
    - TCP 回调：`HandleNewLink()`, `HandleNeqRequest()`, `HandleMsgSent()`, `HandleLinkBroken()`
    - `HandleAllRequests4Response()`: 响应处理
    - `RegisterDecryptHandler()`: 注册 TLS 解密器

**详细文档**: [mmc_net_engine_acc_h.md](mmc_net_engine_acc_h.md)

---

### acc_links_impl/mmc_net_engine_acc.cpp

**功能**: ACC 网络引擎实现

**主要内容**:
- `Start()`: 启动引擎（验证、初始化、启动线程池）
- `Stop()`: 停止引擎
- `VerifyOptions()`: 验证配置选项
- `StartInner()`/`StopInner()`: 内部启动/停止
- `Call()`: RPC 调用（查找链接、创建等待器、发送、等待响应）
- `Send()`: 单向发送（占位实现）
- `Initialize()`/`UnInitialize()`: 初始化/清理
- `RegisterTcpServerHandler()`: 注册 TCP 回调
- `HandleNewLink()`: 新链接处理
- `HandleNeqRequest()`: 请求处理（提交到线程池）
- `HandleLinkBroken()`: 链接断开处理
- `ConnectToPeer()`: 连接对端
- `HandleAllRequests4Response()`: 响应处理
- `RegisterDecryptHandler()`: 注册解密器
- `TraceSendRecord()`/`TraceSendWaitRecord()`: 性能追踪

**详细文档**: [mmc_net_engine_acc_cpp.md](mmc_net_engine_acc_cpp.md)

---

## 数据流图

```
客户端                          服务端
    |                              |
    | 1. 创建 NetWaitHandler       |
    | 2. 存储到 ctxStore           |
    | 3. 发送请求                  |
    |    ---------------------->    |
    |                              | 4. 接收请求
    |                              | 5. 提交到线程池处理
    |                              | 6. 处理业务逻辑
    |                              | 7. 发送响应
    |    <----------------------    |
    | 8. Notify() 唤醒             |
    | 9. 获取响应数据              |
    |                              |
```

---

## 类层次结构

```
NetEngine (抽象接口)
    |
    +-- NetEngineAcc (ACC 实现)
            |
            +-- TcpServer (ACC TCP 服务器)
            +-- NetLinkMapAcc (链接映射)
            +-- NetContextStore (上下文存储)
            +-- MmcThreadPool (线程池)

NetLink (抽象接口)
    |
    +-- NetLinkAcc (ACC 实现)
            |
            +-- TcpLink (ACC TCP 链接)

NetContext (抽象接口)
    |
    +-- NetContextAcc (ACC 实现)
            |
            +-- TcpReqContext (ACC TCP 请求上下文)

MmcReferable (引用计数基类)
    |
    +-- NetEngine
    +-- NetLink
    +-- NetContext
    +-- NetLinkMap
    +-- NetContextStore
    +-- NetWaitHandler
```

---

## 使用示例

### 创建网络引擎

```cpp
#include "mmc_net_engine.h"

// 创建引擎
auto engine = NetEngine::Create();

// 配置选项
NetEngineOptions options;
options.name = "my_engine";
options.ip = "0.0.0.0";
options.port = 5000;
options.threadCount = 4;
options.rankId = 0;
options.startListener = true;

// 启动引擎
Result ret = engine->Start(options);
if (ret != MMC_OK) {
    // 处理错误
}
```

### 注册请求处理器

```cpp
engine->RegRequestReceivedHandler(ML_ALLOC_REQ, [](NetContextPtr &ctx) {
    AllocRequest req;
    AllocResponse resp;

    // 获取请求数据
    ctx->GetRequest(req);

    // 处理业务逻辑...

    // 响应
    ctx->Reply(ML_ALLOC_RESP, resp);
    return MMC_OK;
});

// 注册链接事件处理器
engine->RegNewLinkHandler([](const NetLinkPtr &link) {
    MMC_LOG_INFO("New link established, id: " << link->Id());
    return MMC_OK;
});

engine->RegLinkBrokenHandler([](const NetLinkPtr &link) {
    MMC_LOG_WARN("Link broken, id: " << link->Id());
    return MMC_OK;
});
```

### 连接到对端

```cpp
NetLinkPtr link;
Result ret = engine->ConnectToPeer(peerId, "192.168.1.100", 5000, link, false);
if (ret != MMC_OK) {
    // 处理错误
}
```

### RPC 调用

```cpp
// POD 类型
struct AllocRequest {
    uint64_t size;
    uint32_t rankId;
};

struct AllocResponse {
    int32_t result;
    uint64_t address;
};

AllocRequest req{size, rankId};
AllocResponse resp;
Result ret = engine->Call<AllocRequest, AllocResponse>(peerId, ML_ALLOC_REQ, req, resp, 60);

// 非 POD 类型（需要实现 Serialize/Deserialize）
class ComplexRequest {
public:
    void Serialize(NetMsgPacker &packer) const {
        packer.PackInt32(value);
        packer.PackString(name);
    }
    void Deserialize(NetMsgUnpacker &unpacker) {
        value = unpacker.UnpackInt32();
        name = unpacker.UnpackString();
    }
private:
    int32_t value;
    std::string name;
};

ComplexRequest req;
ComplexResponse resp;
Result ret = engine->Call<ComplexRequest, ComplexResponse>(peerId, ML_COMPLEX_REQ, req, resp, 60);
```

### 单向发送

```cpp
NotificationRequest noti{event, data};
Result ret = engine->Send<NotificationRequest>(peerId, noti, 10);
```

### 停止引擎

```cpp
engine->Stop();
```

---

## 文档索引

| 文件 | 说明 |
|------|------|
| [mmc_net_common_h.md](mmc_net_common_h.md) | 网络通用定义 |
| [mmc_net_engine_h.md](mmc_net_engine_h.md) | 网络引擎接口 |
| [mmc_net_engine_cpp.md](mmc_net_engine_cpp.md) | 网络引擎工厂 |
| [mmc_net_link_map_h.md](mmc_net_link_map_h.md) | 链接映射 |
| [mmc_net_common_acc_h.md](mmc_net_common_acc_h.md) | ACC 通用定义 |
| [mmc_net_ctx_store_h.md](mmc_net_ctx_store_h.md) | 上下文存储 |
| [mmc_net_ctx_acc_h.md](mmc_net_ctx_acc_h.md) | ACC 上下文 |
| [mmc_net_link_acc_h.md](mmc_net_link_acc_h.md) | ACC 链接 |
| [mmc_net_wait_handle_h.md](mmc_net_wait_handle_h.md) | 等待处理器 |
| [mmc_net_engine_acc_h.md](mmc_net_engine_acc_h.md) | ACC 网络引擎头文件 |
| [mmc_net_engine_acc_cpp.md](mmc_net_engine_acc_cpp.md) | ACC 网络引擎实现 |
