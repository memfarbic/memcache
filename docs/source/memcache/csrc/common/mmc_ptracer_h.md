# mmc_ptracer.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_ptracer.h`
- **文件用途**: 定义 MemCache 项目的性能跟踪点（Trace Points）枚举
- **依赖项**: `ptracer.h` 或 `ut_ptracer.h`（根据 UT_ENABLED 宏决定）

---

## 条件编译

```cpp
#ifdef UT_ENABLED
#include "ut_ptracer.h"
#else
#include "ptracer.h"
#endif
```

**声明位置**: 行 15-19

**功能描述**: 根据编译模式包含不同的 ptracer 头文件

**说明**:
- `UT_ENABLED` 定义时: 使用单元测试版本的 ptracer
- 未定义时: 使用生产版本的 ptracer

---

## 枚举定义

### TP_MMC_MOD

```cpp
enum TP_MMC_MOD {
    TP_MMC_START = PTRACER_ID(1, 0U),
    // ... (详见下文)
};
```

**声明位置**: 行 21-120

**功能描述**: 定义所有 MMC 模块的跟踪点 ID

**宏说明**: `PTRACER_ID(major, minor)` 用于生成跟踪点 ID

---

## 跟踪点分类

### 1. Python 接口跟踪点

```cpp
TP_MMC_PY_PUT,                  // Python 单个数据写入
TP_MMC_PY_BATCH_PUT,            // Python 批量数据写入
TP_MMC_PYBIND_PUT_LAYERS,       // PyBind 层写入
TP_MMC_PY_PUT_LAYERS,           // Python 层写入
TP_MMC_PY_PUT_LAYERS_2D,        // Python 2D 层写入
TP_MMC_PYBIND_BATCH_PUT_LAYERS, // PyBind 批量层写入
TP_MMC_PY_BATCH_PUT_LAYERS,     // Python 批量层写入
TP_MMC_PY_BATCH_PUT_LAYERS_2D,  // Python 批量 2D 层写入
```

**说明**: Python API 相关的 Put 操作跟踪点

---

```cpp
TP_MMC_PY_GET,                  // Python 单个数据读取
TP_MMC_PY_BATCH_GET,            // Python 批量数据读取
TP_MMC_PYBIND_GET_LAYERS,       // PyBind 层读取
TP_MMC_PY_GET_LAYERS,           // Python 层读取
TP_MMC_PY_GET_LAYERS_2D,        // Python 2D 层读取
TP_MMC_PYBIND_BATCH_GET_LAYERS, // PyBind 批量层读取
TP_MMC_PY_BATCH_GET_LAYERS,     // Python 批量层读取
TP_MMC_PY_BATCH_GET_LAYERS_2D,  // Python 批量 2D 层读取
```

**说明**: Python API 相关的 Get 操作跟踪点

---

```cpp
TP_MMC_PY_UPDATE,               // Python 更新操作
TP_MMC_PY_BATCH_UPDATE,         // Python 批量更新
TP_MMC_PY_REMOVE,               // Python 删除操作
TP_MMC_PY_BATCH_REMOVE,         // Python 批量删除
TP_MMC_PY_EXIST,                // Python 存在性检查
TP_MMC_PY_BATCH_EXIST,          // Python 批量存在性检查
TP_MMC_PY_QUERY,                // Python 查询操作
TP_MMC_PY_BATCH_QUERY,          // Python 批量查询
```

**说明**: Python API 其他操作的跟踪点

---

### 2. 本地存储跟踪点

```cpp
TP_MMC_LOCAL_PUT,               // 本地单个数据写入
TP_MMC_LOCAL_BATCH_PUT,         // 本地批量数据写入
TP_MMC_LOCAL_BATCH_PUT_SIZE,    // 本地批量写入大小
TP_MMC_LOCAL_BATCH_PUT_SYNC,    // 本地批量写入同步
TP_MMC_LOCAL_GET,               // 本地单个数据读取
TP_MMC_LOCAL_BATCH_GET,         // 本地批量数据读取
TP_MMC_LOCAL_BATCH_GET_SIZE,    // 本地批量读取大小
TP_MMC_LOCAL_BATCH_GET_SYNC,    // 本地批量读取同步
TP_MMC_LOCAL_UPDATE,            // 本地更新
TP_MMC_LOCAL_BATCH_UPDATE,      // 本地批量更新
TP_MMC_LOCAL_REMOVE,            // 本地删除
TP_MMC_LOCAL_BATCH_REMOVE,      // 本地批量删除
TP_MMC_LOCAL_PUT_WAIT_FUTURE,   // 本地 Put 等待 Future
TP_MMC_LOCAL_GET_WAIT_FUTURE,   // 本地 Get 等待 Future
```

**说明**: 本地存储操作的性能跟踪点

---

### 3. 元数据管理器跟踪点

```cpp
TP_MMC_META_MGR_ALLOC,          // 元数据管理器分配
TP_MMC_META_MGR_GET,            // 元数据管理器获取
```

**说明**: 元数据管理器的核心操作

---

### 4. 元数据操作跟踪点

```cpp
TP_MMC_META_PUT,                // 元数据写入
TP_MMC_META_BATCH_PUT,          // 元数据批量写入
TP_MMC_META_GET,                // 元数据读取
TP_MMC_META_BATCH_GET,          // 元数据批量读取
TP_MMC_META_UPDATE,             // 元数据更新
TP_MMC_META_BATCH_UPDATE,       // 元数据批量更新
TP_MMC_META_REMOVE,             // 元数据删除
TP_MMC_META_BATCH_REMOVE,       // 元数据批量删除
TP_MMC_META_REMOVE_ALL,         // 元数据全部删除
TP_MMC_META_EXIST,              // 元数据存在性检查
TP_MMC_META_BATCH_EXIST,        // 元数据批量存在性检查
TP_MMC_META_QUERY,              // 元数据查询
TP_MMC_META_BATCH_QUERY,        // 元数据批量查询
TP_MMC_META_BM_REGISTER,        // 元数据 Buffer Manager 注册
TP_MMC_META_BM_UNREGISTER,      // 元数据 Buffer Manager 注销
TP_MMC_META_CLEAR_RESOURCE,     // 元数据清除资源
```

**说明**: 元数据服务的各种操作跟踪点

---

### 5. SMEM BM 跟踪点

```cpp
TP_SMEM_BM_PUT,                 // SMEM BM 单个数据写入
TP_SMEM_BM_PUT_2D,              // SMEM BM 2D 数据写入
TP_SMEM_BM_GET,                 // SMEM BM 单个数据读取
TP_SMEM_BM_GET_2D,              // SMEM BM 2D 数据读取
```

**说明**: 共享内存 Buffer Manager 的跟踪点

---

### 6. 加速器发送跟踪点

```cpp
TP_ACC_SEND_ALLOC,              // 加速器发送分配请求
TP_ACC_SEND_UPDATE,             // 加速器发送更新请求
TP_ACC_SEND_GET,                // 加速器发送获取请求
TP_ACC_SEND_REMOVE,             // 加速器发送删除请求
TP_ACC_SEND_EXIST,              // 加速器发送存在性检查
TP_ACC_SEND_QUERY,              // 加速器发送查询
TP_ACC_SEND_ALLOC_BAT,          // 加速器批量发送分配
TP_ACC_SEND_UPDATE_BAT,         // 加速器批量发送更新
TP_ACC_SEND_GET_BAT,            // 加速器批量发送获取
TP_ACC_SEND_REMOVE_BAT,         // 加速器批量发送删除
TP_ACC_SEND_EXIST_BAT,          // 加速器批量发送存在性检查
TP_ACC_SEND_QUERY_BAT,          // 加速器批量发送查询
```

**说明**: 加速器通信的发送操作跟踪点

---

### 7. 加速器等待跟踪点

```cpp
TP_ACC_SEND_WAIT_ALLOC,         // 等待分配响应
TP_ACC_SEND_WAIT_UPDATE,        // 等待更新响应
TP_ACC_SEND_WAIT_GET,           // 等待获取响应
TP_ACC_SEND_WAIT_REMOVE,        // 等待删除响应
TP_ACC_SEND_WAIT_EXIST,         // 等待存在性检查响应
TP_ACC_SEND_WAIT_QUERY,         // 等待查询响应
TP_ACC_SEND_WAIT_ALLOC_BAT,     // 等待批量分配响应
TP_ACC_SEND_WAIT_UPDATE_BAT,    // 等待批量更新响应
TP_ACC_SEND_WAIT_GET_BAT,       // 等待批量获取响应
TP_ACC_SEND_WAIT_REMOVE_BAT,    // 等待批量删除响应
TP_ACC_SEND_WAIT_EXIST_BAT,     // 等待批量存在性检查响应
TP_ACC_SEND_WAIT_QUERY_BAT,     // 等待批量查询响应
TP_ACC_SEND_BATCH_GET,          // 批量获取
```

**说明**: 加速器通信的等待响应跟踪点

---

### 8. 客户端跟踪点

```cpp
TP_MMC_CLIENT_BATCH_PUT,        // 客户端批量写入
TP_MMC_CLIENT_BATCH_GET,        // 客户端批量读取
TP_MMC_CLIENT_GET,              // 客户端单个读取
TP_MMC_CLIENT_PUT,              // 客户端单个写入
```

**说明**: 客户端操作的跟踪点

---

## 文件级别的关系图

```
mmc_ptracer.h
    |
    +-- TP_MMC_MOD (跟踪点枚举)
    |   |
    |   +-- Python 接口跟踪点
    |   |   +-- PUT/GET/UPDATE/REMOVE/EXIST/QUERY
    |   |   +-- 单个/批量/层/2D 变体
    |   |
    |   +-- 本地存储跟踪点
    |   |   +-- PUT/GET/UPDATE/REMOVE
    |   |   +-- 批量操作
    |   |   +-- 同步等待
    |   |
    |   +-- 元数据跟踪点
    |   |   +-- 管理器操作
    |   |   +-- CRUD 操作
    |   |   +-- BM 注册/注销
    |   |
    |   +-- SMEM BM 跟踪点
    |   +-- 加速器跟踪点
    |   +-- 客户端跟踪点
    |
    +-- 依赖 ptracer.h
```

---

## 使用示例

### 定义跟踪点

```cpp
#include "mmc_ptracer.h"

void PutData(const std::string& key, const void* data, size_t size) {
    TP_BEGIN(TP_MMC_PY_PUT);
    // 执行 put 操作
    TP_END(TP_MMC_PY_PUT);
}
```

### 批量操作跟踪

```cpp
void BatchPut(const std::vector<std::string>& keys, const std::vector<void*>& data) {
    TP_BEGIN(TP_MMC_PY_BATCH_PUT);
    // 执行批量 put 操作
    TP_END(TP_MMC_PY_BATCH_PUT);
}
```

### 嵌套跟踪

```cpp
void ProcessRequest() {
    TP_BEGIN(TP_MMC_PY_PUT);
    {
        TP_BEGIN(TP_MMC_LOCAL_PUT);
        // 本地存储操作
        TP_END(TP_MMC_LOCAL_PUT);
    }
    TP_END(TP_MMC_PY_PUT);
}
```

---

## 跟踪点命名规则

```
TP_MMC_<模块>_<操作>_<变体>
     |    |      |      |
     |    |      |      +-- 变体（可选）
     |    |      +-- 操作类型（PUT/GET/UPDATE/REMOVE/EXIST/QUERY）
     |    +-- 模块名（PY/LOCAL/META/ACC/CLIENT）
     +-- 固定前缀
```

**示例**:
- `TP_MMC_PY_PUT` - Python 模块的 Put 操作
- `TP_MMC_LOCAL_BATCH_GET` - 本地模块的批量 Get 操作
- `TP_MMC_META_UPDATE` - 元数据模块的 Update 操作

---

## 注意事项

1. **性能影响**: 跟踪点通常在发布版本中编译为空代码
2. **ID 唯一性**: 每个跟踪点必须有唯一的 ID
3. **嵌套支持**: 支持跟踪点的嵌套调用
4. **编译控制**: 通过 `UT_ENABLED` 宏控制使用哪个 ptracer 实现
5. **宏定义**: `TP_BEGIN()` 和 `TP_END()` 是 ptracer 提供的宏，不在本文件中定义
