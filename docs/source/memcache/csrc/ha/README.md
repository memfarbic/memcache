# ha 模块文档

## 模块概述

`ha` 模块实现元服务的高可用（High Availability）功能，通过 Leader 选举机制实现主备切换，确保元服务的连续性和可靠性。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/ha/`

## 文件列表

### 头文件 (.h)
- `mmc_leader_election.h` - Leader 选举类定义

### 源文件 (.cpp)
- `mmc_leader_election.cpp` - Leader 选举实现
- `mmc_leader.cpp` - Leader 相关功能实现

---

## 详细文档

### mmc_leader_election.h

**功能**: 定义元服务 Leader 选举类，实现基于 Kubernetes Lease 的主备选举机制。

**MmcMetaServiceLeaderElection 类**:
```cpp
class MmcMetaServiceLeaderElection {
public:
    // 构造函数
    explicit MmcMetaServiceLeaderElection(const std::string &name,
                                          const std::string &pod,
                                          const std::string &ns,
                                          const std::string &lease);

    // 析构函数
    ~MmcMetaServiceLeaderElection();

    // 启动选举
    Result Start(const mmc_meta_service_config_t &options);

    // 停止选举
    void Stop();

private:
    // 选举循环
    void ElectionLoop();

    // 检查 Leader 状态
    void CheckLeaderStatus();

    // 续约循环
    void RenewLoop();

    // 开始成为 Leader 时的回调
    void OnStartLeading();

    // 停止成为 Leader 时的回调
    void OnStopLeading();

    // 成员变量
    std::mutex mutex_;
    std::thread electionThread_;      // 选举线程
    std::thread renewThread_;         // 续约线程
    std::atomic<bool> running_{false}; // 运行状态
    std::atomic<bool> isLeader_{false}; // 是否为 Leader
    std::string podName_;             // Pod 名称
    std::string leaseName_;           // Lease 名称
    std::string ns_;                  // 命名空间
    std::string name_;                // 实例名称
    pybind11::object leaderElection_; // Python Leader 选举对象
};
```

**常量定义**:
```cpp
constexpr uint32_t LEASE_RETRY_PERIOD = 3;  // Lease 重试周期（秒）
```

**Python 模块**:
```cpp
std::string g_leaderElectionModule = "memcache_hybrid.meta_service_leader_election";
```

---

### mmc_leader_election.cpp

**功能**: Leader 选举的具体实现。

**MmcMetaServiceLeaderElection::Start**

```cpp
Result MmcMetaServiceLeaderElection::Start(const mmc_meta_service_config_t &options)
```

**功能**: 启动 Leader 选举

**参数**:
- `options`: 元服务配置选项

**返回值**: `Result` - 成功返回 `MMC_OK`

**代码逻辑**:
1. 检查是否已启动，如果已启动则直接返回
2. 验证环境变量（Pod 名称、Lease 名称、命名空间）
3. 导入 Python Leader 选举模块
4. 创建 Python `MetaServiceLeaderElection` 对象
5. 验证 Python 对象的方法存在性
6. 启动选举线程和续约线程

**异常处理**: 捕获所有异常并返回错误

---

**MmcMetaServiceLeaderElection::ElectionLoop**

```cpp
void MmcMetaServiceLeaderElection::ElectionLoop()
```

**功能**: 选举循环线程，定期检查 Leader 状态

**代码逻辑**:
1. 每 `LEASE_RETRY_PERIOD` 秒执行一次
2. 调用 `CheckLeaderStatus()` 检查状态
3. 根据状态变化触发 `OnStartLeading()` 或 `OnStopLeading()`

**线程安全**: 使用 GIL 保护 Python 调用

---

**MmcMetaServiceLeaderElection::CheckLeaderStatus**

```cpp
void MmcMetaServiceLeaderElection::CheckLeaderStatus()
```

**功能**: 检查当前 Pod 是否为 Leader

**代码逻辑**:
1. 调用 Python 的 `check_leader_status()` 获取当前 Leader
2. 如果当前 Pod 是 Leader 且 `isLeader_` 为 false，调用 `OnStartLeading()`
3. 如果当前 Pod 不是 Leader 且 `isLeader_` 为 true，调用 `OnStopLeading()`
4. 如果当前 Pod 不是 Leader，尝试调用 `update_lease(false)` 获取 Leader 权

---

**MmcMetaServiceLeaderElection::RenewLoop**

```cpp
void MmcMetaServiceLeaderElection::RenewLoop()
```

**功能**: Leader 续约循环线程

**代码逻辑**:
1. 只有 Leader 才执行续约
2. 每 `LEASE_RETRY_PERIOD` 秒执行一次续约
3. 调用 Python 的 `update_lease(true)` 续约
4. 如果续约失败，调用 `OnStopLeading()` 降级为 Backup

---

**MmcMetaServiceLeaderElection::OnStartLeading**

```cpp
void MmcMetaServiceLeaderElection::OnStartLeading()
```

**功能**: 成为 Leader 时的回调

**代码逻辑**: 调用 Python 的 `update_pod_to_master()` 更新 Pod 状态为 Master

---

**MmcMetaServiceLeaderElection::OnStopLeading**

```cpp
void MmcMetaServiceLeaderElection::OnStopLeading()
```

**功能**: 停止成为 Leader 时的回调

**代码逻辑**: 调用 Python 的 `update_pod_to_backup()` 更新 Pod 状态为 Backup

---

**MmcMetaServiceLeaderElection::Stop**

```cpp
void MmcMetaServiceLeaderElection::Stop()
```

**功能**: 停止选举

**代码逻辑**:
1. 设置 `running_` 为 false
2. 等待选举线程和续约线程结束

---

## 数据流和关系

```
┌─────────────────────────────────────────────────────────────┐
│          MmcMetaServiceLeaderElection                        │
│                    (Leader 选举)                              │
└───────────────────────────┬─────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
┌──────────────────┐ ┌─────────────┐ ┌──────────────┐
│ ElectionLoop     │ │ RenewLoop   │ │ Python       │
│ (选举循环)         │ │ (续约循环)    │ │ Leader       │
│                  │ │             │ │ Election     │
│ - 检查Leader状态   │ │ - 续约Lease  │ │ 模块          │
│ - 触发回调         │ │ - 失败降级    │ │              │
└──────────────────┘ └─────────────┘ └──────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Kubernetes    │
                    │ Lease API     │
                    └───────────────┘
```

---

## 状态转换图

```
                    ┌─────────────┐
                    │   Initial   │
                    └──────┬──────┘
                           │ Start()
                           ▼
                    ┌─────────────┐
                    │   Running   │◄────────────────┐
                    └──────┬──────┘                 │
                           │                        │
                           │ CheckLeaderStatus()    │
                           │                        │
            ┌──────────────┴──────────────┐        │
            │                             │        │
            ▼                             ▼        │
      ┌───────────┐                  ┌───────────┐ │
      │  Leader   │                  │  Backup   │ │
      │           │                  │           │ │
      │ - 续约Lease │                  │ - 等待选举  │ │
      └─────┬─────┘                  └─────┬─────┘ │
            │                              │       │
            │ 续约失败                      │       │
            │                              │       │
            └──────────────┬───────────────┘       │
                           │                       │
                           ▼                       │
                    ┌─────────────┐               │
                    │   Stop()    │               │
                    └─────────────┘               │
                           │                       │
                           ▼                       │
                    ┌─────────────┐                │
                    │   Stopped   │────────────────┘
                    └─────────────┘
```

---

## 使用示例

```cpp
#include "mmc_leader_election.h"

// 创建 Leader 选举实例
MmcMetaServiceLeaderElection election("meta_election",
                                      "meta-pod-1",      // podName
                                      "default",         // namespace
                                      "meta-lease");     // leaseName

// 配置选项
mmc_meta_service_config_t options;
options.logLevel = 1;
options.logPath = "/var/log/mmc";

// 启动选举
Result ret = election.Start(options);
if (ret != MMC_OK) {
    // 处理错误
}

// 运行...
// 选举会在后台自动运行，处理主备切换

// 停止选举
election.Stop();
```

---

## 环境变量要求

| 环境变量 | 说明 | 示例 |
|---------|------|------|
| `META_POD_NAME` | 当前 Pod 名称 | `meta-pod-1` |
| `META_NAMESPACE` | Kubernetes 命名空间 | `default` |
| `META_LEASE_NAME` | Lease 对象名称 | `meta-lease` |

---

## Python 依赖

Leader 选举功能依赖 Python 模块 `memcache_hybrid.meta_service_leader_election`，该模块需提供以下类和方法：

```python
class MetaServiceLeaderElection:
    def __init__(self, lease_name, namespace, pod_name, retry_period, log_level, log_path):
        pass

    def update_lease(self, renew: bool) -> bool:
        """更新或获取 Lease，返回是否成功成为 Leader"""
        pass

    def check_leader_status(self) -> str:
        """检查当前 Leader，返回 Leader Pod 名称"""
        pass

    def update_pod_to_master(self):
        """更新 Pod 状态为 Master"""
        pass

    def update_pod_to_backup(self):
        """更新 Pod 状态为 Backup"""
        pass
```
