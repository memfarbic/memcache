# Python 模块文档

## 模块概述

Python 模块提供 Python 绑定，使 MemCache_Hybrid 可以在 Python 中使用。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/python/`

## 文件列表

- `memcache_hybrid/__init__.py` - 包入口
  - [__init__.py 详细文档](__init___py.md)
- `memcache_hybrid/meta_service_leader_election.py` - Leader 选举
  - [meta_service_leader_election.py 详细文档](meta_service_leader_election_py.md)
- `setup.py` - 安装配置
  - [setup.py 详细文档](setup_py.md)

---

## __init__.py

### 功能

Python 包的入口点，负责:
1. 加载 C++ 扩展库 `libmf_memcache.so`
2. 导出主要类和函数

### 导出的类

```python
from _pymmc import DistributedObjectStore, ReplicateConfig, KeyInfo, MetaService
```

**说明**: `_pymmc` 是由 Pybind11 生成的 C++ 扩展模块。

---

## meta_service_leader_election.py

### MetaServiceLeaderElection 类

基于 Kubernetes Lease 机制实现 Leader 选举。

**构造函数**:
```python
def __init__(self, lease_name, namespace, pod_name, retry_period=3, log_level=1, log_path="/home/memcache")
```

**参数**:
- `lease_name`: Lease 资源名称
- `namespace`: Kubernetes 命名空间
- `pod_name`: 当前 Pod 名称
- `retry_period`: 重试间隔(秒)

**主要方法**:

| 方法 | 功能 |
|------|------|
| `start_election()` | 开始选举循环 |
| `stop_election()` | 停止选举 |
| `update_lease(is_renew=False)` | 更新/续约 Lease |
| `check_leader_status()` | 检查 Leader 状态 |
| `update_pod_to_master()` | 升主时更新 Pod 标签 |
| `update_pod_to_backup()` | 降备时更新 Pod 标签 |

**选举流程**:
1. 定期检查 Lease 状态
2. 如果 Lease 无持有者或已过期，尝试竞选
3. 竞选成功后更新为 master 标签
4. 定期续约保持 Leader 身份
5. 失去 Leader 后更新为 backup 标签

**使用示例**:
```python
from memcache_hybrid import MetaServiceLeaderElection
import os

election = MetaServiceLeaderElection(
    lease_name=os.getenv("META_LEASE_NAME", "default"),
    namespace=os.getenv("META_NAMESPACE", "default"),
    pod_name=os.getenv("META_POD_NAME", "meta-pod-0"),
    retry_period=5
)

election.start_election()
```

---

## Python 使用示例

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

# 查询信息
info = store.GetKeyInfo("model")
print(f"Size: {info.Size()}, Blobs: {info.GetBlobNum()}")

# 清理
store.TearDown()
```

### 分层存储

```python
# 存储模型层
layers = [layer1_data, layer2_data, layer3_data]
sizes = [len(layer1_data), len(layer2_data), len(layer3_data)]
buffers = [layer1_ctypes, layer2_ctypes, layer3_ctypes]

store.PutFromLayers("model", buffers, sizes, 3)

# 获取模型层
output_buffers = [out1_ctypes, out2_ctypes, out3_ctypes]
output_sizes = [out1_size, out2_size, out3_size]
store.GetIntoLayers("model", output_buffers, output_sizes, 2)
```

### 批量操作

```python
keys = ["model1", "model2", "model3"]
buffers = [buf1, buf2, buf3]
sizes = [size1, size2, size3]

# 批量存储
results = store.BatchPutFrom(keys, buffers, sizes, 3)

# 批量获取
results = store.BatchGetInto(keys, out_buffers, out_sizes, 2)
```
