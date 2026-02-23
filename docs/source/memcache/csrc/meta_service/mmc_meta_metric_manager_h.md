# mmc_meta_metric_manager.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_metric_manager.h`
- **文件用途**: 定义元数据管理器的监控指标管理类，集成 Prometheus 指标收集
- **依赖项**: `prometheus/simpleapi.h`

---

## 类定义

### MmcMetaMetricManager

监控指标管理器类，使用单例模式，集成 Prometheus 指标收集。

---

#### GetInstance

```cpp
static MmcMetaMetricManager &GetInstance()
{
    static MmcMetaMetricManager staticInstance;
    return staticInstance;
}
```

**声明位置**: 行 25-29

**功能描述**: 获取单例实例

**返回值**: 单例引用

**实现**: 使用 Meyer's Singleton（函数内静态变量）

---

#### 禁用的方法

```cpp
MmcMetaMetricManager(const MmcMetaMetricManager &) = delete;
MmcMetaMetricManager &operator=(const MmcMetaMetricManager &) = delete;
MmcMetaMetricManager(MmcMetaMetricManager &&) = delete;
MmcMetaMetricManager &operator=(MmcMetaMetricManager &&) = delete;
```

**声明位置**: 行 31-34

**功能描述**: 禁止拷贝和移动操作

---

### GetSummary

```cpp
std::string GetSummary() const;
```

**声明位置**: 行 37

**功能描述**: 获取人类可读的指标摘要

**返回值**: 格式化的字符串摘要

---

### GetPrometheusSummary

```cpp
static std::string GetPrometheusSummary();
```

**声明位置**: 行 39

**功能描述**: 获取 Prometheus 格式的指标

**返回值**: Prometheus 文本格式的字符串

---

### IncrementAllocCounter

```cpp
void IncrementAllocCounter()
{
    allocCounter_++;
}
```

**声明位置**: 行 41-44

**功能描述**: 增加分配操作计数

---

### IncrementRemoveCounter

```cpp
void IncrementRemoveCounter()
{
    removeCounter_++;
}
```

**声明位置**: 行 45-48

**功能描述**: 增加删除操作计数

---

### IncrementGetCounter

```cpp
void IncrementGetCounter()
{
    getCounter_++;
}
```

**声明位置**: 行 49-52

**功能描述**: 增加获取操作计数

---

### IncrementEvictCounter

```cpp
void IncrementEvictCounter()
{
    evictCounter_++;
}
```

**声明位置**: 行 53-56

**功能描述**: 增加淘汰操作计数

---

### SetKeyCount

```cpp
void SetKeyCount(const size_t count)
{
    keyCountGauge_ = static_cast<int64_t>(count);
}
```

**声明位置**: 行 57-60

**功能描述**: 设置当前存储的键数量

**参数**:
- `count`: 键数量

---

## 私有成员

### 构造函数

```cpp
MmcMetaMetricManager();
```

**声明位置**: 行 63

**功能描述**: 私有构造函数，初始化 Prometheus 指标

---

### 析构函数

```cpp
~MmcMetaMetricManager() = default;
```

**声明位置**: 行 64

**功能描述**: 默认析构函数

---

### 成员变量

```cpp
private:
    prometheus::simpleapi::counter_metric_t allocCounter_;
    prometheus::simpleapi::counter_metric_t removeCounter_;
    prometheus::simpleapi::counter_metric_t getCounter_;
    prometheus::simpleapi::counter_metric_t evictCounter_;
    prometheus::simpleapi::gauge_metric_t keyCountGauge_;
```

**声明位置**: 行 66-70

**说明**:
- `allocCounter_`: 分配操作计数器
- `removeCounter_`: 删除操作计数器
- `getCounter_`: 获取操作计数器
- `evictCounter_`: 淘汰操作计数器
- `keyCountGauge_`: 键数量仪表

---

## Prometheus 指标

| 指标名 | 类型 | 描述 |
|--------|------|------|
| `memcache_alloc_operations_total` | Counter | 分配操作总数 |
| `memcache_remove_operations_total` | Counter | 删除操作总数 |
| `memcache_get_operations_total` | Counter | 获取操作总数 |
| `memcache_evict_operations_total` | Counter | 淘汰操作总数 |
| `memcache_stored_keys` | Gauge | 当前存储的键数量 |

---

## 文件级别的关系图

```
mmc_meta_metric_manager.h (监控指标管理器)
    |
    +-- 使用 prometheus/simpleapi (Prometheus 客户端)
    |
    +-- 单例模式
    |
    +-- 被 MmcMetaManager 使用
    |
    +-- 提供:
    |   +-- 操作计数器 (分配/删除/获取/淘汰)
    |   +-- 键数量仪表
    |   +-- Prometheus 格式导出
    |   +-- 人类可读摘要
```

---

## 使用示例

```cpp
// 增加计数
MmcMetaMetricManager::GetInstance().IncrementAllocCounter();

// 设置键数量
MmcMetaMetricManager::GetInstance().SetKeyCount(1000);

// 获取 Prometheus 格式
std::string prometheus = MmcMetaMetricManager::GetPrometheusSummary();

// 获取人类可读摘要
std::string summary = MmcMetaMetricManager::GetInstance().GetSummary();
```
