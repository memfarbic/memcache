# mmc_env.h / mmc_env.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_env.h` / `src/memcache/csrc/common/mmc_env.cpp`
- **文件用途**: 声明和初始化环境变量配置，用于 MemCache 的运行时配置
- **依赖项**: `mmc_functions.h`, `<string>`

---

## 头文件 (mmc_env.h)

### 环境变量声明

```cpp
namespace ock {
namespace mmc {
extern std::string MMC_META_CONF_PATH;
extern std::string MMC_LOCAL_CONF_PATH;
extern std::string META_POD_NAME;
extern std::string META_NAMESPACE;
extern std::string META_LEASE_NAME;
}
}
```

**声明位置**: 行 17-23

**功能描述**: 声明全局环境变量字符串

**变量说明**:

| 变量名 | 用途 | 来源环境变量 |
|--------|------|--------------|
| `MMC_META_CONF_PATH` | 元数据配置文件路径 | `MMC_META_CONFIG_PATH` |
| `MMC_LOCAL_CONF_PATH` | 本地配置文件路径 | `MMC_LOCAL_CONFIG_PATH` |
| `META_POD_NAME` | 元数据 Pod 名称 | `META_POD_NAME` |
| `META_NAMESPACE` | 命名空间 | `META_NAMESPACE` |
| `META_LEASE_NAME` | 租约名称 | `META_LEASE_NAME` |

---

## 实现文件 (mmc_env.cpp)

### 环境变量初始化

```cpp
std::string MMC_META_CONF_PATH = SafeGetEnv("MMC_META_CONFIG_PATH");
std::string MMC_LOCAL_CONF_PATH = SafeGetEnv("MMC_LOCAL_CONFIG_PATH");
std::string META_POD_NAME = SafeGetEnv("META_POD_NAME");
std::string META_NAMESPACE = SafeGetEnv("META_NAMESPACE");
std::string META_LEASE_NAME = SafeGetEnv("META_LEASE_NAME");
```

**声明位置**: 行 20-24

**功能描述**: 程序启动时从环境变量中读取配置

**初始化时机**: 程序启动时（全局变量初始化阶段）

**函数调用**: 使用 `SafeGetEnv()` 安全获取环境变量（定义在 `mmc_functions.h`）

---

## 辅助函数 (来自 mmc_functions.h)

### SafeGetEnv()

```cpp
inline std::string SafeGetEnv(const char *name) noexcept
{
    const auto value = std::getenv(name);
    if (value == nullptr) {
        return "";
    }
    return value;
}
```

**声明位置**: mmc_functions.h 行 148-155

**功能描述**: 安全获取环境变量

**参数**:
- `name`: 环境变量名称

**返回值**: 环境变量值，如果不存在则返回空字符串

**特点**:
- 使用 `noexcept` 保证不会抛出异常
- 空指针安全

---

## 文件级别的关系图

```
mmc_env.h / mmc_env.cpp
    |
    +-- 全局环境变量
    |   +-- MMC_META_CONF_PATH    [元数据配置路径]
    |   +-- MMC_LOCAL_CONF_PATH   [本地配置路径]
    |   +-- META_POD_NAME         [Pod名称]
    |   +-- META_NAMESPACE        [命名空间]
    |   +-- META_LEASE_NAME       [租约名称]
    |
    +-- 依赖 mmc_functions.h
        +-- SafeGetEnv()  [安全获取环境变量]
```

---

## 使用示例

### 读取配置路径

```cpp
#include "mmc_env.h"

void LoadConfig() {
    if (!mmc::MMC_META_CONF_PATH.empty()) {
        LoadMetaConfig(mmc::MMC_META_CONF_PATH);
    }

    if (!mmc::MMC_LOCAL_CONF_PATH.empty()) {
        LoadLocalConfig(mmc::MMC_LOCAL_CONF_PATH);
    }
}
```

### 检查环境变量

```cpp
#include "mmc_env.h"

void PrintConfig() {
    std::cout << "Meta Config: " << ock::mmc::MMC_META_CONF_PATH << std::endl;
    std::cout << "Local Config: " << ock::mmc::MMC_LOCAL_CONF_PATH << std::endl;
    std::cout << "Pod Name: " << ock::mmc::META_POD_NAME << std::endl;
    std::cout << "Namespace: " << ock::mmc::META_NAMESPACE << std::endl;
    std::cout << "Lease Name: " << ock::mmc::META_LEASE_NAME << std::endl;
}
```

### 程序启动前设置环境变量

```bash
# Shell 环境下设置
export MMC_META_CONFIG_PATH=/etc/mmc/meta.conf
export MMC_LOCAL_CONFIG_PATH=/etc/mmc/local.conf
export META_POD_NAME=mmc-pod-0
export META_NAMESPACE=mmc-system
export META_LEASE_NAME=mmc-lease

# 然后运行程序
./mmc_application
```

---

## 环境变量说明

### MMC_META_CONFIG_PATH

- **类型**: 字符串（文件路径）
- **默认值**: 空字符串
- **用途**: 指定元数据服务器的配置文件路径
- **格式**: 绝对路径或相对路径

### MMC_LOCAL_CONFIG_PATH

- **类型**: 字符串（文件路径）
- **默认值**: 空字符串
- **用途**: 指定本地存储的配置文件路径
- **格式**: 绝对路径或相对路径

### META_POD_NAME

- **类型**: 字符串
- **默认值**: 空字符串
- **用途**: 在 Kubernetes 环境中标识当前 Pod 的名称
- **用途场景**: 分布式环境下的节点标识

### META_NAMESPACE

- **类型**: 字符串
- **默认值**: 空字符串
- **用途**: Kubernetes 命名空间
- **用途场景**: 多租户隔离

### META_LEASE_NAME

- **类型**: 字符串
- **默认值**: 空字符串
- **用途**: 租约标识符
- **用途场景**: 分布式锁、领导选举

---

## 注意事项

1. **初始化时机**: 环境变量在程序启动时读取，运行时修改环境变量不会影响这些变量
2. **空值处理**: 如果环境变量未设置，对应变量为空字符串，使用前应检查
3. **线程安全**: 这些是全局变量，读取操作是线程安全的，但不应在运行时修改
4. **命名空间**: 所有变量都在 `ock::mmc` 命名空间内
