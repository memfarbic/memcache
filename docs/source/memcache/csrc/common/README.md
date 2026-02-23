# common 模块文档

## 模块概述

`common` 模块包含 MemCache_Hybrid 项目的核心公共工具类、宏定义和基础设施代码，为其他模块提供通用的功能支持。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/common/`

## 文件列表

### 头文件 (.h)
- `mmc_common_includes.h` - 公共头文件聚合
- `mmc_define.h` - 宏定义和常量
- `mmc_types.h` - 类型定义和枚举
- `mmc_version.h` - 版本信息
- `mmc_logger.h` - 日志系统
- `mmc_last_error.h` - 错误处理
- `mmc_functions.h` - 工具函数
- `mmc_lock.h` - 互斥锁
- `mmc_read_write_lock.h` - 读写锁
- `mmc_spinlock.h` - 自旋锁
- `mmc_ref.h` - 引用计数智能指针
- `mmc_thread_pool.h` - 线程池
- `mmc_env.h` - 环境变量
- `mmc_montotonic.h` - 单调时间
- `mmc_smem_bm_helper.h` - SMEM BM 辅助类
- `mmc_ptracer.h` - 性能追踪

### 源文件 (.cpp)
- `mmc_last_error.cpp` - 错误处理实现
- `mmc_env.cpp` - 环境变量实现

---

## 详细文档

### mmc_common_includes.h

**功能**: 公共头文件聚合入口，统一包含常用的公共头文件。

**包含内容**:
```cpp
#include "mmc_define.h"
#include "mmc_functions.h"
#include "mmc_last_error.h"
#include "mmc_logger.h"
#include "mmc_types.h"
#include "mmc_ref.h"
#include "mmc_spinlock.h"
```

**使用说明**: 其他模块只需包含此文件即可获得所有公共功能。

---

### mmc_define.h

**功能**: 定义系统常量和宏。

**常量定义**:
- `MMC_DATA_TTL_MS` (2000) - 数据存活时间(毫秒)
- `MMC_THRESHOLD_PRINT_SECONDS` (30) - 阈值打印间隔(秒)
- `MMC_THREAD_POOL_MAX_THREADS` (1024) - 线程池最大线程数

**分支预测宏**:
```cpp
#define LIKELY(x)    (__builtin_expect(!!(x), 1) != 0)  // 表达式很可能为真
#define UNLIKELY(x)  (__builtin_expect(!!(x), 0) != 0)  // 表达式很可能为假
```

**错误处理宏**:
- `MMC_LOG_AND_SET_LAST_ERROR(msg)` - 记录并设置最后错误
- `MMC_SET_LAST_ERROR(msg)` - 设置最后错误
- `MMC_COUT_AND_SET_LAST_ERROR(msg)` - 打印并设置最后错误
- `MMC_VALIDATE_RETURN(expression, msg, returnValue)` - 验证表达式，失败则返回
- `MMC_VALIDATE_RETURN_VOID(expression, msg)` - 验证表达式，失败则返回void

**API 导出宏**:
```cpp
#define MMC_API __attribute__((visibility("default")))
```

**内存安全宏**:
```cpp
#define SAFE_DELETE(p) \
    do {               \
        delete (p);    \
        p = nullptr;   \
    } while (0)
```

---

### mmc_types.h

**功能**: 定义核心数据类型和枚举。

**错误码枚举** (`MmcErrorCode`):
```cpp
enum MmcErrorCode : int32_t {
    MMC_OK = 0,                    // 成功
    MMC_ERROR = -1,                // 一般错误
    MMC_INVALID_PARAM = -3000,     // 无效参数
    MMC_MALLOC_FAILED = -3001,     // 内存分配失败
    MMC_NEW_OBJECT_FAILED = -3002, // 对象创建失败
    MMC_NOT_STARTED = -3003,       // 未启动
    MMC_TIMEOUT = -3004,           // 超时
    // ... 更多错误码
};
```

**常量定义**:
```cpp
constexpr int32_t N16 = 16;
constexpr int32_t N64 = 64;
constexpr int32_t N256 = 256;
constexpr uint32_t UN2 = 2;
constexpr uint32_t UN32 = 32;
constexpr uint32_t MMC_DEFAUT_WAIT_TIME = 120; // 默认等待时间120秒
```

**介质类型枚举** (`MediaType`):
```cpp
enum MediaType : uint8_t {
    MEDIA_HBM,    // 高带宽内存
    MEDIA_DRAM,   // 动态随机存取存储器
    MEDIA_NONE,   // 无效类型
};
```

**介质类型转换函数**:
- `MoveUp(MediaType)` - 向上升级介质类型 (DRAM->HBM)
- `MoveDown(MediaType)` - 向下降级介质类型 (HBM->DRAM)

**位置结构** (`MmcLocation`):
```cpp
struct MmcLocation {
    uint32_t rank_;       // Rank ID
    MediaType mediaType_; // 介质类型

    bool operator<(const MmcLocation &other) const;  // 比较运算符
    bool operator==(const MmcLocation &other) const; // 相等运算符
};
```

**操作 ID 联合** (`MmcOperateIdUnion`):
```cpp
union MmcOperateIdUnion {
    uint64_t operateId_;
    struct {
        uint32_t sequence_;  // 序列号
        uint32_t rankid_;    // Rank ID
    };
};
```

**操作 ID 生成函数**:
- `GenerateOperateId(rankid)` - 生成唯一操作 ID
- `GetRankIdByOperateId(operateId)` - 从操作 ID 获取 Rank ID
- `GetSequenceByOperateId(operateId)` - 从操作 ID 获取序列号

**缓冲区数组** (`MmcBufferArray`):
```cpp
class MmcBufferArray {
    void AddBuffer(const mmc_buffer &buf);           // 添加缓冲区
    const std::vector<mmc_buffer> &Buffers() const;  // 获取缓冲区列表
    size_t TotalSize() const;                        // 获取总大小
};
```

**批量拷贝描述** (`BatchCopyDesc`):
```cpp
class BatchCopyDesc {
    std::vector<void *> srcs{};   // 源地址列表
    std::vector<void *> dsts{};   // 目标地址列表
    std::vector<uint64_t> sizes{}; // 大小列表

    void Append(const BatchCopyDesc &desc);  // 追加描述
    void Clear();                             // 清空
};
```

---

### mmc_version.h

**功能**: 定义版本信息。

**版本字符串宏**:
```cpp
#define MMC_VERSION STR2(CONCAT2(VERSION_MAJOR, VERSION_MINOR, VERSION_FIX))
```

**版本信息字符串**:
```cpp
static const char *LIB_VERSION =
    "library version: " MMC_VERSION ", build time: " __DATE__ " " __TIME__ ", commit: " STR2(GIT_LAST_COMMIT);
```

**说明**: 编译时自动生成包含版本号、构建时间和 Git 提交 ID 的版本字符串。

---

### mmc_logger.h

**功能**: 提供统一的日志记录系统。

**日志级别枚举**:
```cpp
enum LogLevel : int {
    DEBUG_LEVEL = 0,  // 调试
    INFO_LEVEL,       // 信息
    WARN_LEVEL,       // 警告
    ERROR_LEVEL,      // 错误
    BUTT_LEVEL        // 边界值(无实际用途)
};
```

**MmcOutLogger 类**:
```cpp
class MmcOutLogger {
public:
    static MmcOutLogger &Instance();                    // 单例
    LogLevel GetLogLevel() const;                       // 获取日志级别
    void SetLogLevel(LogLevel level);                   // 设置日志级别
    void SetExternalLogFunction(ExternalLog func);      // 设置外部日志函数
    void Log(int level, const std::ostringstream &oss); // 记录日志
    int32_t GetLogLevel(const std::string &logLevelDesc) const; // 字符串转级别

private:
    LogLevel logLevel_ = INFO_LEVEL;           // 当前日志级别
    ExternalLog logFunc_ = nullptr;            // 外部日志函数
    ExternalAuditLog auditLogFunc_ = nullptr;  // 外部审计日志函数
};
```

**日志宏**:
```cpp
#define MMC_LOG_DEBUG(ARGS)   // 记录 DEBUG 级别日志
#define MMC_LOG_INFO(ARGS)    // 记录 INFO 级别日志
#define MMC_LOG_WARN(ARGS)    // 记录 WARN 级别日志
#define MMC_LOG_ERROR(ARGS)   // 记录 ERROR 级别日志
#define MMC_LOG_ERROR_WITH_ERRCODE(ARGS, ERRCODE)  // 带错误码的日志
#define MMC_AUDIT_LOG(MSG)    // 记录审计日志
```

**断言宏**:
```cpp
#define MMC_ASSERT_RETURN(ARGS, RET)      // 断言失败返回值
#define MMC_ASSERT_RET_VOID(ARGS)         // 断言失败返回void
#define MMC_ASSERT(ARGS)                  // 简单断言
```

**错误返回宏**:
```cpp
#define MMC_RETURN_ERROR(result, msg)     // 检查结果并返回错误
#define MMC_FALSE_ERROR(result, msg)      // 检查布尔结果并返回错误
```

**日志格式**:
```
[时间戳] [日志级别] [PID-TID] [文件:行 函数] 消息内容
```

---

### mmc_last_error.h

**功能**: 提供线程本地存储的最后错误信息。

**MmcLastError 类**:
```cpp
class MmcLastError {
public:
    static void Set(const std::string &msg);  // 设置错误信息
    static void Set(const char *msg);         // 设置错误信息 (C字符串)
    static const char *GetAndClear(bool clear); // 获取并清除错误信息

private:
    static thread_local bool have_;       // 是否有错误
    static thread_local std::string msg_; // 错误消息
};
```

**说明**:
- 使用 `thread_local` 实现线程本地存储
- 每个线程有独立的错误信息
- `GetAndClear(true)` 获取后清除，`GetAndClear(false)` 仅获取

**实现** (`mmc_last_error.cpp`):
```cpp
thread_local bool MmcLastError::have_ = false;
thread_local std::string MmcLastError::msg_;
```

---

### mmc_functions.h

**功能**: 提供通用工具函数。

**动态加载宏** (`DL_LOAD_SYM`):
```cpp
#define DL_LOAD_SYM(TARGET_FUNC_VAR, TARGET_FUNC_TYPE, FILE_HANDLE, SYMBOL_NAME)
```
**说明**: 从动态库加载符号，失败时清理并返回错误。

**Func 类**:
```cpp
class Func {
public:
    static bool Realpath(std::string &path);  // 获取真实路径(解析符号链接)
    static Result LibraryRealPath(const std::string &libDirPath,
                                  const std::string &libName,
                                  std::string &realPath);  // 获取库真实路径
};
```

**路径验证函数**:
```cpp
inline int ValidatePathNotSymlink(const char *path)
```
**功能**: 验证路径存在且不是软链接。

**安全拷贝函数**:
```cpp
inline void SafeCopy(const std::string &src, char *dst, const size_t dstSize)
```
**功能**: 安全地将字符串拷贝到固定大小缓冲区，确保以 null 结尾。

**环境变量获取函数**:
```cpp
inline std::string SafeGetEnv(const char *name) noexcept
```
**功能**: 安全获取环境变量，不存在时返回空字符串。

---

### mmc_lock.h

**功能**: 提供互斥锁和 RAII 锁管理器。

**Lock 类**:
```cpp
class Lock {
public:
    void DoLock();   // 加锁
    void Unlock();   // 解锁

private:
    std::mutex mLock;  // 标准 mutex
};
```

**Locker 模板类** (RAII):
```cpp
template<class T>
class Locker {
public:
    explicit Locker(T *lock);  // 构造时自动加锁
    ~Locker();                 // 析构时自动解锁
};
```

**GUARD 宏**:
```cpp
#define GUARD(lLock, alias) Locker<Lock> __l##alias(lLock)
```
**使用示例**:
```cpp
GUARD(&lock, unique);  // 创建名为 __lunique 的守卫
// 作用域结束时自动解锁
```

---

### mmc_read_write_lock.h

**功能**: 提供读写锁，支持多个读者或单个写者。

**ReadWriteLock 类**:
```cpp
class ReadWriteLock {
public:
    void LockRead();    // 获取读锁
    void UnlockRead();  // 释放读锁
    void LockWrite();   // 获取写锁
    void UnlockWrite(); // 释放写锁

private:
    std::mutex mutex_;              // 保护内部状态
    std::condition_variable cv_;    // 条件变量
    uint16_t numReaders_{0};        // 当前读者数量
    bool isWriting_{false};         // 是否正在写入
};
```

**ReadLock RAII 类**:
```cpp
class ReadLock {
public:
    explicit ReadLock(ReadWriteLock &rwLock);  // 构造时获取读锁
    ~ReadLock();                                // 析构时释放读锁
};
```

**WriteLock RAII 类**:
```cpp
class WriteLock {
public:
    explicit WriteLock(ReadWriteLock &rwLock); // 构造时获取写锁
    ~WriteLock();                                // 析构时释放写锁
};
```

**使用示例**:
```cpp
ReadWriteLock rwLock;
{
    ReadLock readLock(rwLock);  // 自动获取读锁
    // 读取操作...
} // 自动释放读锁
```

---

### mmc_spinlock.h

**功能**: 提供自旋锁，使用原子操作实现。

**Spinlock 类**:
```cpp
class Spinlock {
public:
    void lock();   // 自旋获取锁
    void unlock(); // 释放锁

private:
    std::atomic<uint32_t> lock_{0};  // 原子变量，0=未锁定，1=已锁定
};
```

**实现原理**:
- `lock()`: 使用 `exchange` 操作原子地设置为 1，循环等待直到成功
- `unlock()`: 原子地设置为 0，使用 `memory_order_release` 语义

**兼容性**: 使用小写 `lock/unlock` 名称，可与 `std::lock_guard` 配合使用。

---

### mmc_ref.h

**功能**: 提供引用计数智能指针，类似于 `std::shared_ptr`。

**MmcReferable 类** (可被引用的对象):
```cpp
class MmcReferable {
public:
    MmcReferable() = default;
    virtual ~MmcReferable() = default;

    void IncreaseRef();  // 增加引用计数
    void DecreaseRef();  // 减少引用计数，计数为0时自动删除

protected:
    int32_t mRefCount = 0;  // 引用计数
};
```

**MmcRef 模板类** (智能指针):
```cpp
template<typename T>
class MmcRef {
public:
    MmcRef() noexcept = default;
    MmcRef(T *newObj) noexcept;              // 从裸指针构造
    MmcRef(const MmcRef<T> &other) noexcept; // 拷贝构造
    MmcRef(MmcRef<T> &&other) noexcept;      // 移动构造
    ~MmcRef();                               // 析构，自动减引用

    MmcRef<T> &operator=(T *newObj);         // 赋值裸指针
    MmcRef<T> &operator=(const MmcRef<T> &other);  // 拷贝赋值
    MmcRef<T> &operator=(MmcRef<T> &&other) noexcept; // 移动赋值

    bool operator==(const MmcRef<T> &other) const; // 相等比较
    bool operator!=(const MmcRef<T> &other) const; // 不等比较

    T *operator->() const;   // 箭头操作符
    T *Get() const;          // 获取裸指针
    void Set(T *newObj);     // 设置新对象

private:
    T *mObj = nullptr;  // 指向的对象
};
```

**辅助函数**:
```cpp
template<class Src, class Des>
static MmcRef<Des> Convert(const MmcRef<Src> &child);  // 类型转换

template<typename C, typename... ARGS>
inline MmcRef<C> MmcMakeRef(ARGS... args);  // 创建对象并返回智能指针
```

**使用示例**:
```cpp
class MyClass : public MmcReferable {
    // ...
};

MmcRef<MyClass> obj = MmcMakeRef<MyClass>(args);
MmcRef<MyClass> obj2 = obj;  // 引用计数增加
```

---

### mmc_thread_pool.h

**功能**: 提供可配置的线程池，支持任务调度。

**MmcThreadPool 类**:
```cpp
class MmcThreadPool : public MmcReferable {
public:
    MmcThreadPool(std::string name, size_t numThreads);  // 构造函数

    static void TrySetProcessNice(int nice_value);              // 设置进程优先级
    static int32_t NextCpu() noexcept;                          // 获取下一个CPU ID
    static void TrySetThreadAffinityAndPriority();              // 设置线程亲和性和优先级

    int32_t Start();                                            // 启动线程池
    template<typename F, typename... Args>
    auto Enqueue(F &&f, Args &&...args) -> std::future<invoke_result_t<F, Args...>>;  // 提交任务
    void Destroy();                                             // 销毁线程池

private:
    std::vector<std::thread> workers;                    // 工作线程
    std::queue<std::function<void()>> taskQueue;         // 任务队列
    std::mutex queueMutex;                               // 保护队列的互斥锁
    std::condition_variable queueCondition;              // 条件变量
    std::string mmcPoolName;                             // 线程池名称
    size_t numThreads;                                   // 线程数量
    bool stop;                                           // 停止标志
};
```

**常量**:
```cpp
constexpr int32_t MIN_NICE = -20;  // 最高优先级
constexpr int32_t MAX_NICE = 19;   // 最低优先级
```

**使用示例**:
```cpp
MmcThreadPoolPtr pool = MmcMakeRef<MmcThreadPool>("my_pool", 4);
pool->Start();

auto future = pool->Enqueue([](int x) { return x * 2; }, 21);
int result = future.get();  // 等待结果，result = 42
```

**类型别名**:
```cpp
using MmcThreadPoolPtr = MmcRef<MmcThreadPool>;
```

---

### mmc_env.h / mmc_env.cpp

**功能**: 定义环境变量相关的全局变量。

**环境变量**:
```cpp
extern std::string MMC_META_CONF_PATH;   // 元服务配置路径 (环境变量 MMC_META_CONFIG_PATH)
extern std::string MMC_LOCAL_CONF_PATH;  // 本地服务配置路径 (环境变量 MMC_LOCAL_CONFIG_PATH)
extern std::string META_POD_NAME;        // Meta Pod 名称 (环境变量 META_POD_NAME)
extern std::string META_NAMESPACE;       // Meta 命名空间 (环境变量 META_NAMESPACE)
extern std::string META_LEASE_NAME;      // Meta Lease 名称 (环境变量 META_LEASE_NAME)
```

**实现** (`mmc_env.cpp`):
```cpp
std::string MMC_META_CONF_PATH = SafeGetEnv("MMC_META_CONFIG_PATH");
std::string MMC_LOCAL_CONF_PATH = SafeGetEnv("MMC_LOCAL_CONFIG_PATH");
std::string META_POD_NAME = SafeGetEnv("META_POD_NAME");
std::string META_NAMESPACE = SafeGetEnv("META_NAMESPACE");
std::string META_LEASE_NAME = SafeGetEnv("META_LEASE_NAME");
```

---

### mmc_montotonic.h

**功能**: 提供高精度单调时间测量。

**Monotonic 类**:
```cpp
class Monotonic {
public:
    // ARM64 平台实现
    template<int32_t FAILURE_RET>
    static int32_t InitTickUs();           // 初始化 tick 频率
    static inline uint64_t TimeUs();       // 获取微秒时间
    static inline uint64_t TimeNs();       // 获取纳秒时间
    static inline uint64_t TimeSec();      // 获取秒时间

    // x86_64 平台实现
    // (同上，使用 RDTSC 指令)

    // 使用 clock_gettime 的备用实现
    // (使用 CLOCK_MONOTONIC)
};
```

**实现方式**:
- **ARM64**: 使用 `cntvct_el0` (虚拟计数器) 和 `cntfrq_el0` (频率寄存器)
- **x86_64**: 使用 `rdtsc` 指令，从 `/proc/cpuinfo` 读取 CPU 频率
- **备用方案**: 使用 `clock_gettime(CLOCK_MONOTONIC)`

**命名空间**: `ock::dagger::Monotonic`

---

### mmc_smem_bm_helper.h

**功能**: 提供与 SMEM (Shared Memory) Blob Manager 交互的辅助函数。

**MmcSmemBmHelper 类**:
```cpp
class MmcSmemBmHelper {
public:
    static smem_bm_data_op_type TransSmemBmDataOpType(const std::string &dataOpType);  // 转换数据操作类型
    static smem_tls_config TransSmemTlsConfig(const mmc_tls_config &config);          // 转换 TLS 配置
};
```

**数据操作类型转换**:
- `"device_sdma"` → `SMEMB_DATA_OP_SDMA`
- `"device_rdma"` → `SMEMB_DATA_OP_DEVICE_RDMA`
- `"host_tcp"` → `SMEMB_DATA_OP_HOST_TCP`
- `"host_rdma"` → `SMEMB_DATA_OP_HOST_RDMA`
- `"host_urma"` → `SMEMB_DATA_OP_HOST_URMA`

---

### mmc_ptracer.h

**功能**: 定义性能追踪点 ID。

**追踪点枚举** (`TP_MMC_MOD`):
```cpp
enum TP_MMC_MOD {
    // Python API 追踪点
    TP_MMC_PY_PUT,                  // Python Put 操作
    TP_MMC_PY_BATCH_PUT,            // Python 批量 Put
    TP_MMC_PY_GET,                  // Python Get 操作
    TP_MMC_PY_BATCH_GET,            // Python 批量 Get
    // ... 更多追踪点

    // 本地服务追踪点
    TP_MMC_LOCAL_PUT,               // 本地 Put
    TP_MMC_LOCAL_GET,               // 本地 Get
    // ... 更多追踪点

    // 元服务追踪点
    TP_MMC_META_MGR_ALLOC,          // 元管理器分配
    TP_MMC_META_PUT,                // 元服务 Put
    // ... 更多追踪点

    // SMEM BM 追踪点
    TP_SMEM_BM_PUT,                 // SMEM BM Put
    // ... 更多追踪点

    // ACC (接入层) 追踪点
    TP_ACC_SEND_ALLOC,              // ACC 发送分配请求
    // ... 更多追踪点
};
```

**说明**: 使用 ptracer 框架进行性能分析和追踪。

---

## 数据流和关系

```
                    ┌─────────────────┐
                    │  mmc_logger.h   │
                    │  (日志系统)      │
                    └────────┬────────┘
                             │ 被
                             ▼
┌──────────────┐     ┌──────────────────┐
│mmc_define.h  │────▶│mmc_common_includes│
│(宏定义)       │     │     .h           │
└──────────────┘     └─────────┬────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐    ┌──────────────┐    ┌──────────────┐
│ mmc_types.h   │    │mmc_functions.h│    │mmc_lock.h    │
│ (类型定义)      │    │ (工具函数)     │    │ (锁机制)      │
└───────────────┘    └──────────────┘    └──────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐    ┌──────────────┐    ┌──────────────┐
│ mmc_ref.h     │    │mmc_thread_   │    │mmc_read_     │
│ (引用计数)      │    │ pool.h       │    │ write_lock.h │
└───────────────┘    │ (线程池)       │    │ (读写锁)      │
                     └──────────────┘    └──────────────┘
```

---

## 使用示例

### 日志记录
```cpp
#include "mmc_common_includes.h"

void SomeFunction() {
    MMC_LOG_INFO("This is an info message");
    MMC_LOG_ERROR("Error occurred: " << errorCode);

    int result = SomeCall();
    MMC_RETURN_ERROR(result, "SomeCall failed");
}
```

### 锁使用
```cpp
#include "mmc_common_includes.h"

Lock myLock;
{
    GUARD(&myLock, unique);  // RAII 自动加锁/解锁
    // 临界区代码...
}

ReadWriteLock rwLock;
{
    ReadLock readLock(rwLock);  // 读锁
    // 读取操作...
}
{
    WriteLock writeLock(rwLock);  // 写锁
    // 写入操作...
}
```

### 线程池
```cpp
#include "mmc_common_includes.h"

MmcThreadPoolPtr pool = MmcMakeRef<MmcThreadPool>("worker", 8);
pool->Start();

auto future = pool->Enqueue([]() {
    MMC_LOG_INFO("Task running");
    return 42;
});

int result = future.get();
```

### 引用计数
```cpp
class MyClass : public MmcReferable {
    // ...
};

MmcRef<MyClass> obj1 = MmcMakeRef<MyClass>();
MmcRef<MyClass> obj2 = obj1;  // 引用计数变为 2
obj1 = nullptr;                // 引用计数变为 1
// obj2 析构时，对象会被自动删除
```
