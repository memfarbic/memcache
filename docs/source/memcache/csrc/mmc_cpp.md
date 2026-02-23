# mmc.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/mmc.cpp`
- **文件用途**: 实现 MemCache 系统的主初始化和配置接口 (mmc.h)
- **依赖项**: `mmc.h`, `mmc_client.h`, `mmc_common_includes.h`, `mmc_configuration.h`, `mmc_def.h`, `mmc_env.h`, `mmc_service.h`, `mmc_thread_pool.h`

---

## 全局变量

### g_localService (第23行)

```cpp
static mmc_local_service_t g_localService;
```

**声明位置**: 行 23

**功能描述**: 全局本地服务句柄，用于跟踪当前启动的本地服务

**作用域**: 文件内部 (static)

---

### gMmcMutex (第25行)

```cpp
static std::mutex gMmcMutex;
```

**声明位置**: 行 25

**功能描述**: 全局互斥锁，保护 MMC 初始化/反初始化过程

---

### mmcInit (第26行)

```cpp
static bool mmcInit = false;
```

**声明位置**: 行 26

**功能描述**: MMC 初始化状态标志

---

## 函数

### mmc_init (第28-86行)

```cpp
MMC_API int32_t mmc_init(const mmc_init_config *config)
```

**声明位置**: 行 28-86

**功能描述**: 初始化 memcache 客户端和本地服务

**参数**:
- `config` [in]: 初始化配置，指向 `mmc_init_config` 结构体

**返回值**:
- `MMC_OK (0)`: 成功
- 其他值: 失败

**代码逻辑**:
1. **设置进程优先级** (第30-31行):
   - 将进程 nice 值设为 -5，提高进程优先级

2. **参数验证** (第33-34行):
   - 验证 config 非空
   - 验证 deviceId 有效 (<= MAX_DEVICE_ID)

3. **检查重复初始化** (第35-39行):
   - 使用互斥锁保护
   - 如果已初始化则直接返回成功

4. **加载配置文件** (第41-52行):
   - 从 `MMC_LOCAL_CONF_PATH` 环境变量指定路径加载配置
   - 验证配置文件内容有效性
   - 配置错误时返回 `MMC_INVALID_PARAM`

5. **准备本地服务配置** (第54-60行):
   - 从配置管理器获取本地服务配置
   - 验证本地服务配置

6. **设置日志** (第62-66行):
   - 设置日志级别
   - 如果配置了外部日志函数则使用

7. **启动本地服务** (第68-72行):
   - 如果 `initBm=true`，启动本地服务
   - 本地服务管理本地内存资源

8. **初始化客户端 SDK** (第74-83行):
   - 获取客户端配置
   - 调用 `mmcc_init()` 初始化客户端
   - 失败时清理已启动的本地服务

9. **设置初始化标志** (第84行)

**调用关系**:
- 调用: `mmcs_local_service_start()`, `mmcc_init()`
- 被调用: 用户代码

---

### mmc_set_extern_logger (第88-95行)

```cpp
MMC_API int32_t mmc_set_extern_logger(void (*func)(int level, const char *msg))
```

**声明位置**: 行 88-95

**功能描述**: 设置外部日志函数

**参数**:
- `func` [in]: 外部日志函数指针

**返回值**:
- `MMC_OK (0)`: 成功
- `MMC_INVALID_PARAM`: func 为 nullptr

**代码逻辑**:
1. 验证函数指针非空
2. 设置到 `MmcOutLogger` 单例

---

### mmc_set_log_level (第97-104行)

```cpp
MMC_API int32_t mmc_set_log_level(int level)
```

**声明位置**: 行 97-104

**功能描述**: 设置日志打印级别

**参数**:
- `level` [in]: 日志级别
  - `0`: DEBUG
  - `1`: INFO
  - `2`: WARN
  - `3`: ERROR

**返回值**:
- `MMC_OK (0)`: 成功
- `MMC_INVALID_PARAM`: level 无效

**代码逻辑**:
1. 验证 level 在有效范围内
2. 设置到 `MmcOutLogger` 单例

---

### mmc_uninit (第106-120行)

```cpp
MMC_API void mmc_uninit(void)
```

**声明位置**: 行 106-120

**功能描述**: 反初始化 smem 运行环境

**参数**: 无

**返回值**: 无

**代码逻辑**:
1. **获取锁** (第108行): 保护反初始化过程

2. **检查初始化状态** (第109-112行):
   - 如果未初始化则直接返回

3. **停止本地服务** (第114-117行):
   - 如果本地服务存在则停止
   - 将句柄置为 nullptr

4. **反初始化客户端** (第118行):
   - 调用 `mmcc_uninit()`

5. **清除初始化标志** (第119行)

**注意事项**: 与 `mmc_init()` 必须配对使用

---

## 初始化流程图

```
用户调用 mmc_init()
    |
    v
设置进程优先级 (-5)
    |
    v
加载配置文件 (MMC_LOCAL_CONF_PATH)
    |
    v
验证配置
    |
    v
<---------------------+
|  initBm == true?     |
+---------------------+
    | Yes              | No
    v                  v
启动本地服务      跳过本地服务
    |
    v
初始化客户端 SDK (mmcc_init)
    |
    v
设置 mmcInit = true
    |
    v
返回 MMC_OK
```

---

## 反初始化流程图

```
用户调用 mmc_uninit()
    |
    v
获取互斥锁
    |
    v
检查初始化状态
    |
    v
<---------------------+
|  本地服务存在?       |
+---------------------+
    | Yes              | No
    v                  |
停止本地服务           |
    |                  |
    v                  v
反初始化客户端 (mmcc_uninit)
    |
    v
清除初始化标志
    |
    v
释放锁
```

---

## 文件级别的关系图

```
mmc.cpp (主初始化实现)
    |
    +-- 依赖: mmc_client.h (客户端接口)
    +-- 依赖: mmc_service.h (服务接口)
    +-- 依赖: mmc_configuration.h (配置管理)
    +-- 依赖: mmc_env.h (环境变量)
    |
    +-- 调用: mmcs_local_service_start()
    +-- 调用: mmcc_init()
    +-- 调用: MmcOutLogger::Instance()
```

---

## 使用示例

```c
#include "mmc.h"

int main() {
    // 初始化配置
    mmc_init_config config = {
        .deviceId = 0,
        .initBm = true
    };

    // 初始化
    int ret = mmc_init(&config);
    if (ret != 0) {
        printf("mmc_init failed: %d\n", ret);
        return -1;
    }

    // 设置日志级别
    mmc_set_log_level(1); // INFO

    // 使用 MemCache...
    // mmcc_put(...);
    // mmcc_get(...);

    // 清理
    mmc_uninit();
    return 0;
}
```
