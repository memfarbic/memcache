# mmc_montotonic.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_montotonic.h`
- **文件用途**: 提供高精度单调时间测量功能，支持 ARM64 和 x86_64 架构
- **依赖项**: `<cstdio>`, `<fstream>`, `<string>`, ARM64/ x86_64 特定头文件
- **命名空间**: `ock::dagger`

---

## 类定义

### Monotonic

```cpp
class Monotonic {
    // ... (详见下文)
};
```

**声明位置**: 行 19-225

**功能描述**: 提供跨平台的高精度单调时间接口

**编译选项控制**:
- `USE_PROCESS_MONOTONIC` 定义时：使用 CPU 指令获取高精度时间
- 未定义时：使用 `clock_gettime(CLOCK_MONOTONIC)` 系统调用

---

## ARM64 实现 (USE_PROCESS_MONOTONIC)

### InitTickUs<FAILURE_RET>()

```cpp
template<int32_t FAILURE_RET>
static int32_t InitTickUs()
{
    /* get frequ */
    uint64_t tmpFreq = 0;
    __asm__ volatile("mrs %0, cntfrq_el0" : "=r"(tmpFreq));
    auto freq = static_cast<uint32_t>(tmpFreq);

    /* calculate */
    freq = freq / 1000L / 1000L;
    if (freq == 0) {
        printf("Failed to get tick as freq is %d\n", freq);
        return FAILURE_RET;
    }

    return freq;
}
```

**声明位置**: 行 26-42 (ARM64)

**功能描述**: 初始化 ARM64 的计时器频率，返回每微秒的 tick 数

**代码逻辑**:
1. 使用 `mrs` 指令读取系统计数器频率寄存器 `cntfrq_el0`
2. 将频率转换为每微秒的 tick 数
3. 验证结果有效性

**返回值**: 每微秒的 tick 数，失败返回 `FAILURE_RET`

---

### TimeUs() [ARM64]

```cpp
static inline uint64_t TimeUs()
{
    const static int32_t TICK_PER_US = InitTickUs<1>();
    uint64_t timeValue = 0;
    __asm__ volatile("mrs %0, cntvct_el0" : "=r"(timeValue));
    return timeValue / TICK_PER_US;
}
```

**声明位置**: 行 47-53 (ARM64)

**功能描述**: 获取单调时间（微秒）

**代码逻辑**:
1. 静态初始化每微秒 tick 数
2. 使用 `mrs` 指令读取虚拟计数器 `cntvct_el0`
3. 将 tick 转换为微秒

**特性**: 使用内联汇编，无系统调用开销

---

### TimeNs() [ARM64]

```cpp
static inline uint64_t TimeNs()
{
    const static int32_t TICK_PER_US = InitTickUs<1>();
    uint64_t timeValue = 0;
    __asm__ volatile("mrs %0, cntvct_el0" : "=r"(timeValue));
    return timeValue * 1000L / TICK_PER_US;
}
```

**声明位置**: 行 58-64 (ARM64)

**功能描述**: 获取单调时间（纳秒）

---

### TimeSec() [ARM64]

```cpp
static inline uint64_t TimeSec()
{
    const static int32_t TICK_PER_US = InitTickUs<1>();
    uint64_t timeValue = 0;
    __asm__ volatile("mrs %0, cntvct_el0" : "=r"(timeValue));
    return timeValue / (TICK_PER_US * 100000L);
}
```

**声明位置**: 行 69-75 (ARM64)

**功能描述**: 获取单调时间（秒）

---

## x86_64 实现 (USE_PROCESS_MONOTONIC)

### InitTickUs<FAILURE_RET>() [x86_64]

```cpp
template<int32_t FAILURE_RET>
static int32_t InitTickUs()
{
    const std::string path = "/proc/cpuinfo";
    const std::string prefix = "model name";
    const std::string gHZ = "GHz";

    std::ifstream inConfFile(path);
    if (!inConfFile) {
        printf("Failed to get tick as failed to open %s\n", path.c_str());
        return FAILURE_RET;
    }

    bool found = false;
    std::string strLine;
    while (getline(inConfFile, strLine)) {
        if (strLine.compare(0, prefix.size(), prefix) == 0) {
            found = true;
            break;
        }
    }

    if (!found) {
        printf("Failed to get tick as failed to find %s\n", prefix.c_str());
        return FAILURE_RET;
    }

    std::vector<std::string> splitVec;
    NN_SplitStr(strLine, " ", splitVec);
    if (splitVec.empty()) {
        printf("Failed to get tick as failed to get line %s\n", prefix.c_str());
        return FAILURE_RET;
    }

    std::string lastWord = splitVec[splitVec.size() - 1];
    auto index = lastWord.find(gHZ);
    if (index == std::string::npos) {
        printf("Failed to get tick as failed to get %s\n", gHZ.c_str());
        return FAILURE_RET;
    }

    auto strGhz = lastWord.substr(0, index);
    float fhz = 0.0f;
    if (!NN_Stof(strGhz, fhz)) {
        printf("Failed to get tick as failed to convert %s to float\n", strGhz.c_str());
        return FAILURE_RET;
    }

    return static_cast<int32_t>(fhz * 1000L);
}
```

**声明位置**: 行 78-127 (x86_64)

**功能描述**: 从 /proc/cpuinfo 解析 CPU 频率，计算每微秒的 tick 数

**代码逻辑**:
1. 打开 `/proc/cpuinfo` 文件
2. 查找 `model name` 行
3. 解析 CPU 频率（格式如 "3.50GHz"）
4. 将 GHz 转换为每微秒的 tick 数

**注意事项**: 依赖于 /proc/cpuinfo 的格式

---

### TimeUs() [x86_64]

```cpp
static inline uint64_t TimeUs()
{
    const static int32_t TICK_PER_US = InitTickUs<1>();
    return __rdtsc() / TICK_PER_US;
}
```

**声明位置**: 行 132-136 (x86_64)

**功能描述**: 使用 RDTSC 指令获取单调时间（微秒）

**代码逻辑**:
1. 使用 `__rdtsc()` 读取时间戳计数器
2. 转换为微秒

---

### TimeNs() [x86_64]

```cpp
static inline uint64_t TimeNs()
{
    const static int32_t TICK_PER_US = InitTickUs<1>();
    return __rdtsc() * 1000L / TICK_PER_US;
}
```

**声明位置**: 行 141-145 (x86_64)

**功能描述**: 获取单调时间（纳秒）

---

### TimeSec() [x86_64]

```cpp
static inline uint64_t TimeSec()
{
    const static int32_t TICK_PER_US = InitTickUs<1>();
    return __rdtsc() / (TICK_PER_US * 1000000L);
}
```

**声明位置**: 行 150-154 (x86_64)

**功能描述**: 获取单调时间（秒）

---

## 系统调用实现 (未定义 USE_PROCESS_MONOTONIC)

### InitTickUs<FAILURE_RET>()

```cpp
template<int32_t FAILURE_RET>
static int32_t InitTickUs()
{
    return 0;
}
```

**声明位置**: 行 160-164

**功能描述**: 系统调用模式下不需要初始化，直接返回 0

---

### TimeUs() [系统调用]

```cpp
static inline uint64_t TimeUs()
{
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return static_cast<uint64_t>(ts.tv_sec * 1000000L + ts.tv_nsec / 1000L);
}
```

**声明位置**: 行 166-171

**功能描述**: 使用 `clock_gettime` 获取单调时间（微秒）

---

### TimeNs() [系统调用]

```cpp
static inline uint64_t TimeNs()
{
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return static_cast<uint64_t>(ts.tv_sec * 1000000000L + ts.tv_nsec);
}
```

**声明位置**: 行 173-178

**功能描述**: 使用 `clock_gettime` 获取单调时间（纳秒）

---

### TimeSec() [系统调用]

```cpp
static inline uint64_t TimeSec()
{
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return static_cast<uint64_t>(ts.tv_sec + ts.tv_nsec / 1000000000L);
}
```

**声明位置**: 行 180-185

**功能描述**: 使用 `clock_gettime` 获取单调时间（秒）

---

## x86_64 辅助函数

### NN_SplitStr()

```cpp
static void NN_SplitStr(const std::string &str, const std::string &separator, std::vector<std::string> &result)
{
    result.clear();
    std::string::size_type pos1 = 0;
    std::string::size_type pos2 = str.find(separator);

    std::string tmpStr;
    while (pos2 != std::string::npos) {
        tmpStr = str.substr(pos1, pos2 - pos1);
        result.emplace_back(tmpStr);
        pos1 = pos2 + separator.size();
        pos2 = str.find(separator, pos1);
    }

    if (pos1 != str.length()) {
        tmpStr = str.substr(pos1);
        result.emplace_back(tmpStr);
    }
}
```

**声明位置**: 行 190-208

**功能描述**: 字符串分割函数（私有辅助函数）

**参数**:
- `str`: 要分割的字符串
- `separator`: 分隔符
- `result`: 输出的分割结果

---

### NN_Stof()

```cpp
static bool NN_Stof(const std::string &str, float &value)
{
    constexpr float EPSINON = 0.000001;
    char *remain = nullptr;
    errno = 0;
    value = std::strtof(str.c_str(), &remain);
    if (remain == nullptr || strlen(remain) > 0 ||
        ((value - HUGE_VALF) >= -EPSINON && (value - HUGE_VALF) <= EPSINON && errno == ERANGE)) {
        return false;
    } else if ((value >= -EPSINON && value <= EPSINON) && (str != "0.0")) {
        return false;
    }
    return true;
}
```

**声明位置**: 行 210-223

**功能描述**: 安全的字符串转浮点数函数（私有辅助函数）

**参数**:
- `str`: 输入字符串
- `value`: 输出的浮点数值

**返回值**: 转换是否成功

---

## 文件级别的关系图

```
mmc_montotonic.h (ock::dagger::Monotonic)
    |
    +-- 编译选项: USE_PROCESS_MONOTONIC
    |   |
    |   +-- ARM64 实现
    |   |   +-- InitTickUs()   [读取 cntfrq_el0]
    |   |   +-- TimeUs()       [使用 cntvct_el0]
    |   |   +-- TimeNs()
    |   |   +-- TimeSec()
    |   |
    |   +-- x86_64 实现
    |       +-- InitTickUs()   [解析 /proc/cpuinfo]
    |       +-- TimeUs()       [使用 __rdtsc()]
    |       +-- TimeNs()
    |       +-- TimeSec()
    |
    +-- 系统调用实现 (默认)
        +-- TimeUs()   [使用 clock_gettime]
        +-- TimeNs()
        +-- TimeSec()
```

---

## 使用示例

### 基本用法

```cpp
#include "mmc_montotonic.h"

using namespace ock::dagger;

void MeasureTime() {
    uint64_t start = Monotonic::TimeUs();
    // 执行一些操作
    uint64_t end = Monotonic::TimeUs();
    uint64_t elapsed = end - start;
    std::cout << "Elapsed: " << elapsed << " microseconds" << std::endl;
}
```

### 性能测量

```cpp
void Benchmark() {
    uint64_t startNs = Monotonic::TimeNs();

    // 被测试的代码
    for (int i = 0; i < 1000000; i++) {
        // ...
    }

    uint64_t endNs = Monotonic::TimeNs();
    std::cout << "Time: " << (endNs - startNs) / 1000.0 << " us" << std::endl;
}
```

---

## 实现对比

| 实现方式 | 精度 | 开销 | 可移植性 | 说明 |
|----------|------|------|----------|------|
| ARM64 CPU 指令 | 纳秒级 | 极低 | ARM64 | 使用系统计数器 |
| x86_64 RDTSC | 纳秒级 | 极低 | x86_64 | 使用时间戳计数器 |
| clock_gettime | 微秒级 | 中等 | POSIX | 标准系统调用 |

---

## 注意事项

1. **单调时间**: 返回的是单调递增的时间，不是墙上时钟时间
2. **线程安全**: 所有函数都是线程安全的
3. **初始化**: `InitTickUs()` 使用模板静态变量，确保只初始化一次
4. **x86_64 限制**: 依赖 /proc/cpuinfo 格式，可能在某些系统上不工作
5. **命名空间**: 类在 `ock::dagger` 命名空间，而非 `ock::mmc`
