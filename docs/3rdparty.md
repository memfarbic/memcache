# 第三方依赖文档

## 概述

MemCache_Hybrid 项目依赖多个第三方库来提供核心功能。

**依赖位置**: `/home/xuruiyuan/project/mini-proj/memcache/3rdparty/`

---

## 核心依赖

### 1. acc_links (华为 ACC Links 框架)

**用途**: 提供高性能网络通信能力

**功能**:
- TCP/RDMA 网络传输
- 连接管理
- 消息收发
- 线程池管理

**使用的头文件**:
- `acc_driver.h`
- `acc_net.h`
- `acc_thread.h`

**集成方式**: 通过 `mmc_net_engine_acc.cpp` 集成

### 2. smem (华为 Shared Memory)

**用途**: 提供共享内存和 Blob Manager 功能

**功能**:
- 共享内存管理
- Blob 分配和释放
- 跨节点内存拷贝 (RDMA/SDMA)
- 内存池管理

**使用的头文件**:
- `smem.h`
- `smem_bm.h`

**集成方式**: 通过 `MmcBmProxy` 和 `smem_bm_api.cpp` 封装

### 3. spdlog

**用途**: C++ 日志库

**功能**:
- 高性能日志记录
- 日志轮转
- 多种日志级别
- 异步日志

**集成方式**: 通过 `spdlogger.h/cpp` 封装

### 4. nlohmann/json

**用途**: JSON 序列化和反序列化

**功能**:
- JSON 解析和生成
- 与 C++ 对象的转换

**使用场景**:
- HTTP API 响应格式
- 配置文件解析
- 元数据序列化

### 5. Pybind11

**用途**: Python/C++ 绑定

**功能**:
- 暴露 C++ 类和函数到 Python
- 自动类型转换
- GIL 释放

**集成方式**: 通过 `pymmc.cpp` 创建 Python 模块

### 6. Kubernetes Python Client

**用途**: Kubernetes API 访问

**功能**:
- Pod 管理
- Lease 资源操作
- 配置管理

**使用场景**: Leader 选举 (`meta_service_leader_election.py`)

### 7. Civetweb

**用途**: HTTP 服务器

**功能**:
- HTTP 请求处理
- RESTful API
- WebSocket 支持

**使用场景**: HTTP 监控服务 (`mmc_http_server.cpp`)

---

## 依赖关系图

```
MemCache_Hybrid
    │
    ├─→ acc_links ──→ 网络通信 (TCP/RDMA)
    │
    ├─→ smem ──→ 共享内存管理
    │
    ├─→ spdlog ──→ 日志记录
    │
    ├─→ nlohmann/json ──→ JSON 序列化
    │
    ├─→ Pybind11 ──→ Python 绑定
    │
    ├─→ Kubernetes Client ──→ Leader 选举
    │
    └─→ Civetweb ──→ HTTP 服务
```

---

## 构建依赖

### CMake 配置

项目通过 CMake 管理依赖，主要配置项:
- `ACC_LINKS_PATH`: acc_links 库路径
- `SMEM_PATH`: smem 库路径
- `SPDLOG_PATH`: spdlog 路径

### 依赖版本要求

| 依赖 | 最低版本 |
|------|----------|
| C++ | C++11 |
| GCC | 7.0+ |
| acc_links | 根据硬件型号 |
| smem | 根据硬件型号 |
| spdlog | 1.8+ |
| Python | 3.6+ |
| Pybind11 | 2.6+ |
| Kubernetes | 1.19+ |
