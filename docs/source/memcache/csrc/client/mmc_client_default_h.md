# mmc_client_default.h 逐函数解读

## 文件概述

- **文件路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/client/mmc_client_default.h`
- **文件用途**: 定义 `MmcClientDefault` 类，这是 MemCache 客户端的默认实现，提供数据的存储（Put）、读取（Get）、删除（Remove）、查询（Query）等核心功能
- **依赖项**:
  - `mmc_common_includes.h` - 公共头文件
  - `mmc_meta_net_client.h` - 元数据网络客户端
  - `mmc_def.h` - 基础定义
  - `mmc_bm_proxy.h` - Blob Manager 代理
  - `mmc_thread_pool.h` - 线程池
  - `mmc_msg_client_meta.h` - 客户端元数据消息定义

## 类: MmcClientDefault

**声明位置**: 行 24-149

**完整签名**:
```cpp
class MmcClientDefault : public MmcReferable {
    // ... 成员函数和变量
};
```

**功能描述**: MemCache 客户端的默认实现类，继承自 `MmcReferable` 支持引用计数，提供与元服务器和网络通信的接口

**继承关系**:
```
MmcReferable
    ^
    |
MmcClientDefault
```

---

### 公共成员函数

#### ~MmcClientDefault()

**声明位置**: 行 26

**完整签名**:
```cpp
~MmcClientDefault() override = default;
```

**功能描述**: 析构函数，使用默认实现

---

#### Start()

**声明位置**: 行 28

**完整签名**:
```cpp
Result Start(const mmc_client_config_t &config);
```

**功能描述**: 启动客户端，初始化网络连接和线程池

**参数**:
- `config` - 客户端配置结构，包含线程数、RPC 超时、聚合 IO 配置等

**返回值**: `Result` - 成功返回 `MMC_OK`，失败返回相应错误码

**注意事项**: 重复调用会直接返回成功（幂等操作）

---

#### Stop()

**声明位置**: 行 30

**完整签名**:
```cpp
void Stop();
```

**功能描述**: 停止客户端，清理资源（线程池、网络连接等）

**注意事项**: 如果客户端未启动，会记录警告日志

---

#### Name()

**声明位置**: 行 32

**完整签名**:
```cpp
const std::string &Name() const;
```

**功能描述**: 获取客户端名称

**返回值**: `const std::string&` - 客户端名称的常量引用

---

#### Put() (单缓冲区版本)

**声明位置**: 行 34

**完整签名**:
```cpp
Result Put(const char *key, mmc_buffer *buf, mmc_put_options &options, uint32_t flags);
```

**功能描述**: 存储单个数据缓冲区到 MemCache

**参数**:
- `key` - 数据的键名（C 字符串）
- `buf` - 数据缓冲区
- `options` - 存储选项（副本数、亲和性策略等）
- `flags` - 标志位

**返回值**: `Result` - 操作结果

---

#### Get() (单缓冲区版本)

**声明位置**: 行 36

**完整签名**:
```cpp
Result Get(const char *key, mmc_buffer *buf, uint32_t flags);
```

**功能描述**: 从 MemCache 获取单个数据缓冲区

**参数**:
- `key` - 数据的键名
- `buf` - 输出数据缓冲区
- `flags` - 标志位

**返回值**: `Result` - 操作结果

---

#### Put() (缓冲区数组版本)

**声明位置**: 行 38

**完整签名**:
```cpp
Result Put(const std::string &key, const MmcBufferArray &bufArr, mmc_put_options &options, uint32_t flags);
```

**功能描述**: 存储多个缓冲区（分散/聚集 I/O）到 MemCache

**参数**:
- `key` - 数据的键名
- `bufArr` - 缓冲区数组
- `options` - 存储选项
- `flags` - 标志位

**返回值**: `Result` - 操作结果

---

#### Get() (缓冲区数组版本)

**声明位置**: 行 40

**完整签名**:
```cpp
Result Get(const std::string &key, const MmcBufferArray &bufArr, uint32_t flags);
```

**功能描述**: 从 MemCache 获取多个缓冲区

**参数**:
- `key` - 数据的键名
- `bufArr` - 输出缓冲区数组
- `flags` - 标志位

**返回值**: `Result` - 操作结果

---

#### BatchPut() (单缓冲区批量版本)

**声明位置**: 行 42-43

**完整签名**:
```cpp
Result BatchPut(const std::vector<std::string> &keys, const std::vector<mmc_buffer> &bufs,
                mmc_put_options &options, uint32_t flags, std::vector<int> &batchResult);
```

**功能描述**: 批量存储多个键值对

**参数**:
- `keys` - 键名列表
- `bufs` - 缓冲区列表
- `options` - 存储选项
- `flags` - 标志位
- `batchResult` - 输出每个键的操作结果

**返回值**: `Result` - 整体操作结果

---

#### BatchGet() (单缓冲区批量版本)

**声明位置**: 行 45-46

**完整签名**:
```cpp
Result BatchGet(const std::vector<std::string> &keys, std::vector<mmc_buffer> &bufs,
                uint32_t flags, std::vector<int> &batchResult);
```

**功能描述**: 批量获取多个键值对

**参数**:
- `keys` - 键名列表
- `bufs` - 输出缓冲区列表
- `flags` - 标志位
- `batchResult` - 输出每个键的操作结果

**返回值**: `Result` - 整体操作结果

---

#### BatchPut() (缓冲区数组批量版本)

**声明位置**: 行 48-49

**完整签名**:
```cpp
Result BatchPut(const std::vector<std::string> &keys, const std::vector<MmcBufferArray> &bufArrs,
                mmc_put_options &options, uint32_t flags, std::vector<int> &batchResult);
```

**功能描述**: 批量存储多个缓冲区数组

**参数**:
- `keys` - 键名列表
- `bufArrs` - 缓冲区数组列表
- `options` - 存储选项
- `flags` - 标志位
- `batchResult` - 输出每个键的操作结果

**返回值**: `Result` - 整体操作结果

---

#### BatchGet() (缓冲区数组批量版本)

**声明位置**: 行 51-52

**完整签名**:
```cpp
Result BatchGet(const std::vector<std::string> &keys, const std::vector<MmcBufferArray> &bufArrs,
                uint32_t flags, std::vector<int> &batchResult);
```

**功能描述**: 批量获取多个缓冲区数组

**参数**:
- `keys` - 键名列表
- `bufArrs` - 输出缓冲区数组列表
- `flags` - 标志位
- `batchResult` - 输出每个键的操作结果

**返回值**: `Result` - 整体操作结果

---

#### Remove()

**声明位置**: 行 54

**完整签名**:
```cpp
Result Remove(const char *key, uint32_t flags) const;
```

**功能描述**: 删除指定键的数据

**参数**:
- `key` - 要删除的键名
- `flags` - 标志位

**返回值**: `Result` - 操作结果

---

#### BatchRemove()

**声明位置**: 行 56

**完整签名**:
```cpp
Result BatchRemove(const std::vector<std::string> &keys, std::vector<Result> &remove_results, uint32_t flags) const;
```

**功能描述**: 批量删除多个键的数据

**参数**:
- `keys` - 要删除的键名列表
- `remove_results` - 输出每个键的删除结果
- `flags` - 标志位

**返回值**: `Result` - 整体操作结果

---

#### RemoveAll()

**声明位置**: 行 58

**完整签名**:
```cpp
Result RemoveAll(uint32_t flags) const;
```

**功能描述**: 删除所有数据

**参数**:
- `flags` - 标志位

**返回值**: `Result` - 操作结果

**注意事项**: 危险操作，慎用

---

#### IsExist()

**声明位置**: 行 60

**完整签名**:
```cpp
Result IsExist(const std::string &key, uint32_t flags) const;
```

**功能描述**: 检查指定键是否存在

**参数**:
- `key` - 要检查的键名
- `flags` - 标志位

**返回值**: `Result` - 存在返回 `MMC_OK`，否则返回错误码

---

#### BatchIsExist()

**声明位置**: 行 62-63

**完整签名**:
```cpp
Result BatchIsExist(const std::vector<std::string> &keys, std::vector<int32_t> &exist_results,
                    uint32_t flags) const;
```

**功能描述**: 批量检查多个键是否存在

**参数**:
- `keys` - 要检查的键名列表
- `exist_results` - 输出每个键的存在结果
- `flags` - 标志位

**返回值**: `Result` - 整体操作结果

---

#### Query()

**声明位置**: 行 65

**完整签名**:
```cpp
Result Query(const std::string &key, mmc_data_info &query_info, uint32_t flags) const;
```

**功能描述**: 查询指定键的元数据信息

**参数**:
- `key` - 要查询的键名
- `query_info` - 输出查询到的元数据信息
- `flags` - 标志位

**返回值**: `Result` - 操作结果

---

#### BatchQuery()

**声明位置**: 行 67-68

**完整签名**:
```cpp
Result BatchQuery(const std::vector<std::string> &keys, std::vector<mmc_data_info> &query_infos,
                  uint32_t flags) const;
```

**功能描述**: 批量查询多个键的元数据信息

**参数**:
- `keys` - 要查询的键名列表
- `query_infos` - 输出查询到的元数据信息列表
- `flags` - 标志位

**返回值**: `Result` - 整体操作结果

---

#### RegisterBuffer()

**声明位置**: 行 70

**完整签名**:
```cpp
Result RegisterBuffer(uint64_t addr, uint64_t size);
```

**功能描述**: 注册内存缓冲区用于 RDMA 等零拷贝操作

**参数**:
- `addr` - 缓冲区地址
- `size` - 缓冲区大小

**返回值**: `Result` - 操作结果

---

#### UnRegisterBuffer()

**声明位置**: 行 72

**完整签名**:
```cpp
Result UnRegisterBuffer(uint64_t addr, uint64_t size);
```

**功能描述**: 注销已注册的内存缓冲区

**参数**:
- `addr` - 缓冲区地址
- `size` - 缓冲区大小

**返回值**: `Result` - 操作结果

---

#### RankId()

**声明位置**: 行 74-77

**完整签名**:
```cpp
uint32_t RankId() const
{
    return rankId_;
}
```

**功能描述**: 获取当前客户端的 Rank ID

**返回值**: `uint32_t` - Rank ID

---

### 静态公共方法

#### RegisterInstance()

**声明位置**: 行 79-92

**完整签名**:
```cpp
static Result RegisterInstance()
```

**功能描述**: 注册全局客户端实例（单例模式）

**代码逻辑**:
1. 加锁保护
2. 检查是否已注册，已注册则直接返回成功
3. 创建默认名称为 "mmc_client" 的实例
4. 失败则返回错误

**返回值**: `Result` - 操作结果

---

#### UnregisterInstance()

**声明位置**: 行 94-104

**完整签名**:
```cpp
static Result UnregisterInstance()
```

**功能描述**: 注销全局客户端实例

**代码逻辑**:
1. 加锁保护
2. 检查是否已注销
3. 删除实例并置空

**返回值**: `Result` - 操作结果

---

#### GetInstance()

**声明位置**: 行 106-110

**完整签名**:
```cpp
static MmcClientDefault *GetInstance()
```

**功能描述**: 获取全局客户端实例指针

**返回值**: `MmcClientDefault*` - 客户端实例指针

---

### 私有成员函数

#### MmcClientDefault() (构造函数)

**声明位置**: 行 113

**完整签名**:
```cpp
explicit MmcClientDefault(const std::string &name) : name_(name) {}
```

**功能描述**: 私有构造函数，通过名称初始化

**参数**:
- `name` - 客户端名称

**注意事项**: 私有构造函数支持单例模式

---

#### RankId(const affinity_policy&) (策略版本)

**声明位置**: 行 117

**完整签名**:
```cpp
inline uint32_t RankId(const affinity_policy &policy);
```

**功能描述**: 根据亲和性策略获取 Rank ID

**参数**:
- `policy` - 亲和性策略（如 `NATIVE_AFFINITY`）

**返回值**: `uint32_t` - Rank ID

**注意事项**: 定义在头文件行 151-159

---

#### PrepareAllocOpt()

**声明位置**: 行 118-119

**完整签名**:
```cpp
Result PrepareAllocOpt(const MmcBufferArray &bufArr, const mmc_put_options &options, uint32_t flags,
                       AllocOptions &allocOpt);
```

**功能描述**: 准备分配选项，将用户配置转换为内部分配参数

**参数**:
- `bufArr` - 缓冲区数组
- `options` - 用户配置选项
- `flags` - 标志位
- `allocOpt` - 输出内部分配选项

**返回值**: `Result` - 操作结果

---

#### PrepareBlob()

**声明位置**: 行 120-121

**完整签名**:
```cpp
Result PrepareBlob(const MmcBufferArray &bufArr, const MmcMemBlobDesc &blob, MediaType &mediaType,
                   BatchCopyDesc &copyDesc, bool blobIsSrc);
```

**功能描述**: 准备单个 Blob 的拷贝描述

**参数**:
- `bufArr` - 缓冲区数组
- `blob` - Blob 描述符
- `mediaType` - 介质类型
- `copyDesc` - 输出拷贝描述
- `blobIsSrc` - true 表示 Blob 是源，false 表示 Blob 是目标

**返回值**: `Result` - 操作结果

---

#### PrepareMultiBlobs()

**声明位置**: 行 122-123

**完整签名**:
```cpp
Result PrepareMultiBlobs(const MmcBufferArray &bufArr, const std::vector<MmcMemBlobDesc> &blobs,
                         MediaType &mediaType, BatchCopyDesc &copyDesc, bool blobIsSrc);
```

**功能描述**: 准备多个 Blob 的拷贝描述

**参数**:
- `bufArr` - 缓冲区数组
- `blobs` - Blob 描述符列表
- `mediaType` - 介质类型
- `copyDesc` - 输出拷贝描述
- `blobIsSrc` - true 表示 Blob 是源，false 表示 Blob 是目标

**返回值**: `Result` - 操作结果

---

#### PutData2Blobs()

**声明位置**: 行 124-125

**完整签名**:
```cpp
Result PutData2Blobs(const std::vector<std::string> &keys, const std::vector<MmcBufferArray> &bufArrs,
                     const BatchAllocResponse &allocResponse, std::vector<int> &batchResult);
```

**功能描述**: 将数据写入多个 Blob（用于批量操作）

**参数**:
- `keys` - 键名列表
- `bufArrs` - 缓冲区数组列表
- `allocResponse` - 分配响应
- `batchResult` - 输出批处理结果

**返回值**: `Result` - 操作结果

---

#### WaitFeatures()

**声明位置**: 行 126-127

**完整签名**:
```cpp
void WaitFeatures(std::vector<std::tuple<uint32_t, uint32_t, std::future<int32_t>>> &futures,
                  std::vector<int> &batchResult);
```

**功能描述**: 等待所有异步任务完成并处理结果

**参数**:
- `futures` - 异步任务 future 列表（包含起始键索引和结束键索引）
- `batchResult` - 输入/输出批处理结果

---

#### SyncUpdateState()

**声明位置**: 行 128

**完整签名**:
```cpp
void SyncUpdateState(BatchUpdateRequest &updateRequest);
```

**功能描述**: 同步更新元数据状态

**参数**:
- `updateRequest` - 更新请求

---

#### AsyncUpdateState()

**声明位置**: 行 129

**完整签名**:
```cpp
void AsyncUpdateState(BatchUpdateRequest &updateRequest);
```

**功能描述**: 异步更新元数据状态

**参数**:
- `updateRequest` - 更新请求

---

#### SubmitPutTask()

**声明位置**: 行 130

**完整签名**:
```cpp
std::future<int32_t> SubmitPutTask(BatchCopyDesc &copyDesc, MediaType mediaType, bool asyncExec);
```

**功能描述**: 提交写任务到线程池

**参数**:
- `copyDesc` - 拷贝描述
- `mediaType` - 介质类型
- `asyncExec` - 是否异步执行

**返回值**: `std::future<int32_t>` - 可获取任务结果的 future

---

#### SubmitGetTask()

**声明位置**: 行 131

**完整签名**:
```cpp
std::future<int32_t> SubmitGetTask(BatchCopyDesc &copyDesc, MediaType mediaType, bool asyncExec);
```

**功能描述**: 提交读任务到线程池

**参数**:
- `copyDesc` - 拷贝描述
- `mediaType` - 介质类型
- `asyncExec` - 是否异步执行

**返回值**: `std::future<int32_t>` - 可获取任务结果的 future

---

## 成员变量

| 变量名 | 类型 | 描述 |
|--------|------|------|
| `gClientHandlerMtx` | `static std::mutex` | 全局实例互斥锁 |
| `gClientHandler` | `static MmcClientDefault*` | 全局客户端实例指针 |
| `mutex_` | `std::mutex` | 实例互斥锁 |
| `started_` | `bool` | 是否已启动 |
| `metaNetClient_` | `MetaNetClientPtr` | 元数据网络客户端 |
| `bmProxy_` | `MmcBmProxyPtr` | Blob Manager 代理 |
| `name_` | `std::string` | 客户端名称 |
| `rankId_` | `uint32_t` | Rank ID |
| `rpcRetryTimeOut_` | `uint32_t` | RPC 重试超时时间 |
| `defaultTtlMs_` | `uint64_t` | 默认 TTL（毫秒） |
| `threadPool_` | `MmcThreadPoolPtr` | 通用线程池 |
| `readThreadPool_` | `MmcThreadPoolPtr` | 读线程池 |
| `writeThreadPool_` | `MmcThreadPoolPtr` | 写线程池 |
| `aggregateIO_` | `bool` | 是否启用 IO 聚合 |
| `aggregateNum_` | `size_t` | 聚合数量 |

---

## 类型别名

**声明位置**: 行 160

```cpp
using MmcClientDefaultPtr = MmcRef<MmcClientDefault>;
```

**功能描述**: 客户端智能指针类型别名

---

## 数据结构关系图

```
┌────────────────────────────────────────────────────────────────────┐
│                        MmcClientDefault                             │
├────────────────────────────────────────────────────────────────────┤
│  成员变量:                                                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │ MetaNetClient   │  │  MmcBmProxy     │  │ MmcThreadPool   │    │
│  │   (metaNetClient_)│  │   (bmProxy_)    │  │(threadPool_等)  │    │
│  └────────┬────────┘  └────────┬────────┘  └─────────────────┘    │
│           │                    │                                    │
│           └─────────┬──────────┘                                    │
│                     ▼                                               │
│              元数据 & Blob 操作                                       │
├────────────────────────────────────────────────────────────────────┤
│  单例模式:                                                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  static gClientHandler: MmcClientDefault*                    │  │
│  │  static gClientHandlerMtx: std::mutex                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

---

## 使用示例

```cpp
// 注册全局实例
MmcClientDefault::RegisterInstance();

// 获取实例
auto* client = MmcClientDefault::GetInstance();

// 启动客户端
mmc_client_config_t config;
config.readThreadPoolNum = 4;
config.writeThreadPoolNum = 4;
config.aggregateIO = true;
config.aggregateNum = 16;
client->Start(config);

// 存储数据
mmc_buffer buf = { .addr = reinterpret_cast<uint64_t>(data), .len = size };
mmc_put_options options;
options.replicaNum = 2;
options.policy = NATIVE_AFFINITY;
client->Put("my_key", buf, options, 0);

// 获取数据
mmc_buffer outBuf = { .addr = reinterpret_cast<uint64_t>(outData), .len = size };
client->Get("my_key", outBuf, 0);

// 批量操作
std::vector<std::string> keys = {"key1", "key2", "key3"};
std::vector<MmcBufferArray> bufArrs = ...;
std::vector<int> results;
client->BatchPut(keys, bufArrs, options, 0, results);

// 查询数据
mmc_data_info info;
client->Query("my_key", info, 0);

// 停止客户端
client->Stop();

// 注销实例
MmcClientDefault::UnregisterInstance();
```
