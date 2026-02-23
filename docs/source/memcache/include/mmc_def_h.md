# mmc_def.h 文档

## 文件概述

`mmc_def.h` 是 MemCache_Hybrid 项目的核心定义头文件，包含了公共数据结构、类型定义和常量宏定义。

**文件路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/include/mmc_def.h`

## 依赖/包含

```c
#include <stdint.h>
```

## 常量定义 (第17-22行)

```c
#define DISCOVERY_URL_SIZE 1024      // 服务发现URL最大长度
#define PATH_MAX_SIZE      1024      // 路径最大长度
#define PROTOCOL_SIZE      64        // 协议名称最大长度
#define MAX_BATCH_OP_COUNT 16384     // 最大批量操作数量
#define TLS_PATH_SIZE      256       // TLS路径大小
#define TLS_PATH_MAX_LEN   (TLS_PATH_SIZE - 1)  // TLS路径最大长度
```

**说明**: 这些常量定义了系统各种配置参数的最大长度限制。

---

## 类型定义

### 不透明类型定义 (第28-30行)

```c
typedef void *mmc_meta_service_t;      // 元数据服务句柄类型
typedef void *mmc_local_service_t;     // 本地服务句柄类型
typedef void *mmc_client_t;            // 客户端句柄类型
```

**说明**: 使用不透明指针模式隐藏内部实现细节，提供良好的封装性。

### 外部日志函数类型 (第31-33行)

```c
#ifndef MMC_OUT_LOGGER
typedef void (*ExternalLog)(int level, const char *msg);
#endif
```

**说明**: 外部日志回调函数类型定义，用于自定义日志处理。

---

## 数据结构

### mmc_tls_config (第35-44行)

```c
typedef struct {
    bool tlsEnable;                    // 是否启用TLS
    char caPath[TLS_PATH_SIZE];        // CA证书路径
    char crlPath[TLS_PATH_SIZE];       // CRL(证书吊销列表)路径
    char certPath[TLS_PATH_SIZE];      // 客户端证书路径
    char keyPath[TLS_PATH_SIZE];       // 客户端私钥路径
    char keyPassPath[TLS_PATH_SIZE];   // 私钥密码路径
    char packagePath[TLS_PATH_SIZE];   // 加密包路径
    char decrypterLibPath[TLS_PATH_SIZE]; // 解密库路径
} mmc_tls_config;
```

**说明**: TLS (Transport Layer Security) 安全传输配置结构体，用于配置加密通信参数。

---

### mmc_meta_service_config_t (第46-59行)

```c
typedef struct {
    char discoveryURL[DISCOVERY_URL_SIZE];   // 服务发现URL，由协议和URL组成，如 tcp://、etcd://、zk://
    char configStoreURL[DISCOVERY_URL_SIZE]; // 配置存储URL
    char httpURL[DISCOVERY_URL_SIZE];        // HTTP服务URL
    bool haEnable;                            // 是否启用高可用
    int32_t logLevel;                         // 日志级别
    char logPath[PATH_MAX_SIZE];              // 日志文件路径
    int32_t logRotationFileSize;              // 日志轮转文件大小
    int32_t logRotationFileCount;             // 日志轮转文件数量
    uint16_t evictThresholdHigh;              // 高驱逐阈值(百分比)
    uint16_t evictThresholdLow;               // 低驱逐阈值(百分比)
    mmc_tls_config accTlsConfig;              // ACC(接入层) TLS配置
    mmc_tls_config configStoreTlsConfig;      // 配置存储 TLS配置
} mmc_meta_service_config_t;
```

**说明**: 元数据服务配置结构体，包含服务发现、日志、TLS等配置。

---

### mmc_local_service_config_t (第61-80行)

```c
typedef struct {
    char discoveryURL[DISCOVERY_URL_SIZE];  // 服务发现URL
    uint32_t deviceId;                       // 设备ID
    uint32_t rankId;                         // BM全局统一编号(Rank ID)
    uint32_t worldSize;                      // 总Rank数量
    char bmIpPort[DISCOVERY_URL_SIZE];       // Blob Manager IP和端口
    char bmHcomUrl[DISCOVERY_URL_SIZE];      // BM HCOM通信URL
    uint32_t createId;                       // 创建ID
    char dataOpType[PROTOCOL_SIZE];          // 数据操作类型
    uint64_t localDRAMSize;                  // 本地DRAM大小
    uint64_t localMaxDRAMSize;               // 本地最大DRAM大小
    uint64_t localHBMSize;                   // 本地HBM(高带宽内存)大小
    uint64_t localMaxHBMSize;                // 本地最大HBM大小
    uint32_t flags;                           // 标志位
    mmc_tls_config accTlsConfig;             // ACC TLS配置
    int32_t logLevel;                        // 日志级别
    ExternalLog logFunc;                     // 外部日志函数
    mmc_tls_config hcomTlsConfig;            // HCOM TLS配置
    mmc_tls_config configStoreTlsConfig;     // 配置存储TLS配置
} mmc_local_service_config_t;
```

**说明**: 本地服务配置结构体，包含设备信息、内存配置、通信配置等。

---

### mmc_client_config_t (第82-94行)

```c
typedef struct {
    char discoveryURL[DISCOVERY_URL_SIZE];  // 服务发现URL
    uint32_t rankId;                         // Rank ID
    uint32_t rpcRetryTimeOut;                // RPC重试超时时间
    uint32_t timeOut;                        // 操作超时时间
    uint32_t readThreadPoolNum;              // 读线程池数量
    uint32_t writeThreadPoolNum;             // 写线程池数量
    bool aggregateIO;                        // 是否聚合IO
    int32_t aggregateNum;                    // 聚合数量
    int32_t logLevel;                        // 日志级别
    ExternalLog logFunc;                     // 外部日志函数
    mmc_tls_config tlsConfig;                // TLS配置
} mmc_client_config_t;
```

**说明**: 客户端配置结构体，包含网络配置、线程池配置、聚合IO配置等。

---

### mmc_buffer (第96-101行)

```c
typedef struct {
    uint64_t addr;     // 缓冲区地址
    uint32_t type;     // 介质类型 (enum MediaType)
    uint64_t offset;   // 偏移量
    uint64_t len;      // 长度
} mmc_buffer;
```

**说明**: 数据缓冲区描述结构体，用于描述内存数据的位置和大小。

---

### affinity_policy (第103-105行)

```c
enum affinity_policy : int {
    NATIVE_AFFINITY = 0,  // 原生亲和性策略
};
```

**说明**: 数据亲和性策略枚举，用于控制数据放置的位置偏好。

---

### mmc_put_options (第107-113行)

```c
#define MAX_BLOB_COPIES 8  // 最大Blob副本数

typedef struct {
    uint16_t mediaType;                              // 介质类型
    affinity_policy policy;                          // 亲和性策略
    uint16_t replicaNum;                             // 副本数量(<= MAX_BLOB_COPIES)
    int32_t preferredLocalServiceIDs[MAX_BLOB_COPIES]; // 首选本地服务ID列表
} mmc_put_options;
```

**说明**: Put 操作选项结构体，用于控制数据存储时的副本策略和位置偏好。

---

### mmc_data_info (第115-122行)

```c
typedef struct {
    uint64_t size;                       // 数据大小
    uint16_t prot;                       // 保护属性
    uint8_t numBlobs;                    // Blob数量
    bool valid;                          // 是否有效
    uint32_t ranks[MAX_BLOB_COPIES];     // Blob所在的Rank列表
    uint16_t types[MAX_BLOB_COPIES];     // Blob的介质类型列表
} mmc_data_info;
```

**说明**: 数据信息结构体，用于查询数据的元信息，包括大小、副本位置等。

---

## C/C++ 兼容性处理

**第24-26行**: C++ 兼容性处理开始
```c
#ifdef __cplusplus
extern "C" {
#endif
```

**第124-126行**: C++ 兼容性处理结束
```c
#ifdef __cplusplus
}
#endif
```

---

## 宏定义

**第12行**: 头文件保护宏
```c
#ifndef __MEMFABRIC_MMC_DEF_H__
#define __MEMFABRIC_MMC_DEF_H__
```

**第128行**: 头文件保护宏结束
```c
#endif //__MEMFABRIC_MMC_DEF_H__
```
