# mmc_net_common_acc.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/acc_links_impl/mmc_net_common_acc.h`
- **文件用途**: 定义 ACC (华为 acc_links) 网络实现的通用类型和序列号结构
- **依赖项**:
  - `acc_tcp_server.h` - ACC TCP 服务器
  - `mmc_common_includes.h` - 公共头文件
  - `mmc_net_link_map.h` - 链接映射

---

## 类型别名

### TCP 相关类型别名

```cpp
using TcpServerOptions = ock::acc::AccTcpServerOptions;
using TcpServerPtr = ock::acc::AccTcpServerPtr;
using TcpServer = ock::acc::AccTcpServer;
using TcpLinkPtr = ock::acc::AccTcpLinkComplexPtr;
using TcpReqContext = ock::acc::AccTcpRequestContext;
using TcpMsgSentResult = ock::acc::AccMsgSentResult;
using TcpMsgHeader = ock::acc::AccMsgHeader;
using TcpDataBufPtr = ock::acc::AccDataBufferPtr;
using TcpConnReq = ock::acc::AccConnReq;
using TcpConnResp = ock::acc::AccConnResp;
using TcpNewReqHandler = ock::acc::AccNewReqHandler;
using TcpReqSentHandler = ock::acc::AccReqSentHandler;
using TcpLinkBrokenHandler = ock::acc::AccLinkBrokenHandler;
using TcpNewLinkHandler = ock::acc::AccNewLinkHandler;
using TcpTlsOption = ock::acc::AccTlsOption;
```

**说明**: 将 ACC 框架的类型别名为简短名称，便于使用

---

### 前置声明

```cpp
class NetLinkAcc;
class NetWaitHandler;
class NetContextStore;
class NetEngineAcc;
```

**说明**: ACC 网络实现类的前置声明

---

### 智能指针类型别名

```cpp
using NetWaitHandlerPtr = MmcRef<NetWaitHandler>;
using NetLinkAccPtr = MmcRef<NetLinkAcc>;
using NetContextStorePtr = MmcRef<NetContextStore>;
using NetEngineAccPtr = MmcRef<NetEngineAcc>;

using NetLinkMapAcc = NetLinkMap<NetLinkAccPtr>;
using NetLinkMapAccPtr = MmcRef<NetLinkMapAcc>;
```

**说明**: ACC 实现类的智能指针类型别名

---

## 常量

### MSG_TYPE_DATA / MSG_TYPE_CTRL

```cpp
constexpr int16_t MSG_TYPE_DATA = 0;
constexpr int16_t MSG_TYPE_CTRL = 1;
```

**说明**: 消息类型枚举
- `MSG_TYPE_DATA`: 数据消息
- `MSG_TYPE_CTRL`: 控制消息

---

## 联合体

### NetSeqNo

网络序列号联合体，用于编码/解码序列号

```cpp
union NetSeqNo {
    struct {
        /* low address */
        uint32_t realSeq : 24; /* real seq no */
        uint32_t version : 6;  /* request version */
        uint32_t fromFlat : 1; /* allocated from flat or hash map */
        uint32_t isResp : 1;   /* request or reply, 0 for request, 1 for reply */
        /* high address */
    };
    uint32_t wholeSeq = 0;

    explicit NetSeqNo(uint32_t whole) : wholeSeq(whole) {}

    void SetValue(uint32_t flat, uint32_t ver, uint32_t seq);
    std::string ToString() const;
    bool IsResp() const;
};
```

**位域说明**:
- `realSeq` (24 位): 真实序列号，支持约 1600 万个并发请求
- `version` (6 位): 版本号，支持 64 个版本轮回
- `fromFlat` (1 位): 是否从扁平数组分配（1=扁平数组，0=哈希表）
- `isResp` (1 位): 是否为响应（1=响应，0=请求）

**设计说明**:
- 使用 32 位整数编码多个信息，节省存储空间
- 支持版本机制，可以复用序列号槽位
- 区分请求和响应，方便双向通信

---

### NetSeqNo::SetValue()

```cpp
inline void NetSeqNo::SetValue(uint32_t flat, uint32_t ver, uint32_t seq)
{
    fromFlat = flat;
    version = ver;
    realSeq = seq;
}
```

**声明位置**: 行 74-79

**功能描述**: 设置序列号各个字段的值

**参数**:
- `flat`: 是否从扁平数组分配 (0 或 1)
- `ver`: 版本号
- `seq`: 真实序列号

---

### NetSeqNo::ToString()

```cpp
inline std::string NetSeqNo::ToString() const
{
    std::ostringstream oss;
    oss << "NetSeqNo info=[wholeSeq: " << wholeSeq << ", isResp: " << isResp << ", fromFlat: " << fromFlat
        << ", version: " << version << ", realSeq: " << realSeq << "]";
    return oss.str();
}
```

**声明位置**: 行 81-87

**功能描述**: 将序列号转换为字符串，用于调试和日志

**返回值**: 包含序列号各字段信息的字符串

---

### NetSeqNo::IsResp()

```cpp
inline bool NetSeqNo::IsResp() const
{
    return isResp == 1L;
}
```

**声明位置**: 行 89-92

**功能描述**: 判断是否为响应序列号

**返回值**: `bool` - true 表示是响应，false 表示是请求

---

## 文件级别的关系图

```
mmc_net_common_acc.h (ACC 网络通用定义)
    |
    +-- TCP 类型别名 -> ACC TCP 框架类型
    +-- ACC 类智能指针 -> NetLinkAccPtr, NetEngineAccPtr 等
    |
    +-- NetSeqNo -> 序列号联合体
            |
            +-- SetValue() -> 设置字段值
            +-- ToString() -> 调试输出
            +-- IsResp() -> 判断是否为响应
            |
            +-> 位域: realSeq(24) + version(6) + fromFlat(1) + isResp(1)
```

---

## 依赖关系

**依赖以下文件**:
- `acc_tcp_server.h` - ACC TCP 服务器框架
- `mmc_common_includes.h` - 公共头文件
- `mmc_net_link_map.h` - 链接映射

**被以下文件依赖**:
- `mmc_net_ctx_store.h` - 上下文存储
- `mmc_net_wait_handle.h` - 等待处理器
- `mmc_net_link_acc.h` - ACC 链接
- `mmc_net_ctx_acc.h` - ACC 上下文
- `mmc_net_engine_acc.h` - ACC 网络引擎

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有 ACC 网络通用定义都在 ock::mmc 命名空间内
}
}
```

---

## 序列号设计说明

NetSeqNo 采用位域设计，在一个 32 位整数中编码多个信息：

```
位31             位24 位23             位18 位17      位16 位15                                   位0
+------------------+-------------------+------------+------------+----------------------------------+
|   realSeq (24)   |   version (6)     | fromFlat(1)|  isResp(1) |                                  |
+------------------+-------------------+------------+------------+----------------------------------+
```

**特点**:
1. **realSeq (24 位)**: 支持约 16M 个并发请求
2. **version (6 位)**: 版本号 0-63，用于复用序列号槽
3. **fromFlat (1 位)**: 标识存储位置，快速判断访问路径
4. **isResp (1 位)**: 区分请求和响应

**使用场景**:
- 请求发送时生成序列号存储上下文
- 响应返回时根据序列号查找原始上下文
- 超时后根据序列号清理上下文
