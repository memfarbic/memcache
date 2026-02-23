# 核心实现文件文档

## 概述

本文档描述 MemCache_Hybrid 的核心 C++ 实现文件，这些文件提供公共 API 的具体实现。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/`

---

## 详细文档链接

### mmc.cpp
**功能**: 实现 MemCache 的主初始化和配置接口 (`mmc.h`)
- [mmc.cpp 详细文档](mmc_cpp.md)

**主要函数**:
- `mmc_init()` - 初始化 memcache 客户端和本地服务
- `mmc_set_extern_logger()` - 设置外部日志函数
- `mmc_set_log_level()` - 设置日志级别
- `mmc_uninit()` - 反初始化

---

### mmc_client.cpp
**功能**: 实现客户端 API (`mmc_client.h`)
- [mmc_client.cpp 详细文档](mmc_client_cpp.md)

**主要函数**:
- `mmcc_init()` / `mmcc_uninit()` - 客户端初始化/反初始化
- `mmcc_register_buffer()` / `mmcc_unregister_buffer()` - 缓冲区注册
- `mmcc_put()` / `mmcc_get()` - 单个对象存取
- `mmcc_batch_put()` / `mmcc_batch_get()` - 批量对象存取
- `mmcc_query()` / `mmcc_batch_query()` - 查询数据信息
- `mmcc_remove()` / `mmcc_batch_remove()` - 删除对象
- `mmcc_exist()` / `mmcc_batch_exist()` - 检查对象存在

---

### mmc_service.cpp
**功能**: 实现服务端 API (`mmc_service.h`)
- [mmc_service.cpp 详细文档](mmc_service_cpp.md)

**主要函数**:
- `mmcs_meta_service_start()` / `mmcs_meta_service_stop()` - 元数据服务启停
- `mmcs_local_service_start()` / `mmcs_local_service_stop()` - 本地服务启停

---

### mmcache_store.h
**功能**: 定义 C++ ObjectStore 接口的实现类
- [mmcache_store.h 详细文档](mmcache_store_h.md)

**主要类**:
- `ResourceTracker` - 全局资源跟踪器
- `MmcacheStore` - ObjectStore 接口的实现类

---

### mmcache_store.cpp
**功能**: 实现 MmcacheStore 类
- [mmcache_store.cpp 详细文档](mmcache_store_cpp.md)

**主要方法**:
- `Init()` / `TearDown()` - 初始化/反初始化
- `RegisterBuffer()` / `UnRegisterBuffer()` - 缓冲区注册
- `GetInto()` / `BatchGetInto()` - 获取数据
- `PutFrom()` / `BatchPutFrom()` - 存储数据
- `GetIntoLayers()` / `BatchGetIntoLayers()` - 获取分层数据
- `PutFromLayers()` / `BatchPutFromLayers()` - 存储分层数据
- `Remove()` / `BatchRemove()` - 删除数据
- `IsExist()` / `BatchIsExist()` - 检查存在
- `GetKeyInfo()` / `BatchGetKeyInfo()` - 获取元信息

---

## 数据流

```
Python/C++ API
    ↓
核心实现文件 (mmc.cpp, mmc_client.cpp, mmc_service.cpp)
    ↓
内部实现类 (MmcClientDefault, MmcMetaService, MmcLocalServiceDefault)
    ↓
底层服务 (网络引擎, Blob Manager, 元数据管理)
```
