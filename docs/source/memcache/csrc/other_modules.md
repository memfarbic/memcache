# ha 模块文档

## 模块概述

`ha` 模块提供高可用性支持，包括 Leader 选举等功能。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/ha/`

## 文件列表

- `mmc_leader_election.h` / `mmc_leader_election.cpp` - Leader 选举
- `mmc_leader.h` / `mmc_leader.cpp` - Leader 管理

---

## 功能说明

### mmc_leader_election.h

Leader 选举机制，用于在多个元数据服务实例中选择主节点。

**主要类**:
```cpp
class MmcLeaderElection : public MmcReferable {
public:
    // 选举相关方法
    Result StartElection();
    Result StopElection();
    bool IsLeader();
    uint32_t GetLeaderId();

private:
    uint32_t currentLeaderId_;
    // ...
};
```

### mmc_leader.h

Leader 管理，处理 Leader 的选举和状态管理。

---

## 使用场景

- 多个元数据服务实例部署时，选举一个主节点
- 主节点负责处理关键的元数据操作
- 从节点作为备份，可随时接管

---

## under_api 模块文档

## 模块概述

`under_api` 模块封装底层 SMEM (Shared Memory) API。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/under_api/`

## 文件列表

- `mf_smem/smem_bm_api.h` / `smem_bm_api.cpp` - Blob Manager API
- `mf_smem/smem_bm_def.h` - Blob Manager 定义

---

## smem_bm_api.h

### SMEM BM API

封装华为 SMEM Blob Manager 的 API:

```cpp
// Blob Manager 初始化
smem_bm_t smem_bm_init(const smem_bm_config_t *config);

// Blob Manager 销毁
int smem_bm_destroy(smem_bm_t bm);

// 内存分配
uint64_t smem_bm_alloc(smem_bm_t bm, uint64_t size, smem_bm_media_type_t type);

// 内存释放
int smem_bm_free(smem_bm_t bm, uint64_t gva);

// 数据拷贝
int smem_bm_copy(smem_bm_t bm, uint64_t src, uint64_t dst,
                uint64_t size, smem_bm_copy_type_t type);

// 批量操作
int smem_bm_batch_put(...);
int smem_bm_batch_get(...);

// 缓冲区注册
int smem_bm_register_buffer(smem_bm_t bm, const void *addr, uint64_t size);
int smem_bm_unregister_buffer(smem_bm_t bm, const void *addr);
```

---

## log 模块文档

## 模块概述

`log` 模块提供日志记录功能，基于 spdlog 实现。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/log/`

## 文件列表

- `spdlogger.h` / `spdlogger.cpp` - C++ 日志封装
- `spdlogger4c.h` / `spdlogger4c.cpp` - C 日志封装

---

## spdlogger.h

### SpdLogger 类

基于 spdlog 的日志封装:
```cpp
class SpdLogger {
public:
    static SpdLogger &Instance();

    void Init(const std::string &logPath, int32_t logLevel,
             int32_t rotationFileSize, int32_t rotationFileCount);

    void SetLogLevel(int32_t level);
    void Log(int32_t level, const std::string &msg);

private:
    std::shared_ptr<spdlog::logger> logger_;
    int32_t logLevel_;
};
```

**日志级别**:
- 0: DEBUG
- 1: INFO
- 2: WARN
- 3: ERROR

---

## daemon 模块文档

## 模块概述

`daemon` 模块提供守护进程功能。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/daemon/`

## 文件列表

- `mmc_daemon.cpp` - 守护进程实现

---

## mmc_daemon.cpp

### 功能

实现 MemCache 服务的守护进程模式，支持:
- 后台运行
- 进程监控
- 自动重启

---

## python_wrapper 模块文档

## 模块概述

`python_wrapper` 模块提供 Python 绑定，使用 Pybind11 实现。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/python_wrapper/`

## 文件列表

- `pymmc.h` / `pymmc.cpp` - Python 绑定实现

---

## pymmc.cpp

### Python 绑定

使用 Pybind11 暴露 C++ 接口到 Python:

```cpp
// 模块定义
PYBIND11_MODULE(pymmc, m) {
    // 暴露 ObjectStore 类
    py::class_<ock::mmc::ObjectStore>(m, "ObjectStore")
        .def(py::init<>())
        .def("CreateObjectStore", &ObjectStore::CreateObjectStore)
        .def("Init", &ObjectStore::Init)
        .def("TearDown", &ObjectStore::TearDown)
        .def("RegisterBuffer", &ObjectStore::RegisterBuffer)
        .def("UnRegisterBuffer", &ObjectStore::UnRegisterBuffer)
        .def("GetInto", &ObjectStore::GetInto)
        .def("BatchGetInto", &ObjectStore::BatchGetInto)
        .def("GetIntoLayers", &ObjectStore::GetIntoLayers)
        .def("BatchGetIntoLayers", &ObjectStore::BatchGetIntoLayers)
        .def("PutFrom", &ObjectStore::PutFrom)
        .def("BatchPutFrom", &ObjectStore::BatchPutFrom)
        .def("PutFromLayers", &ObjectStore::PutFromLayers)
        .def("BatchPutFromLayers", &ObjectStore::BatchPutFromLayers)
        .def("Remove", &ObjectStore::Remove)
        .def("BatchRemove", &ObjectStore::BatchRemove)
        .def("RemoveAll", &ObjectStore::RemoveAll)
        .def("IsExist", &ObjectStore::IsExist)
        .def("BatchIsExist", &ObjectStore::BatchIsExist)
        .def("GetKeyInfo", &ObjectStore::GetKeyInfo)
        .def("BatchGetKeyInfo", &ObjectStore::BatchGetKeyInfo)
        .def("GetLocalServiceId", &ObjectStore::GetLocalServiceId);
}
```

**Python 使用示例**:
```python
import pymmc

# 创建对象存储
store = pymmc.ObjectStore.CreateObjectStore()
store.Init(device_id=0, init_bm=True)

# 存储数据
store.PutFrom("key", buffer, size, 3)

# 获取数据
store.GetInto("key", buffer, size, 2)

# 清理
store.TearDown()
```
