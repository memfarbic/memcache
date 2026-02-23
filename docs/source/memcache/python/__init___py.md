# __init__.py 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/python/memcache_hybrid/__init__.py`
- **文件用途**: Python 包入口，加载 C++ 扩展库并导出主要类和函数
- **依赖项**: `os`, `sys`, `ctypes`, `memfabric_hybrid`

---

## 模块导入

### 标准库导入 (第15-22行)

```python
import os
import sys
import ctypes

import memfabric_hybrid

current_dir = os.path.dirname(os.path.abspath(__file__))
sys.path.append(current_dir)
```

**功能**:
- 导入 os, sys, ctypes 模块
- 导入 memfabric_hybrid 依赖
- 将当前目录添加到 sys.path

---

## C++ 扩展库加载

### 库文件路径 (第24-27行)

```python
lib_dir = os.path.join(current_dir, 'lib')
lib_list = ['libmf_memcache.so']
for lib_source in lib_list:
    ctypes.CDLL(os.path.join(lib_dir, lib_source))
```

**声明位置**: 行 24-27

**功能**: 预加载 C++ 扩展库

**库文件**: `libmf_memcache.so`

**路径**: `{current_dir}/lib/libmf_memcache.so`

**说明**: 使用 ctypes.CDLL 预加载共享库，确保后续导入的 _pymmc 模块可以找到依赖

---

## 模块导出

### __all__ 声明 (第13行)

```python
__all__ = ["DistributedObjectStore", "ReplicateConfig", "KeyInfo", "MetaService"]
```

**声明位置**: 行 13

**功能**: 定义公开的 API 列表

**导出的类**:
- `DistributedObjectStore`: 分布式对象存储主类
- `ReplicateConfig`: 副本配置类
- `KeyInfo`: 键信息类
- `MetaService`: 元数据服务类

---

### 实际导入 (第29行)

```python
from _pymmc import DistributedObjectStore, ReplicateConfig, KeyInfo, MetaService
```

**声明位置**: 行 29

**功能**: 从 _pymmc 模块导入主要类

**说明**: _pymmc 是由 Pybind11 生成的 C++ 扩展模块

---

## 模块架构

```
memcache_hybrid/
├── __init__.py              # 包入口（本文件）
├── meta_service_leader_election.py  # Leader 选举
├── lib/
│   └── libmf_memcache.so    # C++ 扩展库
├── _pymmc.so                # Pybind11 绑定（动态生成）
└── ...
```

---

## 加载流程

```
import memcache_hybrid
    |
    v
执行 __init__.py
    |
    +-- 设置 sys.path
    |
    +-- 预加载 libmf_memcache.so
    |
    v
导入 _pymmc 模块
    |
    v
导出主要类
    |
    v
用户可用: DistributedObjectStore, ReplicateConfig, KeyInfo, MetaService
```

---

## 导出的类说明

### DistributedObjectStore

分布式对象存储类，提供数据存储和检索功能。

**主要方法**:
- `CreateObjectStore()`: 创建对象存储实例
- `Init(device_id, init_bm)`: 初始化
- `TearDown()`: 清理
- `PutFrom()`, `GetInto()`: 数据存取
- `PutFromLayers()`, `GetIntoLayers()`: 分层数据存取
- `BatchPutFrom()`, `BatchGetInto()`: 批量操作
- `Remove()`, `BatchRemove()`: 删除操作
- `IsExist()`, `BatchIsExist()`: 存在性检查
- `GetKeyInfo()`, `BatchGetKeyInfo()`: 获取元信息

### ReplicateConfig

副本配置类，用于控制数据副本策略。

**成员**:
- `replicaNum`: 副本数量 (1-8)
- `preferredLocalServiceIDs`: 首选本地服务 ID 列表

### KeyInfo

键信息类，存储数据对象的元信息。

**方法**:
- `Size()`: 获取数据大小
- `GetBlobNum()`: 获取 Blob 数量
- `GetLocs()`: 获取位置列表
- `GetTypes()`: 获取介质类型列表

### MetaService

元数据服务类，用于管理元数据服务。

**方法**:
- 服务启停
- 配置管理
- Leader 选举

---

## 使用示例

### 基础用法

```python
from memcache_hybrid import DistributedObjectStore

# 创建对象存储
store = DistributedObjectStore.CreateObjectStore()
store.Init(device_id=0, init_bm=True)

# 存储数据
import numpy as np
data = np.random.rand(1024, 1024).astype(np.float32)
store.PutFrom("model", data.ctypes.data, data.nbytes, 3)

# 获取数据
output = np.zeros((1024, 1024), dtype=np.float32)
store.GetInto("model", output.ctypes.data, output.nbytes, 2)

# 清理
store.TearDown()
```

### 副本配置

```python
from memcache_hybrid import DistributedObjectStore, ReplicateConfig

store = DistributedObjectStore.CreateObjectStore()

# 配置副本策略
replicate_config = ReplicateConfig()
replicate_config.replicaNum = 2  # 2副本
replicate_config.preferredLocalServiceIDs = [0, 1]  # 首选 Rank 0 和 1

# 使用副本配置存储
store.PutFrom("key", buffer, size, 3, replicate_config)
```

### 查询元信息

```python
from memcache_hybrid import DistributedObjectStore

store = DistributedObjectStore.CreateObjectStore()

# 获取键信息
info = store.GetKeyInfo("model")
print(f"Size: {info.Size()}")
print(f"Blobs: {info.GetBlobNum()}")
print(f"Locations: {info.GetLocs()}")
print(f"Types: {info.GetTypes()}")
```

---

## 依赖关系

```
__init__.py
    |
    +-- 依赖: ctypes (库加载)
    +-- 依赖: memfabric_hybrid (依赖包)
    |
    +-- 加载: libmf_memcache.so (C++ 扩展)
    +-- 导入: _pymmc.so (Pybind11 绑定)
```

---

## 注意事项

1. **库路径**: 确保 `lib/libmf_memcache.so` 存在于包目录中
2. **Python 版本**: 需要 Python 3.7+
3. **平台兼容性**: 库文件与平台相关 (manylinux)
4. **初始化**: 使用前必须调用 `Init()`
5. **清理**: 使用完毕后应调用 `TearDown()`
