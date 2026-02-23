# mmcache_store.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/mmcache_store.cpp`
- **文件用途**: 实现 MmcacheStore 类，提供 C++ ObjectStore 接口的具体实现
- **依赖项**: `mmc_client.h`, `mmc_client_default.h`, `mmc.h`, `mmc_logger.h`, `mmc_types.h`, `mmc_ptracer.h`, `smem_bm_def.h`, `mmcache_store.h`

---

## 常量定义

### MAX_LAYER_NUM (第28行)

```cpp
constexpr int MAX_LAYER_NUM = 255;
```

**说明**: 最大层数限制（用于分层存储）

---

### MAX_BATCH_SIZE (第29行)

```cpp
constexpr int MAX_BATCH_SIZE = 512;
```

**说明**: 批量操作的最大数量

---

### MAX_KEY_LEN (第30行)

```cpp
constexpr int MAX_KEY_LEN = 256;
```

**说明**: 键的最大长度

---

### 设备地址范围 (第31-32行)

```cpp
constexpr uint64_t MMC_DEVICE_VA_START = 0x100000000000UL;      // NPU上的地址空间起始: 16T
constexpr uint64_t MMC_DEVICE_VA_SIZE = 0x80000000000UL;        // NPU上的地址空间范围: 8T
```

**说明**: 定义 NPU 设备地址空间的范围

---

## 工具函数

### CopyPutOptions() (第34-57行)

```cpp
static bool CopyPutOptions(const ReplicateConfig &replicateConfig, mmc_put_options &options)
```

**声明位置**: 行 34-57

**功能描述**: 将 C++ 副本配置转换为 C mmc_put_options 结构

**参数**:
- `replicateConfig` [in]: C++ 副本配置
- `options` [out]: 输出 C 结构体选项

**返回值**:
- `true`: 转换成功
- `false`: 参数无效

**代码逻辑**:
1. 验证首选服务 ID 数量不超过 `MAX_BLOB_COPIES` (8)
2. 验证副本数在 1-8 范围内
3. 设置 mediaType 为 0（由客户端代理设置）
4. 设置策略为 `NATIVE_AFFINITY`
5. 复制首选服务 ID 列表

---

## ResourceTracker 实现

### getInstance() (第60-64行)

```cpp
ResourceTracker &ResourceTracker::getInstance()
{
    static ResourceTracker instance;
    return instance;
}
```

**声明位置**: 行 60-64

**功能描述**: 获取 ResourceTracker 单例（Meyer's Singleton）

---

### 构造函数 (第66-81行)

```cpp
ResourceTracker::ResourceTracker()
{
    // 设置信号处理器
    struct sigaction sa{};
    sa.sa_handler = signalHandler;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;

    // 注册常见终止信号
    sigaction(SIGINT, &sa, nullptr);  // Ctrl+C
    sigaction(SIGTERM, &sa, nullptr); // kill 命令
    sigaction(SIGHUP, &sa, nullptr);  // 终端关闭

    // 注册退出处理器
    std::atexit(exitHandler);
}
```

**声明位置**: 行 66-81

**功能描述**: 构造 ResourceTracker，注册信号和退出处理器

**捕获的信号**:
- `SIGINT`: 用户按下 Ctrl+C
- `SIGTERM`: kill 命令发送的终止信号
- `SIGHUP`: 终端关闭信号

---

### registerInstance() / unregisterInstance() (第88-98行)

```cpp
void ResourceTracker::registerInstance(MmcacheStore *instance)
{
    std::lock_guard<std::mutex> lock(mutex_);
    instances_.insert(instance);
}

void ResourceTracker::unregisterInstance(MmcacheStore *instance)
{
    std::lock_guard<std::mutex> lock(mutex_);
    instances_.erase(instance);
}
```

**声明位置**: 行 88-98

**功能描述**: 注册/反注册 MmcacheStore 实例

---

### cleanupAllResources() (第100-112行)

```cpp
void ResourceTracker::cleanupAllResources()
{
    std::lock_guard<std::mutex> lock(mutex_);

    // 在锁外执行清理避免潜在死锁
    for (void *instance : instances_) {
        auto *store = static_cast<MmcacheStore *>(instance);
        if (store) {
            std::cout << "Cleaning up MmcacheStore instance" << std::endl;
            store->TearDown();
        }
    }
}
```

**声明位置**: 行 100-112

**功能描述**: 清理所有注册的资源

**注意**: 实际清理在锁外执行，避免死锁

---

### signalHandler() (第114-126行)

```cpp
void ResourceTracker::signalHandler(int signal)
{
    std::cout << "Received signal " << signal << ", cleaning up resources" << std::endl;
    getInstance().cleanupAllResources();

    // 用默认处理器重新触发信号以允许正常终止
    struct sigaction sa{};
    sa.sa_handler = SIG_DFL;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(signal, &sa, nullptr);
    raise(signal);
}
```

**声明位置**: 行 114-126

**功能描述**: 信号处理函数

**行为**:
1. 打印接收到的信号
2. 清理所有资源
3. 恢复默认信号处理
4. 重新触发信号

---

## MmcacheStore 实现

### 构造/析构函数 (第133-143行)

```cpp
MmcacheStore::MmcacheStore()
{
    // 将此实例注册到全局跟踪器
    ResourceTracker::getInstance().registerInstance(this);
}

MmcacheStore::~MmcacheStore()
{
    // 清理前从跟踪器反注册
    ResourceTracker::getInstance().unregisterInstance(this);
}
```

**声明位置**: 行 133-143

**功能描述**: 构造时自动注册，析构时自动反注册

---

### CreateObjectStore() (第147-150行)

```cpp
std::shared_ptr<ObjectStore> ObjectStore::CreateObjectStore()
{
    return std::make_shared<MmcacheStore>();
}
```

**声明位置**: 行 147-150

**功能描述**: 创建对象存储实例的工厂方法

---

### Init() (第152-156行)

```cpp
int MmcacheStore::Init(const uint32_t deviceId, const bool initBm)
{
    mmc_init_config config{deviceId, initBm};
    return mmc_init(&config);
}
```

**声明位置**: 行 152-156

**功能描述**: 初始化对象存储

**实现**: 直接调用 C API `mmc_init()`

---

### TearDown() (第158-162行)

```cpp
int MmcacheStore::TearDown()
{
    mmc_uninit();
    return 0;
}
```

**声明位置**: 行 158-162

**功能描述**: 反初始化对象存储

---

### GetInto() (第174-196行)

```cpp
int MmcacheStore::GetInto(const std::string &key, void *buffer, size_t size, const int32_t direct)
```

**声明位置**: 行 174-196

**功能描述**: 获取对象数据到预分配的缓冲区

**代码逻辑**:
1. **映射 direct 到介质类型** (第176-187行):
   - `SMEMB_COPY_G2L` (1) → `MEDIA_HBM`
   - `SMEMB_COPY_G2H` (2) → `MEDIA_DRAM`
   - 其他值返回错误

2. **构建 mmc_buffer** (第188行):
   ```cpp
   mmc_buffer mmcBuffer = {
       .addr = reinterpret_cast<uint64_t>(buffer),
       .type = type,
       .offset = 0,
       .len = size
   };
   ```

3. **调用 C API** (第189-191行):
   ```cpp
   TP_TRACE_BEGIN(TP_MMC_PY_GET);
   auto res = mmcc_get(key.c_str(), &mmcBuffer, 0);
   TP_TRACE_END(TP_MMC_PY_GET, res);
   ```

4. **错误处理** (第192-195行)

---

### PutFrom() (第203-227行)

```cpp
int MmcacheStore::PutFrom(const std::string &key, void *buffer, size_t size, const int32_t direct,
                          const ReplicateConfig &replicateConfig)
```

**声明位置**: 行 203-227

**功能描述**: 从缓冲区存储对象数据

**代码逻辑**:
1. **映射 direct 到介质类型** (第206-217行):
   - `SMEMB_COPY_L2G` (0) → `MEDIA_HBM`
   - `SMEMB_COPY_H2G` (3) → `MEDIA_DRAM`

2. **复制副本配置** (第220-221行):
   ```cpp
   mmc_put_options options{};
   MMC_ASSERT_RETURN(CopyPutOptions(replicateConfig, options), MMC_ERROR);
   ```

3. **调用 C API** (第222-225行)

---

### Remove() (第229-235行)

```cpp
int MmcacheStore::Remove(const std::string &key)
{
    TP_TRACE_BEGIN(TP_MMC_PY_REMOVE);
    auto ret = mmcc_remove(key.c_str(), 0);
    TP_TRACE_END(TP_MMC_PY_REMOVE, ret);
    return ret;
}
```

**声明位置**: 行 229-235

**功能描述**: 移除对象

---

### BatchRemove() (第237-266行)

```cpp
std::vector<int> MmcacheStore::BatchRemove(const std::vector<std::string> &keys)
```

**声明位置**: 行 237-266

**功能描述**: 批量移除多个对象

**代码逻辑**:
1. 验证键列表非空且不超过最大批量数
2. 分配 C 字符串数组
3. 调用 `mmcc_batch_remove()`
4. 转换结果为 vector
5. 释放临时数组

---

### IsExist() (第276-289行)

```cpp
int MmcacheStore::IsExist(const std::string &key)
{
    TP_TRACE_BEGIN(TP_MMC_PY_EXIST);
    int32_t res = mmcc_exist(key.c_str(), 0);
    TP_TRACE_END(TP_MMC_PY_EXIST, res);
    if (res == MMC_OK) {
        // 对齐 mooncake: 1 表示存在
        return 1;
    } else if (res == MMC_UNMATCHED_KEY) {
        // 对齐 mooncake: 0 表示不存在
        return 0;
    }
    return res;
}
```

**声明位置**: 行 276-289

**功能描述**: 检查对象是否存在

**返回值对齐**:
- 与 mooncake 接口对齐
- `1`: 存在
- `0`: 不存在
- `-1`: 错误

---

### GetKeyInfo() (第332-354行)

```cpp
KeyInfo MmcacheStore::GetKeyInfo(const std::string &key)
{
    mmc_data_info info;
    TP_TRACE_BEGIN(TP_MMC_PY_QUERY);
    auto res = mmcc_query(key.c_str(), &info, 0);
    TP_TRACE_END(TP_MMC_PY_QUERY, res);
    if (res != MMC_OK) {
        MMC_LOG_ERROR("Failed to query key " << key << ", error code: " << res);
        return {0, 0};
    }

    if (!info.valid) {
        MMC_LOG_ERROR("Failed to query key " << key << ", info invalid");
        return {0, 0};
    }

    KeyInfo keyInfo{info.size, info.numBlobs};
    for (int i = 0; i < info.numBlobs; i++) {
        keyInfo.AddLoc(info.ranks[i]);
        keyInfo.AddType(info.types[i]);
    }
    return keyInfo;
}
```

**声明位置**: 行 332-354

**功能描述**: 获取对象信息

**返回的 KeyInfo 包含**:
- 数据大小
- Blob 数量
- 每个 Blob 的 Rank 位置
- 每个 Blob 的介质类型

---

### BatchPutFrom() (第408-455行)

```cpp
std::vector<int> MmcacheStore::BatchPutFrom(const std::vector<std::string> &keys,
                                            const std::vector<void *> &buffers,
                                            const std::vector<size_t> &sizes,
                                            const int32_t direct,
                                            const ReplicateConfig &replicateConfig)
```

**声明位置**: 行 408-455

**功能描述**: 批量存储多个对象数据

**代码逻辑**:
1. 验证输入向量大小一致
2. 映射 direct 到介质类型
3. 构建 keyArray 和 bufferArray
4. 复制副本配置
5. 调用 `mmcc_batch_put()`
6. 包装每个结果（处理重复对象情况）

---

### PutFromLayers() (第505-558行)

```cpp
int MmcacheStore::PutFromLayers(const std::string &key,
                                const std::vector<void *> &buffers,
                                const std::vector<size_t> &sizes,
                                const int32_t direct,
                                const ReplicateConfig &replicateConfig)
```

**声明位置**: 行 505-558

**功能描述**: 存储分层对象数据（如神经网络模型层）

**代码逻辑**:
1. **验证 direct 参数** (第510-514行):
   - 支持: `SMEMB_COPY_L2G` (0), `SMEMB_COPY_H2G` (3), `SMEMB_COPY_AUTO` (9)

2. **确定介质类型** (第516-525行):
   - `L2G` → `MEDIA_HBM`
   - `H2G` → `MEDIA_DRAM`
   - `AUTO` → 根据地址自动检测

3. **验证参数** (第527-541行):
   - 验证 key 长度
   - 验证层数不超过 `MAX_LAYER_NUM`
   - 验证 buffers 和 sizes 数量匹配

4. **构建缓冲区数组** (第546-552行):
   ```cpp
   MmcBufferArray bufArr;
   for (size_t i = 0; i < layerNum; i += 1) {
       bufArr.AddBuffer({
           .addr = reinterpret_cast<uint64_t>(buffers[i]),
           .type = type,
           .offset = 0,
           .len = static_cast<uint64_t>(sizes[i])
       });
   }
   ```

5. **调用客户端** (第553-555行)

---

### GetIntoLayers() (第625-674行)

```cpp
int MmcacheStore::GetIntoLayers(const std::string &key,
                                const std::vector<void *> &buffers,
                                const std::vector<size_t> &sizes,
                                const int32_t direct)
```

**声明位置**: 行 625-674

**功能描述**: 获取分层对象数据

**代码逻辑**: 类似 `PutFromLayers`，但是获取方向

1. 验证 direct 参数
2. 确定介质类型（支持自动检测）
3. 验证输入参数
4. 构建 mmc_buffer 向量
5. 调用客户端的 `Get()` 方法

---

### IsInHybmDeviceRange() (第500-503行)

```cpp
bool MmcacheStore::IsInHybmDeviceRange(uint64_t va)
{
    return (va >= MMC_DEVICE_VA_START) && (va < (MMC_DEVICE_VA_START + MMC_DEVICE_VA_SIZE));
}
```

**声明位置**: 行 500-503

**功能描述**: 判断虚拟地址是否在 NPU 设备地址范围内

**地址范围**: 16T - 24T (0x100000000000 - 0x180000000000)

---

### ReturnWrapper() (第770-782行)

```cpp
int MmcacheStore::ReturnWrapper(const int result, const std::string &key)
{
    if (result != MMC_OK) {
        if (result == MMC_DUPLICATED_OBJECT) {
            MMC_LOG_DEBUG("Duplicated key " << key << ", put operation skipped");
            return MMC_OK;
        } else {
            MMC_LOG_ERROR("Failed to put key " << key << ", error code=" << result);
            return result;
        }
    }
    return MMC_OK;
}
```

**声明位置**: 行 770-782

**功能描述**: 包装返回值，将重复对象视为成功

---

### CheckInput() (第733-748行)

```cpp
int MmcacheStore::CheckInput(const size_t batchSize, const std::vector<std::vector<void *>> &buffers,
                             const std::vector<std::vector<size_t>> &sizes)
```

**声明位置**: 行 733-748

**功能描述**: 验证批量分层操作的输入参数

**验证内容**:
- 每个键的层数不超过 `MAX_LAYER_NUM`
- buffers 和 sizes 的数量匹配

---

### GetBufferArrays() (第750-768行)

```cpp
void MmcacheStore::GetBufferArrays(const size_t batchSize, const uint32_t type,
                                   const std::vector<std::vector<void *>> &bufferLists,
                                   const std::vector<std::vector<size_t>> &sizeLists,
                                   std::vector<MmcBufferArray> &bufferArrays)
```

**声明位置**: 行 750-768

**功能描述**: 构建批量操作的 MmcBufferArray 向量

---

### Python API 专用方法

#### Get() (第830-859行)

```cpp
mmc_buffer MmcacheStore::Get(const std::string &key)
```

**声明位置**: 行 830-859

**功能描述**: Python API 专用，自动分配内存并获取数据

**代码逻辑**:
1. 调用 `mmcc_query()` 查询数据大小
2. 分配相应大小的内存
3. 构建 mmc_buffer
4. 调用 `mmcc_get()` 获取数据
5. 失败时释放内存并返回空缓冲区

**注意**: 调用者负责释放返回的内存

#### GetBatch() (第861-903行)

```cpp
std::vector<mmc_buffer> MmcacheStore::GetBatch(const std::vector<std::string> &keys)
```

**声明位置**: 行 861-903

**功能描述**: 批量获取数据，自动分配内存

**代码逻辑**:
1. 调用 `BatchGetKeyInfo()` 获取所有键的大小信息
2. 为每个键分配内存
3. 调用 `mmcc_batch_get()` 批量获取
4. 返回缓冲区向量

---

## 性能追踪

代码中使用 `TP_TRACE_BEGIN` 和 `TP_TRACE_END` 宏进行性能追踪:

```cpp
TP_TRACE_BEGIN(TP_MMC_PY_PUT);
// ... 操作 ...
TP_TRACE_END(TP_MMC_PY_PUT, ret);
```

追踪点包括:
- `TP_MMC_PY_PUT`: Python Put 操作
- `TP_MMC_PY_GET`: Python Get 操作
- `TP_MMC_PY_BATCH_PUT`: 批量 Put
- `TP_MMC_PY_BATCH_GET`: 批量 Get
- `TP_MMC_PY_REMOVE`: 删除操作
- 等等

---

## direct 参数与介质类型映射

| direct 值 | 宏定义 | Put 操作 | Get 操作 |
|-----------|--------|----------|----------|
| 0 | SMEMB_COPY_L2G | MEDIA_HBM | - |
| 1 | SMEMB_COPY_G2L | - | MEDIA_HBM |
| 2 | SMEMB_COPY_G2H | - | MEDIA_DRAM |
| 3 | SMEMB_COPY_H2G | MEDIA_DRAM | - |
| 9 | SMEMB_COPY_AUTO | 自动检测 | 自动检测 |

---

## 文件级别的关系图

```
mmcache_store.cpp (C++ ObjectStore 实现)
    |
    +-- 依赖: mmcache_store.h (类定义)
    +-- 依赖: mmc_client.h (C API)
    +-- 依赖: mmc_client_default.h (C++ 客户端)
    +-- 依赖: mmc_ptracer.h (性能追踪)
    |
    +-- 调用: mmc_init(), mmc_uninit()
    +-- 调用: mmcc_put(), mmcc_get()
    +-- 调用: mmcc_batch_put(), mmcc_batch_get()
    +-- 调用: mmcc_remove(), mmcc_exist()
    +-- 调用: mmcc_query()
    +-- 调用: MmcClientDefault::GetInstance()
```
