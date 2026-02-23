# mmc_net_wait_handle.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/acc_links_impl/mmc_net_wait_handle.h`
- **文件用途**: 定义网络请求等待处理器，用于同步等待 RPC 响应
- **依赖项**:
  - `ctime` - 时间处理
  - `pthread.h` - POSIX 线程库
  - `mmc_net_ctx_store.h` - 上下文存储

---

## 类定义

### NetWaitHandler 类

网络请求等待处理器，用于同步等待响应

```cpp
class NetWaitHandler : public MmcReferable {
public:
    explicit NetWaitHandler(const NetContextStorePtr &ctxStore);

    ~NetWaitHandler() override;

    Result Initialize();
    Result TimedWait(int32_t second = UINT32_MAX) noexcept;
    Result Notify(int32_t result, const TcpDataBufPtr &data) noexcept;

    inline int32_t GetResult() const;
    inline const TcpDataBufPtr &Data() const;

private:
    NetContextStore *ctxStore_ = nullptr;
    TcpDataBufPtr data_;
    int32_t result_ = INT32_MAX;
    bool notified = false;
    pthread_mutex_t mutex_ = PTHREAD_MUTEX_INITIALIZER;
    pthread_cond_t cond_{};
};
```

**设计说明**:
- 使用条件变量实现同步等待
- 支持超时机制
- 使用 pthread 的 CLOCK_MONOTONIC 避免系统时间调整影响

---

## 方法

### NetWaitHandler::NetWaitHandler()

```cpp
explicit NetWaitHandler(const NetContextStorePtr &ctxStore)
{
    /* hold the reference */
    ctxStore_ = ctxStore.Get();
    if (LIKELY(ctxStore_ != nullptr)) {
        ctxStore_->IncreaseRef();
    }
}
```

**声明位置**: 行 24-31

**功能描述**: 构造函数，持有上下文存储的引用

**参数**: `ctxStore` - 上下文存储智能指针

**代码逻辑**:
1. 获取裸指针
2. 如果非空，增加引用计数

**注意**: 持有引用防止上下文存储在使用过程中被释放

---

### NetWaitHandler::~NetWaitHandler()

```cpp
~NetWaitHandler() override
{
    /* decrease the reference count */
    if (LIKELY(ctxStore_ != nullptr)) {
        ctxStore_->DecreaseRef();
        ctxStore_ = nullptr;
    }
    pthread_cond_destroy(&cond_);
}
```

**声明位置**: 行 33-41

**功能描述**: 析构函数，释放资源

**代码逻辑**:
1. 减少上下文存储的引用计数
2. 销毁条件变量

---

### NetWaitHandler::Initialize()

```cpp
Result Initialize()
{
    /* init pthread condition attr with relative time, instead of abs time */
    pthread_condattr_t attr;
    pthread_condattr_init(&attr);
    auto err = pthread_condattr_setclock(&attr, CLOCK_MONOTONIC);
    if (UNLIKELY(err != 0)) {
        MMC_LOG_ERROR("Failed to init pthread condition, error " << err);
        return MMC_ERROR;
    }

    /* init condition */
    pthread_cond_init(&cond_, &attr);
    return MMC_OK;
}
```

**声明位置**: 行 48-62

**功能描述**: 初始化等待处理器

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 初始化条件变量属性
2. 设置使用 CLOCK_MONOTONIC（单调时钟）
3. 初始化条件变量

**为什么使用 CLOCK_MONOTONIC**:
- 不受系统时间调整影响
- 保证超时计算的准确性

---

### NetWaitHandler::TimedWait()

```cpp
Result TimedWait(int32_t second = UINT32_MAX) noexcept
{
    pthread_mutex_lock(&mutex_);
    /* already notified */
    if (notified) {
        pthread_mutex_unlock(&mutex_);
        return MMC_OK;
    }

    /* relative time instead of abs */
    struct timespec currentTime{};
    clock_gettime(CLOCK_MONOTONIC, &currentTime);

    struct timespec futureTime{};
    if (currentTime.tv_sec > std::numeric_limits<time_t>::max() - static_cast<time_t>(second)) {
        pthread_mutex_unlock(&mutex_);
        MMC_LOG_ERROR("Time overflow");
        return MMC_ERROR;
    }
    futureTime.tv_sec = currentTime.tv_sec + second;
    futureTime.tv_nsec = currentTime.tv_nsec;
    auto waitResult = pthread_cond_timedwait(&cond_, &mutex_, &futureTime);
    if (waitResult == ETIMEDOUT) {
        pthread_mutex_unlock(&mutex_);
        return MMC_TIMEOUT;
    }

    /* notify by other */
    pthread_mutex_unlock(&mutex_);
    return MMC_OK;
}
```

**声明位置**: 行 71-101

**功能描述**: 等待响应或超时

**参数**: `second` - 超时时间（秒），默认最大值

**返回值**:
- `MMC_OK`: 收到响应
- `MMC_TIMEOUT`: 超时
- `MMC_ERROR`: 错误（如时间溢出）

**代码逻辑**:
1. 加锁
2. 如果已通知，直接返回
3. 获取当前单调时间
4. 计算超时时间点
5. 等待条件变量信号
6. 处理超时或正常返回

---

### NetWaitHandler::Notify()

```cpp
Result Notify(int32_t result, const TcpDataBufPtr &data) noexcept
{
    pthread_mutex_lock(&mutex_);
    /* already notified */
    if (notified) {
        pthread_mutex_unlock(&mutex_);
        return MMC_ALREADY_NOTIFIED;
    }

    data_ = data;
    result_ = result;
    pthread_cond_signal(&cond_);
    notified = true;
    pthread_mutex_unlock(&mutex_);
    return MMC_OK;
}
```

**声明位置**: 行 110-125

**功能描述**: 通知等待的线程

**参数**:
- `result`: RPC 调用结果
- `data`: 响应数据缓冲区

**返回值**:
- `MMC_OK`: 成功通知
- `MMC_ALREADY_NOTIFIED`: 已经被通知过

**代码逻辑**:
1. 加锁
2. 检查是否已通知
3. 保存结果和数据
4. 发送条件变量信号
5. 设置已通知标志

---

### NetWaitHandler::GetResult()

```cpp
inline int32_t GetResult() const
{
    return result_;
}
```

**声明位置**: 行 132-135

**功能描述**: 获取 RPC 调用结果

**返回值**: `int32_t` - 结果值

---

### NetWaitHandler::Data()

```cpp
inline const TcpDataBufPtr &Data() const
{
    return data_;
}
```

**声明位置**: 行 142-145

**功能描述**: 获取响应数据

**返回值**: `const TcpDataBufPtr&` - 数据缓冲区智能指针引用

---

## 成员变量

```cpp
private:
    NetContextStore *ctxStore_ = nullptr; /* hold the reference ctx store, in case of use after free */
    TcpDataBufPtr data_;                  /* the data that replied from peer */
    int32_t result_ = INT32_MAX;          /* the result from peer */
    bool notified = false;                /* notified or not to prevent notify again */
    pthread_mutex_t mutex_ = PTHREAD_MUTEX_INITIALIZER;
    pthread_cond_t cond_{};
```

- `ctxStore_`: 上下文存储指针（持有引用）
- `data_`: 响应数据
- `result_`: 调用结果
- `notified`: 是否已通知标志
- `mutex_`: 互斥锁
- `cond_`: 条件变量

---

## 文件级别的关系图

```
mmc_net_wait_handle.h (等待处理器)
    |
    +-- NetWaitHandler -> RPC 响应等待处理
            |
            +-- Initialize() -> 初始化条件变量
            +-- TimedWait() -> 等待响应或超时
            +-- Notify() -> 通知等待的线程
            +-- GetResult() -> 获取结果
            +-- Data() -> 获取响应数据
            |
            +-> 使用 pthread 条件变量
            +-> 使用 CLOCK_MONOTONIC 避免时间调整影响
```

---

## 依赖关系

**依赖以下文件**:
- `ctime` - C++ 时间库
- `pthread.h` - POSIX 线程库
- `mmc_net_ctx_store.h` - 上下文存储

**被以下文件依赖**:
- `mmc_net_engine_acc.cpp` - ACC 网络引擎实现

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有等待处理器定义都在 ock::mmc 命名空间内
}
}
```

---

## 使用流程

```
发起请求线程                    响应处理线程
     |                              |
     | 1. 创建 NetWaitHandler       |
     | 2. 存储到 ctxStore           |
     | 3. 发送请求                  |
     |                              |
     | 4. TimedWait() 等待          |
     |    <------                   |
     |           收到响应            |
     |    <------                   |
     |           Notify(result, data)
     |    |                          |
     | 5. 唤醒                      |
     | 6. GetResult()/Data()        |
     |                              |
```

---

## 使用示例

```cpp
// 发送请求线程
auto waiter = MmcMakeRef<NetWaitHandler>(ctxStore);
waiter->Initialize();

// 存储到上下文存储
uint32_t seqNo = 0;
ctxStore->PutAndGetSeqNo<NetWaitHandler>(waiter.Get(), seqNo);

// 发送请求
link->RealLink()->NonBlockSend(MSG_TYPE_DATA, opCode, seqNo, dataBuf, nullptr);

// 等待响应
Result ret = waiter->TimedWait(60);  // 60秒超时
if (ret == MMC_OK) {
    auto result = waiter->GetResult();
    auto data = waiter->Data();
    // 处理响应...
}

// 响应处理线程
void OnResponse(const TcpReqContext &context) {
    // 获取等待处理器
    NetWaitHandler *waiter = nullptr;
    ctxStore->GetSeqNoAndRemove<NetWaitHandler>(context.SeqNo(), waiter);

    // 通知等待线程
    waiter->Notify(result, dataBuf);
}
```
