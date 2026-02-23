# mmc_meta_net_client.h 逐函数解读

## 文件概述

- **文件路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/client/mmc_meta_net_client.h`
- **文件用途**: 定义 `MetaNetClient` 类，负责客户端与元数据服务器之间的网络通信，支持同步调用、自动重连和请求处理
- **依赖项**:
  - `<chrono>` - 时间相关
  - `<thread>` - 线程相关
  - `mmc_local_common.h` - 本地公共定义
  - `mmc_net_engine.h` - 网络引擎
  - `mmc_blob_common.h` - Blob 公共定义

---

## 常量定义

### NET_RETRY_COUNT

**声明位置**: 行 25

```cpp
constexpr int NET_RETRY_COUNT = 180;
```

**功能描述**: 网络重连尝试次数

---

### TIMEOUT_60_SECONDS

**声明位置**: 行 26

```cpp
constexpr int TIMEOUT_60_SECONDS = 60;
```

**功能描述**: 单次 RPC 调用超时时间（60 秒）

---

### SYNC_CALL_INTERVAL

**声明位置**: 行 27

```cpp
constexpr int SYNC_CALL_INTERVAL = 200;
```

**功能描述**: 同步调用重试间隔（毫秒）

---

### FIRST_RETRY

**声明位置**: 行 28

```cpp
constexpr int FIRST_RETRY = 1;
```

**功能描述**: 第一次重试标记

---

### RETRY_LOG_INTERVAL

**声明位置**: 行 29

```cpp
constexpr int RETRY_LOG_INTERVAL = 10;
```

**功能描述**: 重试日志打印间隔（每 10 次打印一次）

---

## 类型别名

### ClientRetryHandler

**声明位置**: 行 31

```cpp
using ClientRetryHandler = std::function<int32_t(void)>;
```

**功能描述**: 重试处理器类型，无参数返回 `int32_t`

---

### ClientReplicateHandler

**声明位置**: 行 32-33

```cpp
using ClientReplicateHandler = std::function<int32_t(
    const std::vector<uint32_t> &ops, const std::vector<std::string> &keys, const std::vector<MmcMemBlobDesc> &blobs)>;
```

**功能描述**: 副本处理器类型，处理元数据复制请求

**参数**:
- `ops` - 操作列表
- `keys` - 键列表
- `blobs` - Blob 描述符列表

---

### ClientBlobCopyHandler

**声明位置**: 行 34

```cpp
using ClientBlobCopyHandler = std::function<int32_t(const MmcMemBlobDesc &src, const MmcMemBlobDesc &dst)>;
```

**功能描述**: Blob 拷贝处理器类型

**参数**:
- `src` - 源 Blob 描述符
- `dst` - 目标 Blob 描述符

---

## 类: MetaNetClient

**声明位置**: 行 35-151

**完整签名**:
```cpp
class MetaNetClient : public MmcReferable {
    // ... 成员函数和变量
};
```

**功能描述**: 元数据网络客户端，负责与元数据服务器的网络通信，提供同步调用、自动重连等功能

**继承关系**:
```
MmcReferable
    ^
    |
MetaNetClient
```

---

### 公共成员函数

#### MetaNetClient() (构造函数)

**声明位置**: 行 37

**完整签名**:
```cpp
explicit MetaNetClient(const std::string &serverUrl, const std::string &inputName = "");
```

**功能描述**: 构造函数，初始化网络客户端

**参数**:
- `serverUrl` - 服务器 URL
- `inputName` - 可选的客户端名称

---

#### ~MetaNetClient() (析构函数)

**声明位置**: 行 39

**完整签名**:
```cpp
~MetaNetClient() override;
```

**功能描述**: 析构函数

---

#### Start()

**声明位置**: 行 47

**完整签名**:
```cpp
Result Start(const NetEngineOptions &config);
```

**功能描述**: 启动网络客户端，初始化网络引擎

**参数**:
- `config` - 网络引擎配置选项

**返回值**: `Result` - 成功返回 `0`，失败返回错误码

---

#### Stop()

**声明位置**: 行 52

**完整签名**:
```cpp
void Stop();
```

**功能描述**: 停止网络客户端，清理资源

---

#### Connect()

**声明位置**: 行 60

**完整签名**:
```cpp
Result Connect(const std::string &url);
```

**功能描述**: 连接到元数据服务器

**参数**:
- `url` - 服务器 URL

**返回值**: `Result` - 成功返回 `0`，失败返回错误码

---

#### SyncCall()

**声明位置**: 行 72-112

**完整签名**:
```cpp
template<typename REQ, typename RESP>
Result SyncCall(const REQ &req, RESP &resp, int32_t timeoutInMilliSecond)
```

**功能描述**: 同步调用元数据服务器，支持自动重试

**模板参数**:
- `REQ` - 请求类型
- `RESP` - 响应类型

**参数**:
- `req` - 请求对象（需包含 `msgId` 成员）
- `resp` - 响应对象
- `timeoutInMilliSecond` - 超时时间（毫秒）

**返回值**: `Result` - 成功返回 `0`，失败返回错误码

**代码逻辑**:
1. 记录开始时间，计算超时时间点
2. 循环调用网络引擎：
   - 如果返回 `MMC_LINK_NOT_FOUND` 或 `MMC_TIMEOUT`，则重试
   - 检查是否已超时，避免无效休眠
   - 计算剩余时间，休眠适当时长
   - 记录重试次数和日志
3. 记录最终耗时和结果

**重试逻辑**:
- 每次重试间隔 `SYNC_CALL_INTERVAL` (200ms)
- 只对 `MMC_LINK_NOT_FOUND` 和 `MMC_TIMEOUT` 重试
- 超时后停止重试

**注意事项**:
- 模板函数定义在头文件中
- 每 10 次重试打印一次调试日志

---

#### Status()

**声明位置**: 行 119 和 153-157

**完整签名**:
```cpp
bool Status()
```

**功能描述**: 获取客户端状态

**返回值**: `bool` - 已启动返回 `true`，否则返回 `false`

**注意事项**: inline 函数，定义在类外部

---

#### RegisterRetryHandler()

**声明位置**: 行 121-127

**完整签名**:
```cpp
void RegisterRetryHandler(const ClientRetryHandler &retryHandler,
                          const ClientReplicateHandler &replicateHandler,
                          const ClientBlobCopyHandler &blobCopyHandler)
```

**功能描述**: 注册回调处理器

**参数**:
- `retryHandler` - 重试回调
- `replicateHandler` - 副本处理回调
- `blobCopyHandler` - Blob 拷贝回调

**用途**:
- `retryHandler`: 重连成功后调用
- `replicateHandler`: 处理元数据复制请求
- `blobCopyHandler`: 处理 Blob 拷贝请求

---

### 私有成员函数

#### HandleMetaReplicate()

**声明位置**: 行 130

**完整签名**:
```cpp
Result HandleMetaReplicate(const NetContextPtr &context);
```

**功能描述**: 处理元数据复制请求

**参数**:
- `context` - 网络上下文

**返回值**: `Result` - 操作结果

**注意事项**: 作为回调注册到网络引擎

---

#### HandlePing()

**声明位置**: 行 131

**完整签名**:
```cpp
Result HandlePing(const NetContextPtr &context);
```

**功能描述**: 处理 Ping 请求

**参数**:
- `context` - 网络上下文

**返回值**: `Result` - 操作结果

**注意事项**: 用于心跳检测

---

#### HandleLinkBroken()

**声明位置**: 行 132

**完整签名**:
```cpp
Result HandleLinkBroken(const NetLinkPtr &link);
```

**功能描述**: 处理连接断开事件，自动重连

**参数**:
- `link` - 网络连接

**返回值**: `Result` - 重连成功返回 `0`，失败返回错误码

**重连逻辑**:
1. 最多尝试 `NET_RETRY_COUNT` 次
2. 每次失败后休眠 2 秒
3. 重连成功后调用 `retryHandler_`

---

#### HandleBlobCopy()

**声明位置**: 行 133

**完整签名**:
```cpp
Result HandleBlobCopy(const NetContextPtr &context);
```

**功能描述**: 处理 Blob 拷贝请求

**参数**:
- `context` - 网络上下文

**返回值**: `Result` - 操作结果

---

## 成员变量

| 变量名 | 类型 | 描述 |
|--------|------|------|
| `engine_` | `NetEnginePtr` | 网络引擎 |
| `link2Index_` | `NetLinkPtr` | 到索引服务器的连接 |
| `rankId_` | `uint16_t` | Rank ID |
| `ip_` | `std::string` | 服务器 IP 地址 |
| `port_` | `uint64_t` | 服务器端口 |
| `retryCount_` | `const uint32_t` | 重试次数（180） |
| `retryHandler_` | `ClientRetryHandler` | 重试回调 |
| `replicateHandler_` | `ClientReplicateHandler` | 副本处理回调 |
| `blobCopyHandler_` | `ClientBlobCopyHandler` | Blob 拷贝回调 |
| `serverUrl_` | `std::string` | 服务器 URL |
| `mutex_` | `std::mutex` | 互斥锁（非高频使用） |
| `started_` | `bool` | 启动标志（非高频使用） |
| `name_` | `std::string` | 客户端名称（非高频使用） |

---

## 类型别名

**声明位置**: 行 159

```cpp
using MetaNetClientPtr = MmcRef<MetaNetClient>;
```

**功能描述**: 元数据网络客户端智能指针类型别名

---

## 类: MetaNetClientFactory

**声明位置**: 行 161-183

**完整签名**:
```cpp
class MetaNetClientFactory : public MmcReferable {
public:
    static MmcRef<MetaNetClient> GetInstance(const std::string &serverUrl, const std::string inputName = "")
    {
        // ...
    }

private:
    static std::map<std::string, MmcRef<MetaNetClient>> instances_;
    static std::mutex instanceMutex_;
};
```

**功能描述**: 元数据网络客户端工厂，提供单例模式管理多个客户端实例

**继承关系**:
```
MmcReferable
    ^
    |
MetaNetClientFactory
```

---

### GetInstance()

**声明位置**: 行 163-178

**完整签名**:
```cpp
static MmcRef<MetaNetClient> GetInstance(const std::string &serverUrl, const std::string inputName = "")
```

**功能描述**: 获取或创建客户端实例

**参数**:
- `serverUrl` - 服务器 URL
- `inputName` - 可选的客户端名称

**返回值**: `MmcRef<MetaNetClient>` - 客户端实例的智能指针

**代码逻辑**:
1. 加锁保护
2. 根据 `serverUrl + inputName` 生成唯一键
3. 查找已存在的实例
4. 如果不存在则创建新实例并保存
5. 返回实例

**设计模式**: 工厂模式 + 单例模式（多实例单例）

**注意事项**:
- 使用 `new (std::nothrow)` 避免 `std::bad_alloc` 异常
- 内存不足时返回 `nullptr`

---

### 私有静态成员

**声明位置**: 行 181-182

```cpp
static std::map<std::string, MmcRef<MetaNetClient>> instances_;
static std::mutex instanceMutex_;
```

**功能描述**:
- `instances_`: 存储客户端实例的映射表（键 = serverUrl + inputName）
- `instanceMutex_`: 保护实例映射表的互斥锁

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MetaNetClient                                 │
├─────────────────────────────────────────────────────────────────────┤
│  网络通信:                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     NetEngine                                 │    │
│  │                        (engine_)                              │    │
│  └────────────────────────────────┬────────────────────────────┘    │
│                                   │                                   │
│  ┌────────────────────────────────▼─────────────────────────────┐    │
│  │                         SyncCall                              │    │
│  │  ┌────────────────────────────────────────────────────────┐  │    │
│  │  │  自动重试:                                              │  │    │
│  │  │  - 遇到 MMC_LINK_NOT_FOUND 或 MMC_TIMEOUT 时重试        │  │    │
│  │  │  - 每次间隔 SYNC_CALL_INTERVAL (200ms)                  │  │    │
│  │  │  - 超时后停止重试                                       │  │    │
│  │  └────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  回调处理:                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  HandlePing()          - 处理心跳请求                         │    │
│  │  HandleMetaReplicate() - 处理元数据复制                       │    │
│  │  HandleBlobCopy()      - 处理 Blob 拷贝                       │    │
│  │  HandleLinkBroken()    - 处理连接断开（自动重连）              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  自动重连:                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  HandleLinkBroken:                                          │    │
│  │    1. 检测到连接断开                                         │    │
│  │    2. 最多重试 NET_RETRY_COUNT (180) 次                      │    │
│  │    3. 每次失败后休眠 2 秒                                    │    │
│  │    4. 重连成功后调用 retryHandler_                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    MetaNetClientFactory                               │
├─────────────────────────────────────────────────────────────────────┤
│  单例管理:                                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  instances_: map<键, MmcRef<MetaNetClient>>                  │    │
│  │                                                              │    │
│  │  键生成规则: serverUrl + inputName                           │    │
│  │                                                              │    │
│  │  GetInstance():                                              │    │
│  │    - 如果键存在，返回已有实例                                 │    │
│  │    - 如果键不存在，创建新实例并保存                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 使用示例

```cpp
// 从工厂获取客户端实例
auto metaClient = MetaNetClientFactory::GetInstance("tcp://192.168.1.100:5000", "MyClient");

// 启动客户端
NetEngineOptions options;
options.name = "MyClient";
options.rankId = 0;
options.threadCount = 2;
options.startListener = false;
Result ret = metaClient->Start(options);

// 连接到服务器
ret = metaClient->Connect("tcp://192.168.1.100:5000");

// 注册回调处理器
metaClient->RegisterRetryHandler(
    []() -> int32_t {
        std::cout << "Reconnected to server" << std::endl;
        return MMC_OK;
    },
    [](const std::vector<uint32_t> &ops,
       const std::vector<std::string> &keys,
       const std::vector<MmcMemBlobDesc> &blobs) -> int32_t {
        // 处理复制请求
        return MMC_OK;
    },
    [](const MmcMemBlobDesc &src, const MmcMemBlobDesc &dst) -> int32_t {
        // 处理 Blob 拷贝
        return MMC_OK;
    }
);

// 同步调用
AllocRequest allocReq("my_key", {}, GenerateOperateId(0));
AllocResponse allocResp;
ret = metaClient->SyncCall(allocReq, allocResp, 5000); // 5秒超时

// 检查状态
if (metaClient->Status()) {
    std::cout << "Client is running" << std::endl;
}

// 停止客户端
metaClient->Stop();
```
