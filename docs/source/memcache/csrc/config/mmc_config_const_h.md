# mmc_config_const.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/config/mmc_config_const.h`
- **文件用途**: 定义配置系统的常量和默认值，包含所有支持的配置项及其默认值
- **依赖项**: `<utility>` (标准库)

---

## 配置常量定义

### 命名空间 ConfConstant

配置常量定义在 `ock::mmc::ConfConstant` 命名空间中，每个配置项使用 `std::make_pair` 定义：
- **first**: 配置项名称（键）
- **second**: 默认值

---

## 元服务配置常量

### OCK_MMC_META_SERVICE_URL
**声明位置**: 行 21
**完整签名**:
```cpp
constexpr auto OCK_MMC_META_SERVICE_URL = std::make_pair("ock.mmc.meta_service_url", "tcp://127.0.0.1:5000");
```
**功能描述**: 元服务发现 URL
**默认值**: `tcp://127.0.0.1:5000`

### OCK_MMC_META_SERVICE_CONFIG_STORE_URL
**声明位置**: 行 22-23
**完整签名**:
```cpp
constexpr auto OCK_MMC_META_SERVICE_CONFIG_STORE_URL =
    std::make_pair("ock.mmc.meta_service.config_store_url", "tcp://127.0.0.1:6000");
```
**功能描述**: 配置存储服务 URL
**默认值**: `tcp://127.0.0.1:6000`

### OCK_MMC_META_SERVICE_HTTP_URL
**声明位置**: 行 24
**完整签名**:
```cpp
constexpr auto OCK_MMC_META_SERVICE_HTTP_URL = std::make_pair("ock.mmc.meta_service.metrics_url", "127.0.0.1:8000");
```
**功能描述**: 元服务 HTTP 指标 URL
**默认值**: `127.0.0.1:8000`

### OCK_MMC_META_HA_ENABLE
**声明位置**: 行 25
**完整签名**:
```cpp
constexpr auto OCK_MMC_META_HA_ENABLE = std::make_pair("ock.mmc.meta.ha.enable", false);
```
**功能描述**: 元服务高可用（HA）开关
**默认值**: `false`

---

## 驱逐阈值配置常量

### OKC_MMC_EVICT_THRESHOLD_HIGH
**声明位置**: 行 26
**完整签名**:
```cpp
constexpr auto OKC_MMC_EVICT_THRESHOLD_HIGH = std::make_pair("ock.mmc.evict_threshold_high", 70);
```
**功能描述**: 高驱逐阈值（内存使用百分比）
**默认值**: `70`
**取值范围**: 1-100

### OKC_MMC_EVICT_THRESHOLD_LOW
**声明位置**: 行 27
**完整签名**:
```cpp
constexpr auto OKC_MMC_EVICT_THRESHOLD_LOW = std::make_pair("ock.mmc.evict_threshold_low", 60);
```
**功能描述**: 低驱逐阈值（内存使用百分比）
**默认值**: `60`
**取值范围**: 1-99（必须小于高阈值）

---

## 日志配置常量

### OCK_MMC_LOG_LEVEL
**声明位置**: 行 28
**完整签名**:
```cpp
constexpr auto OCK_MMC_LOG_LEVEL = std::make_pair("ock.mmc.log_level", "info");
```
**功能描述**: 日志级别
**默认值**: `"info"`
**可选值**: `debug`, `info`, `warn`, `error`

### OCK_MMC_LOG_PATH
**声明位置**: 行 29
**完整签名**:
```cpp
constexpr auto OCK_MMC_LOG_PATH = std::make_pair("ock.mmc.log_path", "/var/log/memcache_hybrid");
```
**功能描述**: 日志文件路径
**默认值**: `/var/log/memcache_hybrid`

### OCK_MMC_LOG_ROTATION_FILE_SIZE
**声明位置**: 行 30
**完整签名**:
```cpp
constexpr auto OCK_MMC_LOG_ROTATION_FILE_SIZE = std::make_pair("ock.mmc.log_rotation_file_size", 20);
```
**功能描述**: 单个日志文件最大大小（MB）
**默认值**: `20`
**取值范围**: 1-500

### OCK_MMC_LOG_ROTATION_FILE_COUNT
**声明位置**: 行 31
**完整签名**:
```cpp
constexpr auto OCK_MMC_LOG_ROTATION_FILE_COUNT = std::make_pair("ock.mmc.log_rotation_file_count", 50);
```
**功能描述**: 保留的日志文件数量
**默认值**: `50`
**取值范围**: 1-50

---

## TLS 配置常量（ACC 层）

### OCK_MMC_TLS_ENABLE
**声明位置**: 行 33
**完整签名**:
```cpp
constexpr auto OCK_MMC_TLS_ENABLE = std::make_pair("ock.mmc.tls.enable", false);
```
**功能描述**: 是否启用 TLS 加密
**默认值**: `false`

### OCK_MMC_TLS_CA_PATH
**声明位置**: 行 34
**完整签名**:
```cpp
constexpr auto OCK_MMC_TLS_CA_PATH = std::make_pair("ock.mmc.tls.ca.path", "");
```
**功能描述**: CA 证书文件路径
**默认值**: `""`（空字符串）

### OCK_MMC_TLS_CRL_PATH
**声明位置**: 行 35
**完整签名**:
```cpp
constexpr auto OCK_MMC_TLS_CRL_PATH = std::make_pair("ock.mmc.tls.ca.crl.path", "");
```
**功能描述**: CRL（证书吊销列表）文件路径
**默认值**: `""`

### OCK_MMC_TLS_CERT_PATH
**声明位置**: 行 36
**完整签名**:
```cpp
constexpr auto OCK_MMC_TLS_CERT_PATH = std::make_pair("ock.mmc.tls.cert.path", "");
```
**功能描述**: 客户端证书文件路径
**默认值**: `""`

### OCK_MMC_TLS_KEY_PATH
**声明位置**: 行 37
**完整签名**:
```cpp
constexpr auto OCK_MMC_TLS_KEY_PATH = std::make_pair("ock.mmc.tls.key.path", "");
```
**功能描述**: 私钥文件路径
**默认值**: `""`

### OCK_MMC_TLS_KEY_PASS_PATH
**声明位置**: 行 38
**完整签名**:
```cpp
constexpr auto OCK_MMC_TLS_KEY_PASS_PATH = std::make_pair("ock.mmc.tls.key.pass.path", "");
```
**功能描述**: 私钥密码文件路径
**默认值**: `""`

### OCK_MMC_TLS_PACKAGE_PATH
**声明位置**: 行 39
**完整签名**:
```cpp
constexpr auto OCK_MMC_TLS_PACKAGE_PATH = std::make_pair("ock.mmc.tls.package.path", "");
```
**功能描述**: OpenSSL 动态库目录路径
**默认值**: `""`

### OCK_MMC_TLS_DECRYPTER_PATH
**声明位置**: 行 40
**完整签名**:
```cpp
constexpr auto OCK_MMC_TLS_DECRYPTER_PATH = std::make_pair("ock.mmc.tls.decrypter.path", "");
```
**功能描述**: 解密器库文件路径
**默认值**: `""`

---

## TLS 配置常量（Config Store）

### OCK_MMC_CS_TLS_ENABLE
**声明位置**: 行 41
**完整签名**:
```cpp
constexpr auto OCK_MMC_CS_TLS_ENABLE = std::make_pair("ock.mmc.config_store.tls.enable", false);
```
**功能描述**: 配置存储 TLS 开关
**默认值**: `false`

### OCK_MMC_CS_TLS_CA_PATH ~ OCK_MMC_CS_TLS_DECRYPTER_PATH
**声明位置**: 行 42-48
**功能描述**: 配置存储相关的 TLS 证书路径配置
**默认值**: 全部为 `""`

---

## 本地服务配置常量

### OKC_MMC_LOCAL_SERVICE_WORLD_SIZE
**声明位置**: 行 50
**完整签名**:
```cpp
constexpr auto OKC_MMC_LOCAL_SERVICE_WORLD_SIZE = std::make_pair("ock.mmc.local_service.world_size", 16);
```
**功能描述**: 本地服务 world size（节点数量）
**默认值**: `16`
**取值范围**: 1-1024

### OKC_MMC_LOCAL_SERVICE_BM_IP_PORT
**声明位置**: 行 51-52
**完整签名**:
```cpp
constexpr auto OKC_MMC_LOCAL_SERVICE_BM_IP_PORT =
    std::make_pair("ock.mmc.local_service.config_store_url", "tcp://127.0.0.1:6000");
```
**功能描述**: 本地服务 Blob Manager IP:Port
**默认值**: `tcp://127.0.0.1:6000`

### OKC_MMC_LOCAL_SERVICE_PROTOCOL
**声明位置**: 行 53
**完整签名**:
```cpp
constexpr auto OKC_MMC_LOCAL_SERVICE_PROTOCOL = std::make_pair("ock.mmc.local_service.protocol", "host_rdma");
```
**功能描述**: 数据传输协议
**默认值**: `"host_rdma"`
**可选值**: `host_rdma`, `host_urma`, `host_tcp`, `device_rdma`, `device_sdma`

### OKC_MMC_LOCAL_SERVICE_DRAM_SIZE
**声明位置**: 行 54
**完整签名**:
```cpp
constexpr auto OKC_MMC_LOCAL_SERVICE_DRAM_SIZE = std::make_pair("ock.mmc.local_service.dram.size", "128MB");
```
**功能描述**: DRAM 分配大小
**默认值**: `"128MB"`
**格式**: 支持 KB/MB/GB/TB 单位

### OKC_MMC_LOCAL_SERVICE_MAX_DRAM_SIZE
**声明位置**: 行 55
**完整签名**:
```cpp
constexpr auto OKC_MMC_LOCAL_SERVICE_MAX_DRAM_SIZE = std::make_pair("ock.mmc.local_service.max.dram.size", "64GB");
```
**功能描述**: DRAM 最大分配大小
**默认值**: `"64GB"`

### OKC_MMC_LOCAL_SERVICE_HBM_SIZE
**声明位置**: 行 56
**完整签名**:
```cpp
constexpr auto OKC_MMC_LOCAL_SERVICE_HBM_SIZE = std::make_pair("ock.mmc.local_service.hbm.size", "0");
```
**功能描述**: HBM 分配大小
**默认值**: `"0"`（表示不使用 HBM）

### OKC_MMC_LOCAL_SERVICE_MAX_HBM_SIZE
**声明位置**: 行 57
**完整签名**:
```cpp
constexpr auto OKC_MMC_LOCAL_SERVICE_MAX_HBM_SIZE = std::make_pair("ock.mmc.local_service.max.hbm.size", "0");
```
**功能描述**: HBM 最大分配大小
**默认值**: `"0"`

### OKC_MMC_LOCAL_SERVICE_BM_HCOM_URL
**声明位置**: 行 58-59
**完整签名**:
```cpp
constexpr auto OKC_MMC_LOCAL_SERVICE_BM_HCOM_URL =
    std::make_pair("ock.mmc.local_service.hcom_url", "tcp://127.0.0.1:7000");
```
**功能描述**: HCOM（集合通信）URL
**默认值**: `tcp://127.0.0.1:7000`

---

## HCOM TLS 配置常量

### OCK_MMC_HCOM_TLS_ENABLE ~ OCK_MMC_HCOM_TLS_DECRYPTER_PATH
**声明位置**: 行 60-66
**功能描述**: HCOM 通信的 TLS 配置
**默认值**: 全部为 `false` 或 `""`

---

## 客户端配置常量

### OKC_MMC_CLIENT_RETRY_MILLISECONDS
**声明位置**: 行 68
**完整签名**:
```cpp
constexpr auto OKC_MMC_CLIENT_RETRY_MILLISECONDS = std::make_pair("ock.mmc.client.retry_milliseconds", 0);
```
**功能描述**: 客户端 RPC 重试超时时间（毫秒）
**默认值**: `0`（不重试）
**取值范围**: 0-600000

### OCK_MMC_CLIENT_TIMEOUT_SECONDS
**声明位置**: 行 69
**完整签名**:
```cpp
constexpr auto OCK_MMC_CLIENT_TIMEOUT_SECONDS = std::make_pair("ock.mmc.client.timeout.seconds", 60);
```
**功能描述**: 客户端超时时间（秒）
**默认值**: `60`
**取值范围**: 1-600

### OCK_MMC_CLIENT_READ_THREAD_POOL_SIZE
**声明位置**: 行 70
**完整签名**:
```cpp
constexpr auto OCK_MMC_CLIENT_READ_THREAD_POOL_SIZE = std::make_pair("ock.mmc.client.read_thread_pool.size", 32);
```
**功能描述**: 读线程池大小
**默认值**: `32`
**取值范围**: 1-64

### OCK_MMC_CLIENT_AGGREGATE_IO
**声明位置**: 行 71
**完整签名**:
```cpp
constexpr auto OCK_MMC_CLIENT_AGGREGATE_IO = std::make_pair("ock.mmc.client.aggregate.io", true);
```
**功能描述**: 是否启用聚合 IO
**默认值**: `true`

### OCK_MMC_CLIENT_AGGREGATE_NUM
**声明位置**: 行 72
**完整签名**:
```cpp
constexpr auto OCK_MMC_CLIENT_AGGREGATE_NUM = std::make_pair("ock.mmc.client.aggregate.num", 122);
```
**功能描述**: 聚合 IO 批次大小
**默认值**: `122`
**取值范围**: 1-131072

### OCK_MMC_CLIENT_WRITE_THREAD_POOL_SIZE
**声明位置**: 行 73
**完整签名**:
```cpp
constexpr auto OCK_MMC_CLIENT_WRITE_THREAD_POOL_SIZE = std::make_pair("ock.mmc.client.write_thread_pool.size", 4);
```
**功能描述**: 写线程池大小
**默认值**: `4`
**取值范围**: 1-64

---

## 配置验证范围常量

### 日志相关范围
**声明位置**: 行 76-80
```cpp
constexpr int MIN_LOG_ROTATION_FILE_SIZE = 1;
constexpr int MAX_LOG_ROTATION_FILE_SIZE = 500;
constexpr int MIN_LOG_ROTATION_FILE_COUNT = 1;
constexpr int MAX_LOG_ROTATION_FILE_COUNT = 50;
```

### 设备和 World Size 范围
**声明位置**: 行 82-86
```cpp
constexpr int MIN_DEVICE_ID = 0;
constexpr int MAX_DEVICE_ID = 383;
constexpr int MIN_WORLD_SIZE = 1;
constexpr int MAX_WORLD_SIZE = 1024;
```

### 驱逐阈值范围
**声明位置**: 行 88-89
```cpp
constexpr int MIN_EVICT_THRESHOLD = 1;
constexpr int MAX_EVICT_THRESHOLD = 100;
```

### 客户端超时范围
**声明位置**: 行 91-95
```cpp
constexpr int MIN_RETRY_MS = 0;
constexpr int MAX_RETRY_MS = 600000;
constexpr int MIN_TIMEOUT_SEC = 1;
constexpr int MAX_TIMEOUT_SEC = 600;
```

### 线程池和聚合范围
**声明位置**: 行 97-99
```cpp
constexpr int MIN_THREAD_POOL_SIZE = 1;
constexpr int MAX_THREAD_POOL_SIZE = 64;
constexpr int MAX_AGGREGATE_NUM = 131072; // 128K
```

### 内存大小限制
**声明位置**: 行 101-102
```cpp
constexpr uint64_t MAX_DRAM_SIZE = 1024ULL * 1024ULL * 1024ULL * 1024ULL; // 1TB
constexpr uint64_t MAX_HBM_SIZE = 1024ULL * 1024ULL * 1024ULL * 1024ULL;  // 1TB
```

---

## 内存单位转换常量

**声明位置**: 行 104-111
```cpp
constexpr uint64_t KB_MEM_BYTES = 1024ULL;
constexpr uint64_t MB_MEM_BYTES = 1024ULL * 1024ULL;
constexpr uint64_t GB_MEM_BYTES = 1024ULL * 1024ULL * 1024ULL;
constexpr uint64_t TB_MEM_BYTES = 1024ULL * 1024ULL * 1024ULL * 1024ULL;

constexpr int MB_NUM = 1024 * 1024;
constexpr uint64_t MEM_2MB_BYTES = 2ULL * 1024ULL * 1024ULL;
constexpr uint64_t MEM_128MB_BYTES = 128ULL * 1024ULL * 1024ULL;
```

**功能描述**: 内存单位与字节的转换常量

### 路径长度限制
**声明位置**: 行 113
```cpp
constexpr unsigned long PATH_MAX_LEN = 1023;
```

---

## 使用示例

```cpp
#include "mmc_config_const.h"

using namespace ock::mmc::ConfConstant;

// 使用配置常量获取默认值
std::string defaultUrl = OCK_MMC_META_SERVICE_URL.second;  // "tcp://127.0.0.1:5000"
int defaultThreshold = OKC_MMC_EVICT_THRESHOLD_HIGH.second; // 70

// 使用配置常量获取配置项名称
std::string configKey = OCK_MMC_LOG_LEVEL.first;  // "ock.mmc.log_level"
```

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ConfConstant 命名空间                              │
├─────────────────────────────────────────────────────────────────────┤
│  元服务配置                                                           │
│  ├── OCK_MMC_META_SERVICE_URL                                        │
│  ├── OCK_MMC_META_SERVICE_CONFIG_STORE_URL                           │
│  ├── OCK_MMC_META_SERVICE_HTTP_URL                                   │
│  └── OCK_MMC_META_HA_ENABLE                                          │
├─────────────────────────────────────────────────────────────────────┤
│  日志配置                                                             │
│  ├── OCK_MMC_LOG_LEVEL                                               │
│  ├── OCK_MMC_LOG_PATH                                                │
│  ├── OCK_MMC_LOG_ROTATION_FILE_SIZE                                  │
│  └── OCK_MMC_LOG_ROTATION_FILE_COUNT                                 │
├─────────────────────────────────────────────────────────────────────┤
│  TLS 配置（ACC）                                                      │
│  ├── OCK_MMC_TLS_ENABLE                                              │
│  ├── OCK_MMC_TLS_CA_PATH                                             │
│  ├── OCK_MMC_TLS_CERT_PATH                                           │
│  └── ...                                                             │
├─────────────────────────────────────────────────────────────────────┤
│  TLS 配置（Config Store）                                             │
│  ├── OCK_MMC_CS_TLS_ENABLE                                           │
│  └── ...                                                             │
├─────────────────────────────────────────────────────────────────────┤
│  本地服务配置                                                         │
│  ├── OKC_MMC_LOCAL_SERVICE_WORLD_SIZE                                │
│  ├── OKC_MMC_LOCAL_SERVICE_PROTOCOL                                  │
│  ├── OKC_MMC_LOCAL_SERVICE_DRAM_SIZE                                 │
│  └── ...                                                             │
├─────────────────────────────────────────────────────────────────────┤
│  客户端配置                                                           │
│  ├── OKC_MMC_CLIENT_RETRY_MILLISECONDS                               │
│  ├── OKC_MMC_CLIENT_TIMEOUT_SECONDS                                  │
│  ├── OCK_MMC_CLIENT_READ_THREAD_POOL_SIZE                            │
│  └── ...                                                             │
├─────────────────────────────────────────────────────────────────────┤
│  验证范围常量                                                         │
│  ├── MIN/MAX_LOG_ROTATION_FILE_SIZE                                  │
│  ├── MIN/MAX_WORLD_SIZE                                              │
│  ├── MIN/MAX_EVICT_THRESHOLD                                         │
│  └── ...                                                             │
├─────────────────────────────────────────────────────────────────────┤
│  内存单位常量                                                         │
│  ├── KB_MEM_BYTES                                                    │
│  ├── MB_MEM_BYTES                                                    │
│  ├── GB_MEM_BYTES                                                    │
│  └── TB_MEM_BYTES                                                    │
└─────────────────────────────────────────────────────────────────────┘
```
