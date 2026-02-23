# mmcache_store.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/mmcache_store.h`
- **文件用途**: 定义 C++ ObjectStore 接口的实现类和资源跟踪器
- **依赖项**: `mmc_def.h`, `mmc_types.h`, `mmcache.h`, `<mutex>`, `<unordered_set>`, `<csignal>`

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 类定义
}
}
```

**声明位置**: 行 25-26

---

## ResourceTracker 类

### 类概述

全局资源跟踪器，用于处理异常终止时的清理工作。

**声明位置**: 行 30-60

### getInstance() (第32-33行)

```cpp
static ResourceTracker &getInstance();
```

**功能**: 获取 ResourceTracker 单例实例

**实现**: 使用 Meyer's Singleton 模式

### registerInstance() (第39-40行)

```cpp
void registerInstance(MmcacheStore *instance);
```

**功能**: 注册 MmcacheStore 实例用于清理

**参数**:
- `instance`: 要注册的实例指针

### unregisterInstance() (第42-43行)

```cpp
void unregisterInstance(MmcacheStore *instance);
```

**功能**: 反注册 MmcacheStore 实例

**参数**:
- `instance`: 要反注册的实例指针

### signalHandler() (第52-53行)

```cpp
static void signalHandler(int signal);
```

**功能**: 信号处理函数，处理 SIGINT、SIGTERM、SIGHUP

**行为**:
1. 清理所有注册的资源
2. 重新触发信号使用默认处理

### exitHandler() (第55-56行)

```cpp
static void exitHandler();
```

**功能**: 进程退出处理函数

**行为**: 清理所有注册的资源

### 成员变量 (第58-59行)

```cpp
std::mutex mutex_;
std::unordered_set<MmcacheStore *> instances_;
```

- `mutex_`: 保护实例集合的互斥锁
- `instances_`: 所有注册的 MmcacheStore 实例集合

---

## MmcacheStore 类

### 类概述

ObjectStore 接口的实现类，提供具体的数据存储功能。

**声明位置**: 行 62-143

**继承**: `public ock::mmc::ObjectStore`

### 构造函数 (第64行)

```cpp
MmcacheStore();
```

**功能**: 构造 MmcacheStore 对象

**行为**: 自动注册到 ResourceTracker

---

### 析构函数 (第65行)

```cpp
~MmcacheStore() override;
```

**功能**: 析构 MmcacheStore 对象

**行为**: 从 ResourceTracker 反注册

---

### Init() (第67行)

```cpp
int Init(const uint32_t deviceId, const bool initBm = true) override;
```

**功能**: 初始化对象存储

**参数**:
- `deviceId`: 设备 ID
- `initBm`: 是否初始化 Blob Manager，默认为 true

**返回值**: 成功返回 0，失败返回其他值

**实现**: 调用 `mmc_init()` 进行初始化

---

### TearDown() (第69行)

```cpp
int TearDown() override;
```

**功能**: 反初始化对象存储

**返回值**: 成功返回 0

**实现**: 调用 `mmc_uninit()` 进行清理

---

### RegisterBuffer() (第71行)

```cpp
int RegisterBuffer(void *buffer, size_t size) override;
```

**功能**: 为零拷贝操作注册缓冲区

**参数**:
- `buffer`: 预分配的缓冲区指针
- `size`: 缓冲区大小（字节）

**返回值**: 成功返回 0

**实现**: 调用 `mmcc_register_buffer()`

---

### UnRegisterBuffer() (第73行)

```cpp
int UnRegisterBuffer(void *buffer, size_t size) override;
```

**功能**: 注销缓冲区

**参数**:
- `buffer`: 缓冲区指针
- `size`: 缓冲区大小

**返回值**: 成功返回 0

**实现**: 调用 `mmcc_unregister_buffer()`

---

### GetInto() (第75行)

```cpp
int GetInto(const std::string &key, void *buffer, size_t size, const int32_t direct = 2) override;
```

**功能**: 获取对象数据到预分配的缓冲区

**参数**:
- `key`: 对象的键
- `buffer`: 预分配的缓冲区指针
- `size`: 缓冲区大小
- `direct`: 数据位置指示器 (2=G2H, 1=G2L)

**返回值**: 成功返回 0，失败返回错误码

**实现逻辑**:
1. 根据 direct 映射到介质类型
2. 构建 `mmc_buffer` 结构
3. 调用 `mmcc_get()` 获取数据

---

### BatchGetInto() (第77-78行)

```cpp
std::vector<int> BatchGetInto(const std::vector<std::string> &keys,
                              const std::vector<void *> &buffers,
                              const std::vector<size_t> &sizes,
                              const int32_t direct = 2) override;
```

**功能**: 批量获取多个对象数据

**参数**:
- `keys`: 键向量
- `buffers`: 缓冲区指针向量
- `sizes`: 缓冲区大小向量
- `direct`: 数据位置指示器

**返回值**: 每个操作的结果向量

---

### GetIntoLayers() (第80-81行)

```cpp
int GetIntoLayers(const std::string &key,
                  const std::vector<void *> &buffers,
                  const std::vector<size_t> &sizes,
                  const int32_t direct = 9) override;
```

**功能**: 获取分层对象数据（如模型层）

**参数**:
- `key`: 对象的键
- `buffers`: 每层的缓冲区指针
- `sizes`: 每层的缓冲区大小
- `direct`: 数据位置指示器

**返回值**: 成功返回 0

**实现**: 直接调用 `MmcClientDefault::GetInstance()->Get()`

---

### BatchGetIntoLayers() (第83-86行)

```cpp
std::vector<int> BatchGetIntoLayers(const std::vector<std::string> &keys,
                                    const std::vector<std::vector<void *>> &buffers,
                                    const std::vector<std::vector<size_t>> &sizes,
                                    const int32_t direct = 2) override;
```

**功能**: 批量获取多个分层对象数据

**参数**:
- `keys`: 键向量
- `buffers`: 二维缓冲区指针向量
- `sizes`: 二维缓冲区大小向量
- `direct`: 数据位置指示器

**返回值**: 每个操作的结果向量

---

### PutFrom() (第88-89行)

```cpp
int PutFrom(const std::string &key, void *buffer, size_t size,
            const int32_t direct = 3,
            const ReplicateConfig &replicateConfig = {}) override;
```

**功能**: 从缓冲区存储对象数据

**参数**:
- `key`: 对象的键
- `buffer`: 包含数据的缓冲区指针
- `size`: 缓冲区大小
- `direct`: 数据位置指示器 (3=H2G, 0=L2G)
- `replicateConfig`: 副本配置

**返回值**: 成功返回 0

**实现逻辑**:
1. 根据 direct 映射到介质类型
2. 复制副本配置到 mmc_put_options
3. 调用 `mmcc_put()` 存储数据

---

### GetLocalServiceId() (第91行)

```cpp
int GetLocalServiceId(uint32_t &localServiceId) override;
```

**功能**: 获取当前服务实例 ID

**参数**:
- `localServiceId`: [out] 输出本地服务 ID

**返回值**: 成功返回 0

**实现**: 调用 `mmcc_local_service_id()`

---

### BatchPutFrom() (第93-95行)

```cpp
std::vector<int> BatchPutFrom(const std::vector<std::string> &keys,
                              const std::vector<void *> &buffers,
                              const std::vector<size_t> &sizes,
                              const int32_t direct = 3,
                              const ReplicateConfig &replicateConfig = {}) override;
```

**功能**: 批量存储多个对象数据

**参数**:
- `keys`: 键向量
- `buffers`: 缓冲区指针向量
- `sizes`: 缓冲区大小向量
- `direct`: 数据位置指示器
- `replicateConfig`: 副本配置

**返回值**: 每个操作的结果向量

---

### PutFromLayers() (第97-98行)

```cpp
int PutFromLayers(const std::string &key,
                  const std::vector<void *> &buffers,
                  const std::vector<size_t> &sizes,
                  const int32_t direct = 9,
                  const ReplicateConfig &replicateConfig = {}) override;
```

**功能**: 存储分层对象数据

**参数**:
- `key`: 对象的键
- `buffers`: 每层的缓冲区指针
- `sizes`: 每层的缓冲区大小
- `direct`: 数据位置指示器
- `replicateConfig`: 副本配置

**返回值**: 成功返回 0

---

### BatchPutFromLayers() (第100-103行)

```cpp
std::vector<int> BatchPutFromLayers(const std::vector<std::string> &keys,
                                    const std::vector<std::vector<void *>> &buffers,
                                    const std::vector<std::vector<size_t>> &sizes,
                                    const int32_t direct = 3,
                                    const ReplicateConfig &replicateConfig = {}) override;
```

**功能**: 批量存储多个分层对象数据

---

### Remove() (第105行)

```cpp
int Remove(const std::string &key) override;
```

**功能**: 移除对象

**参数**:
- `key`: 要移除的对象的键

**返回值**: 成功返回 0

**实现**: 调用 `mmcc_remove()`

---

### BatchRemove() (第107行)

```cpp
std::vector<int> BatchRemove(const std::vector<std::string> &keys) override;
```

**功能**: 批量移除多个对象

**参数**:
- `keys`: 要移除的键列表

**返回值**: 每个操作的结果向量

---

### RemoveAll() (第109行)

```cpp
int RemoveAll() override;
```

**功能**: 移除所有对象

**返回值**: 成功返回 0

**实现**: 调用 `MmcClientDefault::GetInstance()->RemoveAll()`

---

### IsExist() (第111行)

```cpp
int IsExist(const std::string &key) override;
```

**功能**: 检查对象是否存在

**参数**:
- `key`: 要检查的键

**返回值**:
- `1`: 存在
- `0`: 不存在
- `-1`: 错误

**实现**: 调用 `mmcc_exist()` 并转换返回值

---

### BatchIsExist() (第113行)

```cpp
std::vector<int> BatchIsExist(const std::vector<std::string> &keys) override;
```

**功能**: 批量检查多个对象是否存在

---

### GetKeyInfo() (第115行)

```cpp
KeyInfo GetKeyInfo(const std::string &key) override;
```

**功能**: 获取对象信息

**参数**:
- `key`: 要查询的键

**返回值**: KeyInfo 对象，包含大小、副本位置等信息

**实现**: 调用 `mmcc_query()`

---

### BatchGetKeyInfo() (第117行)

```cpp
std::vector<KeyInfo> BatchGetKeyInfo(const std::vector<std::string> &keys) override;
```

**功能**: 批量获取多个对象信息

---

### Python API 专用方法

#### Get() (第120行)

```cpp
mmc_buffer Get(const std::string &key);
```

**功能**: Python API 专用，自动分配内存并获取数据

**返回值**: mmc_buffer，包含自动分配的内存地址

**注意**: 调用者负责释放返回的内存

#### GetBatch() (第122行)

```cpp
std::vector<mmc_buffer> GetBatch(const std::vector<std::string> &key);
```

**功能**: 批量获取数据，自动分配内存

#### Put() (第124行)

```cpp
int Put(const std::string &key, mmc_buffer &buffer, const ReplicateConfig &replicateConfig = {});
```

**功能**: Python API 专用的 Put 方法

#### PutBatch() (第126-127行)

```cpp
int PutBatch(const std::vector<std::string> &keys, std::vector<mmc_buffer> &buffers,
             const ReplicateConfig &replicateConfig);
```

**功能**: 批量 Put 方法

---

### 私有方法

#### CheckInput() (第130-131行)

```cpp
int CheckInput(size_t batchSize, const std::vector<std::vector<void *>> &buffers,
               const std::vector<std::vector<size_t>> &sizes);
```

**功能**: 验证批量操作的输入参数

#### GetBufferArrays() (第133-135行)

```cpp
void GetBufferArrays(size_t batchSize, uint32_t type,
                     const std::vector<std::vector<void *>> &bufferLists,
                     const std::vector<std::vector<size_t>> &sizeLists,
                     std::vector<ock::mmc::MmcBufferArray> &bufferArrays);
```

**功能**: 构建批量操作的缓冲区数组

#### IsInHybmDeviceRange() (第137行)

```cpp
bool IsInHybmDeviceRange(uint64_t va);
```

**功能**: 判断虚拟地址是否在设备地址范围内

#### ReturnWrapper() (第139行)

```cpp
static int ReturnWrapper(const int result, const std::string &key);
```

**功能**: 包装返回值，处理重复对象情况

---

## direct 参数说明

`direct` 参数指示数据的位置和传输方向：

| 值 | 宏定义 | 含义 |
|---|--------|------|
| 0 | SMEMB_COPY_L2G | 本地到全局 (HBM) |
| 1 | SMEMB_COPY_G2L | 全局到本地 (HBM) |
| 2 | SMEMB_COPY_G2H | 全局到主机 (DRAM) |
| 3 | SMEMB_COPY_H2G | 主机到全局 (DRAM) |
| 9 | SMEMB_COPY_AUTO | 自动检测 |

---

## 类关系图

```
ObjectStore (抽象接口)
    ↑
    | 继承
    |
MmcacheStore (实现类)
    |
    +-- 使用 ResourceTracker (资源管理)
    +-- 调用 mmc.h API (C 接口)
    +-- 调用 MmcClientDefault (C++ 实现)
```

---

## 内存管理

### 构造/析构时的资源管理

```
构造 MmcacheStore
    |
    v
注册到 ResourceTracker
    |
    v
... 使用对象 ...
    |
    v
析构 MmcacheStore
    |
    v
从 ResourceTracker 反注册
```

### 异常终止时的清理

```
收到信号 (SIGINT/SIGTERM/SIGHUP)
    |
    v
signalHandler() 被调用
    |
    v
cleanupAllResources()
    |
    v
调用所有注册实例的 TearDown()
    |
    v
重新触发信号 (默认处理)
```
