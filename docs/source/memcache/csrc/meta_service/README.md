# meta_service 模块文档

## 模块概述

`meta_service` 模块包含 MemCache_Hybrid 项目的元数据管理服务，负责管理所有元数据的生命周期、内存分配和淘汰策略。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/meta_service/`

## 文件列表

### 头文件 (.h)
- `mmc_blob_allocator.h` - Blob 内存分配器
- `mmc_global_allocator.h` - 全局内存分配器
- `mmc_http_server.h` - HTTP 监控服务器
- `mmc_locality_strategy.h` - 内存分配策略
- `mmc_meta_backup_mgr.h` - 元数据备份管理器接口
- `mmc_meta_backup_mgr_default.h` - 元数据备份管理器默认实现
- `mmc_meta_backup_mgr_factory.h` - 元数据备份管理器工厂
- `mmc_meta_common.h` - 元服务公共头文件
- `mmc_meta_container.h` - 元数据容器接口
- `mmc_meta_manager.h` - 元数据管理器
- `mmc_meta_metric_manager.h` - 监控指标管理器
- `mmc_meta_mgr_proxy.h` - 元数据管理器代理
- `mmc_meta_net_server.h` - 元数据网络服务器
- `mmc_meta_service.h` - 元数据服务
- `mmc_meta_service_process.h` - 元服务进程管理

### 源文件 (.cpp)
- `mmc_blob_allocator.cpp` - Blob 内存分配器实现
- `mmc_http_server.cpp` - HTTP 监控服务器实现
- `mmc_meta_backup_mgr_default.cpp` - 元数据备份管理器默认实现
- `mmc_meta_container_lru.cpp` - 元数据容器 LRU 实现
- `mmc_meta_manager.cpp` - 元数据管理器实现
- `mmc_meta_metric_manager.cpp` - 监控指标管理器实现
- `mmc_meta_mgr_proxy.cpp` - 元数据管理器代理实现
- `mmc_meta_net_server.cpp` - 元数据网络服务器实现
- `mmc_meta_service.cpp` - 元数据服务实现
- `mmc_meta_service_process.cpp` - 元服务进程管理实现

---

## 详细文档链接

每个文件都有独立的逐函数解读文档：

### 内存分配相关
- [mmc_blob_allocator.h](mmc_blob_allocator_h.md) / [mmc_blob_allocator.cpp](mmc_blob_allocator_cpp.md) - 单点内存分配器，使用首次适应算法和空闲块合并
- [mmc_global_allocator.h](mmc_global_allocator_h.md) - 全局内存分配器，管理所有 Rank 和介质类型的分配器
- [mmc_locality_strategy.h](mmc_locality_strategy_h.md) - 内存分配策略（亲和性/强制/随机）

### 网络服务相关
- [mmc_meta_net_server.h](mmc_meta_net_server_h.md) / [mmc_meta_net_server.cpp](mmc_meta_net_server_cpp.md) - 元数据网络服务器，处理客户端请求
- [mmc_http_server.h](mmc_http_server_h.md) / [mmc_http_server.cpp](mmc_http_server_cpp.md) - HTTP 监控服务器，提供 RESTful API

### 元数据管理相关
- [mmc_meta_container.h](mmc_meta_container_h.md) / [mmc_meta_container_lru.cpp](mmc_meta_container_lru_cpp.md) - 元数据容器，提供 LRU 淘汰功能
- [mmc_meta_manager.h](mmc_meta_manager_h.md) / [mmc_meta_manager.cpp](mmc_meta_manager_cpp.md) - 元数据管理器，核心业务逻辑
- [mmc_meta_mgr_proxy.h](mmc_meta_mgr_proxy_h.md) / [mmc_meta_mgr_proxy.cpp](mmc_meta_mgr_proxy_cpp.md) - 元数据管理器代理，处理网络请求

### 服务管理相关
- [mmc_meta_service.h](mmc_meta_service_h.md) / [mmc_meta_service.cpp](mmc_meta_service_cpp.md) - 元数据服务，顶层入口
- [mmc_meta_service_process.h](mmc_meta_service_process_h.md) / [mmc_meta_service_process.cpp](mmc_meta_service_process_cpp.md) - 元服务进程管理

### 备份相关
- [mmc_meta_backup_mgr.h](mmc_meta_backup_mgr_h.md) - 元数据备份管理器接口
- [mmc_meta_backup_mgr_default.h](mmc_meta_backup_mgr_default_h.md) / [mmc_meta_backup_mgr_default.cpp](mmc_meta_backup_mgr_default_cpp.md) - 元数据备份管理器默认实现
- [mmc_meta_backup_mgr_factory.h](mmc_meta_backup_mgr_factory_h.md) - 元数据备份管理器工厂

### 监控相关
- [mmc_meta_metric_manager.h](mmc_meta_metric_manager_h.md) / [mmc_meta_metric_manager.cpp](mmc_meta_metric_manager_cpp.md) - 监控指标管理器，集成 Prometheus

### 通用文件
- [mmc_meta_common.h](mmc_meta_common_h.md) - 元服务公共头文件

---

## 数据流和关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    MmcMetaServiceProcess                        │
│  (进程管理: 配置加载/信号处理/日志初始化)                       │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       MmcMetaService                            │
│  (元服务: 顶层入口/组件管理)                                    │
└──┬──────────────────┬──────────────────┬───────────────────────┘
   │                  │                  │
   ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│MetaNetServer │  │MmcMetaMgrProxy│  │BackupManager │
│(网络服务器)   │  │(代理层)        │  │(备份管理)     │
└──────┬───────┘  └──────┬───────┘  └──────────────┘
       │                  │
       │           ┌──────┴──────┐
       │           │             │
       ▼           ▼             ▼
┌──────────────────────────────────────────┐
│         MmcMetaManager                   │
│  (元数据管理: CRUD/淘汰/内存管理)         │
└──┬───────────────────────┬───────────────┘
   │                       │
   ▼                       ▼
┌──────────────┐    ┌──────────────┐
│GlobalAllocator│    │MetaContainer │
│(全局分配器)    │    │(元数据容器)   │
└──────┬───────┘    └──────┬───────┘
       │                   │
       ▼                   ▼
┌──────────────────────────────────┐
│      MmcBlobAllocator           │
│  (单点分配器: per Rank per Type) │
└─────────────────────────────────┘
```

---

## 核心功能

### 1. 内存分配管理

- **MmcBlobAllocator**: 单点内存分配器，使用双树结构实现高效分配
  - `addressTree_`: 按地址索引，便于合并相邻空闲块
  - `sizeTree_`: 按大小索引，快速找到合适的空闲块

- **MmcGlobalAllocator**: 全局内存分配器，管理所有分配器
  - 支持多 Rank、多介质类型
  - 三种分配策略：亲和性/强制/随机

### 2. 元数据管理

- **MmcMetaContainer**: 元数据容器，提供 LRU 淘汰
  - 每种介质类型独立的 LRU 链表
  - 支持多层级淘汰（向下移动或删除）

- **MmcMetaManager**: 元数据管理器
  - 元数据 CRUD 操作
  - 自动内存淘汰
  - Blob 复制/移动

### 3. 网络服务

- **MetaNetServer**: 处理客户端网络请求
  - 支持 15+ 种消息类型
  - 性能追踪集成

- **MmcHttpServer**: HTTP 监控服务
  - 健康检查端点
  - 指标导出（Prometheus 格式）
  - 元数据查询 API

### 4. 备份管理

- **MMCMetaBackUpMgr**: 元数据备份管理
  - 异步备份线程
  - 批量备份（最多 1024 个）
  - 按 Rank 分组发送

---

## HTTP API 示例

```bash
# 健康检查
curl http://localhost:8080/health

# 获取所有 key
curl http://localhost:8080/get_all_keys

# 查询 key 信息
curl http://localhost:8080/query_key?key=mykey

# 获取所有分段信息
curl http://localhost:8080/get_all_segments

# 获取 Prometheus 指标
curl http://localhost:8080/metrics

# 获取指标摘要
curl http://localhost:8080/metrics/summary

# 获取性能追踪信息
curl http://localhost:8080/metrics/ptracer
```

---

## 元数据操作流程

### 分配流程 (Alloc)
```
1. 客户端发送 AllocRequest (key, size, replicaNum)
2. MetaNetServer 接收并转发到 MmcMetaMgrProxy
3. 触发淘汰检查 CheckAndEvict()
4. MmcGlobalAllocator 根据 AllocOptions 分配 Blob
5. 创建 MmcMemObjMeta 并存储到 MmcMetaContainer
6. 返回 AllocResponse (blob 列表)
```

### 获取流程 (Get)
```
1. 客户端发送 GetRequest (key)
2. MmcMetaContainer 查找元数据
3. 更新 LRU 位置（Promote）
4. 返回 Blob 信息
```

### 淘汰流程 (Evict)
```
1. 定期检查内存使用率
2. 超过高水位时触发淘汰
3. 从 LRU 链表尾部选择对象
4. 尝试向下移动（HBM -> DRAM）
5. 无法移动则删除
6. 更新指标计数器
```
