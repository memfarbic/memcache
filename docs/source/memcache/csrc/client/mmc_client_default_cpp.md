# mmc_client_default.cpp 逐函数解读

## 文件概述

- **文件路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/client/mmc_client_default.cpp`
- **文件用途**: 实现 `MmcClientDefault` 类的具体功能，包括客户端启动/停止、数据的存储/读取/删除/查询等操作
- **依赖项**:
  - `mmc_client_default.h` - 客户端头文件
  - `mmc_msg_client_meta.h` - 客户端元数据消息定义
  - `mmc_mem_obj_meta.h` - 内存对象元数据
  - `mmc_bm_proxy.h` - Blob Manager 代理
  - `mmc_montotonic.h` - 单调时间
  - `mmc_ptracer.h` - 性能追踪

---

## 常量定义

### CLIENT_THREAD_COUNT

**声明位置**: 行 22

```cpp
constexpr int CLIENT_THREAD_COUNT = 2;
```

**功能描述**: 客户端网络引擎的线程数量

---

### MMC_REGISTER_SET_MARK_BIT

**声明位置**: 行 23

```cpp
constexpr uint32_t MMC_REGISTER_SET_MARK_BIT = 1U;
```

**功能描述**: 注册设置的标志位

---

### MMC_REGISTER_SET_LEFT_MARK

**声明位置**: 行 24

```cpp
constexpr uint32_t MMC_REGISTER_SET_LEFT_MARK = 1U;
```

**功能描述**: 注册左侧标志位

---

### MMC_BATCH_TRANSPORT

**声明位置**: 行 25

```cpp
constexpr int32_t MMC_BATCH_TRANSPORT = 1U;
```

**功能描述**: 批量传输模式标志

---

### MMC_ASYNC_TRANSPORT

**声明位置**: 行 26

```cpp
constexpr int32_t MMC_ASYNC_TRANSPORT = 2U;
```

**功能描述**: 异步传输模式标志

---

### KEY_MAX_LENTH

**声明位置**: 行 27

```cpp
constexpr uint32_t KEY_MAX_LENTH = 256U;
```

**功能描述**: 键名最大长度（字符数）

---

## 静态成员初始化

### gClientHandler / gClientHandlerMtx

**声明位置**: 行 29-30

```cpp
MmcClientDefault *MmcClientDefault::gClientHandler = nullptr;
std::mutex MmcClientDefault::gClientHandlerMtx;
```

**功能描述**: 初始化全局单例指针和互斥锁

---

## 函数实现

### Start()

**声明位置**: 行 32-79

**完整签名**:
```cpp
Result MmcClientDefault::Start(const mmc_client_config_t &config)
```

**功能描述**: 启动客户端，初始化 BM 代理、线程池和网络客户端

**代码逻辑**:
1. 加锁检查是否已启动，避免重复启动
2. 获取 BM 代理实例并获取 Rank ID
3. 创建并启动通用线程池（1 个线程）
4. 创建并启动读线程池（可配置数量）
5. 创建并启动写线程池（可配置数量）
6. 从工厂获取元数据网络客户端实例
7. 如果网络客户端未启动，则启动并连接到发现服务
8. 保存 RPC 重试超时时间，设置启动标志

**参数**:
- `config` - 客户端配置结构

**返回值**: `Result` - 成功返回 `MMC_OK`，失败返回相应错误码

**注意事项**:
- 聚合 IO 和聚合数量从配置读取
- 发现 URL 必须以 null 结尾

---

### Stop()

**声明位置**: 行 81-103

**完整签名**:
```cpp
void MmcClientDefault::Stop()
```

**功能描述**: 停止客户端，清理所有资源

**代码逻辑**:
1. 加锁检查是否已启动
2. 销毁读线程池
3. 销毁写线程池
4. 销毁通用线程池
5. 停止并清空元数据网络客户端
6. 重置启动标志

**注意事项**: 如果未启动，仅记录警告日志

---

### Name()

**声明位置**: 行 105-108

**完整签名**:
```cpp
const std::string &MmcClientDefault::Name() const
```

**功能描述**: 返回客户端名称

**返回值**: `const std::string&` - 客户端名称的常量引用

---

### Put() (C 字符串版本)

**声明位置**: 行 110-123

**完整签名**:
```cpp
Result MmcClientDefault::Put(const char *key, mmc_buffer *buf, mmc_put_options &options, uint32_t flags)
```

**功能描述**: 存储单个数据缓冲区（C 字符串键名）

**代码逻辑**:
1. 验证 BM 代理和元数据客户端已初始化
2. 验证参数（缓冲区、键名非空且长度合法）
3. 创建 `MmcBufferArray` 并添加缓冲区
4. 调用 `MmcBufferArray` 版本的 Put 方法

**参数验证**:
- `buf != nullptr`
- `key != nullptr`
- `key[0] != '\0'`
- `strnlen(key, KEY_MAX_LENTH + 1) != KEY_MAX_LENTH + 1`（键名不超过最大长度）

---

### PrepareAllocOpt()

**声明位置**: 行 125-148

**完整签名**:
```cpp
Result MmcClientDefault::PrepareAllocOpt(const MmcBufferArray &bufArr, const mmc_put_options &options, uint32_t flags,
                                         AllocOptions &allocOpt)
```

**功能描述**: 准备分配选项，将用户配置转换为内部分配参数

**代码逻辑**:
1. 设置 Blob 大小为缓冲区总大小
2. 设置 Blob 数量为 `max(replicaNum, 1)`
3. 设置介质类型为 `MEDIA_NONE`（自动选择）
4. 复制标志位
5. 使用 `std::copy_if` 复制有效的 `preferredLocalServiceIDs`（非负数）
6. 验证 preferredRank 数量不超过副本数
7. 如果有首选 Rank，设置 `ALLOC_FORCE_BY_RANK` 标志
8. 否则使用亲和性策略获取 Rank ID

**参数**:
- `bufArr` - 缓冲区数组
- `options` - 用户配置选项
- `flags` - 标志位
- `allocOpt` - 输出内部分配选项

**返回值**: `Result` - 成功返回 `MMC_OK`，参数错误返回 `MMC_INVALID_PARAM`

**注意事项**: `preferredRank_` 数量必须小于等于 `replicaNum`

---

### Put() (BufferArray 版本)

**声明位置**: 行 150-194

**完整签名**:
```cpp
Result MmcClientDefault::Put(const std::string &key, const MmcBufferArray &bufArr, mmc_put_options &options,
                             uint32_t flags)
```

**功能描述**: 存储缓冲区数组数据

**代码逻辑**:
1. 验证 BM 代理和元数据客户端已初始化
2. 验证缓冲区数组非空
3. 生成操作 ID
4. 准备分配请求和选项
5. 调用元数据服务分配 Blob
6. 检查分配结果（处理重复对象情况）
7. 遍历所有 Blob，使用 BM 代理写入数据
8. 构建更新请求（记录每个 Blob 的写入结果）
9. 同步更新元数据状态

**返回值**: `Result` - 成功返回 `MMC_OK`，失败返回错误码

**注意事项**: 元数据状态同步更新，避免立即读取时 Blob 不可读

---

### BatchPut() (单缓冲区批量版本)

**声明位置**: 行 196-217

**完整签名**:
```cpp
Result MmcClientDefault::BatchPut(const std::vector<std::string> &keys, const std::vector<mmc_buffer> &bufs,
                                  mmc_put_options &options, uint32_t flags, std::vector<int> &batchResult)
```

**功能描述**: 批量存储单缓冲区数据

**代码逻辑**:
1. 验证 BM 代理和元数据客户端已初始化
2. 验证参数（键列表和缓冲区列表非空且大小相等）
3. 将每个单缓冲区包装成 `MmcBufferArray`
4. 调用 `MmcBufferArray` 版本的 `BatchPut`

**参数验证**:
- `!keys.empty() && !bufs.empty() && keys.size() == bufs.size()`

---

### BatchPut() (BufferArray 批量版本)

**声明位置**: 行 219-270

**完整签名**:
```cpp
Result MmcClientDefault::BatchPut(const std::vector<std::string> &keys, const std::vector<MmcBufferArray> &bufArrs,
                                  mmc_put_options &options, uint32_t flags, std::vector<int> &batchResult)
```

**功能描述**: 批量存储缓冲区数组数据

**代码逻辑**:
1. 验证 BM 代理和元数据客户端已初始化
2. 验证参数（键列表和缓冲区数组列表非空且大小相等）
3. 生成操作 ID
4. 为每个键准备分配选项，构建批量分配请求
5. 调用元数据服务批量分配 Blob
6. 验证分配响应大小匹配
7. 调用 `PutData2Blobs` 写入数据
8. 构建批量更新请求
9. **同步更新**元数据状态（避免立即读取问题）

**返回值**: `Result` - 成功返回 `MMC_OK`

**注意事项**: 写操作必须同步更新状态，否则会出现立即读查询 Blob 不可读的情况

---

### Get() (C 字符串版本)

**声明位置**: 行 272-282

**完整签名**:
```cpp
Result MmcClientDefault::Get(const char *key, mmc_buffer *buf, uint32_t flags)
```

**功能描述**: 获取单个数据（C 字符串键名）

**代码逻辑**:
1. 验证参数（缓冲区、键名非空且长度合法）
2. 创建 `MmcBufferArray` 并添加缓冲区
3. 调用 `MmcBufferArray` 版本的 Get 方法

---

### Get() (BufferArray 版本)

**声明位置**: 行 284-315

**完整签名**:
```cpp
Result MmcClientDefault::Get(const std::string &key, const MmcBufferArray &bufArr, uint32_t flags)
```

**功能描述**: 获取缓冲区数组数据

**代码逻辑**:
1. 验证 BM 代理和元数据客户端已初始化
2. 生成操作 ID
3. 构建获取请求
4. 调用元数据服务查询 Blob 位置
5. 验证响应中包含 Blob
6. 使用 BM 代理读取数据
7. 构建更新请求并异步更新元数据状态

**返回值**: `Result` - 成功返回 `MMC_OK`，失败返回错误码

**注意事项**: 读操作使用异步更新状态，不阻塞返回

---

### BatchGet() (单缓冲区批量版本)

**声明位置**: 行 317-336

**完整签名**:
```cpp
Result MmcClientDefault::BatchGet(const std::vector<std::string> &keys, std::vector<mmc_buffer> &bufs, uint32_t flags,
                                  std::vector<int> &batchResult)
```

**功能描述**: 批量获取单缓冲区数据

**代码逻辑**:
1. 验证 BM 代理和元数据客户端已初始化
2. 验证参数（键列表和缓冲区列表非空且大小相等）
3. 将每个单缓冲区包装成 `MmcBufferArray`
4. 调用 `MmcBufferArray` 版本的 `BatchGet`

---

### BatchGet() (BufferArray 批量版本)

**声明位置**: 行 338-424

**完整签名**:
```cpp
Result MmcClientDefault::BatchGet(const std::vector<std::string> &keys, const std::vector<MmcBufferArray> &bufArrs,
                                  uint32_t flags, std::vector<int> &batchResult)
```

**功能描述**: 批量获取缓冲区数组数据

**代码逻辑**:
1. 验证 BM 代理和元数据客户端已初始化
2. 验证参数（键列表和缓冲区数组列表非空且大小相等）
3. 初始化批处理结果为 `MMC_ERROR`
4. 生成操作 ID
5. 构建批量获取请求
6. 调用元数据服务查询所有 Blob 位置
7. 验证响应大小匹配
8. **聚合读取优化**:
   - 遍历每个键，准备拷贝描述
   - 如果启用聚合 IO 且未达到聚合数量，继续累积
   - 达到聚合数量或最后一个键时，提交读取任务
9. 等待所有异步任务完成
10. 异步更新元数据状态

**返回值**: `Result` - 成功返回 `MMC_OK`

**聚合 IO 策略**:
- 避免单流读取地址太少（减少调用栈开销）
- 避免聚合地址太多（保持并发性能）
- 实测单流性能低于多流

---

### Remove()

**声明位置**: 行 426-435

**完整签名**:
```cpp
Result MmcClientDefault::Remove(const char *key, uint32_t flags) const
```

**功能描述**: 删除指定键的数据

**代码逻辑**:
1. 验证元数据客户端已初始化
2. 构建删除请求
3. 调用元数据服务执行删除

**返回值**: `Result` - 操作结果

---

### BatchRemove()

**声明位置**: 行 437-457

**完整签名**:
```cpp
Result MmcClientDefault::BatchRemove(const std::vector<std::string> &keys, std::vector<Result> &remove_results,
                                     uint32_t flags) const
```

**功能描述**: 批量删除多个键的数据

**代码逻辑**:
1. 验证元数据客户端已初始化
2. 构建批量删除请求
3. 调用元数据服务执行批量删除
4. 验证响应大小，填充结果

**参数验证**: 响应结果数量必须与键数量相等

**返回值**: `Result` - 整体操作结果

---

### RemoveAll()

**声明位置**: 行 459-470

**完整签名**:
```cpp
Result MmcClientDefault::RemoveAll(uint32_t flags) const
```

**功能描述**: 删除所有数据

**代码逻辑**:
1. 验证元数据客户端已初始化
2. 构建删除所有请求
3. 调用元数据服务执行删除所有

**返回值**: `Result` - 操作结果

**注意事项**: 危险操作，会删除所有数据

---

### IsExist()

**声明位置**: 行 472-486

**完整签名**:
```cpp
Result MmcClientDefault::IsExist(const std::string &key, uint32_t flags) const
```

**功能描述**: 检查指定键是否存在

**代码逻辑**:
1. 验证元数据客户端已初始化
2. 验证键非空
3. 构建存在性检查请求
4. 调用元数据服务检查

**返回值**: `Result` - 存在返回 `MMC_OK`，否则返回错误码

---

### BatchIsExist()

**声明位置**: 行 488-512

**完整签名**:
```cpp
Result MmcClientDefault::BatchIsExist(const std::vector<std::string> &keys, std::vector<int32_t> &exist_results,
                                      uint32_t flags) const
```

**功能描述**: 批量检查多个键是否存在

**代码逻辑**:
1. 验证元数据客户端已初始化
2. 验证键列表非空
3. 构建批量存在性检查请求
4. 调用元数据服务批量检查
5. 验证响应大小，填充结果

**返回值**: `Result` - 整体操作结果

---

### Query()

**声明位置**: 行 514-537

**完整签名**:
```cpp
Result MmcClientDefault::Query(const std::string &key, mmc_data_info &query_info, uint32_t flags) const
```

**功能描述**: 查询指定键的元数据信息

**代码逻辑**:
1. 验证元数据客户端已初始化
2. 验证键非空
3. 构建查询请求
4. 调用元数据服务查询
5. 填充 `mmc_data_info` 结构：
   - `size` - 数据大小
   - `prot` - 保护属性
   - `numBlobs` - Blob 数量
   - `valid` - 是否有效
   - `ranks[]` - Rank ID 列表
   - `types[]` - 介质类型列表

**返回值**: `Result` - 操作结果

---

### BatchQuery()

**声明位置**: 行 539-580

**完整签名**:
```cpp
Result MmcClientDefault::BatchQuery(const std::vector<std::string> &keys, std::vector<mmc_data_info> &query_infos,
                                    uint32_t flags) const
```

**功能描述**: 批量查询多个键的元数据信息

**代码逻辑**:
1. 验证元数据客户端已初始化
2. 验证键列表非空
3. 构建批量查询请求
4. 调用元数据服务批量查询
5. 验证响应大小
6. 遍历响应，为每个键填充 `mmc_data_info`

**返回值**: `Result` - 操作结果

---

### WaitFeatures()

**声明位置**: 行 582-599

**完整签名**:
```cpp
void MmcClientDefault::WaitFeatures(std::vector<std::tuple<uint32_t, uint32_t, std::future<int32_t>>> &futures,
                                    std::vector<int> &batchResult)
```

**功能描述**: 等待所有异步任务完成并处理失败结果

**代码逻辑**:
1. 遍历所有 future 元组
2. 获取每个 future 的结果
3. 如果结果不是 `MMC_OK`：
   - 记录错误日志
   - 将错误码更新到对应范围的 `batchResult`
4. 成功的结果保持不变

**参数**:
- `futures` - 元组列表：`(起始键索引, 结束键索引, future)`
- `batchResult` - 输入/输出批处理结果

**注意事项**: 只覆盖原本为 `MMC_OK` 的结果，保留已有的错误码

---

### SyncUpdateState()

**声明位置**: 行 601-618

**完整签名**:
```cpp
void MmcClientDefault::SyncUpdateState(BatchUpdateRequest &updateRequest)
```

**功能描述**: 同步更新元数据状态

**代码逻辑**:
1. 开始性能追踪
2. 调用元数据服务同步更新
3. 结束性能追踪
4. 检查更新结果，记录错误日志

**参数**:
- `updateRequest` - 更新请求（包含键、Rank、介质类型、操作结果）

---

### AsyncUpdateState()

**声明位置**: 行 620-627

**完整签名**:
```cpp
void MmcClientDefault::AsyncUpdateState(BatchUpdateRequest &updateRequest)
```

**功能描述**: 异步更新元数据状态

**代码逻辑**:
1. 将更新任务提交到通用线程池
2. 如果提交失败（future 无效），降级为同步更新

**参数**:
- `updateRequest` - 更新请求

**注意事项**: 使用值捕获避免引用失效

---

### PrepareBlob()

**声明位置**: 行 629-660

**完整签名**:
```cpp
Result MmcClientDefault::PrepareBlob(const MmcBufferArray &bufArr, const MmcMemBlobDesc &blob, MediaType &mediaType,
                                     BatchCopyDesc &copyDesc, bool blobIsSrc)
```

**功能描述**: 准备单个 Blob 的拷贝描述

**代码逻辑**:
1. 验证缓冲区数组非空
2. 遍历缓冲区数组中的每个缓冲区：
   - 验证缓冲区类型有效（非 `MEDIA_NONE`）
   - 确定并验证介质类型一致性
   - 根据 `blobIsSrc` 设置源地址和目标地址：
     - `blobIsSrc=true`: Blob 是源，缓冲区是目标（读操作）
     - `blobIsSrc=false`: 缓冲区是源，Blob 是目标（写操作）
   - 记录每个段的大小
   - 累加偏移量

**参数**:
- `bufArr` - 缓冲区数组
- `blob` - Blob 描述符（包含 GVA 地址）
- `mediaType` - 介质类型（输入/输出）
- `copyDesc` - 输出拷贝描述
- `blobIsSrc` - true 表示 Blob 是源，false 表示 Blob 是目标

**返回值**: `Result` - 成功返回 `MMC_OK`，失败返回错误码

---

### PrepareMultiBlobs()

**声明位置**: 行 662-672

**完整签名**:
```cpp
Result MmcClientDefault::PrepareMultiBlobs(const MmcBufferArray &bufArr, const std::vector<MmcMemBlobDesc> &blobs,
                                           MediaType &mediaType, BatchCopyDesc &copyDesc, bool blobIsSrc)
```

**功能描述**: 准备多个 Blob 的拷贝描述

**代码逻辑**:
1. 遍历所有 Blob
2. 对每个 Blob 调用 `PrepareBlob`
3. 任何失败则立即返回错误码

**参数**:
- `bufArr` - 缓冲区数组
- `blobs` - Blob 描述符列表
- `mediaType` - 介质类型（输入/输出）
- `copyDesc` - 输出拷贝描述
- `blobIsSrc` - true 表示 Blob 是源，false 表示 Blob 是目标

**返回值**: `Result` - 成功返回 `MMC_OK`，失败返回错误码

---

### SubmitPutTask()

**声明位置**: 行 674-695

**完整签名**:
```cpp
std::future<int32_t> MmcClientDefault::SubmitPutTask(BatchCopyDesc &copyDesc, MediaType mediaType, bool asyncExec)
```

**功能描述**: 提交写任务到线程池

**代码逻辑**:
1. 如果允许异步执行：
   - 将任务提交到写线程池
   - 如果提交成功，返回 future
2. 如果异步失败或不允许异步：
   - 创建已完成的 promise
   - 直接执行写操作（带性能追踪）
   - 设置 promise 值
   - 返回 future

**参数**:
- `copyDesc` - 拷贝描述
- `mediaType` - 介质类型
- `asyncExec` - 是否异步执行

**返回值**: `std::future<int32_t>` - 可获取任务结果的 future

**注意事项**: 同步执行时返回已就绪的 future

---

### SubmitGetTask()

**声明位置**: 行 697-717

**完整签名**:
```cpp
std::future<int32_t> MmcClientDefault::SubmitGetTask(BatchCopyDesc &copyDesc, MediaType mediaType, bool asyncExec)
```

**功能描述**: 提交读任务到线程池

**代码逻辑**:
1. 如果允许异步执行：
   - 将任务提交到读线程池
   - 如果提交成功，返回 future
2. 如果异步失败或不允许异步：
   - 创建已完成的 promise
   - 直接执行读操作（带性能追踪）
   - 设置 promise 值
   - 返回 future

**参数**:
- `copyDesc` - 拷贝描述
- `mediaType` - 介质类型
- `asyncExec` - 是否异步执行

**返回值**: `std::future<int32_t>` - 可获取任务结果的 future

---

### PutData2Blobs()

**声明位置**: 行 719-773

**完整签名**:
```cpp
Result MmcClientDefault::PutData2Blobs(const std::vector<std::string> &keys, const std::vector<MmcBufferArray> &bufArrs,
                                       const BatchAllocResponse &allocResponse, std::vector<int> &batchResult)
```

**功能描述**: 将数据写入多个 Blob（用于批量 Put 操作）

**代码逻辑**:
1. 初始化介质类型和 future 列表
2. 遍历所有键：
   - 跳过分配失败的键（保留错误码）
   - 验证 Blob 数量
   - 准备多 Blob 拷贝描述
   - 累积到总拷贝描述
   - 如果启用聚合 IO 且未达到聚合数量，继续累积
   - 达到条件时提交写任务
3. 处理剩余的拷贝描述
4. 等待所有任务完成

**参数**:
- `keys` - 键名列表
- `bufArrs` - 缓冲区数组列表
- `allocResponse` - 分配响应
- `batchResult` - 输入/输出批处理结果

**返回值**: `Result` - 成功返回 `MMC_OK`

**聚合策略**: 与批量读取类似，平衡调用开销和并发性能

---

### RegisterBuffer()

**声明位置**: 行 775-778

**完整签名**:
```cpp
Result MmcClientDefault::RegisterBuffer(uint64_t addr, uint64_t size)
```

**功能描述**: 注册内存缓冲区

**代码逻辑**: 直接委托给 BM 代理

---

### UnRegisterBuffer()

**声明位置**: 行 780-783

**完整签名**:
```cpp
Result MmcClientDefault::UnRegisterBuffer(uint64_t addr, uint64_t size)
```

**功能描述**: 注销内存缓冲区

**代码逻辑**: 直接委托给 BM 代理（注意：只使用地址参数）

---

## 数据流和关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         MmcClientDefault                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Put 操作流程:                                                            │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────┐   │
│  │ 用户数据  │───▶│PrepareAllocOpt│───▶│MetaNetClient│───▶│Alloc Blob│   │
│  └──────────┘    └──────────────┘    └─────────────┘    └────┬─────┘   │
│                                                           │             │
│                                                           ▼             │
│                                                  ┌──────────────┐       │
│                                                  │PrepareBlob   │       │
│                                                  └──────┬───────┘       │
│                                                         │               │
│                                                         ▼               │
│                                                  ┌──────────────┐       │
│                                                  │SubmitPutTask │       │
│                                                  └──────┬───────┘       │
│                                                         │               │
│                                    ┌────────────────────┴────────────┐  │
│                                    ▼                                 ▼  │
│                             ┌─────────────┐                   ┌──────────┐│
│                             │WriteThread  │                   │直接执行  ││
│                             │   Pool      │                   └────┬─────┘│
│                             └──────┬──────┘                        │      │
│                                    │                               │      │
│                                    ▼                               │      │
│                             ┌─────────────┐                       │      │
│                             │MmcBmProxy   │◄──────────────────────┘      │
│                             └──────┬──────┘                              │
│                                    │                                     │
│                                    ▼                                     │
│                             ┌─────────────┐                              │
│                             │SyncUpdate   │◄──── 更新元数据               │
│                             │State        │                              │
│                             └─────────────┘                              │
│                                                                          │
│  Get 操作流程:                                                            │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────┐   │
│  │ 查询请求  │───▶│MetaNetClient │───▶│ Get Blob    │───▶│PrepareBlob│   │
│  └──────────┘    └──────────────┘    └─────────────┘    └────┬─────┘   │
│                                                           │             │
│                                                           ▼             │
│                                                  ┌──────────────┐       │
│                                                  │SubmitGetTask │       │
│                                                  └──────┬───────┘       │
│                                                         │               │
│                                    ┌────────────────────┴────────────┐  │
│                                    ▼                                 ▼  │
│                             ┌─────────────┐                   ┌──────────┐│
│                             │ReadThread   │                   │直接执行  ││
│                             │   Pool      │                   └────┬─────┘│
│                             └──────┬──────┘                        │      │
│                                    │                               │      │
│                                    ▼                               │      │
│                             ┌─────────────┐                       │      │
│                             │MmcBmProxy   │◄──────────────────────┘      │
│                             └──────┬──────┘                              │
│                                    │                                     │
│                                    ▼                                     │
│                             ┌─────────────┐                              │
│                             │AsyncUpdate  │◄──── 异步更新元数据           │
│                             │State        │                              │
│                             └─────────────┘                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 使用示例

```cpp
// 创建并启动客户端
auto* client = new MmcClientDefault("my_client");
mmc_client_config_t config = {
    .readThreadPoolNum = 4,
    .writeThreadPoolNum = 4,
    .aggregateIO = true,
    .aggregateNum = 16,
    .rpcRetryTimeOut = 5000,
};
client->Start(config);

// 单个数据存储
char data[] = "Hello World";
mmc_buffer buf = {
    .addr = reinterpret_cast<uint64_t>(data),
    .len = sizeof(data),
    .offset = 0,
    .type = MEDIA_DRAM
};
mmc_put_options options = {
    .replicaNum = 2,
    .policy = NATIVE_AFFINITY,
};
client->Put("key1", buf, options, 0);

// 批量存储
std::vector<std::string> keys = {"key1", "key2", "key3"};
std::vector<MmcBufferArray> bufArrs = ...;
std::vector<int> results;
client->BatchPut(keys, bufArrs, options, 0, results);

// 读取数据
char outData[1024];
mmc_buffer outBuf = {
    .addr = reinterpret_cast<uint64_t>(outData),
    .len = 1024,
    .offset = 0,
    .type = MEDIA_DRAM
};
client->Get("key1", outBuf, 0);

// 查询元数据
mmc_data_info info;
client->Query("key1", info, 0);
printf("Size: %lu, Blobs: %u\n", info.size, info.numBlobs);

// 停止客户端
client->Stop();
```
