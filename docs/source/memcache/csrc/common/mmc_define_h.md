# mmc_define.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_define.h`
- **文件用途**: 定义 MemCache 项目使用的核心宏定义和常量，包括编译优化宏、错误处理宏、API 导出宏等
- **依赖项**: 无（基础头文件）

---

## 宏定义

### MMC_DATA_TTL_MS

```cpp
#define MMC_DATA_TTL_MS 2000
```

**说明**: 数据的默认生存时间（Time To Live），单位为毫秒，默认值为 2000ms（2秒）

---

### MMC_THRESHOLD_PRINT_SECONDS

```cpp
#define MMC_THRESHOLD_PRINT_SECONDS 30
```

**说明**: 打印日志的时间阈值，单位为秒，默认值为 30 秒

---

### MMC_THREAD_POOL_MAX_THREADS

```cpp
#define MMC_THREAD_POOL_MAX_THREADS 1024
```

**说明**: 线程池允许的最大线程数量，默认值为 1024

---

### LIKELY / UNLIKELY

```cpp
#ifndef LIKELY
#define LIKELY(x) (__builtin_expect(!!(x), 1) != 0)
#endif

#ifndef UNLIKELY
#define UNLIKELY(x) (__builtin_expect(!!(x), 0) != 0)
#endif
```

**说明**: GCC 分支预测优化宏
- `LIKELY(x)`: 告诉编译器表达式 `x` 很可能为真
- `UNLIKELY(x)`: 告诉编译器表达式 `x` 很可能为假

**用途**: 用于优化 if/else 分支预测，提高代码执行效率

**使用示例**:
```cpp
if (UNLIKELY(error_code != 0)) {
    // 错误处理路径（不常执行）
}
// 正常路径（常执行）
```

---

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

**声明位置**: 行 35-41

**功能描述**: 设置最后的错误消息并同时记录错误日志

**参数**:
- `msg`: 要设置和记录的错误消息（支持流式输出）

**代码逻辑**:
1. 创建一个 `std::stringstream` 临时对象
2. 将消息写入流中
3. 调用 `MmcLastError::Set()` 设置最后错误
4. 调用 `MMC_LOG_ERROR()` 记录错误日志

**使用示例**:
```cpp
MMC_LOG_AND_SET_LAST_ERROR("Failed to allocate memory, size=" << size);
```

**注意事项**: 使用 `do { ... } while (0)` 包装确保宏在使用时像语句一样安全

---

### MMC_SET_LAST_ERROR

```cpp
#define MMC_SET_LAST_ERROR(msg)          \
    do {                                 \
        std::stringstream tmpStr;        \
        tmpStr << msg;                   \
        MmcLastError::Set(tmpStr.str()); \
    } while (0)
```

**声明位置**: 行 48-53

**功能描述**: 仅设置最后的错误消息，不记录日志

**参数**:
- `msg`: 要设置的错误消息（支持流式输出）

**使用场景**: 当只需要设置错误但不需日志记录时使用

---

### MMC_COUT_AND_SET_LAST_ERROR

```cpp
#define MMC_COUT_AND_SET_LAST_ERROR(msg) \
    do {                                 \
        std::stringstream tmpStr;        \
        tmpStr << msg;                   \
        MmcLastError::Set(tmpStr.str()); \
        std::cout << msg << std::endl;   \
    } while (0)
```

**声明位置**: 行 60-66

**功能描述**: 设置最后的错误消息并输出到标准输出

**参数**:
- `msg`: 要设置和打印的错误消息（支持流式输出）

**使用场景**: 调试或需要直接输出到终端的场景

---

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

**声明位置**: 行 78-85

**功能描述**: 验证表达式，如果表达式为假则：
1. 设置最后的错误消息
2. 记录错误日志
3. 返回指定的值

**参数**:
- `expression`: 要验证的表达式
- `msg`: 验证失败时的错误消息
- `returnValue`: 验证失败时的返回值

**代码逻辑**:
1. 使用 `UNLIKELY` 包装表达式（假设验证通常成功）
2. 如果表达式为假，执行错误处理并返回

**使用示例**:
```cpp
MMC_VALIDATE_RETURN(ptr != nullptr, "Null pointer detected", MMC_INVALID_PARAM);
```

---

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

**声明位置**: 行 96-103

**功能描述**: 验证表达式，如果表达式为假则：
1. 设置最后的错误消息
2. 记录错误日志
3. 直接返回（无返回值）

**参数**:
- `expression`: 要验证的表达式
- `msg`: 验证失败时的错误消息

**使用示例**:
```cpp
MMC_VALIDATE_RETURN_VOID(initialized_, "Module not initialized");
```

---

### MMC_API

```cpp
#define MMC_API __attribute__((visibility("default")))
```

**声明位置**: 行 105

**功能描述**: GCC/Clang 属性宏，用于标记符号为动态可见

**用途**: 控制符号在动态库中的可见性，`visibility("default")` 表示该符号可被动态库外部访问

**使用场景**: 标记需要导出的公共 API 函数或类

---

### SAFE_DELETE

```cpp
#define SAFE_DELETE(p) \
    do {               \
        delete (p);    \
        p = nullptr;   \
    } while (0)
```

**声明位置**: 行 112-116

**功能描述**: 安全删除指针，删除后将指针置为 nullptr

**参数**:
- `p`: 要删除的指针

**代码逻辑**:
1. 调用 `delete` 释放指针指向的内存
2. 将指针置为 `nullptr` 防止悬空指针

**使用示例**:
```cpp
int* ptr = new int(42);
SAFE_DELETE(ptr);  // ptr 现在是 nullptr
```

**注意事项**:
- 只适用于用 `new` 分配的单个对象
- 不适用于数组（应使用 `delete[]`）
- 删除后指针自动置空，避免重复删除

---

## 文件级别的关系图

```
mmc_define.h (基础宏定义)
    |
    +-- 被几乎所有其他头文件包含
    |
    +-- LIKELY/UNLIKELY -> 用于分支预测优化
    +-- MMC_*_LAST_ERROR -> 用于错误处理
    +-- MMC_VALIDATE_* -> 用于参数验证
    +-- SAFE_DELETE -> 用于内存管理
```

---

## 依赖关系

**被以下文件依赖**:
- `mmc_logger.h`
- `mmc_types.h`
- `mmc_functions.h`
- 以及几乎所有其他 common 模块的源文件

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有宏定义都在 ock::mmc 命名空间内定义
}
}
```
