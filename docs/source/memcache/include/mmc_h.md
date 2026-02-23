# mmc.h 文档

## 文件概述

`mmc.h` 是 MemCache_Hybrid 项目的公共 C API 头文件，定义了客户端和本地服务的初始化和配置接口。

**文件路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/include/mmc.h`

## 依赖/包含

```c
#include "stdint.h"
```

## 数据结构

### mmc_init_config (第21-24行)

```c
typedef struct {
    uint32_t deviceId;    // 设备ID，标识当前设备
    bool initBm;          // 是否初始化 Blob Manager (BM)
} mmc_init_config;
```

**说明**: MemCache 初始化配置结构体，用于 `mmc_init` 函数。

## 函数接口

### mmc_init (第31行)

```c
int32_t mmc_init(const mmc_init_config *config);
```

**功能**: 初始化 memcache 客户端和本地服务

**参数**:
- `config` [in]: 初始化配置，指向 `mmc_init_config` 结构体的指针

**返回值**:
- `0`: 成功
- 其他值: 失败

**说明**: 这是 MemCache 系统的主入口函数，负责初始化整个运行环境。

---

### mmc_set_extern_logger (第48行)

```c
int32_t mmc_set_extern_logger(void (*func)(int level, const char *msg));
```

**功能**: 设置外部日志函数，允许用户自定义日志记录器

**参数**:
- `func` [in]: 外部日志函数指针
  - `level`: 日志级别
    - `0`: DEBUG
    - `1`: INFO
    - `2`: WARN
    - `3`: ERROR
  - `msg`: 日志消息内容

**返回值**:
- `0`: 成功

**说明**:
- 用户可以设置自定义的日志函数
- 如果未设置，日志将输出到 stdout
- 通过统一日志工具，日志消息可以写入与调用者相同的日志文件

---

### mmc_set_log_level (第56行)

```c
int32_t mmc_set_log_level(int level);
```

**功能**: 设置日志打印级别

**参数**:
- `level` [in]: 日志级别
  - `0`: debug
  - `1`: info
  - `2`: warn
  - `3`: error

**返回值**:
- `0`: 成功

**说明**: 控制日志输出的详细程度，只有大于等于设定级别的日志才会被输出。

---

### mmc_uninit (第61行)

```c
void mmc_uninit(void);
```

**功能**: 反初始化 smem 运行环境

**参数**: 无

**返回值**: 无

**说明**: 清理 MemCache 运行环境，释放资源。应与 `mmc_init` 配对使用。

---

## C 兼容性处理

**第17-19行**: C++ 兼容性处理
```c
#ifdef __cplusplus
extern "C" {
#endif
```

**第63-65行**: C++ 兼容性处理结束
```c
#ifdef __cplusplus
}
#endif
```

**说明**: 使用 `extern "C"` 确保 C++ 编译器使用 C 链接方式，使该头文件可被 C 和 C++ 代码同时使用。

---

## 宏定义

**第12行**: 头文件保护宏
```c
#ifndef __MEMFABRIC_MMC_H__
#define __MEMFABRIC_MMC_H__
```

**第67行**: 头文件保护宏结束
```c
#endif //__MEMFABRIC_MMC_H__
```

**说明**: 防止头文件被重复包含。
