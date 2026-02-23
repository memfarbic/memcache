# mmc_service.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/mmc_service.cpp`
- **文件用途**: 实现服务端管理 API (mmc_service.h)
- **依赖项**: `mmc_service.h`, `mmc_common_includes.h`, `mmc_meta_service.h`, `mmc_local_service_default.h`

---

## 命名空间

```cpp
using namespace ock::mmc;
```

**声明位置**: 行 17

**说明**: 使用 ock::mmc 命名空间，简化类型名称

---

## 函数

### mmcs_meta_service_start (第19-33行)

```cpp
MMC_API mmc_meta_service_t mmcs_meta_service_start(mmc_meta_service_config_t *config)
```

**声明位置**: 行 19-33

**功能描述**: 启动分布式内存缓存的元数据服务（全局服务）

**参数**:
- `config` [in]: 元数据服务配置

**返回值**:
- 成功: 返回元数据服务句柄（`MmcMetaService*`）
- 失败: 返回 `nullptr`

**代码逻辑**:

1. **参数验证** (第21行):
   ```cpp
   MMC_VALIDATE_RETURN(config != nullptr, "invalid param, config is nullptr", nullptr);
   ```
   - 使用宏验证 config 非空
   - 失败时记录错误并返回 nullptr

2. **创建服务对象** (第22-26行):
   ```cpp
   auto *serviceDefault = new (std::nothrow) MmcMetaService("meta_service");
   if (serviceDefault == nullptr) {
       MMC_LOG_AND_SET_LAST_ERROR("create or start meta service failed");
       return nullptr;
   }
   ```
   - 使用 `nothrow` new 创建服务对象
   - 创建失败时设置最后错误并返回

3. **启动服务** (第27-32行):
   ```cpp
   if (serviceDefault->Start(*config) == MMC_OK) {
       return serviceDefault;
   }
   delete serviceDefault;
   MMC_LOG_AND_SET_LAST_ERROR("create or start meta service failed");
   return nullptr;
   ```
   - 调用 `Start()` 方法启动服务
   - 成功则返回服务句柄
   - 失败则删除对象并返回 nullptr

**内存管理**:
- 成功时：调用者负责调用 `mmcs_meta_service_stop()` 释放
- 失败时：函数内部自动释放

---

### mmcs_meta_service_stop (第35-39行)

```cpp
MMC_API void mmcs_meta_service_stop(mmc_meta_service_t handle)
```

**声明位置**: 行 35-39

**功能描述**: 停止元数据服务

**参数**:
- `handle` [in]: 元数据服务句柄，由 `mmcs_meta_service_start` 创建

**返回值**: 无

**代码逻辑**:
```cpp
MMC_VALIDATE_RETURN_VOID(handle != nullptr, "invalid param, handle is nullptr");
static_cast<MmcMetaService *>(handle)->Stop();
```

1. 验证句柄非空
2. 将句柄转换为 `MmcMetaService*` 指针
3. 调用 `Stop()` 方法停止服务

**注意事项**:
- 停止后句柄失效，不应再使用
- 不释放内存（由 MmcMetaService 内部管理）

---

### mmcs_local_service_start (第41-55行)

```cpp
MMC_API mmc_local_service_t mmcs_local_service_start(mmc_local_service_config_t *config)
```

**声明位置**: 行 41-55

**功能描述**: 启动分布式内存缓存的本地服务

**参数**:
- `config` [in]: 本地服务配置

**返回值**:
- 成功: 返回本地服务句柄（`MmcLocalServiceDefault*`）
- 失败: 返回 `nullptr`

**代码逻辑**:

1. **参数验证** (第43行):
   ```cpp
   MMC_VALIDATE_RETURN(config != nullptr, "invalid param, config is nullptr", nullptr);
   ```

2. **创建服务对象** (第44-48行):
   ```cpp
   auto *serviceDefault = new (std::nothrow) MmcLocalServiceDefault("local_service");
   if (serviceDefault == nullptr) {
       MMC_LOG_AND_SET_LAST_ERROR("create or start local service failed");
       return nullptr;
   }
   ```

3. **启动服务** (第49-54行):
   ```cpp
   if (serviceDefault->Start(*config) == MMC_OK) {
       return serviceDefault;
   }
   delete serviceDefault;
   MMC_LOG_AND_SET_LAST_ERROR("create or start local service failed");
   return nullptr;
   ```

**与元数据服务的区别**:
- 本地服务管理本地内存资源
- 每个节点都需要启动本地服务
- 与 Blob Manager 交互进行内存分配

---

### mmcs_local_service_stop (第57-66行)

```cpp
MMC_API void mmcs_local_service_stop(mmc_local_service_t handle)
```

**声明位置**: 行 57-66

**功能描述**: 停止本地服务

**参数**:
- `handle` [in]: 本地服务句柄，由 `mmcs_local_service_start` 创建

**返回值**: 无

**代码逻辑**:
```cpp
MMC_VALIDATE_RETURN_VOID(handle != nullptr, "invalid param, handle is nullptr");
auto service_default = static_cast<MmcLocalServiceDefault *>(handle);
if (service_default != nullptr) {
    service_default->Stop();
    delete service_default;
    service_default = nullptr;
}
```

1. 验证句柄非空
2. 转换为 `MmcLocalServiceDefault*` 指针
3. 调用 `Stop()` 停止服务
4. **删除服务对象**（与元数据服务不同）
5. 将指针置空

**内存管理**:
- 本地服务需要显式 delete
- 元数据服务由引用计数管理

---

## 宏展开说明

### MMC_VALIDATE_RETURN

```cpp
#define MMC_VALIDATE_RETURN(expression, msg, returnValue) \
    do {                                                  \
        if (UNLIKELY(!(expression))) {                    \
            MMC_SET_LAST_ERROR(msg);                      \
            MMC_LOG_ERROR(msg);                           \
            return returnValue;                           \
        }                                                 \
    } while (0)
```

用于验证表达式，失败时设置错误、记录日志并返回值。

### MMC_VALIDATE_RETURN_VOID

```cpp
#define MMC_VALIDATE_RETURN_VOID(expression, msg) \
    do {                                          \
        if (UNLIKELY(!(expression))) {            \
            MMC_SET_LAST_ERROR(msg);              \
            MMC_LOG_ERROR(msg);                   \
            return;                               \
        }                                         \
    } while (0)
```

类似 `MMC_VALIDATE_RETURN`，但无返回值。

### MMC_LOG_AND_SET_LAST_ERROR

```cpp
#define MMC_LOG_AND_SET_LAST_ERROR(msg)  \
    do {                                 \
        std::stringstream tmpStr;        \
        tmpStr << msg;                   \
        MmcLastError::Set(tmpStr.str()); \
        MMC_LOG_ERROR(tmpStr.str());     \
    } while (0)
```

同时设置最后错误和记录错误日志。

---

## 服务启动流程

### 元数据服务启动流程

```
用户调用 mmcs_meta_service_start()
    |
    v
验证配置参数
    |
    v
创建 MmcMetaService 对象
    |
    v
调用 service->Start()
    |
    +-- 初始化网络服务器
    +-- 初始化元数据代理
    +-- 初始化备份管理器
    +-- 注册信号处理
    |
    v
返回服务句柄
```

### 本地服务启动流程

```
用户调用 mmcs_local_service_start()
    |
    v
验证配置参数
    |
    v
创建 MmcLocalServiceDefault 对象
    |
    v
调用 service->Start()
    |
    +-- 初始化网络客户端
    +-- 初始化 Blob Manager 代理
    +-- 注册到元数据服务
    +-- 初始化本地内存
    |
    v
返回服务句柄
```

---

## 服务架构

```
┌──────────────────────────────────────────────────────┐
│                   MemCache 集群                        │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌────────────────────────────────────────────┐     │
│  │          元数据服务 (全局唯一)                │     │
│  │  - 服务发现                                   │     │
│  │  - 元数据管理                                 │     │
│  │  - 分配决策                                   │     │
│  └────────────────────────────────────────────┘     │
│                       ↑                              │
│                       │                              │
│          ┌────────────┼────────────┐                │
│          │            │            │                │
│  ┌───────▼────┐ ┌────▼─────┐ ┌────▼─────┐          │
│  │ 本地服务1   │ │ 本地服务2 │ │ 本地服务N │          │
│  │- Rank 0    │ │- Rank 1  │ │- Rank N  │          │
│  │- BM 管理    │ │- BM 管理  │ │- BM 管理  │          │
│  └────────────┘ └──────────┘ └──────────┘          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 文件级别的关系图

```
mmc_service.cpp (服务 API 实现)
    |
    +-- 依赖: mmc_service.h (C 接口定义)
    +-- 依赖: mmc_meta_service.h (元数据服务类)
    +-- 依赖: mmc_local_service_default.h (本地服务类)
    +-- 依赖: mmc_common_includes.h (公共定义)
    |
    +-- 创建: MmcMetaService
    +-- 创建: MmcLocalServiceDefault
```

---

## 使用示例

```c
#include "mmc_service.h"

// 启动元数据服务
mmc_meta_service_config_t meta_config = {/* ... */};
mmc_meta_service_t meta = mmcs_meta_service_start(&meta_config);
if (meta == nullptr) {
    printf("Failed to start meta service\n");
    return -1;
}

// 启动本地服务
mmc_local_service_config_t local_config = {/* ... */};
mmc_local_service_t local = mmcs_local_service_start(&local_config);
if (local == nullptr) {
    printf("Failed to start local service\n");
    mmcs_meta_service_stop(meta);
    return -1;
}

// 使用服务...

// 清理
mmcs_local_service_stop(local);  // 会 delete 对象
mmcs_meta_service_stop(meta);    // 不 delete，由内部管理
```

---

## 错误处理

所有函数都使用统一的错误处理机制：

1. **参数验证失败**: 设置错误消息并返回错误值
2. **对象创建失败**: 记录日志并返回 nullptr
3. **启动失败**: 清理已创建的对象并返回 nullptr

错误信息可通过 `MmcLastError::GetAndClear()` 获取。
