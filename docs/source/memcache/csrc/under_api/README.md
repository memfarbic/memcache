# under_api 模块文档

## 模块概述

`under_api` 模块提供对底层 SMEM (Shared Memory) Big Memory API 的封装，通过动态加载的方式调用 `libmf_smem.so` 库，实现跨节点内存共享和通信功能。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/under_api/`

## 文件列表

### mf_smem 子目录
- `smem_bm_def.h` - SMEM BM 数据结构定义
- `smem_bm_api.h` - SMEM BM API 封装头文件
- `smem_bm_api.cpp` - SMEM BM API 封装实现

---

## 详细文档

### smem_bm_def.h

**功能**: 定义 SMEM BM 的数据结构和常量。

**SMEM BM 句柄**:
```cpp
typedef void *smem_bm_t;  // BM 对象句柄
```

**常量定义**:
```cpp
#define SMEM_BM_TIMEOUT_MAX   UINT32_MAX  // 最大超时时间
#define ASYNC_COPY_FLAG       (1UL << 0)  // 异步拷贝标志
#define SMEM_BM_INIT_GVM_FLAG (1ULL << 1ULL) // 初始化 GVM 模块
#define SMEM_TLS_PATH_SIZE    256         // TLS 路径大小
```

**内存类型枚举** (`smem_bm_mem_type`):
```cpp
typedef enum {
    SMEM_MEM_TYPE_LOCAL_DEVICE = 0,  // 本地设备内存
    SMEM_MEM_TYPE_LOCAL_HOST,        // 本地主机内存
    SMEM_MEM_TYPE_DEVICE,            // 全局设备内存
    SMEM_MEM_TYPE_HOST,              // 全局主机内存
    SMEM_MEM_TYPE_BUTT
} smem_bm_mem_type;
```

**数据操作类型枚举** (`smem_bm_data_op_type`):
```cpp
typedef enum {
    SMEMB_DATA_OP_SDMA = 1U << 0,        // SDMA 操作
    SMEMB_DATA_OP_HOST_RDMA = 1U << 1,   // Host RDMA
    SMEMB_DATA_OP_HOST_TCP = 1U << 2,    // Host TCP
    SMEMB_DATA_OP_DEVICE_RDMA = 1U << 3, // Device RDMA
    SMEMB_DATA_OP_BUTT
} smem_bm_data_op_type;
```

**拷贝类型枚举** (`smem_bm_copy_type`):
```cpp
typedef enum {
    SMEMB_COPY_L2G = 0,   // Local to Global (本地到全局)
    SMEMB_COPY_G2L = 1,   // Global to Local (全局到本地)
    SMEMB_COPY_G2H = 2,   // Global to Host (全局到主机)
    SMEMB_COPY_H2G = 3,   // Host to Global (主机到全局)
    SMEMB_COPY_G2G = 4,   // Global to Global (全局到全局)
    SMEMB_COPY_AUTO = 9,  // 自动类型
    SMEMB_COPY_BUTT
} smem_bm_copy_type;
```

**TLS 配置结构** (`smem_tls_config`):
```cpp
typedef struct {
    bool tlsEnable;                          // 是否启用 TLS
    char caPath[SMEM_TLS_PATH_SIZE];         // CA 证书路径
    char crlPath[SMEM_TLS_PATH_SIZE];        // CRL 路径
    char certPath[SMEM_TLS_PATH_SIZE];       // 证书路径
    char keyPath[SMEM_TLS_PATH_SIZE];        // 密钥路径
    char keyPassPath[SMEM_TLS_PATH_SIZE];    // 密钥密码路径
    char packagePath[SMEM_TLS_PATH_SIZE];    // 包路径
    char decrypterLibPath[SMEM_TLS_PATH_SIZE]; // 解密库路径
} smem_tls_config;
```

**BM 配置结构** (`smem_bm_config_t`):
```cpp
typedef struct {
    uint32_t initTimeout;             // 初始化超时（默认120s）
    uint32_t createTimeout;           // 创建超时（默认120s）
    uint32_t controlOperationTimeout; // 控制操作超时（默认120s）
    bool startConfigStoreServer;      // 是否启动配置服务器
    bool startConfigStoreOnly;        // 仅启动配置服务器
    bool dynamicWorldSize;            // 动态成员加入
    bool unifiedAddressSpace;         // 统一地址空间（SVM）
    bool autoRanking;                 // 自动分配 Rank ID
    uint16_t rankId;                  // 用户指定 Rank ID
    uint32_t flags;                   // 其他标志
    char hcomUrl[64];                 // HCOM URL
    smem_tls_config hcomTlsConfig;    // HCOM TLS 配置
    smem_tls_config storeTlsConfig;   // 存储 TLS 配置
} smem_bm_config_t;
```

**拷贝参数结构**:

```cpp
// 2D 拷贝参数
typedef struct {
    void *src;         // 源地址
    uint64_t spitch;   // 源跨度
    void *dest;        // 目标地址
    uint64_t dpitch;   // 目标跨度
    uint64_t width;    // 宽度
    uint64_t height;   // 高度
} smem_copy_2d_params;

// 1D 拷贝参数
typedef struct {
    const void *src;   // 源地址
    void *dest;        // 目标地址
    size_t dataSize;   // 数据大小
} smem_copy_params;

// 批量拷贝参数
typedef struct {
    void **sources;           // 源地址数组
    void **destinations;      // 目标地址数组
    const uint64_t *dataSizes; // 大小数组
    uint32_t batchSize;       // 批量数量
} smem_batch_copy_params;
```

---

### smem_bm_api.h

**功能**: 定义 SMEM BM API 的 C++ 封装类，通过动态加载方式调用底层库。

**函数指针类型定义**:
```cpp
// SMEM 基础函数
using smemInitFunc = int32_t (*)(uint32_t);
using smemUnInitFunc = void (*)();
using smemSetExternLoggerFunc = int32_t (*)(void (*)(int level, const char *));
using smemSetLogLevelFunc = int32_t (*)(int);
using smemGetLastErrMsgFunc = const char *(*)();
using smemGetAndClearLastErrMsgFunc = const char *(*)();

// SMEM BM 函数
using smemBmConfigInitFunc = int32_t (*)(smem_bm_config_t *);
using smemBmInitFunc = int32_t (*)(const char *, uint32_t, uint16_t, const smem_bm_config_t *);
using smemBmUnInitFunc = void (*)(uint32_t);
using smemBmGetRankIdFunc = uint32_t (*)();
using smemBmCreateFunc = smem_bm_t (*)(uint32_t, uint32_t, smem_bm_data_op_type, uint64_t, uint64_t, uint32_t);
using smemBmDestroyFunc = void (*)(smem_bm_t);
using smemBmJoinFunc = int32_t (*)(smem_bm_t, uint32_t);
using smemBmLeaveFunc = int32_t (*)(smem_bm_t, uint32_t);
using smemBmPtrFunc = void *(*)(smem_bm_t, uint16_t);
using smemBmCopyFunc = int32_t (*)(smem_bm_t, const void *, void *, uint64_t, smem_bm_copy_type, uint32_t);
```

**MFSmemApi 类**:
```cpp
class MFSmemApi {
public:
    // 加载动态库
    static Result LoadLibrary(const std::string &libDirPath);

    // SMEM 基础 API
    static int32_t SmemInit(uint32_t flags);
    static void SmemUninit();
    static int32_t SmemSetExternLogger(void (*func)(int level, const char *msg));
    static int32_t SmemSetLogLevel(int level);
    static const char *SmemGetLastErrMsg();
    static const char *smem_get_and_clear_last_err_msg();

    // SMEM BM API
    static int32_t SmemBmConfigInit(smem_bm_config_t *config);
    static int32_t SmemBmInit(const char *storeURL, uint32_t worldSize, uint16_t deviceId,
                              const smem_bm_config_t *config);
    static void SmemBmUninit(uint32_t flags);
    static uint32_t SmemBmGetRankId(void);
    static smem_bm_t SmemBmCreate(uint32_t id, uint32_t memberSize, smem_bm_data_op_type dataOpType,
                                  uint64_t localDRAMSize, uint64_t localHBMSize, uint32_t flags);
    static void SmemBmDestroy(smem_bm_t handle);
    static int32_t SmemBmJoin(smem_bm_t handle, uint32_t flags);
    static int32_t SmemBmLeave(smem_bm_t handle, uint32_t flags);
    static void *SmemBmPtr(smem_bm_t handle, uint16_t peerRankId);
    static int32_t SmemBmCopy(smem_bm_t handle, const void *src, void *dest, uint64_t size,
                              smem_bm_copy_type t, uint32_t flags);

private:
    static int32_t GetLibPath(const std::string &libDir, std::string &outputPath);

    // 静态成员变量
    static std::mutex gMutex;
    static bool gLoaded;
    static void *gSmemHandle;
    static const char *gSmemLibName;

    // 函数指针
    static smemInitFunc gSmemInit;
    static smemUnInitFunc gSmemUnInit;
    // ... 其他函数指针
};
```

---

### smem_bm_api.cpp

**功能**: SMEM BM API 实现，负责动态加载和函数绑定。

**静态变量初始化**:
```cpp
bool MFSmemApi::gLoaded = false;
std::mutex MFSmemApi::gMutex;
void *MFSmemApi::gSmemHandle = nullptr;
const char *MFSmemApi::gSmemLibName = "libmf_smem.so";
```

**MFSmemApi::LoadLibrary**

```cpp
Result MFSmemApi::LoadLibrary(const std::string &libDirPath)
```

**功能**: 动态加载 `libmf_smem.so` 库并绑定所有函数指针

**参数**:
- `libDirPath`: 库文件所在目录路径，为空则使用默认搜索路径

**返回值**: `Result` - 成功返回 `MMC_OK`

**代码逻辑**:
1. 检查是否已加载，已加载则直接返回
2. 构建库文件完整路径
3. 使用 `dlopen()` 加载动态库
4. 使用 `DL_LOAD_SYM` 宏加载所有符号
5. 设置 `gLoaded` 为 true

**加载的符号列表**:
| 符号名 | 函数指针 | 说明 |
|--------|----------|------|
| `smem_init` | `gSmemInit` | 初始化 SMEM |
| `smem_uninit` | `gSmemUnInit` | 清理 SMEM |
| `smem_set_extern_logger` | `gSmemSetExternLogger` | 设置外部日志 |
| `smem_set_log_level` | `gSmemSetLogLevel` | 设置日志级别 |
| `smem_bm_config_init` | `gSmemBmConfigInit` | 初始化配置 |
| `smem_bm_init` | `gSmemBmInit` | 初始化 BM |
| `smem_bm_get_rank_id` | `gSmemBmGetRankId` | 获取 Rank ID |
| `smem_bm_create` | `gSmemBmCreate` | 创建 BM 对象 |
| `smem_bm_destroy` | `gSmemBmDestroy` | 销毁 BM 对象 |
| `smem_bm_join` | `gSmemBmJoin` | 加入全局空间 |
| `smem_bm_leave` | `gSmemBmLeave` | 离开全局空间 |
| `smem_bm_ptr` | `gSmemBmPtr` | 获取对端指针 |
| `smem_bm_copy` | `gSmemBmCopy` | 数据拷贝 |

---

## 数据流和关系

```
┌─────────────────────────────────────────────────────────────┐
│                    MFSmemApi                                │
│                 (SMEM API 封装)                              │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ dlopen/dlsym
                            ▼
                    ┌───────────────┐
                    │ libmf_smem.so │
                    │  (SMEM BM库)   │
                    └───────┬───────┘
                            │
                            │
                            ▼
                    ┌───────────────┐
                    │   RDMA/SDMA   │
                    │  网络传输层     │
                    └───────────────┘
```

---

## 使用示例

### 加载库并初始化

```cpp
#include "smem_bm_api.h"

// 加载动态库
Result ret = MFSmemApi::LoadLibrary("/opt/mf/lib");
if (ret != MMC_OK) {
    // 处理错误
}

// 初始化 SMEM
ret = MFSmemApi::SmemInit(0);
if (ret != 0) {
    const char *errMsg = MFSmemApi::SmemGetLastErrMsg();
    // 处理错误
}
```

### 创建 Big Memory 对象

```cpp
// 配置 BM
smem_bm_config_t config;
MFSmemApi::SmemBmConfigInit(&config);
config.startConfigStoreServer = false;

// 初始化 BM
ret = MFSmemApi::SmemBmInit("tcp://192.168.1.1:12345", 4, 0, &config);

// 获取 Rank ID
uint32_t rankId = MFSmemApi::SmemBmGetRankId();

// 创建 BM 对象
smem_bm_t bm = MFSmemApi::SmemBmCreate(1, 4, SMEMB_DATA_OP_SDMA,
                                        1024ULL * 1024 * 1024,  // 1GB DRAM
                                        0,                      // 0 HBM
                                        0);
if (bm == nullptr) {
    // 处理错误
}

// 加入全局空间
ret = MFSmemApi::SmemBmJoin(bm, 0);
```

### 数据拷贝

```cpp
// 获取对端地址
void *peerAddr = MFSmemApi::SmemBmPtr(bm, 1);  // Rank 1

// 本地数据
void *localData = malloc(1024);

// 拷贝: Local to Global
ret = MFSmemApi::SmemBmCopy(bm, localData, globalAddr, 1024, SMEMB_COPY_L2G, 0);

// 拷贝: Global to Local
ret = MFSmemApi::SmemBmCopy(bm, globalAddr, localData, 1024, SMEMB_COPY_G2L, 0);

// 拷贝: Global to Global
ret = MFSmemApi::SmemBmCopy(bm, srcGlobalAddr, dstGlobalAddr, 1024, SMEMB_COPY_G2G, 0);
```

### 清理资源

```cpp
// 离开全局空间
MFSmemApi::SmemBmLeave(bm, 0);

// 销毁 BM 对象
MFSmemApi::SmemBmDestroy(bm);

// 清理 SMEM
MFSmemApi::SmemBmUninit(0);
MFSmemApi::SmemUninit();
```

---

## API 映射表

| C++ API | 说明 |
|---------|------|
| `LoadLibrary()` | 加载动态库 |
| `SmemInit()` | 初始化 SMEM 环境 |
| `SmemUninit()` | 清理 SMEM 环境 |
| `SmemSetLogLevel()` | 设置日志级别 |
| `SmemSetExternLogger()` | 设置外部日志函数 |
| `SmemBmConfigInit()` | 初始化配置结构 |
| `SmemBmInit()` | 初始化 Big Memory |
| `SmemBmUninit()` | 清理 Big Memory |
| `SmemBmGetRankId()` | 获取 Rank ID |
| `SmemBmCreate()` | 创建 BM 对象 |
| `SmemBmDestroy()` | 销毁 BM 对象 |
| `SmemBmJoin()` | 加入全局空间 |
| `SmemBmLeave()` | 离开全局空间 |
| `SmemBmPtr()` | 获取对端内存地址 |
| `SmemBmCopy()` | 数据拷贝 |

---

## 错误处理

所有 API 函数返回错误码：
- `0`: 成功
- `非0`: 失败

获取错误消息：
```cpp
const char *errMsg = MFSmemApi::SmemGetLastErrMsg();
```
