# client 模块文档

## 模块概述

`client` 模块包含 MemCache 客户端实现，负责与元数据服务器通信，提供数据的存储、读取、删除、查询等功能。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/client/`

## 文件列表

### 头文件 (.h)
- `mmc_client_default.h` - 默认客户端实现
- `mmc_meta_net_client.h` - 元数据网络客户端

### 源文件 (.cpp)
- `mmc_client_default.cpp` - 默认客户端实现
- `mmc_meta_net_client.cpp` - 元数据网络客户端实现

---

## 详细文档

### mmc_client_default.h / mmc_client_default.cpp

**功能**: 提供默认的 MemCache 客户端实现

**核心类**: `MmcClientDefault`

**主要功能**:
- 数据存储 (Put)
- 数据读取 (Get)
- 批量操作 (BatchPut, BatchGet)
- 数据删除 (Remove, BatchRemove, RemoveAll)
- 存在性检查 (IsExist, BatchIsExist)
- 元数据查询 (Query, BatchQuery)
- 内存缓冲区注册 (RegisterBuffer, UnRegisterBuffer)

**单例模式**:
```cpp
MmcClientDefault::RegisterInstance();
auto* client = MmcClientDefault::GetInstance();
// 使用客户端...
MmcClientDefault::UnregisterInstance();
```

**线程池**:
- `threadPool_` - 通用线程池（状态更新）
- `readThreadPool_` - 读线程池
- `writeThreadPool_` - 写线程池

**聚合 IO 优化**:
- 支持将多个小请求聚合成一个大请求
- 减少调用栈开销，平衡并发性能

**详细文档**:
- [mmc_client_default_h.md](mmc_client_default_h.md) - 头文件解读
- [mmc_client_default_cpp.md](mmc_client_default_cpp.md) - 实现文件解读

---

### mmc_meta_net_client.h / mmc_meta_net_client.cpp

**功能**: 提供与元数据服务器的网络通信功能

**核心类**: `MetaNetClient`

**主要功能**:
- 启动/停止网络引擎
- 连接到元数据服务器
- 同步 RPC 调用（带自动重试）
- 回调处理器注册
- 自动重连

**工厂模式**: `MetaNetClientFactory`
- 根据服务器 URL 管理多个客户端实例
- 键生成规则: `serverUrl + inputName`

**请求类型**:
- `ML_*` - 客户端 -> 元数据服务
- `LM_*` - 元数据服务 -> 客户端

**重试机制**:
- 对 `MMC_LINK_NOT_FOUND` 和 `MMC_TIMEOUT` 自动重试
- 单次重试间隔 200ms
- 连接断开后最多重试 180 次，每次间隔 2 秒

**详细文档**:
- [mmc_meta_net_client_h.md](mmc_meta_net_client_h.md) - 头文件解读
- [mmc_meta_net_client_cpp.md](mmc_meta_net_client_cpp.md) - 实现文件解读

---

## 模块内文件关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                        client 模块                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                   MmcClientDefault                             │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │  对外接口:                                              │  │  │
│  │  │  - Put / Get                                           │  │  │
│  │  │  - BatchPut / BatchGet                                 │  │  │
│  │  │  - Remove / BatchRemove / RemoveAll                    │  │  │
│  │  │  - IsExist / BatchIsExist                              │  │  │
│  │  │  - Query / BatchQuery                                  │  │  │
│  │  │  - RegisterBuffer / UnRegisterBuffer                   │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                              │                                 │  │
│  │                              ▼                                 │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │              MetaNetClient                              │  │  │
│  │  │  ┌──────────────────────────────────────────────────┐  │  │  │
│  │  │  │  - Start() / Stop()                              │  │  │  │
│  │  │  │  - Connect()                                     │  │  │  │
│  │  │  │  - SyncCall<REQ, RESP>() (带自动重试)             │  │  │  │
│  │  │  │  - RegisterRetryHandler()                         │  │  │  │
│  │  │  └──────────────────────────────────────────────────┘  │  │  │
│  │  │                              │                          │  │  │
│  │  │                              ▼                          │  │  │
│  │  │  ┌──────────────────────────────────────────────────┐  │  │  │
│  │  │  │              NetEngine                            │  │  │  │
│  │  │  │  (网络通信引擎)                                     │  │  │  │
│  │  │  └──────────────────────────────────────────────────┘  │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                              │                                 │  │
│  │                              ▼                                 │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │              MmcBmProxy                                 │  │  │
│  │  │  (Blob Manager 代理)                                    │  │  │
│  │  │  - BatchPut / BatchGet                                 │  │  │
│  │  │  - BatchDataPut / BatchDataGet                         │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  工厂类:                                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              MetaNetClientFactory                            │   │
│  │  - GetInstance(serverUrl, name)                              │   │
│  │  - 管理 MetaNetClient 实例                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 数据流图

### Put 操作流程

```
┌──────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────┐
│ 用户调用  │───▶│MmcClient     │───▶│PrepareAllocOpt  │───▶│构建分配  │
│ Put()     │    │Default       │    │                 │    │请求      │
└──────────┘    └──────────────┘    └─────────────────┘    └────┬─────┘
                                                                  │
                                        ┌─────────────────────────┴────────┐
                                        ▼                                  │
                              ┌──────────────────┐                        │
                              │MetaNetClient     │                        │
                              │SyncCall()        │                        │
                              └────┬─────────────┘                        │
                                   │                                       │
                                   ▼                                       │
                              ┌──────────────────┐                        │
                              │元数据服务器       │                        │
                              │分配 Blob        │                        │
                              └────┬─────────────┘                        │
                                   │                                       │
                                   ▼                                       │
                              ┌──────────────────┐                        │
                              │返回 Blob 列表    │                        │
                              └────┬─────────────┘                        │
                                                                   │       │
                                        ┌──────────┬──────────────┘       │
                                        ▼                                 │
                              ┌──────────────────┐                        │
                              │PrepareBlob      │                        │
                              │准备拷贝描述      │                        │
                              └────┬─────────────┘                        │
                                   │                                       │
                        ┌──────────┴──────────┐                          │
                        ▼                     ▼                          │
               ┌──────────────┐      ┌──────────────┐                   │
               │SubmitPutTask │      │SubmitPutTask │                   │
               │(异步)        │      │(同步)        │                   │
               └──────┬───────┘      └──────┬───────┘                   │
                      │                     │                            │
                      ▼                     ▼                            │
               ┌──────────────┐      ┌──────────────┐                   │
               │WriteThread   │      │直接执行      │                   │
               │Pool          │      │              │                   │
               └──────┬───────┘      └──────┬───────┘                   │
                      │                     │                            │
                      └──────────┬──────────┘                            │
                                 ▼                                       │
                      ┌──────────────────┐                               │
                      │MmcBmProxy        │                               │
                      │BatchDataPut      │                               │
                      └────┬─────────────┘                               │
                           │                                              │
                           ▼                                              │
                      ┌──────────────────┐                               │
                      │SyncUpdateState   │◄───── 更新元数据               │
                      └──────────────────┘                               │
```

### Get 操作流程

```
┌──────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────┐
│ 用户调用  │───▶│MmcClient     │───▶│构建 Get 请求     │───▶│MetaNet   │
│ Get()     │    │Default       │    │                 │    │Client    │
└──────────┘    └──────────────┘    └─────────────────┘    └────┬─────┘
                                                                  │
                                                                   ▼
                              ┌──────────────────┐
                              │元数据服务器       │
                              │查询 Blob 位置    │
                              └────┬─────────────┘
                                   │
                                   ▼
                              ┌──────────────────┐
                              │返回 Blob 信息    │
                              └────┬─────────────┘
                                   │
                                   ▼
                              ┌──────────────────┐
                              │PrepareBlob      │
                              │准备拷贝描述      │
                              └────┬─────────────┘
                                   │
                        ┌──────────┴──────────┐
                        ▼                     ▼
               ┌──────────────┐      ┌──────────────┐
               │SubmitGetTask │      │SubmitGetTask │
               │(异步)        │      │(同步)        │
               └──────┬───────┘      └──────┬───────┘
                      │                     │
                      ▼                     ▼
               ┌──────────────┐      ┌──────────────┐
               │ReadThread    │      │直接执行      │
               │Pool          │      │              │
               └──────┬───────┘      └──────┬───────┘
                      │                     │
                      └──────────┬──────────┘
                                 ▼
                      ┌──────────────────┐
                      │MmcBmProxy        │
                      │BatchDataGet      │
                      └────┬─────────────┘
                           │
                           ▼
                      ┌──────────────────┐
                      │AsyncUpdateState  │◄───── 异步更新元数据
                      └──────────────────┘
```

---

## 使用示例

### 基本使用

```cpp
#include "mmc_client_default.h"

// 注册并获取客户端实例
MmcClientDefault::RegisterInstance();
auto* client = MmcClientDefault::GetInstance();

// 配置客户端
mmc_client_config_t config;
config.readThreadPoolNum = 4;
config.writeThreadPoolNum = 4;
config.aggregateIO = true;
config.aggregateNum = 16;
config.rpcRetryTimeOut = 5000;

// 启动客户端
client->Start(config);

// 存储数据
char data[] = "Hello, MemCache!";
mmc_buffer buf = {
    .addr = reinterpret_cast<uint64_t>(data),
    .len = sizeof(data),
    .offset = 0,
    .type = MEDIA_DRAM
};
mmc_put_options options;
options.replicaNum = 2;
options.policy = NATIVE_AFFINITY;
client->Put("my_key", buf, options, 0);

// 读取数据
char outData[1024];
mmc_buffer outBuf = {
    .addr = reinterpret_cast<uint64_t>(outData),
    .len = 1024,
    .offset = 0,
    .type = MEDIA_DRAM
};
client->Get("my_key", outBuf, 0);

// 查询元数据
mmc_data_info info;
client->Query("my_key", info, 0);

// 停止客户端
client->Stop();
MmcClientDefault::UnregisterInstance();
```

### 批量操作

```cpp
// 批量存储
std::vector<std::string> keys = {"key1", "key2", "key3"};
std::vector<MmcBufferArray> bufArrs = {
    /* ... 缓冲区数组 ... */
};
std::vector<int> results;
client->BatchPut(keys, bufArrs, options, 0, results);

// 批量读取
std::vector<MmcBufferArray> outBufArrs = {
    /* ... 输出缓冲区数组 ... */
};
results.clear();
client->BatchGet(keys, outBufArrs, 0, results);

// 检查结果
for (size_t i = 0; i < keys.size(); ++i) {
    if (results[i] == MMC_OK) {
        std::cout << keys[i] << " succeeded" << std::endl;
    } else {
        std::cout << keys[i] << " failed: " << results[i] << std::endl;
    }
}
```

### 注册内存缓冲区

```cpp
// 注册内存用于 RDMA
void* buffer = malloc(1024 * 1024); // 1MB
uint64_t addr = reinterpret_cast<uint64_t>(buffer);
client->RegisterBuffer(addr, 1024 * 1024);

// 使用已注册的内存...
// ...

// 注销内存
client->UnRegisterBuffer(addr, 1024 * 1024);
free(buffer);
```

---

## 常量定义

| 常量名 | 值 | 说明 |
|--------|---|------|
| `CLIENT_THREAD_COUNT` | 2 | 客户端网络引擎线程数 |
| `KEY_MAX_LENTH` | 256 | 键名最大长度 |
| `NET_RETRY_COUNT` | 180 | 网络重连最大尝试次数 |
| `TIMEOUT_60_SECONDS` | 60 | 单次 RPC 超时时间（秒） |
| `SYNC_CALL_INTERVAL` | 200 | 同步调用重试间隔（毫秒） |
| `RETRY_LOG_INTERVAL` | 10 | 重试日志打印间隔 |

---

## 依赖模块

| 模块 | 用途 |
|------|------|
| `common` | 公共工具类、日志、锁等 |
| `net` | 网络通信引擎 |
| `bm` | Blob Manager 代理 |
| `msg` | 消息定义 |

---

## 相关文档

- [common 模块文档](../common/README.md)
