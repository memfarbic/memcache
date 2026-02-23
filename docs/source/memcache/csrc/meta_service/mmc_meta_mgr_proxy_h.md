# mmc_meta_mgr_proxy.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_mgr_proxy.h`
- **文件用途**: 定义元数据管理器代理类，处理网络请求并调用底层元管理器
- **依赖项**: `mmc_meta_manager.h`, `mmc_msg_client_meta.h`, `mmc_meta_net_server.h`

---

## 类定义

### MmcMetaMgrProxy

元数据管理器代理类，位于网络层和元管理器之间，负责请求处理和参数转换。

---

#### 构造函数

```cpp
explicit MmcMetaMgrProxy(const MetaNetServerPtr &netServerPtr) : netServerPtr_(netServerPtr) {}
```

**声明位置**: 行 26

**功能描述**: 构造代理对象

**参数**:
- `netServerPtr`: 网络服务器指针

---

#### 析构函数

```cpp
~MmcMetaMgrProxy() override = default;
```

**声明位置**: 行 28

**功能描述**: 默认析构函数

---

### Start

```cpp
Result Start(uint64_t defaultTtl, uint16_t evictThresholdHigh, uint16_t evictThresholdLow)
```

**声明位置**: 行 30-46

**功能描述**: 启动代理并初始化元管理器

**参数**:
- `defaultTtl`: 默认 TTL
- `evictThresholdHigh`: 高淘汰阈值
- `evictThresholdLow`: 低淘汰阈值

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 获取互斥锁
2. 如果已启动则直接返回
3. 创建元管理器
4. 设置网络服务器
5. 启动元管理器
6. 设置启动标志

---

### Stop

```cpp
void Stop()
```

**声明位置**: 行 48-57

**功能描述**: 停止代理

**代码逻辑**:
1. 获取互斥锁
2. 如果未启动则直接返回
3. 停止元管理器
4. 清除启动标志

---

### Alloc

```cpp
Result Alloc(const AllocRequest &req, AllocResponse &resp);
```

**声明位置**: 行 59

**功能描述**: 处理分配请求

**参数**:
- `req`: 分配请求
- `resp`: 分配响应

**返回值**: 成功返回 MMC_OK

---

### BatchAlloc

```cpp
Result BatchAlloc(const BatchAllocRequest &req, BatchAllocResponse &resp);
```

**声明位置**: 行 61

**功能描述**: 处理批量分配请求

**参数**:
- `req`: 批量分配请求
- `resp`: 批量分配响应

**返回值**: 成功返回 MMC_OK

---

### UpdateState

```cpp
Result UpdateState(const UpdateRequest &req, Response &resp);
```

**声明位置**: 行 63

**功能描述**: 处理状态更新请求

**参数**:
- `req`: 更新请求
- `resp`: 响应

**返回值**: 成功返回 MMC_OK

---

### BatchUpdateState

```cpp
Result BatchUpdateState(const BatchUpdateRequest &req, BatchUpdateResponse &resp);
```

**声明位置**: 行 65

**功能描述**: 处理批量状态更新请求

**参数**:
- `req`: 批量更新请求
- `resp`: 批量更新响应

**返回值**: 成功返回 MMC_OK

---

### Get

```cpp
Result Get(const GetRequest &req, AllocResponse &resp);
```

**声明位置**: 行 67

**功能描述**: 处理获取请求

**参数**:
- `req`: 获取请求
- `resp`: 分配响应

**返回值**: 成功返回 MMC_OK

---

### BatchGet

```cpp
Result BatchGet(const BatchGetRequest &req, BatchAllocResponse &resp);
```

**声明位置**: 行 69

**功能描述**: 处理批量获取请求

**参数**:
- `req`: 批量获取请求
- `resp`: 批量分配响应

**返回值**: 成功返回 MMC_OK

---

### Remove

```cpp
Result Remove(const RemoveRequest &req, Response &resp)
{
    return resp.ret_ = metaMangerPtr_->Remove(req.key_);
}
```

**声明位置**: 行 71-74

**功能描述**: 处理删除请求

**代码逻辑**: 直接调用元管理器的 Remove 方法

---

### BatchRemove

```cpp
Result BatchRemove(const BatchRemoveRequest &req, BatchRemoveResponse &resp)
{
    resp.results_.reserve(req.keys_.size());
    for (const std::string &key : req.keys_) {
        resp.results_.emplace_back(metaMangerPtr_->Remove(key));
    }
    return MMC_OK;
}
```

**声明位置**: 行 76-83

**功能描述**: 处理批量删除请求

**代码逻辑**: 遍历所有键并删除

---

### RemoveAll

```cpp
Result RemoveAll(const RemoveAllRequest &req, Response &resp)
{
    return resp.ret_ = metaMangerPtr_->RemoveAll();
}
```

**声明位置**: 行 85-88

**功能描述**: 处理删除所有请求

---

### Mount

```cpp
Result Mount(const std::vector<MmcLocation> &loc, const std::vector<MmcLocalMemlInitInfo> &localMemInitInfo,
             std::map<std::string, MmcMemBlobDesc> &blobMap)
{
    return metaMangerPtr_->Mount(loc, localMemInitInfo, blobMap);
}
```

**声明位置**: 行 90-94

**功能描述**: 挂载内存池

---

### Unmount

```cpp
Result Unmount(const MmcLocation &loc)
{
    return metaMangerPtr_->Unmount(loc);
}
```

**声明位置**: 行 96-99

**功能描述**: 卸载内存池

---

### ExistKey

```cpp
Result ExistKey(const IsExistRequest &req, IsExistResponse &resp)
{
    return resp.ret_ = metaMangerPtr_->ExistKey(req.key_);
}
```

**声明位置**: 行 101-104

**功能描述**: 检查键是否存在

---

### BatchExistKey

```cpp
Result BatchExistKey(const BatchIsExistRequest &req, BatchIsExistResponse &resp);
```

**声明位置**: 行 106

**功能描述**: 批量检查键是否存在

---

### Query

```cpp
Result Query(const QueryRequest &req, QueryResponse &resp)
{
    return metaMangerPtr_->Query(req.key_, resp.queryInfo_);
}
```

**声明位置**: 行 108-111

**功能描述**: 查询键信息

---

### BatchQuery

```cpp
Result BatchQuery(const BatchQueryRequest &req, BatchQueryResponse &resp)
{
    for (const std::string &key : req.keys_) {
        MemObjQueryInfo queryInfo;
        metaMangerPtr_->Query(key, queryInfo);
        resp.batchQueryInfos_.push_back(queryInfo);
    }
    return MMC_OK;
}
```

**声明位置**: 行 113-121

**功能描述**: 批量查询键信息

---

### GetMetaManager

```cpp
const MmcMetaManagerPtr &GetMetaManager()
{
    return metaMangerPtr_;
}
```

**声明位置**: 行 123-126

**功能描述**: 获取元管理器指针

---

## 成员变量

```cpp
private:
    std::mutex mutex_;
    bool started_ = false;
    MmcMetaManagerPtr metaMangerPtr_;
    MetaNetServerPtr netServerPtr_;
    const int32_t timeOut_ = 60;
```

**说明**:
- `mutex_`: 互斥锁
- `started_`: 启动标志
- `metaMangerPtr_`: 元管理器指针
- `netServerPtr_`: 网络服务器指针
- `timeOut_`: 超时时间（秒）

---

## 类型别名

```cpp
using MmcMetaMgrProxyPtr = MmcRef<MmcMetaMgrProxy>;
```

**声明位置**: 行 135

**说明**: 代理类智能指针类型

---

## 文件级别的关系图

```
mmc_meta_mgr_proxy.h (元管理器代理)
    |
    +-- 使用 MmcMetaManager (元管理器)
    |
    +-- 使用 MetaNetServer (网络服务器)
    |
    +-- 被 MetaNetServer 调用
    |
    +-- 功能:
    |   +-- 请求/响应转换
    |   +-- 批量操作支持
    |   +-- 参数验证
    |   +-- 淘汰检查触发
```
