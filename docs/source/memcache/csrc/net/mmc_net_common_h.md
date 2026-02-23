# mmc_net_common.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/mmc_net_common.h`
- **文件用途**: 定义网络通信模块的通用类型、枚举、结构和辅助函数
- **依赖项**:
  - `mmc_common_includes.h` - 公共头文件
  - `mf_ipv4_validator.h` - IPv4 地址验证
  - `mmc_def.h` - 基础定义

---

## 常量定义

### PROTOCOL_TCP

```cpp
const std::string PROTOCOL_TCP = "tcp://";
```

**说明**: TCP 协议前缀常量，用于标识使用 TCP 协议的 URL

---

## 枚举类型

### NetProtoVersion

```cpp
enum NetProtoVersion : int16_t {
    VERSION_1 = 1,
};
```

**说明**: 网络协议版本枚举，当前支持版本 1

---

### NetProtocol

```cpp
enum NetProtocol {
    NET_NONE = 1,      // 无协议
    NET_RPC_TCP,       // TCP RPC
    NET_RPC_RDMA,      // RDMA RPC
    NET_RPC_URMA,      // URMA RPC
    NET_IPC_UDS,       // Unix Domain Socket IPC
    NET_IPC_SHM,       // 共享内存 IPC
};
```

**说明**: 网络协议类型枚举

**枚举值说明**:
- `NET_NONE`: 无协议或未初始化
- `NET_RPC_TCP`: 基于 TCP 的远程过程调用
- `NET_RPC_RDMA`: 基于 RDMA 的远程过程调用
- `NET_RPC_URMA`: 基于 URMA 的远程过程调用
- `NET_IPC_UDS`: 基于 Unix Domain Socket 的进程间通信
- `NET_IPC_SHM`: 基于共享内存的进程间通信

---

## 类型别名

### ExternalLog

```cpp
using ExternalLog = void (*)(int, const char *);
```

**说明**: 外部日志函数类型定义

**参数**:
- 第一个参数: 日志级别 (int)
- 第二个参数: 日志消息内容 (const char*)

---

## 前置声明

```cpp
class NetContext;
class NetLink;
class NetEngine;
```

**说明**: 网络核心类的前置声明，用于定义智能指针类型别名

---

## 智能指针类型别名

```cpp
using NetContextPtr = MmcRef<NetContext>;
using NetLinkPtr = MmcRef<NetLink>;
using NetEnginePtr = MmcRef<NetEngine>;
```

**说明**: 网络核心类的智能指针类型别名，基于 MmcRef 引用计数智能指针

---

## 结构体

### NetEngineOptions

```cpp
struct NetEngineOptions {
    std::string name;             /* name of engine */
    std::string ip;               /* ip */
    uint16_t port = 9980L;        /* listen port */
    uint16_t threadCount = 2;     /* worker thread count */
    uint16_t rankId = UINT16_MAX; /* rank id */
    bool startListener = false;   /* start listener or not */
    mmc_tls_config tlsOption;     /* TLS communication options */
    int32_t logLevel = 3;
    ExternalLog logFunc = nullptr;

    /* functions */
    std::string ToString() const;

    static Result ExtractIpPortFromUrl(const std::string &url, NetEngineOptions &option);
};
```

**说明**: 网络引擎配置选项结构

**成员变量**:
- `name`: 引擎名称
- `ip`: 监听 IP 地址
- `port`: 监听端口，默认 9980
- `threadCount`: 工作线程数量，默认 2
- `rankId`: Rank ID，默认 UINT16_MAX
- `startListener`: 是否启动监听器，默认 false
- `tlsOption`: TLS 通信配置选项
- `logLevel`: 日志级别，默认 3
- `logFunc`: 外部日志函数指针

---

## 函数

### NetEngineOptions::ToString()

```cpp
inline std::string NetEngineOptions::ToString() const
{
    std::ostringstream oss;
    oss << "NetEngineOptions [name " << name << ", ip: " << ip << ", port: " << port << ", threadCount: " << threadCount
        << ", rankId " << rankId << ", startListener: " << startListener << ", tlsEnables: " << tlsOption.tlsEnable
        << "]";
    return oss.str();
}
```

**声明位置**: 行 61-68

**功能描述**: 将网络引擎配置选项转换为字符串格式

**返回值**: 包含配置信息的字符串

**代码逻辑**:
1. 创建字符串输出流
2. 将各配置项写入流中
3. 返回流内容字符串

**使用示例**:
```cpp
NetEngineOptions options;
options.name = "my_engine";
options.ip = "0.0.0.0";
options.port = 5000;
std::cout << options.ToString() << std::endl;
// 输出: NetEngineOptions [name my_engine, ip: 0.0.0.0, port: 5000, ...]
```

---

### NetEngineOptions::ExtractIpPortFromUrl()

```cpp
inline Result NetEngineOptions::ExtractIpPortFromUrl(const std::string &url, NetEngineOptions &option)
{
    using namespace mf;
    auto result = SocketAddressParserMgr::getInstance().CreateParser(url);
    MMC_ASSERT_RETURN(result != nullptr, MMC_INVALID_PARAM);
    std::string ipStr = result->GetIp();
    std::string portStr = std::to_string(result->GetPort());
    if (!result->IsIpv6()) {
        /* verify ip and port */
        Ipv4PortValidator validator1("IndexServiceUrl");
        validator1.Initialize();
        if (!(validator1.Validate(ipStr + ":" + portStr))) {
            MMC_LOG_ERROR("Failed to extract url");
            return MMC_INVALID_PARAM;
        }
    }

    /* covert port */
    long tmpPort = 0;
    if (!StrUtil::String2Uint<long>(portStr, tmpPort)) {
        MMC_LOG_ERROR("Failed to extract url");
        return MMC_INVALID_PARAM;
    }

    /* set ip and port */
    option.ip = ipStr;
    option.port = tmpPort;
    return MMC_OK;
}
```

**声明位置**: 行 70-98

**功能描述**: 从 URL 字符串中提取 IP 地址和端口，并设置到选项结构中

**参数**:
- `url`: 输入的 URL 字符串，如 "tcp://192.168.1.1:5000"
- `option`: 输出参数，提取的 IP 和端口将设置到此结构

**返回值**:
- `MMC_OK`: 成功提取
- `MMC_INVALID_PARAM`: URL 格式无效或验证失败

**代码逻辑**:
1. 使用 `SocketAddressParserMgr` 解析 URL
2. 提取 IP 和端口字符串
3. 如果是 IPv4 地址，使用 `Ipv4PortValidator` 验证
4. 将端口字符串转换为数值
5. 设置到输出参数中

**使用示例**:
```cpp
NetEngineOptions options;
Result ret = NetEngineOptions::ExtractIpPortFromUrl("tcp://192.168.1.100:8080", options);
if (ret == MMC_OK) {
    // options.ip = "192.168.1.100"
    // options.port = 8080
}
```

---

## 文件级别的关系图

```
mmc_net_common.h (网络通用定义)
    |
    +-- PROTOCOL_TCP -> TCP 协议前缀
    +-- NetProtoVersion -> 协议版本枚举
    +-- NetProtocol -> 协议类型枚举
    +-- NetEngineOptions -> 网络引擎配置
    +-- ExternalLog -> 外部日志函数类型
    +-- NetContextPtr/NetLinkPtr/NetEnginePtr -> 智能指针类型别名
```

---

## 依赖关系

**依赖以下文件**:
- `mmc_common_includes.h` - 公共头文件聚合
- `mf_ipv4_validator.h` - IP 地址验证
- `mmc_def.h` - 基础定义

**被以下文件依赖**:
- `mmc_net_engine.h`
- `mmc_net_engine_acc.h`
- `mmc_net_link_acc.h`
- `mmc_net_ctx_acc.h`
- 所有网络相关实现文件

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有网络通用定义都在 ock::mmc 命名空间内
}
}
```
