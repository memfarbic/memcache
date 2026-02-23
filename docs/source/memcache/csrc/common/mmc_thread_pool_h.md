# mmc_thread_pool.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_thread_pool.h`
- **文件用途**: 提供线程池实现，支持任务队列、CPU 亲和性、优先级设置等
- **依赖项**: `mmc_logger.h`, `mmc_types.h`, `mmc_ref.h`, 标准库线程相关头文件

---

## 类型别名

### invoke_result_t

```cpp
#if __cplusplus < 201703L
template<typename F, typename... Args>
using invoke_result_t = typename std::result_of<F(Args...)>::type;
#else
template<typename F, typename... Args>
using invoke_result_t = typename std::invoke_result<F, Args...>::type;
#endif
```

**声明位置**: 行 34-40

**功能描述**: C++11/14 与 C++17 兼容的类型推导，用于获取函数调用的返回类型

---

## 常量定义

### MIN_NICE / MAX_NICE

```cpp
constexpr int32_t MIN_NICE = -20;
constexpr int32_t MAX_NICE = 19;
```

**声明位置**: 行 42-43

**功能描述**: 进程优先级（nice值）的最小值和最大值

**说明**: Linux nice 值范围，-20 为最高优先级，19 为最低优先级

---

## 类定义

### MmcThreadPool

```cpp
class MmcThreadPool : public MmcReferable {
public:
    MmcThreadPool(std::string name, size_t numThreads);

    static inline void TrySetProcessNice(int nice_value);
    static inline int32_t NextCpu() noexcept;
    static inline void TrySetThreadAffinityAndPriority();

    int32_t Start();
    template<typename F, typename... Args>
    auto Enqueue(F &&f, Args &&...args) -> std::future<invoke_result_t<F, Args...>>;
    void Destroy();

    ~MmcThreadPool() override;

    MmcThreadPool(const MmcThreadPool &) = delete;
    MmcThreadPool &operator=(const MmcThreadPool &) = delete;

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> taskQueue;
    std::mutex queueMutex;
    std::condition_variable queueCondition;
    std::string mmcPoolName;
    size_t numThreads;
    bool stop;
};
```

**声明位置**: 行 45-170

**功能描述**: 线程池类，支持异步任务执行

---

#### MmcThreadPool::MmcThreadPool()

```cpp
MmcThreadPool(std::string name, size_t numThreads) : mmcPoolName(name), numThreads(numThreads), stop(false) {}
```

**声明位置**: 行 47

**功能描述**: 构造函数

**参数**:
- `name`: 线程池名称（用于命名工作线程）
- `numThreads`: 线程数量

**初始状态**: `stop = false`，线程池未启动

---

#### MmcThreadPool::TrySetProcessNice()

```cpp
static inline void TrySetProcessNice(int nice_value)
{
    if (nice_value < MIN_NICE || nice_value > MAX_NICE) {
        return;
    }
    int result = setpriority(PRIO_PROCESS, 0, nice_value);
    if (result != 0) {
        MMC_LOG_WARN("Failed to set process nice to " << nice_value << " (errno=" << errno
                                                      << "): " << strerror(errno));
    }
}
```

**声明位置**: 行 49-59

**功能描述**: 尝试设置进程优先级

**参数**:
- `nice_value`: nice 值（-20 到 19）

**代码逻辑**:
1. 检查 nice 值是否在有效范围内
2. 调用 `setpriority()` 设置优先级
3. 如果失败，记录警告日志

---

#### MmcThreadPool::NextCpu()

```cpp
static inline int32_t NextCpu() noexcept
{
    auto num_cpus = static_cast<int32_t>(sysconf(_SC_NPROCESSORS_ONLN));
    if (num_cpus <= 0) {
        num_cpus = 1;
    }
    static std::atomic<int32_t> nextId{0};
    if (nextId.fetch_add(1) > std::numeric_limits<int16_t>::max() - 1) {
        nextId.store(0);
    }
    return nextId.load() % num_cpus;
}
```

**声明位置**: 行 61-72

**功能描述**: 获取下一个 CPU 核心编号（轮询分配）

**返回值**: CPU 核心 ID

**代码逻辑**:
1. 获取系统 CPU 核心数
2. 使用原子计数器轮询分配
3. 防止计数器溢出（超过 INT16_MAX 时重置）

**用途**: 用于设置线程 CPU 亲和性

---

#### MmcThreadPool::TrySetThreadAffinityAndPriority()

```cpp
static inline void TrySetThreadAffinityAndPriority()
{
    static thread_local bool initialized = false;
    if (initialized) {
        return;
    }
    initialized = true;
    cpu_set_t cpus;
    CPU_ZERO(&cpus);
    CPU_SET(NextCpu(), &cpus);
    pthread_setaffinity_np(pthread_self(), sizeof(cpus), &cpus);
    setpriority(PRIO_PROCESS, 0, MIN_NICE);
}
```

**声明位置**: 行 74-86

**功能描述**: 设置线程的 CPU 亲和性和优先级

**代码逻辑**:
1. 使用 `thread_local` 确保每个线程只初始化一次
2. 创建 CPU 集合，只包含一个核心
3. 设置线程亲和性（绑定到单个 CPU 核心）
4. 设置线程优先级为最高（MIN_NICE = -20）

**注意事项**:
- 每个线程只会调用一次
- 可以提高缓存局部性

---

#### MmcThreadPool::Start()

```cpp
int32_t Start()
{
    if (numThreads == 0 || numThreads > MMC_THREAD_POOL_MAX_THREADS) {
        MMC_LOG_ERROR("Number of threads must be greater than 0 and less than " << MMC_THREAD_POOL_MAX_THREADS);
        return MMC_ERROR;
    }
    for (size_t i = 0; i < numThreads; ++i) {
        workers.emplace_back([this] {
            while (true) {
                std::function<void()> task;
                {
                    std::unique_lock<std::mutex> lock(queueMutex);
                    queueCondition.wait(lock, [this] { return stop || !taskQueue.empty(); });
                    if (stop && taskQueue.empty()) {
                        break;
                    }
                    task = std::move(taskQueue.front());
                    taskQueue.pop();
                }
                task();
            }
            MMC_LOG_DEBUG("worker thread :" << std::this_thread::get_id() << " exit");
        });

        std::string threadName = mmcPoolName + std::to_string(i);
        int ret = pthread_setname_np(workers.back().native_handle(), threadName.c_str());
        if (ret != 0) {
            MMC_LOG_ERROR("set thread name failed, i:" << i << ", ret:" << ret);
        }
    }
    return MMC_OK;
}
```

**声明位置**: 行 88-119

**功能描述**: 启动线程池，创建工作线程

**返回值**: 成功返回 `MMC_OK`，失败返回 `MMC_ERROR`

**代码逻辑**:
1. 验证线程数量是否合法
2. 创建指定数量的工作线程
3. 每个工作线程循环：
   - 等待任务或停止信号
   - 取出任务执行
   - 如果收到停止信号且队列为空，退出
4. 为每个线程设置名称（格式：`poolName + index`）

**工作线程行为**:
- 等待条件变量（无任务时阻塞）
- 有任务时取出执行
- 收到停止信号后退出

---

#### MmcThreadPool::Enqueue()

```cpp
template<typename F, typename... Args>
auto Enqueue(F &&f, Args &&...args) -> std::future<invoke_result_t<F, Args...>>
{
    using returnType = invoke_result_t<F, Args...>;
    auto func = std::make_shared<std::packaged_task<returnType()>>(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...));
    std::future<returnType> future = func->get_future();
    {
        std::unique_lock<std::mutex> uniqueLock(queueMutex);
        if (stop) {
            MMC_LOG_ERROR("thread pool has stopped.");
            return std::future<returnType>{}; // 返回无效的 future
        }
        taskQueue.emplace([func]() { (*func)(); });
    }
    queueCondition.notify_one();
    return future;
}
```

**声明位置**: 行 121-138

**功能描述**: 向线程池提交任务

**参数**:
- `f`: 可调用对象（函数、lambda 等）
- `args`: 参数

**返回值**: `std::future`，可用于获取任务执行结果

**代码逻辑**:
1. 创建 `packaged_task` 包装任务和参数
2. 获取关联的 `future`
3. 如果线程池已停止，返回无效的 `future`
4. 将任务加入队列
5. 唤醒一个工作线程

**使用示例**:
```cpp
auto result = pool.Enqueue([](int x) { return x * 2; }, 21);
int value = result.get();  // 42
```

---

#### MmcThreadPool::Destroy()

```cpp
void Destroy()
{
    {
        std::unique_lock<std::mutex> lock(queueMutex);
        stop = true;
    }
    queueCondition.notify_all();
    for (std::thread &worker : workers) {
        if (worker.joinable()) {
            worker.join();
        }
    }
}
```

**声明位置**: 行 140-152

**功能描述**: 销毁线程池，等待所有工作线程退出

**代码逻辑**:
1. 设置停止标志
2. 唤醒所有工作线程
3. 等待所有线程退出（join）

**注意事项**: 会等待队列中的任务执行完成

---

#### MmcThreadPool::~MmcThreadPool()

```cpp
~MmcThreadPool() override
{
    Destroy();
}
```

**声明位置**: 行 154-157

**功能描述**: 析构函数，自动调用 `Destroy()`

---

#### MmcThreadPool 禁止的操作

```cpp
MmcThreadPool(const MmcThreadPool &) = delete;
MmcThreadPool &operator=(const MmcThreadPool &) = delete;
```

**声明位置**: 行 159-160

**功能描述**: 禁止拷贝构造和拷贝赋值

---

#### MmcThreadPool 成员变量

```cpp
private:
    std::vector<std::thread> workers;           // 工作线程数组
    std::queue<std::function<void()>> taskQueue; // 任务队列
    std::mutex queueMutex;                      // 保护队列的互斥锁
    std::condition_variable queueCondition;     // 条件变量
    std::string mmcPoolName;                    // 线程池名称
    size_t numThreads;                          // 线程数量
    bool stop;                                  // 停止标志
```

---

## 类型别名

### MmcThreadPoolPtr

```cpp
using MmcThreadPoolPtr = MmcRef<MmcThreadPool>;
```

**声明位置**: 行 172

**功能描述**: 线程池的智能指针类型别名

---

## 文件级别的关系图

```
mmc_thread_pool.h
    |
    +-- MmcThreadPool (继承自 MmcReferable)
    |   +-- mmcPoolName     [线程池名称]
    |   +-- numThreads      [线程数量]
    |   +-- stop            [停止标志]
    |   +-- workers         [工作线程数组]
    |   +-- taskQueue       [任务队列]
    |   +-- queueMutex      [队列互斥锁]
    |   +-- queueCondition  [条件变量]
    |   |
    |   +-- Start()                    [启动线程池]
    |   +-- Enqueue()                  [提交任务]
    |   +-- Destroy()                  [销毁线程池]
    |   |
    |   +-- TrySetProcessNice()        [设置进程优先级]
    |   +-- NextCpu()                  [获取下一个CPU]
    |   +-- TrySetThreadAffinityAndPriority() [设置线程亲和性]
    |
    +-- MmcThreadPoolPtr (智能指针类型)
```

---

## 使用示例

### 基本用法

```cpp
#include "mmc_thread_pool.h"

// 创建线程池
MmcThreadPool pool("MyPool", 4);
pool.Start();

// 提交任务
auto future1 = pool.Enqueue([]() {
    return 42;
});

int result = future1.get();  // 等待结果
```

### 带参数的任务

```cpp
void ProcessData(int id, const std::string& data) {
    // 处理数据
}

auto future = pool.Enqueue(ProcessData, 1, "hello");
future.wait();  // 等待完成
```

### Lambda 任务

```cpp
auto result = pool.Enqueue([](int a, int b) {
    return a + b;
}, 10, 20);

std::cout << result.get() << std::endl;  // 输出 30
```

### 使用智能指针

```cpp
MmcThreadPoolPtr pool = MmcMakeRef<MmcThreadPool>("WorkerPool", 8);
pool->Start();
// 使用 pool
pool->Destroy();
```

---

## 注意事项

1. **先启动再使用**: 必须先调用 `Start()` 才能提交任务
2. **最大线程数**: 线程数不能超过 `MMC_THREAD_POOL_MAX_THREADS`（1024）
3. **异常处理**: 任务中的异常不会传播，需要在任务内部处理
4. **资源清理**: 析构函数会自动调用 `Destroy()`，但建议手动调用以控制清理时机
5. **CPU 亲和性**: `TrySetThreadAffinityAndPriority()` 需要在任务函数内部手动调用
