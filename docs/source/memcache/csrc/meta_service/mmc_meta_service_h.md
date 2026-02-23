# mmc_meta_service.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_service.h`
- **文件用途**: 定义元数据服务类，元服务的顶层入口
- **依赖项**: `mmc_def.h`, `mmc_global_allocator.h`, `mmc_meta_net_server.h`, `mmc_meta_mgr_proxy.h`, `smem_config_store.h`

---

## 类定义

### MmcMetaService

元数据服务类，元服务的顶层入口，负责初始化和管理所有子组件。

---

#### 构造函数

```cpp
explicit MmcMetaService(const std::string &name) : name_(name), options_{} {}
```

**声明位置**: 行 25

**功能描述**: 构造元服务

**参数**:
- `name`: 服务名称

---

### Start

```cpp
Result Start(const mmc_meta_service_config_t &options);
```

**声明位置**: 行 27

**功能描述**: 启动元服务

**参数**:
- `options`: 服务配置选项

**返回值**: 成功返回 MMC_OK

---

### Stop

```cpp
void Stop();
```

**声明位置**: 行 29

**功能描述**: 停止元服务

---

### BmRegister

```cpp
Result BmRegister(uint32_t rank, std::vector<uint16_t> mediaType, std::vector<uint64_t> bm,
                  std::vector<uint64_t> capacity, std::map<std::string, MmcMemBlobDesc> &blobMap);
```

**声明位置**: 行 31-32

**功能描述**: 注册 Blob Manager

**参数**:
- `rank`: Rank ID
- `mediaType`: 介质类型列表
- `bm`: Blob Manager 地址列表
- `capacity`: 容量列表
- `blobMap`: Blob 映射表（用于重建）

**返回值**: 成功返回 MMC_OK

---

### BmUnregister

```cpp
Result BmUnregister(uint32_t rank, uint16_t mediaType);
```

**声明位置**: 行 34

**功能描述**: 注销 Blob Manager

**参数**:
- `rank`: Rank ID
- `mediaType`: 介质类型

**返回值**: 成功返回 MMC_OK

---

### ClearResource

```cpp
Result ClearResource(uint32_t rank);
```

**声明位置**: 行 36

**功能描述**: 清理指定 Rank 的资源

**参数**:
- `rank`: Rank ID

**返回值**: 成功返回 MMC_OK

---

### Name

```cpp
const std::string &Name() const;
```

**声明位置**: 行 38

**功能描述**: 获取服务名称

---

### Options

```cpp
const mmc_meta_service_config_t &Options() const;
```

**声明位置**: 行 40

**功能描述**: 获取服务配置

---

### GetMetaMgrProxy

```cpp
const MmcMetaMgrProxyPtr &GetMetaMgrProxy() const;
```

**声明位置**: 行 42

**功能描述**: 获取元管理器代理

---

## 内联函数实现

### Name

```cpp
inline const std::string &MmcMetaService::Name() const
{
    return name_;
}
```

**声明位置**: 行 56-59

**功能描述**: 获取服务名称

---

### Options

```cpp
inline const mmc_meta_service_config_t &MmcMetaService::Options() const
{
    return options_;
}
```

**声明位置**: 行 61-64

**功能描述**: 获取服务配置

---

### GetMetaMgrProxy

```cpp
inline const MmcMetaMgrProxyPtr &MmcMetaService::GetMetaMgrProxy() const
{
    return metaMgrProxy_;
}
```

**声明位置**: 行 66-69

**功能描述**: 获取元管理器代理

---

## 成员变量

```cpp
private:
    MetaNetServerPtr metaNetServer_;
    MmcMetaMgrProxyPtr metaMgrProxy_;
    MMCMetaBackUpMgrPtr metaBackUpMgrPtr_;

    std::mutex mutex_;
    bool started_ = false;
    std::string name_;
    mmc_meta_service_config_t options_;
    std::unordered_map<uint32_t, std::unordered_set<uint16_t>> rankMediaTypeMap_;
    ock::smem::StorePtr confStore_ = nullptr;
```

**说明**:
- `metaNetServer_`: 网络服务器
- `metaMgrProxy_`: 元管理器代理
- `metaBackUpMgrPtr_`: 备份管理器
- `mutex_`: 互斥锁
- `started_`: 启动标志
- `name_`: 服务名称
- `options_`: 服务配置
- `rankMediaTypeMap_`: Rank 到介质类型的映射
- `confStore_`: 配置存储

---

## 类型别名

```cpp
using MmcMetaServiceDefaultPtr = MmcRef<MmcMetaService>;
```

**声明位置**: 行 71

**说明**: 元服务智能指针类型

---

## 文件级别的关系图

```
mmc_meta_service.h (元服务)
    |
    +-- 包含 MetaNetServer (网络服务器)
    |
    +-- 包含 MmcMetaMgrProxy (元管理器代理)
    |
    +-- 包含 MMCMetaBackUpMgr (备份管理器)
    |
    +-- 被 MmcMetaServiceProcess 使用
    |
    +-- 功能:
    |   +-- 服务启动/停止
    |   +-- BM 注册/注销
    |   +-- 资源清理
    |   +-- 配置管理
```

---

## 服务配置 (mmc_meta_service_config_t)

配置结构包含以下字段（具体定义在 mmc_def.h 中）:
- discoveryURL: 服务发现 URL
- httpURL: HTTP 服务 URL
- configStoreURL: 配置存储 URL
- evictThresholdHigh: 高淘汰阈值
- evictThresholdLow: 低淘汰阈值
- accTlsConfig: 接入层 TLS 配置
- configStoreTlsConfig: 配置存储 TLS 配置
- haEnable: 高可用开关
- logLevel: 日志级别
- logPath: 日志路径
- logRotationFileSize: 日志轮转文件大小
- logRotationFileCount: 日志轮转文件数量
