# 公共 API 头文件目录

本目录包含 MemCache_Hybrid 项目的公共 API 头文件定义。

## 目录结构

```
include/
├── mmc.h              # 主初始化和配置接口
├── mmc_def.h          # 核心数据结构和常量定义
├── mmc_client.h       # 客户端操作接口
├── mmc_service.h      # 服务端操作接口
└── cpp/
    └── mmcache.h      # C++ 面向对象接口
```

## 文件说明

### mmc.h
提供 MemCache 系统的初始化和配置功能：
- `mmc_init()` - 初始化 memcache 客户端和本地服务
- `mmc_set_extern_logger()` - 设置外部日志函数
- `mmc_set_log_level()` - 设置日志级别
- `mmc_uninit()` - 反初始化

### mmc_def.h
定义核心数据结构和常量：
- `mmc_init_config` - 初始化配置结构
- `mmc_meta_service_config_t` - 元数据服务配置
- `mmc_local_service_config_t` - 本地服务配置
- `mmc_client_config_t` - 客户端配置
- `mmc_buffer` - 数据缓冲区描述
- `mmc_put_options` - Put 操作选项
- `mmc_data_info` - 数据信息结构
- `mmc_tls_config` - TLS 配置

### mmc_client.h
提供客户端数据操作接口：
- `mmcc_init()` / `mmcc_uninit()` - 客户端初始化/反初始化
- `mmcc_register_buffer()` / `mmcc_unregister_buffer()` - 缓冲区注册
- `mmcc_put()` / `mmcc_get()` - 单个对象存取
- `mmcc_batch_put()` / `mmcc_batch_get()` - 批量对象存取
- `mmcc_query()` / `mmcc_batch_query()` - 查询数据信息
- `mmcc_exist()` / `mmcc_batch_exist()` - 检查对象存在
- `mmcc_remove()` / `mmcc_batch_remove()` - 删除对象
- `mmcc_local_service_id()` - 获取本地服务 ID

### mmc_service.h
提供服务端管理接口：
- `mmcs_meta_service_start()` / `mmcs_meta_service_stop()` - 元数据服务启停
- `mmcs_local_service_start()` / `mmcs_local_service_stop()` - 本地服务启停

### cpp/mmcache.h
提供 C++ 面向对象接口：
- `ock::mmc::KeyInfo` - 数据对象元信息类
- `ock::mmc::ReplicateConfig` - 副本配置类
- `ock::mmc::ObjectStore` - 对象存储抽象接口类

## API 使用方式

### C API

```c
#include "mmc.h"
#include "mmc_client.h"
#include "mmc_service.h"

// 初始化
mmc_init_config config = {0};
mmc_init(&config);

// 启动服务
mmc_meta_service_t meta = mmcs_meta_service_start(&meta_config);
mmc_local_service_t local = mmcs_local_service_start(&local_config);

// 客户端操作
mmcc_init(&client_config);
mmcc_put("key", &buffer, options, 0);
mmcc_get("key", &buffer, 0);

// 清理
mmcc_uninit();
mmcs_local_service_stop(local);
mmcs_meta_service_stop(meta);
mmc_uninit();
```

### C++ API

```cpp
#include "cpp/mmcache.h"

using namespace ock::mmc;

// 创建对象存储
auto store = ObjectStore::CreateObjectStore();
store->Init(deviceId, true);

// 存取数据
store->PutFrom("key", buffer, size, 3);
store->GetInto("key", buffer, size, 2);

// 获取元信息
KeyInfo info = store->GetKeyInfo("key");

// 清理
store->TearDown();
```

## 详细文档

- [mmc.h 详细文档](mmc_h.md)
- [mmc_def.h 详细文档](mmc_def_h.md)
- [mmc_client.h 详细文档](mmc_client_h.md)
- [mmc_service.h 详细文档](mmc_service_h.md)
- [mmcache.h 详细文档](cpp/mmcache_h.md)
