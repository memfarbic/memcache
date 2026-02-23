# mmc_service.h 文档

## 文件概述

`mmc_service.h` 定义了 MemCache_Hybrid 的服务端 C API 接口，包括元数据服务和本地服务的启动和停止。

**文件路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/include/mmc_service.h`

## 依赖/包含

```c
#include "mmc_def.h"
```

## 函数接口

### mmcs_meta_service_start (第27行)

```c
mmc_meta_service_t mmcs_meta_service_start(mmc_meta_service_config_t *config);
```

**功能**: 启动分布式内存缓存的元数据服务（全局服务）

**参数**:
- `config` [in]: 元数据服务配置，指向 `mmc_meta_service_config_t` 结构体

**返回值**:
- 成功: 返回元数据服务句柄
- 失败: 返回 `NULL`

**说明**:
- 元数据服务是全局服务，负责管理整个集群的元数据
- 包括服务发现、配置管理、数据位置索引等功能
- 返回的句柄用于后续的停止操作

---

### mmcs_meta_service_stop (第34行)

```c
void mmcs_meta_service_stop(mmc_meta_service_t handle);
```

**功能**: 停止元数据服务

**参数**:
- `handle` [in]: 元数据服务句柄，由 `mmcs_meta_service_start` 创建

**返回值**: 无

**说明**: 停止元数据服务并释放相关资源。

---

### mmcs_local_service_start (第43行)

```c
mmc_local_service_t mmcs_local_service_start(mmc_local_service_config_t *config);
```

**功能**: 启动分布式内存缓存的本地服务

**参数**:
- `config` [in]: 本地服务配置，指向 `mmc_local_service_config_t` 结构体

**返回值**:
- 成功: 返回本地服务句柄
- 失败: 返回 `NULL`

**说明**:
- 本地服务负责管理本地内存对象
- 每个节点都需要启动本地服务
- 管理本地的 DRAM/HBM 等内存资源

---

### mmcs_local_service_stop (第50行)

```c
void mmcs_local_service_stop(mmc_local_service_t handle);
```

**功能**: 停止本地服务

**参数**:
- `handle` [in]: 本地服务句柄，由 `mmcs_local_service_start` 创建

**返回值**: 无

**说明**: 停止本地服务并释放相关资源。

---

## 服务架构说明

MemCache_Hybrid 采用两层服务架构：

1. **元数据服务 (Meta Service)**:
   - 全局唯一的集群级服务
   - 管理数据对象的元数据
   - 处理服务发现
   - 维护数据位置索引

2. **本地服务 (Local Service)**:
   - 每个节点一个实例
   - 管理本地内存资源
   - 处理本地数据读写
   - 与 Blob Manager 交互

---

## C/C++ 兼容性处理

**第17-19行**: C++ 兼容性处理开始
```c
#ifdef __cplusplus
extern "C" {
#endif
```

**第52-54行**: C++ 兼容性处理结束
```c
#ifdef __cplusplus
}
#endif
```

---

## 宏定义

**第12行**: 头文件保护宏
```c
#ifndef __MEMFABRIC_MMC_SERVICE_H__
#define __MEMFABRIC_MMC_SERVICE_H__
```

**第56行**: 头文件保护宏结束
```c
#endif // __MEMFABRIC_MMC_SERVICE_H__
```
