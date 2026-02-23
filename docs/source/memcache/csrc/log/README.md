# log 模块文档

## 模块概述

`log` 模块提供基于 spdlog 的日志功能，支持文件日志轮转、多级别日志、审计日志等特性。同时提供 C 和 C++ 两套 API。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/log/`

## 文件列表

### 头文件 (.h)
- `spdlogger.h` - C++ 日志类定义
- `spdlogger4c.h` - C API 定义

### 源文件 (.cpp)
- `spdlogger.cpp` - C++ 日志实现
- `spdlogger4c.cpp` - C API 桥接实现

---

## 详细文档

### spdlogger.h

**功能**: 定义 C++ 日志类 `SpdLogger`，基于 spdlog 库。

**日志级别枚举** (`LogLevel`):
```cpp
enum class LogLevel {
    TRACE = 0,    // 追踪
    DEBUG = 1,    // 调试
    INFO = 2,     // 信息
    WARN = 3,     // 警告
    ERROR = 4,    // 错误
    CRITICAL = 5, // 严重
    LOG_LEVEL_MAX,
};
```

**SpdLogger 类**:
```cpp
class SpdLogger {
public:
    SpdLogger() = default;
    ~SpdLogger() = default;

    // 获取单例实例
    static SpdLogger &GetInstance();
    static SpdLogger &GetAuditInstance();

    // 初始化日志
    int Initialize(const std::string &path, int minLogLevel,
                   int rotationFileSize, int rotationFileCount);

    // 设置日志级别
    int SetLogMinLevel(int minLevel);

    // 记录日志消息
    void LogMessage(int level, const char *message);
    void AuditLogMessage(const char *message);

    // 获取最后错误消息
    static const char *GetLastErrorMessage();

    // 刷新日志
    void Flush(void);

private:
    // 参数验证
    static int ValidateParams(int minLogLevel, const std::string &path,
                             int rotationFileSize, int rotationFileCount);

    // 文件回调
    static void BeforeOpenCallback(const std::string &filename);
    static void AfterOpenCallback(const std::string &filename, std::FILE *file_stream);
    static void AfterCloseCallback(const std::string &filename);

    // 成员变量
    std::mutex mutex_;
    bool started_ = false;
    std::shared_ptr<spdlog::logger> mSPDLogger;
    std::string mFilePath;
    int mRotationFileSize = 0;
    int mRotationFileCount = 0;
    bool mDebugEnabled = false;
    static thread_local std::string gLastErrorMessage;
};
```

**常量定义**:
```cpp
constexpr int ROTATION_FILE_SIZE_MAX = 500 * 1024 * 1024;  // 500MB
constexpr int ROTATION_FILE_SIZE_MIN = 1 * 1024 * 1024;     // 1MB
constexpr int ROTATION_FILE_COUNT_MAX = 50;
constexpr mode_t LOG_FILE_CREATE_MODE = 0640;
constexpr mode_t LOG_FILE_READ_ONLY_MODE = 0440;
```

---

### spdlogger4c.h

**功能**: 定义 C API，方便 C 代码调用日志功能。

**审计事件类型** (`AuditEventType`):
```cpp
enum AuditEventType {
    START_MF_SERVICE = 0,
    STOP_MF_SERVICE,
    CONNECT_ZOOKEEPER,
    DISCONNECT_ZOOKEEPER,
    RW_ZOOKEEPER,
    CLOSE_ZOOKEEPER,
};
```

**审计资源类型** (`AuditResourceType`):
```cpp
enum AuditResourceType {
    BIG_MEMORY = 0,
    META_ZOOKEEPER,
};
```

**C API 函数**:
```cpp
// 初始化普通日志
int SPDLOG_Init(const char *path, int minLogLevel,
                int rotationFileSize, int rotationFileCount);

// 初始化审计日志
int SPDLOG_AuditInit(const char *path, int rotationFileSize, int rotationFileCount);

// 记录普通日志
void SPDLOG_LogMessage(int32_t level, const char *msg);

// 记录审计日志
void SPDLOG_AuditLogMessage(const char *msg);

// 获取最后错误消息
const char *SPDLOG_GetLastErrorMessage();

// 重置日志级别
int SPDLOG_ResetLogLevel(int logLevel);
```

---

### spdlogger.cpp

**功能**: C++ 日志实现。

**SpdLogger::ValidateParams**

```cpp
int SpdLogger::ValidateParams(int minLogLevel, const std::string &path,
                              int rotationFileSize, int rotationFileCount)
```

**功能**: 验证初始化参数

**参数范围**:
- `minLogLevel`: 0-5 (TRACE 到 CRITICAL)
- `rotationFileSize`: 1MB - 500MB
- `rotationFileCount`: 1 - 50

**返回值**: 0=成功, -1=失败

---

**SpdLogger::Initialize**

```cpp
int SpdLogger::Initialize(const std::string &path, int minLogLevel,
                          int rotationFileSize, int rotationFileCount)
```

**功能**: 初始化日志系统

**代码逻辑**:
1. 验证参数
2. 设置文件事件回调（处理权限）
3. 创建轮转日志记录器
4. 设置日志格式: `%Y-%m-%d %H:%M:%S.%f %t %l %v`
5. 设置每秒自动刷新
6. 设置错误级别自动刷新

**日志格式说明**:
- `%Y`: 年
- `%m`: 月
- `%d`: 日
- `%H:%M:%S.%f`: 时间.微秒
- `%t`: 线程 ID
- `%l`: 日志级别
- `%v`: 消息

---

**SpdLogger::LogMessage**

```cpp
void SpdLogger::LogMessage(int level, const char *message)
```

**功能**: 记录指定级别的日志消息

**参数**:
- `level`: 日志级别 (0-5)
- `message`: 日志消息

---

**SpdLogger::AuditLogMessage**

```cpp
void SpdLogger::AuditLogMessage(const char *message)
```

**功能**: 记录审计日志（固定为 WARN 级别）

---

**文件权限回调**:

```cpp
// 打开前设置创建权限
void SpdLogger::BeforeOpenCallback(const std::string &filename)
{
    chmod(filename.c_str(), LOG_FILE_CREATE_MODE);  // 0640
}

// 打开后设置只读权限
void SpdLogger::AfterOpenCallback(const std::string &filename, std::FILE *file_stream)
{
    chmod(filename.c_str(), LOG_FILE_READ_ONLY_MODE);  // 0440
}

// 关闭后设置只读权限
void SpdLogger::AfterCloseCallback(const std::string &filename)
{
    chmod(filename.c_str(), LOG_FILE_READ_ONLY_MODE);  // 0440
}
```

---

### spdlogger4c.cpp

**功能**: C API 实现，桥接到 C++ API。

**SPDLOG_Init**

```cpp
int SPDLOG_Init(const char *path, int minLogLevel,
                int rotationFileSize, int rotationFileCount)
```

**功能**: 初始化普通日志

**说明**: 日志级别参数会自动加 1，以匹配 C 和 C++ 的级别定义差异

---

**SPDLOG_AuditInit**

```cpp
int SPDLOG_AuditInit(const char *path, int rotationFileSize, int rotationFileCount)
```

**功能**: 初始化审计日志

**说明**: 审计日志固定使用 WARN (3) 级别

---

**SPDLOG_LogMessage**

```cpp
void SPDLOG_LogMessage(int32_t level, const char *msg)
```

**功能**: 记录日志消息

---

**SPDLOG_AuditLogMessage**

```cpp
void SPDLOG_AuditLogMessage(const char *msg)
```

**功能**: 记录审计日志消息

---

## 日志格式

### 普通日志格式

```
YYYY-MM-DD HH:MM:SS.uuuuuu thread_id LEVEL message
```

示例:
```
2025-02-24 12:34:56.123456 12345 INFO Starting service
2025-02-24 12:34:57.234567 12345 WARN Connection timeout
2025-02-24 12:34:58.345678 12345 ERROR Failed to allocate memory
```

### 审计日志格式

与普通日志格式相同，但记录到独立的审计日志文件。

---

## 使用示例

### C++ API

```cpp
#include "spdlogger.h"

using namespace ock::mmc::log;

// 初始化日志
SpdLogger &logger = SpdLogger::GetInstance();
int ret = logger.Initialize("/var/log/mmc/app.log",
                            2,  // INFO 级别
                            10 * 1024 * 1024,  // 10MB
                            10);  // 保留 10 个文件
if (ret != 0) {
    const char *errMsg = SpdLogger::GetLastErrorMessage();
    // 处理错误
}

// 记录日志
logger.LogMessage(2, "Application started");  // INFO
logger.LogMessage(3, "Configuration loaded");  // WARN
logger.LogMessage(4, "Operation failed");      // ERROR

// 刷新日志
logger.Flush();

// 初始化审计日志
SpdLogger &auditLogger = SpdLogger::GetAuditInstance();
auditLogger.Initialize("/var/log/mmc/audit.log",
                       3,  // WARN 级别
                       50 * 1024 * 1024,  // 50MB
                       20);  // 保留 20 个文件

// 记录审计日志
auditLogger.AuditLogMessage("User logged in");
auditLogger.AuditLogMessage("Data accessed");
```

### C API

```c
#include "spdlogger4c.h"

// 初始化日志
int ret = SPDLOG_Init("/var/log/mmc/app.log",
                      1,  // INFO 级别 (C API)
                      10 * 1024 * 1024,  // 10MB
                      10);  // 保留 10 个文件
if (ret != 0) {
    const char *errMsg = SPDLOG_GetLastErrorMessage();
    // 处理错误
}

// 记录日志
SPDLOG_LogMessage(1, "Application started");   // INFO
SPDLOG_LogMessage(2, "Configuration loaded");  // WARN
SPDLOG_LogMessage(3, "Operation failed");      // ERROR

// 初始化审计日志
SPDLOG_AuditInit("/var/log/mmc/audit.log",
                 50 * 1024 * 1024,  // 50MB
                 20);  // 保留 20 个文件

// 记录审计日志
SPDLOG_AuditLogMessage("User logged in");
SPDLOG_AuditLogMessage("Data accessed");

// 重置日志级别
SPDLOG_ResetLogLevel(0);  // 改为 DEBUG
```

---

## 日志级别对照表

| C++ LogLevel | C API 值 | 名称 | 说明 |
|--------------|----------|------|------|
| 0 | 1 | DEBUG | 调试信息 |
| 1 | 2 | INFO | 一般信息 |
| 2 | 3 | WARN | 警告信息 |
| 3 | 4 | ERROR | 错误信息 |
| 4 | 5 | CRITICAL | 严重错误 |

---

## 文件权限

日志文件权限设置：

| 状态 | 权限 | 说明 |
|------|------|------|
| 创建前 | 0640 | 用户可读写，组可读 |
| 打开后 | 0440 | 用户只读，组只读 |
| 关闭后 | 0440 | 用户只读，组只读 |

---

## 日志轮转

当日志文件达到指定大小时，会自动轮转：

1. 当前日志文件重命名为 `app.log.1`
2. 旧的 `.1` 文件重命名为 `.2`，依此类推
3. 超过 `rotationFileCount` 的旧文件会被删除

示例（rotationFileCount = 3）:
```
app.log        (当前写入)
app.log.1      (最新的轮转文件)
app.log.2
app.log.3      (最老的轮转文件)
```

---

## 线程安全

- 所有 API 都是线程安全的
- 使用互斥锁保护内部状态
- `GetLastErrorMessage()` 使用 `thread_local` 存储，线程独立

---

## 性能考虑

1. **自动刷新**: 每秒自动刷新一次，避免频繁 I/O
2. **错误刷新**: ERROR 及以上级别立即刷新
3. **异步日志**: 可配置为异步模式（通过 spdlog）
4. **批量刷新**: 调用 `Flush()` 可手动触发刷新

---

## 错误处理

所有初始化函数返回 `int`:
- `0`: 成功
- `-1`: 失败

获取错误信息：
```cpp
const char *errMsg = SpdLogger::GetLastErrorMessage();
// 或 C API
const char *errMsg = SPDLOG_GetLastErrorMessage();
```
