# mmc_meta_net_client.cpp 逐函数解读

## 文件概述

- **文件路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/client/mmc_meta_net_client.cpp`
- **文件用途**: 实现 `MetaNetClient` 类和 `MetaNetClientFactory` 类的具体功能
- **依赖项**:
  - `mmc_meta_net_client.h` - 头文件
  - `mmc_msg_base.h` - 基础消息定义
  - `mmc_msg_client_meta.h` - 客户端元数据消息定义

---

## 静态成员初始化

### instances_ / instanceMutex_

**声明位置**: 行 19-20

```cpp
std::map<std::string, MmcRef<MetaNetClient>> MetaNetClientFactory::instances_;
std::mutex MetaNetClientFactory::instanceMutex_;
```

**功能描述**: 初始化工厂类的实例映射表和互斥锁

---

## 函数实现

### ~MetaNetClient()

**声明位置**: 行 22-23

**完整签名**:
```cpp
MetaNetClient::~MetaNetClient() {}
```

**功能描述**: 析构函数（默认实现）

---

### MetaNetClient() (构造函数)

**声明位置**: 行 23-25

**完整签名**:
```cpp
MetaNetClient::MetaNetClient(const std::string &serverUrl, const std::string &inputName)
    : serverUrl_(serverUrl), name_(inputName)
{}
```

**功能描述**: 构造函数，使用初始化列表设置服务器 URL 和客户端名称

**参数**:
- `serverUrl` - 服务器 URL
- `inputName` - 客户端名称

---

### Start()

**声明位置**: 行 27-70

**完整签名**:
```cpp
Result MetaNetClient::Start(const NetEngineOptions &config)
```

**功能描述**: 启动元数据网络客户端

**代码逻辑**:
1. 加锁检查是否已启动，避免重复启动
2. 创建网络引擎实例
3. 注册各种请求处理器：
   - **请求处理器**（无回调）:
     - `ML_PING_REQ` - Ping 请求
     - `ML_ALLOC_REQ` - 分配请求
     - `ML_UPDATE_REQ` - 更新请求
     - `ML_BATCH_UPDATE_REQ` - 批量更新请求
     - `ML_GET_REQ` - 获取请求
     - `ML_BATCH_GET_REQ` - 批量获取请求
     - `ML_REMOVE_REQ` - 删除请求
     - `ML_BATCH_REMOVE_REQ` - 批量删除请求
     - `ML_BM_REGISTER_REQ` - BM 注册请求
     - `ML_IS_EXIST_REQ` - 存在性检查请求
     - `ML_BATCH_IS_EXIST_REQ` - 批量存在性检查请求
     - `ML_BM_UNREGISTER_REQ` - BM 注销请求
     - `ML_QUERY_REQ` - 查询请求
     - `ML_BATCH_QUERY_REQ` - 批量查询请求
     - `ML_BATCH_ALLOC_REQ` - 批量分配请求
     - `ML_REMOVE_ALL_REQ` - 删除所有请求
   - **回调处理器**:
     - `LM_PING_REQ` → `HandlePing`
     - `LM_META_REPLICATE_REQ` → `HandleMetaReplicate`
     - `LM_BLOB_COPY_REQ` → `HandleBlobCopy`
4. 注册连接断开处理器 → `HandleLinkBroken`
5. 启动网络引擎
6. 保存 Rank ID，设置启动标志

**参数**:
- `config` - 网络引擎配置选项

**返回值**: `Result` - 成功返回 `MMC_OK`，失败返回错误码

**请求类型说明**:
- `ML_*` - 从客户端发往元数据服务的请求
- `LM_*` - 从元数据服务发往客户端的请求

---

### Stop()

**声明位置**: 行 72-86

**完整签名**:
```cpp
void MetaNetClient::Stop()
```

**功能描述**: 停止元数据网络客户端

**代码逻辑**:
1. 加锁检查是否已启动
2. 清空连接指针
3. 如果引擎存在，停止并清空
4. 重置启动标志

**注意事项**: 如果未启动，仅记录警告日志

---

### Connect()

**声明位置**: 行 88-98

**完整签名**:
```cpp
Result MetaNetClient::Connect(const std::string &url)
```

**功能描述**: 连接到元数据服务器

**代码逻辑**:
1. 从 URL 提取 IP 和端口
2. 验证引擎已初始化
3. 调用引擎连接到对端
4. 保存 IP 和端口

**参数**:
- `url` - 服务器 URL（格式如 `tcp://192.168.1.100:5000`）

**返回值**: `Result` - 成功返回 `MMC_OK`，失败返回错误码

---

### HandleMetaReplicate()

**声明位置**: 行 100-113

**完整签名**:
```cpp
Result MetaNetClient::HandleMetaReplicate(const NetContextPtr &context)
```

**功能描述**: 处理元数据复制请求

**代码逻辑**:
1. 从网络上下文获取 `MetaReplicateRequest`
2. 创建 `Response` 对象
3. 如果 `replicateHandler_` 已注册，调用它处理请求
4. 否则记录错误并返回 `MMC_ERROR`
5. 回复响应

**参数**:
- `context` - 网络上下文，包含请求数据

**返回值**: `Result` - 操作结果

**请求结构**:
```cpp
struct MetaReplicateRequest {
    std::vector<uint32_t> ops_;      // 操作列表
    std::vector<std::string> keys_;  // 键列表
    std::vector<MmcMemBlobDesc> blobs_; // Blob 描述符列表
    // ... 其他字段
};
```

---

### HandleBlobCopy()

**声明位置**: 行 115-130

**完整签名**:
```cpp
Result MetaNetClient::HandleBlobCopy(const NetContextPtr &context)
```

**功能描述**: 处理 Blob 拷贝请求

**代码逻辑**:
1. 从网络上下文获取 `BlobCopyRequest`
2. 创建 `Response` 对象
3. 如果 `blobCopyHandler_` 已注册，调用它执行拷贝
4. 如果拷贝失败，记录错误日志
5. 否则记录错误并返回 `MMC_ERROR`
6. 回复响应

**参数**:
- `context` - 网络上下文，包含请求数据

**返回值**: `Result` - 操作结果

**请求结构**:
```cpp
struct BlobCopyRequest {
    MmcMemBlobDesc srcBlob_;  // 源 Blob
    MmcMemBlobDesc dstBlob_;  // 目标 Blob
    // ... 其他字段
};
```

---

### HandlePing()

**声明位置**: 行 132-146

**完整签名**:
```cpp
Result MetaNetClient::HandlePing(const NetContextPtr &context)
```

**功能描述**: 处理 Ping 请求（心跳检测）

**代码逻辑**:
1. 从网络上下文获取原始数据
2. 创建 `NetMsgUnpacker` 并反序列化 `PingMsg`
3. 记录 Ping 消息中的数字
4. 创建 `NetMsgPacker` 并序列化响应
5. 回复响应

**参数**:
- `context` - 网络上下文，包含 Ping 数据

**返回值**: `Result` - 操作结果

**注意事项**: 用于心跳检测，确认连接存活

---

### HandleLinkBroken()

**声明位置**: 行 148-166

**完整签名**:
```cpp
Result MetaNetClient::HandleLinkBroken(const NetLinkPtr &link)
```

**功能描述**: 处理连接断开事件，自动重连

**代码逻辑**:
1. 记录连接断开日志
2. 验证引擎已初始化
3. 循环尝试重连（最多 `retryCount_` 次）:
   - 调用引擎连接到原服务器
   - 如果连接成功:
     - 如果 `retryHandler_` 已注册，调用它
     - 返回成功
   - 如果连接失败:
     - 记录错误日志
     - 休眠 2 秒后继续重试
4. 所有重试失败后返回错误

**参数**:
- `link` - 断开的网络连接（未使用）

**返回值**: `Result` - 重连成功返回 `MMC_OK`，失败返回 `MMC_ERROR`

**重连策略**:
- 最多尝试 180 次
- 每次失败后休眠 2 秒
- 使用保存的 `ip_` 和 `port_` 重连

---

## 数据流和关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         MetaNetClient                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  启动流程:                                                                │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────────────────┐   │
│  │ Start()  │───▶│创建 NetEngine│───▶│注册请求处理器                 │   │
│  └──────────┘    └──────────────┘    │ - ML_ALLOC_REQ                │   │
│                                     │ - ML_GET_REQ                  │   │
│                                     │ - ML_UPDATE_REQ               │   │
│                                     │ - ML_BATCH_*                  │   │
│                                     │ ...                           │   │
│                                     └──────────────┬───────────────┘   │
│                                                    │                    │
│                                     ┌──────────────▼───────────────┐   │
│                                     │注册回调处理器                  │   │
│                                     │ - LM_PING_REQ → HandlePing   │   │
│                                     │ - LM_META_REPLICATE_REQ      │   │
│                                     │ - LM_BLOB_COPY_REQ           │   │
│                                     └──────────────────────────────┘   │
│                                                                          │
│  连接流程:                                                                │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────────────────┐   │
│  │ Connect()│───▶│解析 URL      │───▶│NetEngine::ConnectToPeer      │   │
│  └──────────┘    └──────────────┘    └──────────────────────────────┘   │
│                                                                          │
│  SyncCall 流程:                                                          │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────────────────┐   │
│  │ SyncCall │───▶│NetEngine::   │───▶│等待响应或超时                 │   │
│  │          │    │Call          │    │                              │   │
│  └──────────┘    └──────────────┘    └──────────────┬───────────────┘   │
│                                                      │                    │
│                                         ┌────────────▼─────────────┐      │
│                                         │失败?                       │      │
│                                         │  - MMC_LINK_NOT_FOUND      │      │
│                                         │  - MMC_TIMEOUT            │      │
│                                         └────────────┬─────────────┘      │
│                                                      │                   │
│                                         ┌────────────▼─────────────┐      │
│                                         │重试 (间隔200ms)            │      │
│                                         │  - 检查超时                │      │
│                                         │  - 继续重试或返回          │      │
│                                         └───────────────────────────┘      │
│                                                                          │
│  回调处理流程:                                                            │
│  ┌──────────────────────┐    ┌──────────────────────────────────────┐   │
│  │元数据服务发送请求      │───▶│NetEngine 路由到对应的 Handler       │   │
│  └──────────────────────┘    └──────────────────────────────────────┘   │
│                                                                          │
│  HandleMetaReplicate:                                                    │
│  ┌──────────────┐    ┌──────────────────┐    ┌─────────────────────┐   │
│  │接收复制请求    │───▶│调用replicateHandler│───▶│执行复制操作        │   │
│  └──────────────┘    └──────────────────┘    └─────────────────────┘   │
│                                                                          │
│  HandleBlobCopy:                                                         │
│  ┌──────────────┐    ┌──────────────────┐    ┌─────────────────────┐   │
│  │接收拷贝请求    │───▶│调用blobCopyHandler │───▶│执行拷贝操作        │   │
│  └──────────────┘    └──────────────────┘    └─────────────────────┘   │
│                                                                          │
│  HandlePing:                                                             │
│  ┌──────────────┐    ┌──────────────────┐    ┌─────────────────────┐   │
│  │接收Ping请求    │───▶│反序列化PingMsg    │───▶│发送Ping响应         │   │
│  └──────────────┘    └──────────────────┘    └─────────────────────┘   │
│                                                                          │
│  自动重连流程:                                                            │
│  ┌──────────────┐    ┌──────────────────┐    ┌─────────────────────┐   │
│  │连接断开        │───▶│HandleLinkBroken  │───▶│循环重试 (最多180次)  │   │
│  └──────────────┘    └──────────────────┘    └──────────┬──────────┘   │
│                                                         │               │
│                                         ┌───────────────▼───────────┐    │
│                                         │每次失败休眠2秒              │    │
│                                         │继续尝试ConnectToPeer       │    │
│                                         └──────────────┬────────────┘    │
│                                                        │                 │
│                                         ┌──────────────▼─────────────┐   │
│                                         │重连成功?                    │   │
│                                         │  是 → 调用retryHandler_    │   │
│                                         │  否 → 继续重试或返回错误   │   │
│                                         └───────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 使用示例

```cpp
// 从工厂获取客户端实例
auto metaClient = MetaNetClientFactory::GetInstance(
    "tcp://192.168.1.100:5000",
    "MyMetaClient"
);

// 启动客户端
NetEngineOptions options;
options.name = "MyMetaClient";
options.rankId = 0;
options.threadCount = 2;
options.startListener = false;  // 客户端不需要监听

Result ret = metaClient->Start(options);
if (ret != MMC_OK) {
    std::cerr << "Failed to start meta client" << std::endl;
    return;
}

// 连接到元数据服务器
ret = metaClient->Connect("tcp://192.168.1.100:5000");
if (ret != MMC_OK) {
    std::cerr << "Failed to connect to meta server" << std::endl;
    return;
}

// 注册回调处理器
metaClient->RegisterRetryHandler(
    // 重试回调
    []() -> int32_t {
        std::cout << "Reconnected to meta server, resending pending requests..." << std::endl;
        // 可以在这里重新发送之前的请求
        return MMC_OK;
    },
    // 副本处理回调
    [](const std::vector<uint32_t> &ops,
       const std::vector<std::string> &keys,
       const std::vector<MmcMemBlobDesc> &blobs) -> int32_t {
        std::cout << "Received replicate request for " << keys.size() << " keys" << std::endl;
        // 处理副本创建逻辑
        return MMC_OK;
    },
    // Blob 拷贝回调
    [](const MmcMemBlobDesc &src, const MmcMemBlobDesc &dst) -> int32_t {
        std::cout << "Copying blob from rank " << src.rank_
                  << " to rank " << dst.rank_ << std::endl;
        // 执行 Blob 拷贝逻辑
        return MMC_OK;
    }
);

// 发送分配请求
AllocRequest allocReq("my_key", {}, GenerateOperateId(0));
AllocResponse allocResp;
ret = metaClient->SyncCall(allocReq, allocResp, 5000);
if (ret == MMC_OK) {
    std::cout << "Alloc successful, numBlobs: " << allocResp.numBlobs_ << std::endl;
}

// 发送获取请求
GetRequest getReq("my_key", 0, GenerateOperateId(0), true);
AllocResponse getResp;
ret = metaClient->SyncCall(getReq, getResp, 5000);
if (ret == MMC_OK) {
    std::cout << "Get successful" << std::endl;
}

// 检查状态
if (metaClient->Status()) {
    std::cout << "Meta client is running" << std::endl;
}

// 停止客户端
metaClient->Stop();
```
