# mmc_client.h 文档

## 文件概述

`mmc_client.h` 定义了 MemCache_Hybrid 的客户端 C API 接口，提供了数据存取、查询、删除等核心操作。

**文件路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/include/mmc_client.h`

## 依赖/包含

```c
#include <stddef.h>
#include "mmc_def.h"
```

## 函数接口

### mmcc_init (第29行)

```c
int32_t mmcc_init(mmc_client_config_t *config);
```

**功能**: 使用配置初始化分布式内存缓存客户端（单例模式）

**参数**:
- `config` [in]: 客户端配置，指向 `mmc_client_config_t` 结构体

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 客户端是单例模式，只能初始化一次。

---

### mmcc_uninit (第34行)

```c
void mmcc_uninit(void);
```

**功能**: 反初始化客户端

**参数**: 无

**返回值**: 无

**说明**: 释放客户端资源，应与 `mmcc_init` 配对使用。

---

### mmcc_register_buffer (第42行)

```c
int32_t mmcc_register_buffer(uint64_t addr, uint64_t size);
```

**功能**: 注册内存缓冲区到 Blob Manager，支持 SDMA (Symmetric DMA)

**参数**:
- `addr` [in]: 要注册的缓冲区地址
- `size` [in]: 要注册的缓冲区大小

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 用于零拷贝操作，需要先将缓冲区注册到 BM。

---

### mmcc_unregister_buffer (第50行)

```c
int32_t mmcc_unregister_buffer(uint64_t addr, uint64_t size);
```

**功能**: 从 Blob Manager 注销内存缓冲区

**参数**:
- `addr` [in]: 要注销的缓冲区地址
- `size` [in]: 要注销的缓冲区大小

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 与 `mmcc_register_buffer` 配对使用，释放注册的缓冲区。

---

### mmcc_put (第62行)

```c
int32_t mmcc_put(const char *key, mmc_buffer *buf, mmc_put_options options, uint32_t flags);
```

**功能**: 将数据对象存入分布式内存缓存

**参数**:
- `key` [in]: 数据的键，长度需小于256字节
- `buf` [in]: 要存储的数据缓冲区
- `options` [in]: Put 操作选项（副本策略、位置偏好等）
- `flags` [in]: 可选标志位，保留

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 支持同步和异步两种模式。

---

### mmcc_local_service_id (第70行)

```c
int32_t mmcc_local_service_id(uint32_t *localServiceId);
```

**功能**: 查询本地服务 ID

**参数**:
- `localServiceId` [out]: 输出本地服务 ID

**返回值**:
- `0`: 成功
- 其他值: 失败

---

### mmcc_get (第81行)

```c
int32_t mmcc_get(const char *key, mmc_buffer *buf, uint32_t flags);
```

**功能**: 根据键从分布式内存缓存获取数据对象

**参数**:
- `key` [in]: 数据的键，长度需小于256字节
- `buf` [in/out]: 用于存储获取数据的缓冲区
- `flags` [in]: 可选标志位，保留

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 支持同步和异步两种模式。

---

### mmcc_query (第92行)

```c
int32_t mmcc_query(const char *key, mmc_data_info *info, uint32_t flags);
```

**功能**: 根据键查询数据对象的信息

**参数**:
- `key` [in]: 数据的键，长度需小于256字节
- `info` [out]: 输出数据信息（大小、副本位置等）
- `flags` [in]: 可选标志位，保留

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 只查询元数据，不传输实际数据内容。

---

### mmcc_batch_query (第103行)

```c
int32_t mmcc_batch_query(const char **keys, size_t keys_count, mmc_data_info *info, uint32_t flags);
```

**功能**: 批量查询多个键的数据信息

**参数**:
- `keys` [in]: 键数组，每个键长度需小于256字节
- `keys_count` [in]: 键的数量
- `info` [out]: 输出数据信息数组
- `flags` [in]: 操作标志位

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 批量操作可以提高效率。

---

### mmcc_remove (第113行)

```c
int32_t mmcc_remove(const char *key, uint32_t flags);
```

**功能**: 从分布式内存缓存中移除指定键的对象

**参数**:
- `key` [in]: 要移除的数据的键，长度需小于256字节
- `flags` [in]: 可选标志位，保留

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 支持同步和异步两种模式。

---

### mmcc_batch_remove (第124行)

```c
int32_t mmcc_batch_remove(const char **keys, uint32_t keys_count, int32_t *remove_results, uint32_t flags);
```

**功能**: 批量移除多个键的数据对象

**参数**:
- `keys` [in]: 要移除的键列表
- `keys_count` [in]: 键的数量
- `remove_results` [out]: 每个移除操作的结果数组
- `flags` [in]: 操作标志位

**返回值**:
- `0`: 成功
- 正值: 发生错误

---

### mmcc_exist (第132行)

```c
int32_t mmcc_exist(const char *key, uint32_t flags);
```

**功能**: 判断指定键是否存在于 Blob Manager 中

**参数**:
- `key` [in]: 要判断的键，长度需小于256字节
- `flags` [in]: 可选标志位，保留

**返回值**:
- `0`: 成功

---

### mmcc_batch_exist (第142行)

```c
int32_t mmcc_batch_exist(const char **keys, uint32_t keys_count, int32_t *exist_results, uint32_t flags);
```

**功能**: 批量判断多个键是否存在于 Blob Manager 中

**参数**:
- `keys` [in]: 要判断的键列表，每个键长度需小于256字节
- `keys_count` [in]: 键的数量
- `exist_results` [out]: 每个键在 BM 中的存在状态列表
- `flags` [in]: 操作标志位

**返回值**:
- `0`: 成功

---

### mmcc_batch_put (第155行)

```c
int32_t mmcc_batch_put(const char **keys, uint32_t keys_count, const mmc_buffer *bufs,
                       mmc_put_options &options, uint32_t flags, int *results);
```

**功能**: 批量将多个数据对象存入分布式内存缓存

**参数**:
- `keys` [in]: 数据对象的键数组
- `keys_count` [in]: 键的数量
- `bufs` [in]: 要存储的数据缓冲区数组
- `options` [in]: 批量 Put 操作选项
- `flags` [in]: 可选标志位，保留
- `results` [out]: 每个操作的结果

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 支持同步和异步两种模式，批量操作可提高吞吐量。

---

### mmcc_batch_get (第168行)

```c
int32_t mmcc_batch_get(const char **keys, uint32_t keys_count, mmc_buffer *bufs, uint32_t flags, int *results);
```

**功能**: 批量获取多个数据对象

**参数**:
- `keys` [in]: 数据对象的键数组
- `keys_count` [in]: 键的数量
- `bufs` [out]: 存储获取数据的缓冲区数组
- `flags` [in]: 可选标志位，保留
- `results` [out]: 每个操作的结果

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 支持同步和异步两种模式。

---

## C/C++ 兼容性处理

**第19-21行**: C++ 兼容性处理开始
```c
#ifdef __cplusplus
extern "C" {
#endif
```

**第170-172行**: C++ 兼容性处理结束
```c
#ifdef __cplusplus
}
#endif
```

---

## 宏定义

**第12行**: 头文件保护宏
```c
#ifndef __MEMFABRIC_MMC_CLIENT_H__
#define __MEMFABRIC_MMC_CLIENT_H__
```

**第174行**: 头文件保护宏结束
```c
#endif //__MEMFABRIC_MMC_CLIENT_H__
```
