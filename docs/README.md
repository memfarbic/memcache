# MemCache_Hybrid 项目文档

## 项目概述

MemCache_Hybrid 是一个分布式混合内存缓存系统，支持 DRAM 和 HBM 的多层级内存管理，提供高性能的对象存储和检索能力。

**许可证**: Mulan PSL v2
**版权所有**: 华为技术有限公司 (2025)

---

## 文档导航

### [公共 API 文档](source/memcache/include/README.md)
- C API 接口 (`mmc.h`, `mmc_def.h`, `mmc_client.h`, `mmc_service.h`)
- C++ API 接口 (`cpp/mmcache.h`)

### [核心模块文档](source/memcache/csrc/)

#### [common/](source/memcache/csrc/common/README.md) - 公共工具模块
- 宏定义和常量
- 类型定义
- 日志系统
- 锁机制 (互斥锁、读写锁、自旋锁)
- 线程池
- 引用计数智能指针
- 工具函数

#### [config/](source/memcache/csrc/config/README.md) - 配置管理模块
- 键值对解析器 (`KVParser`)
- 配置验证器 (`Validator`)
- 配置管理类 (`Configuration`)
- 元服务配置 (`MetaServiceConfig`)
- 客户端配置 (`ClientConfig`)

#### [client/](source/memcache/csrc/client/README.md) - 客户端模块
- 默认客户端实现 (`MmcClientDefault`)
- 元数据网络客户端 (`MetaNetClient`)
- 数据存取接口

#### [entities/](source/memcache/csrc/entities/README.md) - 数据实体模块
- Blob 描述 (`MmcMemBlobDesc`)
- Blob 状态机 (`BlobState`, `BlobStateMachine`)
- 内存 Blob (`MmcMemBlob`)
- 内存对象元数据 (`MmcMemObjMeta`)
- 租约管理 (`MmcMetaLeaseManager`)

#### [net/](source/memcache/csrc/net/README.md) - 网络通信模块
- 网络引擎 (`NetEngine`)
- 网络上下文 (`NetContext`)
- 网络链接 (`NetLink`)
- ACC 链接实现

#### [proto/](source/memcache/csrc/proto/README.md) - 协议模块
- 消息基类 (`MsgBase`)
- 消息序列化 (`NetMsgPacker`, `NetMsgUnpacker`)
- 请求/响应消息定义

#### [meta_service/](source/memcache/csrc/meta_service/README.md) - 元数据服务模块
- 元数据服务 (`MmcMetaService`)
- 元数据管理器 (`MmcMetaManager`)
- 元数据容器 (`MmcMetaContainer`)
- 全局分配器 (`MmcGlobalAllocator`)
- 网络服务器 (`MetaNetServer`)
- HTTP 监控服务 (`MmcHttpServer`)

#### [local_service/](source/memcache/csrc/local_service/README.md) - 本地服务模块
- 本地服务接口 (`MmcLocalService`)
- 本地服务实现 (`MmcLocalServiceDefault`)
- Blob Manager 代理 (`MmcBmProxy`)

#### [其他模块](source/memcache/csrc/other_modules.md)
- 高可用模块 (`ha/`)
- 底层 API 封装 (`under_api/`)
- 日志模块 (`log/`)
- Python 绑定 (`python_wrapper/`)

#### [核心实现](source/memcache/csrc/core_impl.md)
- `mmc.cpp` - 主初始化实现
- `mmc_client.cpp` - 客户端 API 实现
- `mmc_service.cpp` - 服务端 API 实现
- `mmcache_store.cpp` - C++ ObjectStore 实现

### [Python 模块文档](source/memcache/python/README.md)
- Python 包结构
- Leader 选举实现
- Python 使用示例

### [第三方依赖](3rdparty.md)
- acc_links (网络框架)
- smem (共享内存)
- spdlog (日志)
- nlohmann/json (JSON)
- Pybind11 (Python 绑定)
- Kubernetes Python Client
- Civetweb (HTTP 服务)

---

## 架构概述

```
┌─────────────────────────────────────────────────────────────┐
│                        应用层                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │   C API      │  │   C++ API    │  │  Python API  │        │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘        │
└─────────┼──────────────────┼──────────────────┼────────────────┘
          │                  │                  │
┌─────────┼──────────────────┼──────────────────┼────────────────┐
│         │                  │                  │                  │
│  ┌──────▼──────────┐  ┌────▼───────────┐  ┌──▼──────────────┐  │
│  │   mmc.cpp      │  │ pymmc.cpp     │  │ mmc_service.cpp │  │
│  │   (初始化)      │  │  (Python绑定)  │  │   (服务端)      │  │
│  └──────┬──────────┘  └────┬───────────┘  └──┬──────────────┘  │
│         │                  │                  │                  │
│  ┌──────▼──────────────────────────────────▼──────────────────┐  │
│  │                    核心实现层                              │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │  │
│  │  │  MmcClient   │  │ MmcMetaMgr   │  │ MmcLocalSvc  │     │  │
│  │  │   Default    │  │    Proxy     │  │   Default    │     │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │  │
│  └─────────┼─────────────────┼─────────────────┼─────────────┘  │
│            │                 │                 │                  │
│  ┌─────────▼─────────────────▼─────────────────▼─────────────┐ │
│  │                      网络层                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │ │
│  │  │  NetEngine   │  │ MetaNetServer│  │  MsgPacker   │     │ │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │ │
│  └─────────┼──────────────────┼─────────────────┼─────────────┘ │
│            │                  │                 │                  │
│  ┌─────────▼──────────────────▼─────────────────▼─────────────┐ │
│  │                    存储层                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │ │
│  │  │MmcMetaMgr    │  │MmcGlobal     │  │ MmcBmProxy   │     │ │
│  │  │              │  │ Allocator    │  │              │     │ │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │ │
│  └─────────┼──────────────────┼─────────────────┼─────────────┘ │
│            │                  │                 │                  │
│  ┌─────────▼──────────────────▼─────────────────▼─────────────┐ │
│  │                    底层层                                  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │ │
│  │  │  SMEM BM     │  │  acc_links   │  │  spdlog      │     │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘     │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

---

## 使用流程

### 1. 编译安装

```bash
cd /home/xuruiyuan/project/mini-proj/memcache
mkdir build && cd build
cmake ..
make -j
```

### 2. 启动元数据服务

```bash
# 配置文件: meta_config.ini
./bin/mmc_meta_service --config meta_config.ini
```

### 3. 启动本地服务

```bash
# 配置文件: local_config.ini
./bin/mmc_local_service --config local_config.ini
```

### 4. 客户端使用

#### C 客户端

```c
#include "mmc.h"

mmc_init_config config = {0};
config.deviceId = 0;
config.initBm = false;
mmc_init(&config);

mmc_client_config_t client_config = {...};
mmcc_init(&client_config);

// 使用 memcache...
mmcc_uninit();
mmc_uninit();
```

#### Python 客户端

```python
from memcache_hybrid import DistributedObjectStore

store = DistributedObjectStore.CreateObjectStore()
store.Init(device_id=0, init_bm=False)

# 存储数据
store.PutFrom("key", buffer, size, 3)

# 获取数据
store.GetInto("key", buffer, size, 2)

store.TearDown()
```

---

## 目录结构

```
memcache/
├── docs/                    # 文档目录
├── src/memcache/
│   ├── include/             # 公共 API 头文件
│   │   ├── mmc.h
│   │   ├── mmc_def.h
│   │   ├── mmc_client.h
│   │   ├── mmc_service.h
│   │   └── cpp/mmcache.h
│   └── csrc/                 # C++ 源码
│       ├── common/           # 公共工具
│       ├── config/          # 配置管理
│       ├── client/          # 客户端
│       ├── entities/        # 数据实体
│       ├── net/             # 网络
│       ├── proto/           # 协议
│       ├── meta_service/    # 元数据服务
│       ├── local_service/   # 本地服务
│       ├── ha/              # 高可用
│       ├── under_api/       # 底层 API
│       ├── daemon/          # 守护进程
│       ├── log/             # 日志
│       ├── python_wrapper/  # Python 绑定
│       ├── mmc.cpp
│       ├── mmc_client.cpp
│       ├── mmc_service.cpp
│       └── mmcache_store.cpp
└── python/                  # Python 源码
    └── memcache_hybrid/
```

---

## 版本信息

- 当前版本: 参考 `mmc_version.h`
- 构建时间: 编译时自动生成
- Git 提交: 编译时自动生成

---

## 许可证

本项目采用 Mulan PSL v2 许可证。详情请见:
http://license.coscl.org.cn/MulanPSL2

---

## 联系方式

如有问题或建议，请联系项目维护者。
