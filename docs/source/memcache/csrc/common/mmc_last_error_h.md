# mmc_last_error.h / mmc_last_error.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_last_error.h` / `src/memcache/csrc/common/mmc_last_error.cpp`
- **文件用途**: 提供线程局部的最后错误消息存储和获取机制
- **依赖项**: `<string>`

---

## 头文件 (mmc_last_error.h)

### MmcLastError

```cpp
class MmcLastError {
public:
    static void Set(const std::string &msg);
    static void Set(const char *msg);
    static const char *GetAndClear(bool clear);

private:
    static thread_local bool have_;
    static thread_local std::string msg_;
};
```

**声明位置**: 行 19-45

**功能描述**: 线程局部的错误消息管理类

**特性**:
- 线程安全（使用 `thread_local` 存储）
- 每个线程有独立的错误消息
- 支持设置、获取和清除错误消息

---

#### MmcLastError::Set(string)

```cpp
static void Set(const std::string &msg);
```

**声明位置**: 行 26

**功能描述**: 设置最后的错误消息（字符串版本）

**参数**:
- `msg`: 错误消息字符串

**实现**:
```cpp
inline void MmcLastError::Set(const std::string &msg)
{
    msg_ = msg;
    have_ = true;
}
```

**代码逻辑**:
1. 将消息保存到线程局部存储
2. 设置错误标志为 true

---

#### MmcLastError::Set(const char*)

```cpp
static void Set(const char *msg);
```

**声明位置**: 行 33

**功能描述**: 设置最后的错误消息（C 字符串版本）

**参数**:
- `msg`: 错误消息 C 字符串

**实现**:
```cpp
inline void MmcLastError::Set(const char *msg)
{
    msg_ = msg;
    have_ = true;
}
```

**代码逻辑**:
1. 将 C 字符串转换为 `std::string` 并保存
2. 设置错误标志为 true

---

#### MmcLastError::GetAndClear()

```cpp
static const char *GetAndClear(bool clear);
```

**声明位置**: 行 40

**功能描述**: 获取并可选地清除最后的错误消息

**参数**:
- `clear`: 是否清除错误消息（true 清除，false 保留）

**返回值**: 错误消息的 C 字符串，如果没有错误则返回空字符串

**实现**:
```cpp
inline const char *MmcLastError::GetAndClear(bool clear)
{
    /* have last error, just set the flag to false */
    if (have_) {
        have_ = !clear;
        return msg_.c_str();
    }

    /* empty string */
    static std::string emptyString;

    return emptyString.c_str();
}
```

**代码逻辑**:
1. 如果有错误消息：
   - 如果 `clear` 为 true，清除错误标志
   - 返回错误消息
2. 如果没有错误消息：
   - 返回静态空字符串

---

#### MmcLastError 成员变量

```cpp
private:
    static thread_local bool have_;       /* thread local flag that indicates if there is last error */
    static thread_local std::string msg_; /* last error message */
```

**声明位置**: 行 43-44

**功能描述**: 线程局部的错误状态存储

**说明**:
- `have_`: 标识是否有错误消息
- `msg_`: 存储错误消息内容
- 使用 `thread_local` 确保每个线程有独立的副本

---

## 实现文件 (mmc_last_error.cpp)

### 静态成员定义

```cpp
thread_local bool MmcLastError::have_ = false;
thread_local std::string MmcLastError::msg_;
```

**声明位置**: 行 16-17

**功能描述**: 定义线程局部静态成员变量

**说明**:
- C++11 要求 `thread_local` 静态成员必须在 .cpp 文件中定义
- 初始状态：无错误（`have_ = false`）

---

## 文件级别的关系图

```
mmc_last_error.h / mmc_last_error.cpp
    |
    +-- MmcLastError (错误管理类)
    |   |
    |   +-- Set()          [设置错误消息]
    |   |   +-- Set(string)
    |   |   +-- Set(const char*)
    |   |
    |   +-- GetAndClear()  [获取并可选清除错误消息]
    |   |
    |   +-- thread_local 成员变量
    |       +-- have_   [错误标志]
    |       +-- msg_    [错误消息]
```

---

## 使用示例

### 基本用法

```cpp
#include "mmc_last_error.h"

using namespace ock::mmc;

void SomeFunction() {
    if (error_occurred) {
        MmcLastError::Set("Failed to allocate memory");
        return;
    }
}

// 调用后检查错误
SomeFunction();
const char* error = MmcLastError::GetAndClear(true);
if (strlen(error) > 0) {
    std::cerr << "Error: " << error << std::endl;
}
```

### 配合宏使用

```cpp
// 使用 MMC_LOG_AND_SET_LAST_ERROR 宏
MMC_LOG_AND_SET_LAST_ERROR("Connection failed: " << error_code);

// 使用 MMC_VALIDATE_RETURN 宏
MMC_VALIDATE_RETURN(ptr != nullptr, "Null pointer", MMC_INVALID_PARAM);
```

### 保留错误消息

```cpp
// 获取错误但不清除
const char* error = MmcLastError::GetAndClear(false);
// 可以再次获取
const char* sameError = MmcLastError::GetAndClear(false);
// 现在清除
const char* error2 = MmcLastError::GetAndClear(true);  // 这会清除
```

### 多线程环境

```cpp
void ThreadFunc(int id) {
    // 每个线程有独立的错误消息
    MmcLastError::Set("Error in thread " + std::to_string(id));

    // 其他线程的错误不会影响当前线程
    const char* error = MmcLastError::GetAndClear(true);
    // error 是当前线程的错误消息
}

std::thread t1(ThreadFunc, 1);
std::thread t2(ThreadFunc, 2);
```

---

## 与其他组件的配合

### 与 mmc_define.h 宏配合

```cpp
// 这些宏内部使用 MmcLastError::Set()
MMC_LOG_AND_SET_LAST_ERROR(msg)
MMC_SET_LAST_ERROR(msg)
MMC_COUT_AND_SET_LAST_ERROR(msg)
MMC_VALIDATE_RETURN(expression, msg, returnValue)
MMC_VALIDATE_RETURN_VOID(expression, msg)
```

---

## 注意事项

1. **线程安全**: 使用 `thread_local` 确保每个线程有独立的错误存储
2. **内存管理**: 返回的 C 字符串指针在下次调用 `Set()` 或线程结束前有效
3. **空字符串**: 无错误时返回指向静态空字符串的指针
4. **清除操作**: `GetAndClear(true)` 只清除标志，不释放消息内容
5. **指针有效期**: 返回的指针在下次调用 `Set()` 前有效，之后可能失效
6. **非全局**: 错误消息不是全局的，每个线程有独立的错误状态
