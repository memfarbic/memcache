# python_wrapper 模块文档

## 模块概述

`python_wrapper` 模块提供 Python 绑定，使用 pybind11 将 C++ 功能暴露给 Python，使 Python 应用能够直接使用 MemCache_Hybrid 的分布式对象存储功能。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/python_wrapper/`

## 文件列表

### 头文件 (.h)
- `pymmc.h` - Python 绑定头文件

### 源文件 (.cpp)
- `pymmc.cpp` - Python 绑定实现

### 构建文件
- `CMakeLists.txt` - CMake 构建配置

---

## 详细文档

### pymmc.h

**功能**: 定义 Python 模块的依赖头文件。

**包含内容**:
```cpp
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wmissing-field-initializers"
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#pragma GCC diagnostic pop

#include <csignal>
#include <mutex>
#include <string>
#include <unordered_set>
#include <sstream>

#include "mmc_def.h"
#include "mmc_types.h"
#include "mmcache.h"
#include "mmcache_store.h"
```

---

### pymmc.cpp

**功能**: 实现 Python 模块绑定，定义 `_pymmc` 扩展模块。

**PYBIND11_MODULE 定义**:

```cpp
PYBIND11_MODULE(_pymmc, m)
{
    // 模块定义
}
```

---

## Python API

### 模块名称

Python 扩展模块名称为 `_pymmc`，通常通过包装器模块 `memcache_hybrid` 导入。

### 类定义

#### KeyInfo

键信息类，提供对象的位置和类型信息。

```python
class KeyInfo:
    def size(self) -> int:
        """获取对象总大小"""

    def loc_list(self) -> List[int]:
        """获取位置列表 (Rank ID 列表)"""

    def type_list(self) -> List[int]:
        """获取介质类型列表"""

    def __str__(self) -> str:
        """字符串表示"""

    def __repr__(self) -> str:
        """调试字符串表示"""
```

#### ReplicateConfig

副本配置类，控制对象的副本分配策略。

```python
class ReplicateConfig:
    replicaNum: int  # 副本数量，<= 8
    preferredLocalServiceIDs: List[int]  # 强制存储的实例 ID 列表
```

**默认值**:
- `replicaNum = 0`
- `preferredLocalServiceIDs = []`

#### DistributedObjectStore

分布式对象存储主类。

```python
class DistributedObjectStore:
    def init(self, device_id: int, init_bm: bool = True) -> None:
        """初始化存储

        Args:
            device_id: 设备 ID
            init_bm: 是否初始化 Blob Manager
        """

    def remove(self, key: str) -> None:
        """删除单个对象"""

    def remove_batch(self, keys: List[str]) -> None:
        """批量删除对象"""

    def remove_all(self) -> None:
        """删除所有对象"""

    def is_exist(self, key: str) -> bool:
        """检查对象是否存在"""

    def batch_is_exist(self, keys: List[str]) -> List[int]:
        """批量检查对象是否存在

        Returns:
            List[int]: 1=存在, 0=不存在, -1=错误
        """

    def get_key_info(self, key: str) -> KeyInfo:
        """获取对象的键信息"""

    def batch_get_key_info(self, keys: List[str]) -> List[KeyInfo]:
        """批量获取键信息"""

    def close(self) -> None:
        """关闭存储"""

    def register_buffer(self, buffer_ptr: int, size: int) -> None:
        """注册内存缓冲区用于 RDMA 操作

        Args:
            buffer_ptr: 缓冲区地址 (整数)
            size: 缓冲区大小
        """

    def unregister_buffer(self, buffer_ptr: int, size: int) -> None:
        """注销内存缓冲区"""

    def get_into(self, key: str, buffer_ptr: int, size: int, direct: int = 2) -> None:
        """获取对象数据到预分配缓冲区

        Args:
            key: 对象键
            buffer_ptr: 缓冲区地址
            size: 缓冲区大小
            direct: 拷贝方向 (默认 G2H=2)
        """

    def batch_get_into(self, keys: List[str], buffer_ptrs: List[int],
                       sizes: List[int], direct: int = 2) -> None:
        """批量获取对象数据到预分配缓冲区"""

    def get_into_layers(self, key: str, buffer_ptrs: List[int],
                        sizes: List[int], direct: int = 2) -> None:
        """获取分层对象数据到预分配缓冲区"""

    def batch_get_into_layers(self, keys: List[str], buffer_ptrs: List[List[int]],
                              sizes: List[List[int]], direct: int = 2) -> None:
        """批量获取分层对象数据"""

    def get_local_service_id(self) -> int:
        """获取本地服务 ID"""

    def put_from(self, key: str, buffer_ptr: int, size: int,
                 direct: int = 3, config: ReplicateConfig = None) -> None:
        """从预分配缓冲区存储对象数据

        Args:
            key: 对象键
            buffer_ptr: 缓冲区地址
            size: 数据大小
            direct: 拷贝方向 (默认 H2G=3)
            config: 副本配置
        """

    def batch_put_from(self, keys: List[str], buffer_ptrs: List[int],
                       sizes: List[int], direct: int = 3,
                       config: ReplicateConfig = None) -> None:
        """批量从预分配缓冲区存储对象数据"""

    def put_from_layers(self, key: str, buffer_ptrs: List[int],
                        sizes: List[int], direct: int = 3,
                        config: ReplicateConfig = None) -> None:
        """存储分层对象数据"""

    def batch_put_from_layers(self, keys: List[str], buffer_ptrs: List[List[int]],
                              sizes: List[List[int]], direct: int = 3,
                              config: ReplicateConfig = None) -> None:
        """批量存储分层对象数据"""
```

#### MetaService

元服务类，用于启动元服务进程。

```python
class MetaService:
    @staticmethod
    def main() -> None:
        """直接启动元服务进程。这是一个阻塞调用。"""
```

---

## 拷贝方向常量

| 常量 | 值 | 说明 |
|------|---|------|
| `SMEMB_COPY_L2G` | 0 | Local to Global |
| `SMEMB_COPY_G2L` | 1 | Global to Local |
| `SMEMB_COPY_G2H` | 2 | Global to Host |
| `SMEMB_COPY_H2G` | 3 | Host to Global |
| `SMEMB_COPY_G2G` | 4 | Global to Global |

---

## 使用示例

### 基本使用

```python
from memcache_hybrid import DistributedObjectStore
import numpy as np

# 创建存储实例
store = DistributedObjectStore()

# 初始化
store.init(device_id=0)

# 存储数据
data = np.random.rand(1000, 1000).astype(np.float32)
store.put("my_key", data)

# 获取数据
result = store.get("my_key")

# 删除数据
store.remove("my_key")

# 关闭存储
store.close()
```

### 使用直接内存访问

```python
from memcache_hybrid import DistributedObjectStore
import ctypes

store = DistributedObjectStore()
store.init(device_id=0)

# 分配内存
data = (ctypes.c_float * 1000)()
# 填充数据...

# 注册内存缓冲区
buffer_ptr = ctypes.addressof(data)
store.register_buffer(buffer_ptr, ctypes.sizeof(data))

# 直接存储
from memcache_hybrid import ReplicateConfig
config = ReplicateConfig()
config.replicaNum = 2
store.put_from("direct_key", buffer_ptr, ctypes.sizeof(data),
               direct=3, config=config)  # H2G

# 直接获取
store.get_into("direct_key", buffer_ptr, ctypes.sizeof(data),
               direct=2)  # G2H

# 注销缓冲区
store.unregister_buffer(buffer_ptr, ctypes.sizeof(data))
store.close()
```

### 批量操作

```python
from memcache_hybrid import DistributedObjectStore

store = DistributedObjectStore()
store.init(device_id=0)

# 批量存储
keys = [f"key_{i}" for i in range(100)]
data_list = [...]  # 100 个数据对象
store.batch_put(keys, data_list)

# 批量获取
results = store.batch_get(keys)

# 批量检查存在性
existence = store.batch_is_exist(keys)
# existence[i]: 1=存在, 0=不存在, -1=错误

# 批量删除
store.remove_batch(keys)
store.close()
```

### 分层存储

```python
from memcache_hybrid import DistributedObjectStore

store = DistributedObjectStore()
store.init(device_id=0)

# 分层存储 (如 Transformer 的各层)
layer_data = [...]  # 每层的数据
buffer_ptrs = [ctypes.addressof(data) for data in layer_data]
sizes = [ctypes.sizeof(data) for data in layer_data]

store.put_from_layers("transformer_layers", buffer_ptrs, sizes)

# 分层获取
output_buffers = [...]  # 预分配的输出缓冲区
output_ptrs = [ctypes.addressof(buf) for buf in output_buffers]
output_sizes = [ctypes.sizeof(buf) for buf in output_buffers]

store.get_into_layers("transformer_layers", output_ptrs, output_sizes)
store.close()
```

### 副本配置

```python
from memcache_hybrid import DistributedObjectStore, ReplicateConfig

store = DistributedObjectStore()
store.init(device_id=0)

# 配置副本策略
config = ReplicateConfig()
config.replicaNum = 3  # 3 个副本
config.preferredLocalServiceIDs = [0, 1, 2]  # 指定存储到 Rank 0, 1, 2

# 使用副本配置存储
data = np.random.rand(100, 100).astype(np.float32)
store.put("replicated_key", data, replicate_config=config)

# 获取键信息查看副本位置
info = store.get_key_info("replicated_key")
print(f"Locations: {info.loc_list()}")  # [0, 1, 2]
print(f"Types: {info.type_list()}")      # 介质类型列表

store.close()
```

### 启动元服务

```python
from memcache_hybrid import MetaService

# 启动元服务 (阻塞调用)
# 通常在独立的 Python 进程中运行
MetaService.main()
```

---

## 错误处理

所有函数在失败时会抛出异常：

```python
try:
    store.get("non_existent_key")
except RuntimeError as e:
    print(f"Error: {e}")
```

---

## 性能考虑

1. **GIL 释放**: 大多数操作会释放 Python GIL，允许其他线程运行
2. **批量操作**: 批量 API 比多次调用单个 API 更高效
3. **直接内存访问**: 使用 `put_from`/`get_into` 避免额外的数据拷贝
4. **内存注册**: 对于频繁访问的内存，使用 `register_buffer` 提高性能

---

## 构建和安装

```bash
# 构建
mkdir build && cd build
cmake ..
make pymmc

# 安装
pip install -e .
```
