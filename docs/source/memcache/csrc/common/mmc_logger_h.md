# mmc_logger.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_logger.h`
- **文件用途**: 提供 MemCache 项目的日志记录功能，支持多级别日志、外部日志函数注入、审计日志等
- **依赖项**: `mmc_define.h`, `<ctime>`, `<cstring>`, `<iostream>`, `<mutex>`, `<unistd.h>`, `<sstream>`, `<sys/time.h>`, `<sys/syscall.h>`

---

## 宏定义

### OBJ_MAX_LOG_FILE_SIZE

```cpp
#define OBJ_MAX_LOG_FILE_SIZE 20971520 // 每个日志文件的最大大小
```

**说明**: 单个日志文件的最大大小限制（20MB）

---

### OBJ_MAX_LOG_FILE_NUM

```cpp
#define OBJ_MAX_LOG_FILE_NUM 50
```

**说明**: 最大日志文件数量

---

### MICROSECOND_WIDTH

```cpp
constexpr int MICROSECOND_WIDTH = 6;
```

**说明**: 微秒显示宽度（6位数字）

---

### PID_TID

```cpp
#define PID_TID " [" << getpid() << "-" << syscall(SYS_gettid) << "]"
```

**声明位置**: 行 27

**功能描述**: 生成包含进程 ID 和线程 ID 的日志前缀

**输出格式**: `[PID-TID]`

---

## 类型别名

### ExternalLog

```cpp
using ExternalLog = void (*)(int, const char *);
```

**声明位置**: 行 35

**功能描述**: 外部日志函数类型指针

**参数**:
- `int`: 日志级别
- `const char*`: 日志内容

---

### ExternalAuditLog

```cpp
using ExternalAuditLog = void (*)(const char *);
```

**声明位置**: 行 36

**功能描述**: 外部审计日志函数类型指针

**参数**:
- `const char*`: 审计日志内容

---

## 枚举定义

### LogLevel

```cpp
enum LogLevel : int {
    DEBUG_LEVEL = 0,  // 调试级别
    INFO_LEVEL,       // 信息级别
    WARN_LEVEL,       // 警告级别
    ERROR_LEVEL,      // 错误级别
    BUTT_LEVEL        // 边界值（无实际用途）
};
```

**声明位置**: 行 38-44

**功能描述**: 定义日志级别枚举

**级别关系**: DEBUG < INFO < WARN < ERROR

---

## 类定义

### MmcOutLogger

```cpp
class MmcOutLogger {
    // ... (详见下文)
};
```

**声明位置**: 行 46-169

**功能描述**: 单例模式的日志记录器，支持日志级别控制、外部日志函数注入等功能

---

#### MmcOutLogger::Instance()

```cpp
static MmcOutLogger &Instance()
{
    static MmcOutLogger gLogger;
    return gLogger;
}
```

**声明位置**: 行 48-52

**功能描述**: 获取日志记录器的单例实例

**返回值**: 日志记录器的引用

**实现方式**: 使用静态局部变量（Meyers Singleton）实现线程安全的单例

---

#### MmcOutLogger::GetLogLevel()

```cpp
inline LogLevel GetLogLevel() const
{
    return logLevel_;
}
```

**声明位置**: 行 54-57

**功能描述**: 获取当前日志级别

**返回值**: 当前日志级别

---

#### MmcOutLogger::GetLogExtraFunc()

```cpp
inline ExternalLog GetLogExtraFunc() const
{
    return logFunc_;
}
```

**声明位置**: 行 59-62

**功能描述**: 获取外部日志函数指针

**返回值**: 外部日志函数指针

---

#### MmcOutLogger::GetAuditLogExtraFunc()

```cpp
inline ExternalAuditLog GetAuditLogExtraFunc() const
{
    return auditLogFunc_;
}
```

**声明位置**: 行 64-67

**功能描述**: 获取外部审计日志函数指针

**返回值**: 外部审计日志函数指针

---

#### MmcOutLogger::SetLogLevel()

```cpp
inline int32_t SetLogLevel(LogLevel level)
{
    if (level < DEBUG_LEVEL || level >= BUTT_LEVEL) {
        return -1;
    }
    logLevel_ = level;
    return 0;
}
```

**声明位置**: 行 69-76

**功能描述**: 设置日志级别

**参数**:
- `level`: 要设置的日志级别

**返回值**: 成功返回 0，失败返回 -1

**代码逻辑**:
1. 检查级别是否在有效范围内
2. 如果有效，更新日志级别
3. 返回操作结果

---

#### MmcOutLogger::SetExternalLogFunction()

```cpp
inline void SetExternalLogFunction(ExternalLog func, bool forceUpdate = false)
{
    if (logFunc_ == nullptr || forceUpdate) {
        logFunc_ = func;
    }
}
```

**声明位置**: 行 78-83

**功能描述**: 设置外部日志处理函数

**参数**:
- `func`: 外部日志函数指针
- `forceUpdate`: 是否强制更新（默认 false）

**代码逻辑**:
- 只有当 `logFunc_` 为空或 `forceUpdate` 为 true 时才设置
- 防止意外覆盖已设置的外部日志函数

---

#### MmcOutLogger::SetExternalAuditLogFunction()

```cpp
inline void SetExternalAuditLogFunction(ExternalAuditLog func, bool forceUpdate = false)
{
    if (auditLogFunc_ == nullptr || forceUpdate) {
        auditLogFunc_ = func;
    }
}
```

**声明位置**: 行 85-90

**功能描述**: 设置外部审计日志处理函数

**参数**:
- `func`: 外部审计日志函数指针
- `forceUpdate`: 是否强制更新（默认 false）

---

#### MmcOutLogger::Log()

```cpp
inline void Log(int level, const std::ostringstream &oss)
{
    if (level < logLevel_) {
        return;
    }

#ifndef UT_ENABLED
    if (logFunc_ != nullptr) {
        logFunc_(level, oss.str().c_str());
        return;
    }

    struct timeval tv{};
    char strTime[24];

    gettimeofday(&tv, nullptr);
    time_t timeStamp = tv.tv_sec;
    struct tm localTime{};
    if (strftime(strTime, sizeof strTime, "%Y-%m-%d %H:%M:%S.", localtime_r(&timeStamp, &localTime)) != 0) {
        std::cout << strTime << std::setw(MICROSECOND_WIDTH) << std::setfill('0') << tv.tv_usec << " "
                  << LogLevelDesc(level) << PID_TID << oss.str() << std::endl;
    } else {
        std::cout << " Invalid time " << LogLevelDesc(level) << PID_TID << oss.str() << std::endl;
    }
#else
    std::cout << LogLevelDesc(level) << oss.str() << std::endl;
#endif
}
```

**声明位置**: 行 92-119

**功能描述**: 记录日志

**参数**:
- `level`: 日志级别
- `oss`: 包含日志内容的字符串流

**代码逻辑**:
1. 检查日志级别是否满足要求
2. 如果设置了外部日志函数，调用外部函数
3. 否则，格式化输出到标准输出
   - 获取当前时间（秒 + 微秒）
   - 格式化时间字符串
   - 输出：时间 + 日志级别 + PID-TID + 日志内容

**输出格式**: `YYYY-MM-DD HH:MM:SS.uuuuuu LEVEL [PID-TID] message`

**注意事项**: UT 模式下简化输出格式

---

#### MmcOutLogger::AuditLog()

```cpp
inline void AuditLog(const std::ostringstream &oss)
{
    if (auditLogFunc_ != nullptr) {
        auditLogFunc_(oss.str().c_str());
        return;
    }
}
```

**声明位置**: 行 121-127

**功能描述**: 记录审计日志

**参数**:
- `oss`: 包含审计日志内容的字符串流

**代码逻辑**:
- 如果设置了外部审计日志函数，调用外部函数
- 否则什么都不做

---

#### MmcOutLogger::GetLogLevel(string)

```cpp
int32_t GetLogLevel(const std::string &logLevelDesc) const
{
    for (uint32_t count = DEBUG_LEVEL; count < BUTT_LEVEL; count++) {
        if (logLevelDesc == logLevelDesc_[count]) {
            return count;
        }
    }
    // 没有匹配到日志级别，使用默认级别INFO
    return INFO_LEVEL;
}
```

**声明位置**: 行 129-138

**功能描述**: 从字符串描述获取日志级别枚举值

**参数**:
- `logLevelDesc`: 日志级别字符串（如 "DEBUG", "INFO"）

**返回值**: 对应的日志级别枚举，未匹配时返回 INFO_LEVEL

---

#### MmcOutLogger::LogLevelDesc()

```cpp
private:
const char *LogLevelDesc(const int level) const
{
    const static std::string invalid = "invalid";
    if (UNLIKELY(level < DEBUG_LEVEL || level >= BUTT_LEVEL)) {
        return invalid.c_str();
    }
    return logLevelDesc_[level];
}
```

**声明位置**: 行 154-161

**功能描述**: 获取日志级别的字符串描述（私有方法）

**参数**:
- `level`: 日志级别枚举值

**返回值**: 日志级别字符串（"DEBUG"/"INFO"/"WARN"/"ERROR"）

---

### MmcOutLogger 成员变量

```cpp
private:
    LogLevel logLevel_ = INFO_LEVEL;                      // 当前日志级别
    ExternalLog logFunc_ = nullptr;                       // 外部日志函数
    ExternalAuditLog auditLogFunc_ = nullptr;             // 外部审计日志函数
    const char *logLevelDesc_[BUTT_LEVEL] = {"DEBUG", "INFO", "WARN", "ERROR"}; // 级别描述
```

---

## 日志宏定义

### MMC_LOG_FILENAME_SHORT

```cpp
#define MMC_LOG_FILENAME_SHORT (strrchr(__FILE__, '/') ? strrchr(__FILE__, '/') + 1 : __FILE__)
```

**声明位置**: 行 174

**功能描述**: 获取文件名（不含路径）

---

### MMC_LOG_FORMAT

```cpp
#define MMC_LOG_FORMAT "[MMC " << MMC_LOG_FILENAME_SHORT << ":" << __LINE__ << " " << __FUNCTION__ << "] "
```

**声明位置**: 行 175

**功能描述**: 生成日志前缀格式

**输出格式**: `[MMC filename:line function] `

---

### MMC_OUT_LOG

```cpp
#define MMC_OUT_LOG(LEVEL, ARGS)                            \
    do {                                                    \
        std::ostringstream oss;                             \
        oss << MMC_LOG_FORMAT << ARGS;                      \
        ock::mmc::MmcOutLogger::Instance().Log(LEVEL, oss); \
    } while (0)
```

**声明位置**: 行 176-181

**功能描述**: 核心日志输出宏

**参数**:
- `LEVEL`: 日志级别
- `ARGS`: 日志内容（支持流式输出）

**使用示例**:
```cpp
MMC_OUT_LOG(ock::mmc::INFO_LEVEL, "Processing " << count << " items");
```

---

### MMC_OUT_AUDIT_LOG

```cpp
#define MMC_OUT_AUDIT_LOG(MSG)                            \
    do {                                                  \
        std::ostringstream oss;                           \
        oss << MMC_LOG_FORMAT << (MSG);                   \
        ock::mmc::MmcOutLogger::Instance().AuditLog(oss); \
    } while (0)
```

**声明位置**: 行 182-187

**功能描述**: 审计日志输出宏

---

### MMC_LOG_ERROR_WITH_ERRCODE

```cpp
#define MMC_LOG_ERROR_WITH_ERRCODE(ARGS, ERRCODE)                           \
    do {                                                                    \
        std::ostringstream oss;                                             \
        oss << MMC_LOG_FORMAT << ARGS << ", error code " << ERRCODE;        \
        ock::mmc::MmcOutLogger::Instance().Log(ock::mmc::ERROR_LEVEL, oss); \
    } while (0)
```

**声明位置**: 行 188-193

**功能描述**: 带错误码的错误日志宏

---

### 便捷日志宏

```cpp
#define MMC_LOG_DEBUG(ARGS) MMC_OUT_LOG(ock::mmc::DEBUG_LEVEL, ARGS)
#define MMC_LOG_INFO(ARGS)  MMC_OUT_LOG(ock::mmc::INFO_LEVEL, ARGS)
#define MMC_LOG_WARN(ARGS)  MMC_OUT_LOG(ock::mmc::WARN_LEVEL, ARGS)
#define MMC_LOG_ERROR(ARGS) MMC_OUT_LOG(ock::mmc::ERROR_LEVEL, ARGS)
#define MMC_AUDIT_LOG(MSG)  MMC_OUT_AUDIT_LOG(MSG)
```

**声明位置**: 行 195-200

**功能描述**: 各级别日志的便捷宏

**使用示例**:
```cpp
MMC_LOG_INFO("Server started on port " << port);
MMC_LOG_ERROR("Failed to connect: " << error_msg);
MMC_LOG_WARN("Memory usage high: " << usage << "%");
```

---

### 断言宏

#### MMC_ASSERT_RETURN

```cpp
#define MMC_ASSERT_RETURN(ARGS, RET)             \
    do {                                         \
        if (__builtin_expect(!(ARGS), 0) != 0) { \
            MMC_LOG_ERROR("Assert " << #ARGS);   \
            return RET;                          \
        }                                        \
    } while (0)
```

**声明位置**: 行 203-209

**功能描述**: 断言宏，断言失败时记录日志并返回值

---

#### MMC_ASSERT_RET_VOID

```cpp
#define MMC_ASSERT_RET_VOID(ARGS)                \
    do {                                         \
        if (__builtin_expect(!(ARGS), 0) != 0) { \
            MMC_LOG_ERROR("Assert " << #ARGS);   \
            return;                              \
        }                                        \
    } while (0)
```

**声明位置**: 行 211-217

**功能描述**: 断言宏，断言失败时记录日志并直接返回

---

#### MMC_ASSERT

```cpp
#define MMC_ASSERT(ARGS)                         \
    do {                                         \
        if (__builtin_expect(!(ARGS), 0) != 0) { \
            MMC_LOG_ERROR("Assert " << #ARGS);   \
        }                                        \
    } while (0)
```

**声明位置**: 行 219-224

**功能描述**: 断言宏，断言失败时仅记录日志

---

### 错误返回宏

#### MMC_RETURN_ERROR

```cpp
#define MMC_RETURN_ERROR(result, msg)                     \
    do {                                                  \
        auto innerResult = (result);                      \
        if (UNLIKELY(innerResult != 0)) {                 \
            MMC_LOG_ERROR_WITH_ERRCODE(msg, innerResult); \
            return innerResult;                           \
        }                                                 \
    } while (0)
```

**声明位置**: 行 226-233

**功能描述**: 检查结果并返回错误

**参数**:
- `result`: 要检查的结果表达式
- `msg`: 错误消息

---

#### MMC_FALSE_ERROR

```cpp
#define MMC_FALSE_ERROR(result, msg)  \
    do {                              \
        auto innerResult = (result);  \
        if (UNLIKELY(!innerResult)) { \
            MMC_LOG_ERROR(msg);       \
            return innerResult;       \
        }                             \
    } while (0)
```

**声明位置**: 行 235-242

**功能描述**: 检查布尔结果并返回错误

---

## 文件级别的关系图

```
mmc_logger.h
    |
    +-- LogLevel (枚举)
    +-- MmcOutLogger (单例日志类)
    |   +-- Instance()
    |   +-- SetLogLevel() / GetLogLevel()
    |   +-- SetExternalLogFunction()
    |   +-- Log() / AuditLog()
    |
    +-- 日志宏
    |   +-- MMC_LOG_DEBUG/INFO/WARN/ERROR
    |   +-- MMC_AUDIT_LOG
    |   +-- MMC_ASSERT_* (断言宏)
    |   +-- MMC_*_ERROR (错误返回宏)
```

---

## 使用示例

```cpp
// 基本日志
MMC_LOG_INFO("Application starting");
MMC_LOG_ERROR("Failed with code: " << err_code);

// 条件断言
MMC_ASSERT_RETURN(ptr != nullptr, MMC_INVALID_PARAM);
MMC_ASSERT_RET_VOID(is_initialized_);

// 带错误码检查
MMC_RETURN_ERROR(connect(server, port), "Connection failed");
```
