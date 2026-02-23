# mmc_meta_service_process.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_service_process.h`
- **文件用途**: 定义元服务进程类，管理元服务的启动和生命周期
- **依赖项**: `mmc_leader_election.h`, `mmc_configuration.h`, `mmc_http_server.h`, `mmc_meta_service.h`

---

## 类定义

### MmcMetaServiceProcess

元服务进程管理类，使用单例模式，负责元服务的完整生命周期管理。

---

#### GetInstance

```cpp
static MmcMetaServiceProcess &getInstance()
{
    static MmcMetaServiceProcess meta;
    return meta;
}
```

**声明位置**: 行 31-35

**功能描述**: 获取单例实例

**返回值**: 单例引用

**实现**: Meyer's Singleton

---

#### 构造/析构函数

```cpp
MmcMetaServiceProcess() = default;
~MmcMetaServiceProcess() = default;
```

**声明位置**: 行 36-37

**功能描述**: 默认构造和析构函数

---

#### 禁用的方法

```cpp
MmcMetaServiceProcess(const MmcMetaServiceProcess &) = delete;
MmcMetaServiceProcess &operator=(const MmcMetaServiceProcess &) = delete;
```

**声明位置**: 行 38-39

**功能描述**: 禁止拷贝和赋值

---

### MainForExecutable

```cpp
int MainForExecutable();
```

**声明位置**: 行 41

**功能描述**: 可执行文件入口

**返回值**: 程序退出码

**代码逻辑**:
1. 创建 Python 解释器
2. 释放 GIL
3. 调用 MainForPython

---

### MainForPython

```cpp
int MainForPython();
```

**声明位置**: 行 42

**功能描述**: Python 模块入口

**返回值**: 程序退出码

**代码逻辑**:
1. 检查是否已运行
2. 加载配置
3. 注册信号处理
4. 初始化性能追踪
5. 初始化日志
6. 启动 Leader 选举（可选）
7. 启动元服务
8. 启动 HTTP 服务
9. 等待退出信号
10. 清理资源

---

## 私有方法

### CheckIsRunning

```cpp
static bool CheckIsRunning();
```

**声明位置**: 行 45

**功能描述**: 检查进程是否已在运行

**实现**: 使用文件锁机制

---

### LoadConfig

```cpp
int LoadConfig();
```

**声明位置**: 行 46

**功能描述**: 加载配置文件

**返回值**: 成功返回 0

---

### RegisterSignal

```cpp
static void RegisterSignal();
```

**声明位置**: 行 47

**功能描述**: 注册信号处理函数

---

### SignalInterruptHandler

```cpp
static void SignalInterruptHandler(const int signal);
```

**声明位置**: 行 48

**功能描述**: 信号中断处理函数

---

### InitLogger

```cpp
static int InitLogger(const mmc_meta_service_config_t &options);
```

**声明位置**: 行 49

**功能描述**: 初始化日志系统

**参数**:
- `options`: 服务配置

**返回值**: 成功返回 0

---

### StartHttpServer

```cpp
int StartHttpServer();
```

**声明位置**: 行 50

**功能描述**: 启动 HTTP 服务器

**返回值**: 成功返回 MMC_OK

---

### Exit

```cpp
void Exit();
```

**声明位置**: 行 51

**功能描述**: 退出并清理资源

---

## 成员变量

```cpp
private:
    mmc_meta_service_config_t config_{};
    MmcMetaService *metaService_{};
    MmcMetaServiceLeaderElection *leaderElection_{};
    MmcHttpServer *httpServer_{};
```

**说明**:
- `config_`: 服务配置
- `metaService_`: 元服务指针
- `leaderElection_`: Leader 选举指针
- `httpServer_`: HTTP 服务器指针

---

## 文件级别的关系图

```
mmc_meta_service_process.h (元服务进程)
    |
    +-- 使用 MmcMetaService (元服务)
    |
    +-- 使用 MmcMetaServiceLeaderElection (Leader 选举)
    |
    +-- 使用 MmcHttpServer (HTTP 服务)
    |
    +-- 使用 MetaServiceConfig (配置管理)
    |
    +-- 单例模式
    |
    +-- 功能:
    |   +-- 进程生命周期管理
    |   +-- 信号处理
    |   +-- 配置加载
    |   +-- 日志初始化
    |   +-- 服务启动/停止
```

---

## 启动流程

```
MainForExecutable/MainForPython
    |
    +-- CheckIsRunning (检查是否已运行)
    |
    +-- LoadConfig (加载配置)
    |
    +-- RegisterSignal (注册信号)
    |
    +-- ptracer_init (初始化性能追踪)
    |
    +-- InitLogger (初始化日志)
    |
    +-- LeaderElection::Start (启动 Leader 选举，可选)
    |
    +-- MmcMetaService::Start (启动元服务)
    |
    +-- StartHttpServer (启动 HTTP 服务)
    |
    +-- wait for exit signal (等待退出信号)
    |
    +-- Exit (清理退出)
```
