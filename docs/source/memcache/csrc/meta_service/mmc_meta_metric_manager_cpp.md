# mmc_meta_metric_manager.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_metric_manager.cpp`
- **文件用途**: MmcMetaMetricManager 类的实现文件
- **依赖项**: `prometheus/text_serializer.h`, `mmc_meta_metric_manager.h`

---

## 全局变量

```cpp
namespace prometheus {
namespace simpleapi {
auto registry_ptr = std::make_shared<Registry>();
Registry &registry = *registry_ptr;
} // namespace simpleapi
} // namespace prometheus
```

**声明位置**: 行 20-25

**说明**: Prometheus 注册表全局实例

---

## 常量定义

```cpp
constexpr int FP_OUTPUT_PRECISION = 2;
```

**声明位置**: 行 27

**说明**: 浮点数输出精度（小数点后位数）

---

## 函数实现

### 构造函数

```cpp
MmcMetaMetricManager::MmcMetaMetricManager()
    : allocCounter_("memcache_alloc_operations_total", "Total number of allocation operations"),
      removeCounter_("memcache_remove_operations_total", "Total number of remove operations"),
      getCounter_("memcache_get_operations_total", "Total number of get operations"),
      evictCounter_("memcache_evict_operations_total", "Total number of eviction operations"),
      keyCountGauge_("memcache_stored_keys", "Current number of stored keys")
{}
```

**声明位置**: 行 32-38

**功能描述**: 构造并初始化 Prometheus 指标

**指标初始化**:
- `allocCounter_`: 分配操作计数器，名称 "memcache_alloc_operations_total"，描述 "Total number of allocation operations"
- `removeCounter_`: 删除操作计数器
- `getCounter_`: 获取操作计数器
- `evictCounter_`: 淘汰操作计数器
- `keyCountGauge_`: 键数量仪表，名称 "memcache_stored_keys"，描述 "Current number of stored keys"

---

### GetSummary

```cpp
std::string MmcMetaMetricManager::GetSummary() const
{
    std::ostringstream oss;

    oss << "=== MemCache Metrics Summary ===" << std::endl;
    oss << std::fixed << std::setprecision(FP_OUTPUT_PRECISION);

    oss << "Allocation Operations: " << allocCounter_.value() << std::endl;
    oss << "Remove Operations: " << removeCounter_.value() << std::endl;
    oss << "Get Operations: " << getCounter_.value() << std::endl;

    return oss.str();
}
```

**声明位置**: 行 40-52

**功能描述**: 获取人类可读的指标摘要

**输出格式**:
```
=== MemCache Metrics Summary ===
Allocation Operations: 1234.00
Remove Operations: 567.00
Get Operations: 8901.00
```

---

### GetPrometheusSummary

```cpp
std::string MmcMetaMetricManager::GetPrometheusSummary()
{
    auto metrics = prometheus::simpleapi::registry.Collect();
    std::ostringstream ss;
    prometheus::TextSerializer::Serialize(ss, metrics);
    return ss.str();
}
```

**声明位置**: 行 54-60

**功能描述**: 获取 Prometheus 文本格式的指标

**输出格式**: Prometheus 文本格式，示例：
```
# HELP memcache_alloc_operations_total Total number of allocation operations
# TYPE memcache_alloc_operations_total counter
memcache_alloc_operations_total 1234
# HELP memcache_remove_operations_total Total number of remove operations
# TYPE memcache_remove_operations_total counter
memcache_remove_operations_total 567
# HELP memcache_get_operations_total Total number of get operations
# TYPE memcache_get_operations_total counter
memcache_get_operations_total 8901
# HELP memcache_stored_keys Current number of stored keys
# TYPE memcache_stored_keys gauge
memcache_stored_keys 1000
```

---

## 总结

此文件实现了监控指标管理器的功能：

1. **Prometheus 集成**: 使用 Prometheus C++ 客户端库收集指标
2. **单例模式**: 全局唯一的指标管理器
3. **多种指标类型**:
   - Counter: 单调递增的计数器（分配、删除、获取、淘汰操作）
   - Gauge: 可增减的仪表（键数量）
4. **多种输出格式**:
   - 人类可读摘要
   - Prometheus 文本格式

**使用场景**:
- HTTP 监控端点 `/metrics` 导出 Prometheus 格式指标
- HTTP 监控端点 `/metrics/summary` 导出人类可读摘要
