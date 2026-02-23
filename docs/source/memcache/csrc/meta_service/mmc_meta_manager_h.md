# mmc_meta_manager.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_manager.h`
- **文件用途**: 定义元数据管理器类，负责元数据的 CRUD 操作和内存淘汰管理
- **依赖项**: `mmc_global_allocator.h`, `mmc_mem_obj_meta.h`, `mmc_meta_container.h`, `mmc_meta_backup_mgr.h`, `mmc_meta_net_server.h`, `mmc_thread_pool.h`

---

## 常量定义

```cpp
constexpr int METAMGR_POOL_BASE = 16;
```

**声明位置**: 行 30

**说明**: 元管理器线程池基础大小

---

## 数据结构

### MmcMemMetaDesc

```cpp
struct MmcMemMetaDesc {
    uint16_t prot_{0};
    uint8_t priority_{0};
    uint8_t numBlobs_{0};
    uint64_t size_{0};
    std::vector<MmcMemBlobDesc> blobs_;

    MmcMemMetaDesc() = default;
    MmcMemMetaDesc(const uint16_t &prot, const uint8_t &priority, const uint8_t &numBlobs, const uint64_t &size)
        : prot_(prot), priority_(priority), numBlobs_(numBlobs), size_(size)
    {}

    MmcMemMetaDesc(const uint16_t &prot, const uint8_t &priority, const uint8_t &numBlobs, const uint64_t &size,
                   const std::vector<MmcMemBlobPtr> &blobs)
        : prot_(prot), priority_(priority), numBlobs_(numBlobs), size_(size)
    {
        for (const auto &blob : blobs) {
            AddBlob(blob);
        }
    }

    void AddBlob(const MmcMemBlobPtr &blob)
    {
        blobs_.push_back(blob->GetDesc());
    }

    void AddBlobs(const std::vector<MmcMemBlobPtr> &blobs)
    {
        for (const auto &blob : blobs) {
            AddBlob(blob);
        }
    }

    uint16_t Prot() { return prot_; };
    uint8_t Priority() { return priority_; };
    uint8_t NumBlobs() { return numBlobs_; };
    uint64_t Size() { return size_; };
};
```

**声明位置**: 行 32-84

**说明**: 元数据描述结构
- `prot_`: 保护标志
- `priority_`: 优先级
- `numBlobs_`: Blob 数量
- `size_`: 大小
- `blobs_`: Blob 描述列表

---

## 类定义

### MmcMetaManager

元数据管理器类，负责管理所有元数据的生命周期。

---

#### 构造函数

```cpp
explicit MmcMetaManager(uint64_t defaultTtl, uint16_t evictThresholdHigh, uint16_t evictThresholdLow)
    : defaultTtlMs_(defaultTtl), evictThresholdHigh_(evictThresholdHigh), evictThresholdLow_(evictThresholdLow)
{}
```

**声明位置**: 行 88-90

**功能描述**: 构造元管理器

**参数**:
- `defaultTtl`: 默认 TTL（毫秒）
- `evictThresholdHigh`: 高淘汰阈值
- `evictThresholdLow`: 低淘汰阈值

---

#### 析构函数

```cpp
~MmcMetaManager() override
{
    Stop();
}
```

**声明位置**: 行 92-95

**功能描述**: 析构函数，自动停止管理器

---

### Start

```cpp
Result Start()
```

**声明位置**: 行 97-114

**功能描述**: 启动元管理器

**代码逻辑**:
1. 创建全局分配器
2. 创建元数据容器（LRU 实现）
3. 创建线程池
4. 启动线程池
5. 设置启动标志

---

### Stop

```cpp
void Stop()
```

**声明位置**: 行 116-125

**功能描述**: 停止元管理器

**代码逻辑**:
1. 销毁线程池
2. 清除启动标志

---

### Get

```cpp
Result Get(const std::string &key, uint64_t operateId, MmcBlobFilterPtr filterPtr, MmcMemMetaDesc &objMeta)
```

**声明位置**: 行 132

**功能描述**: 获取元数据对象并更新 LRU 位置

**参数**:
- `key`: 键
- `operateId`: 操作 ID
- `filterPtr`: Blob 过滤器
- `objMeta`: 输出元数据描述

**返回值**: 成功返回 MMC_OK

---

### Alloc

```cpp
Result Alloc(const std::string &key, const AllocOptions &allocOpt, uint64_t operateId, MmcMemMetaDesc &objMeta)
```

**声明位置**: 行 139

**功能描述**: 分配全局内存空间并创建元数据对象

**参数**:
- `key`: 键
- `allocOpt`: 分配选项
- `operateId`: 操作 ID
- `objMeta`: 输出元数据描述

**返回值**: 成功返回 MMC_OK，已存在返回 MMC_DUPLICATED_OBJECT

---

### UpdateState

```cpp
Result UpdateState(const std::string &key, const MmcLocation &loc, const BlobActionResult &actRet,
                   uint64_t operateId)
```

**声明位置**: 行 145-146

**功能描述**: 更新 Blob 状态

**参数**:
- `key`: 键
- `loc`: 位置
- `actRet`: 操作结果
- `operateId`: 操作 ID

**返回值**: 成功返回 MMC_OK

---

### Remove

```cpp
Result Remove(const std::string &key)
```

**声明位置**: 行 152

**功能描述**: 移除元数据对象

**参数**:
- `key`: 键

**返回值**: 成功返回 MMC_OK

---

### RemoveAll

```cpp
Result RemoveAll()
```

**声明位置**: 行 157

**功能描述**: 移除所有元数据对象

**返回值**: 成功返回 MMC_OK

---

### Mount (single)

```cpp
Result Mount(const MmcLocation &loc, const MmcLocalMemlInitInfo &localMemInitInfo,
             std::map<std::string, MmcMemBlobDesc> &blobMap)
```

**声明位置**: 行 165-166

**功能描述**: 挂载新的内存池贡献者

**参数**:
- `loc`: 位置
- `localMemInitInfo`: 本地内存初始化信息
- `blobMap`: Blob 映射表（用于重建）

**返回值**: 成功返回 MMC_OK

---

### Mount (vector)

```cpp
Result Mount(const std::vector<MmcLocation> &locs, const std::vector<MmcLocalMemlInitInfo> &localMemInitInfos,
             std::map<std::string, MmcMemBlobDesc> &blobMap)
```

**声明位置**: 行 168-169

**功能描述**: 批量挂载内存池贡献者

**参数**:
- `locs`: 位置列表
- `localMemInitInfos`: 本地内存初始化信息列表
- `blobMap`: Blob 映射表

**返回值**: 成功返回 MMC_OK，部分失败会自动回滚

---

### Unmount

```cpp
Result Unmount(const MmcLocation &loc)
```

**声明位置**: 行 174

**功能描述**: 卸载指定位置的内存池

**参数**:
- `loc`: 位置

**返回值**: 成功返回 MMC_OK

---

### GetAllSegmentInfo

```cpp
nlohmann::json GetAllSegmentInfo() const
```

**声明位置**: 行 179

**功能描述**: 获取所有分段信息

**返回值**: JSON 格式的分段信息

---

### ExistKey

```cpp
Result ExistKey(const std::string &key)
```

**声明位置**: 行 185

**功能描述**: 检查键是否存在

**参数**:
- `key`: 键

**返回值**: 存在返回 MMC_OK，不存在返回 MMC_UNMATCHED_KEY

---

### Query

```cpp
Result Query(const std::string &key, MemObjQueryInfo &queryInfo)
```

**声明位置**: 行 192

**功能描述**: 查询键的信息

**参数**:
- `key`: 键
- `queryInfo`: 输出查询信息

**返回值**: 成功返回 MMC_OK

---

### GetAllKeys

```cpp
Result GetAllKeys(std::vector<std::string> &keys)
```

**声明位置**: 行 198

**功能描述**: 获取所有键

**参数**:
- `keys`: 输出键列表

**返回值**: 成功返回 MMC_OK

---

### CheckAndEvict

```cpp
void CheckAndEvict()
```

**声明位置**: 行 203

**功能描述**: 检查并执行内存淘汰

---

### Ttl

```cpp
inline uint64_t Ttl()
{
    return defaultTtlMs_;
}
```

**声明位置**: 行 205-208

**功能描述**: 获取默认 TTL

---

### ReplicateBlob

```cpp
Result ReplicateBlob(const std::string &key, const MmcLocation &loc)
```

**声明位置**: 行 213

**功能描述**: 复制 Blob 到指定位置

**参数**:
- `key`: 键
- `loc`: 目标位置

**返回值**: 成功返回 MMC_OK

---

### MoveBlob

```cpp
Result MoveBlob(const std::string &key, const MmcLocation &src, const MmcLocation &dst)
```

**声明位置**: 行 218

**功能描述**: 从源位置移动 Blob 到目标位置

**参数**:
- `key`: 键
- `src`: 源位置
- `dst`: 目标位置

**返回值**: 成功返回 MMC_OK

---

### SetMetaNetServer

```cpp
void SetMetaNetServer(MetaNetServerPtr metaNetServer)
{
    metaNetServer_ = metaNetServer;
}
```

**声明位置**: 行 221-224

**功能描述**: 设置元网络服务器（临时方案）

---

## 私有方法

### CopyBlob

```cpp
Result CopyBlob(const MmcMemObjMetaPtr &objMeta, const MmcMemBlobDesc &srcBlob, const MmcLocation &dstLoc)
```

**声明位置**: 行 227

**功能描述**: 复制 Blob 到目标位置

---

### RebuildMeta

```cpp
Result RebuildMeta(std::map<std::string, MmcMemBlobDesc> &blobMap)
```

**声明位置**: 行 229

**功能描述**: 从 Blob 映射表重建元数据

---

### PushRemoveList

```cpp
void PushRemoveList(const std::string &key, const MmcMemObjMetaPtr &meta)
```

**声明位置**: 行 231

**功能描述**: 推送到移除列表（异步释放）

---

### EvictCallBackFunction

```cpp
EvictResult EvictCallBackFunction(const std::string &key, const MmcMemObjMetaPtr &objMeta)
```

**声明位置**: 行 233

**功能描述**: 淘汰回调函数

---

## 成员变量

```cpp
private:
    std::mutex mutex_;
    bool started_ = false;
    std::atomic<bool> evictCheck_{false};

    MmcRef<MmcMetaContainer<std::string, MmcMemObjMetaPtr>> metaContainer_;
    MmcGlobalAllocatorPtr globalAllocator_;

    uint64_t defaultTtlMs_;
    uint16_t evictThresholdHigh_;
    uint16_t evictThresholdLow_;
    MetaNetServerPtr metaNetServer_;
    MmcThreadPoolPtr threadPool_;
```

**说明**:
- `mutex_`: 互斥锁
- `started_`: 启动标志
- `evictCheck_`: 淘汰检查标志
- `metaContainer_`: 元数据容器
- `globalAllocator_`: 全局分配器
- `defaultTtlMs_`: 默认 TTL
- `evictThresholdHigh_`: 高淘汰阈值
- `evictThresholdLow_`: 低淘汰阈值
- `metaNetServer_`: 元网络服务器
- `threadPool_`: 线程池

---

## 类型别名

```cpp
using MmcMetaManagerPtr = MmcRef<MmcMetaManager>;
```

**声明位置**: 行 249

**说明**: 元管理器智能指针类型

---

## 文件级别的关系图

```
mmc_meta_manager.h (元数据管理器)
    |
    +-- 使用 MmcGlobalAllocator (全局分配器)
    |
    +-- 使用 MmcMetaContainer (元数据容器)
    |
    +-- 使用 MmcThreadPool (线程池)
    |
    +-- 被 MmcMetaMgrProxy 使用
    |
    +-- 功能:
    |   +-- 元数据 CRUD
    |   +-- 内存分配/释放
    |   +-- LRU 淘汰
    |   +-- 多层级淘汰
    |   +-- Blob 复制/移动
```
