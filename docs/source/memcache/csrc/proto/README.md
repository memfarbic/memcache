# proto 模块文档

## 模块概述

`proto` 模块包含 MemCache_Hybrid 项目的网络通信协议定义，包括消息基类、序列化工具和具体的请求/响应消息类型。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/proto/`

## 文件列表

### 头文件 (.h)
- `mmc_msg_base.h` - 消息基类和操作码定义
- `mmc_msg_packer.h` - 消息序列化/反序列化工具
- `mmc_msg_client_meta.h` - 客户端与元服务通信的消息定义

### 源文件 (.cpp)
- `mmc_msg_client_meta.cpp` - 消息实现（仅包含版权声明）

---

## 详细文档链接

### [mmc_msg_base_h.md](mmc_msg_base_h.md)
**功能**: 定义网络通信协议的基础消息结构和操作码枚举

**主要组件**:
- `MsgBase` 结构体 - 所有网络消息的基类
- `LOCAL_META_OPCODE_REQ` 枚举 - 18种请求操作码
- `LOCAL_META_OPCODE_RESP` 枚举 - 12种响应操作码

---

### [mmc_msg_packer_h.md](mmc_msg_packer_h.md)
**功能**: 提供网络消息的序列化和反序列化功能

**主要组件**:
- `NetMsgPacker` 类 - 序列化器
- `NetMsgUnpacker` 类 - 反序列化器

**支持的类型**:
- POD 类型（基本类型、枚举、简单结构体）
- `std::string` - 字符串
- `std::pair<K,V>` - 键值对
- `std::vector<V>` - 向量
- `std::map<K,V>` - 映射表

---

### [mmc_msg_client_meta_h.md](mmc_msg_client_meta_h.md)
**功能**: 定义客户端与元服务之间通信的具体消息类型

**消息类型数量**: 30+ 种消息类型

**主要内容**:
- `AllocOptions` 结构体 - 分配选项
- 请求消息类型 (18种)
- 响应消息类型 (9种)
- `MemObjQueryInfo` 结构体 - 内存对象查询信息

---

## 数据流和关系

```
┌─────────────────────┐
│  mmc_msg_packer.h   │
│  (序列化工具)        │
└──────────┬──────────┘
           │ 被
           ▼
┌─────────────────────┐      ┌──────────────────────┐
│   mmc_msg_base.h    │─────▶│ mmc_msg_client_meta  │
│   (消息基类)         │      │ .h (具体消息)          │
└─────────────────────┘      └──────────────────────┘
           │                           │
           │ 继承                       │
           ▼                           ▼
    ┌─────────────────────────────────────┐
    │         MsgBase 基类                 │
    │  + msgVer, msgId, destRankId        │
    │  + Serialize() / Deserialize()      │
    └─────────────────────────────────────┘
                          │
                          │ 被继承
                          ▼
    ┌─────────────────────────────────────┐
    │    所有具体请求/响应消息             │
    │  (AllocRequest, BatchGetResponse...) │
    └─────────────────────────────────────┘
```

---

## 序列化格式规范

### 基本类型
```
直接写入内存的二进制表示
```

### 字符串
```
[uint32_t: 长度][char * N: 内容]
```

### 容器 (vector, map)
```
[size_t: 元素个数][元素1][元素2]...[元素N]
```

### 消息格式
```
[msgVer(2字节)][msgId(2字节)][destRankId(4字节)][消息体...]
```

---

## 操作码映射表

### 请求操作码 (LOCAL_META_OPCODE_REQ)

| 值 | 名称 | 说明 |
|----|------|------|
| 0 | ML_PING_REQ | Ping 请求 |
| 1 | ML_ALLOC_REQ | 分配请求 |
| 2 | ML_UPDATE_REQ | 更新请求 |
| 3 | ML_GET_REQ | 获取请求 |
| 4 | ML_REMOVE_REQ | 删除请求 |
| 5 | ML_BM_REGISTER_REQ | BM 注册请求 |
| 6 | LM_PING_REQ | Ping 请求（重复） |
| 7 | LM_META_REPLICATE_REQ | 元复制请求 |
| 8 | ML_IS_EXIST_REQ | 存在性检查请求 |
| 9 | ML_BATCH_IS_EXIST_REQ | 批量存在性检查请求 |
| 10 | ML_BATCH_REMOVE_REQ | 批量删除请求 |
| 11 | ML_BM_UNREGISTER_REQ | BM 注销请求 |
| 12 | ML_BATCH_GET_REQ | 批量获取请求 |
| 13 | ML_QUERY_REQ | 查询请求 |
| 14 | ML_BATCH_QUERY_REQ | 批量查询请求 |
| 15 | ML_BATCH_ALLOC_REQ | 批量分配请求 |
| 16 | ML_BATCH_UPDATE_REQ | 批量更新请求 |
| 17 | LM_BLOB_COPY_REQ | Blob 拷贝请求 |
| 18 | LM_REMOVE_ALL_REQ | 删除所有请求 |

### 响应操作码 (LOCAL_META_OPCODE_RESP)

| 值 | 名称 | 说明 |
|----|------|------|
| 0 | ML_PING_RESP | Ping 响应 |
| 1 | ML_ALLOC_RESP | 分配响应 |
| 2 | ML_UPDATE_RESP | 更新响应 |
| 3 | ML_BM_REGISTER_RESP | BM 注册响应 |
| 4 | ML_IS_EXIST_RESP | 存在性检查响应（未使用） |
| 5 | ML_BATCH_IS_EXIST_RESP | 批量存在性检查响应 |
| 6 | ML_BATCH_REMOVE_RESP | 批量删除响应 |
| 7 | ML_BM_UNREGISTER_RESP | BM 注销响应（未使用） |
| 8 | ML_BATCH_ALLOC_RESP | 批量分配响应 |
| 9 | ML_QUERY_RESP | 查询响应 |
| 10 | ML_BATCH_QUERY_RESP | 批量查询响应 |
| 11 | ML_BATCH_UPDATE_RESP | 批量更新响应 |

---

## 消息类型汇总

### 请求消息 (Request)

| 消息类型 | 操作码 | 主要用途 |
|----------|--------|----------|
| `PingMsg` | `ML_PING_REQ (0)` | 心跳检测 |
| `AllocRequest` | `ML_ALLOC_REQ (1)` | 分配单个对象 |
| `GetRequest` | `ML_GET_REQ (3)` | 获取单个对象 |
| `BatchGetRequest` | `ML_BATCH_GET_REQ (12)` | 批量获取对象 |
| `BatchUpdateRequest` | `ML_BATCH_UPDATE_REQ (16)` | 批量更新对象 |
| `BatchAllocRequest` | `ML_BATCH_ALLOC_REQ (15)` | 批量分配对象 |
| `RemoveRequest` | `ML_REMOVE_REQ (4)` | 删除单个对象 |
| `BatchRemoveRequest` | `ML_BATCH_REMOVE_REQ (10)` | 批量删除对象 |
| `UpdateRequest` | `ML_UPDATE_REQ (2)` | 更新单个对象 |
| `BmRegisterRequest` | `ML_BM_REGISTER_REQ (5)` | 注册 Blob Manager |
| `BmUnregisterRequest` | `ML_BM_UNREGISTER_REQ (11)` | 注销 Blob Manager |
| `MetaReplicateRequest` | `LM_META_REPLICATE_REQ (7)` | 元数据复制 |
| `BlobCopyRequest` | `LM_BLOB_COPY_REQ (17)` | Blob 拷贝 |
| `IsExistRequest` | `ML_IS_EXIST_REQ (8)` | 检查对象是否存在 |
| `BatchIsExistRequest` | `ML_BATCH_IS_EXIST_REQ (9)` | 批量检查存在性 |
| `QueryRequest` | `ML_QUERY_REQ (13)` | 查询对象信息 |
| `BatchQueryRequest` | `ML_BATCH_QUERY_REQ (14)` | 批量查询对象信息 |
| `RemoveAllRequest` | `LM_REMOVE_ALL_REQ (18)` | 删除所有对象 |

### 响应消息 (Response)

| 消息类型 | 操作码 | 主要用途 |
|----------|--------|----------|
| `BatchAllocResponse` | `ML_BATCH_ALLOC_RESP (8)` | 批量分配响应 |
| `BatchRemoveResponse` | `ML_BATCH_REMOVE_RESP (6)` | 批量删除响应 |
| `AllocResponse` | `ML_ALLOC_RESP (1)` | 分配响应 |
| `Response` | `ML_UPDATE_REQ (2)` | 通用响应 |
| `BatchUpdateResponse` | `ML_BATCH_UPDATE_RESP (11)` | 批量更新响应 |
| `IsExistResponse` | `ML_IS_EXIST_RESP (4)` | 存在性检查响应 |
| `BatchIsExistResponse` | `ML_BATCH_IS_EXIST_RESP (5)` | 批量存在性检查响应 |
| `QueryResponse` | `ML_QUERY_RESP (9)` | 查询响应 |
| `BatchQueryResponse` | `ML_BATCH_QUERY_RESP (10)` | 批量查询响应 |

---

## 使用示例

### 创建自定义消息

```cpp
#include "mmc_msg_client_meta.h"

// 创建分配请求
AllocOptions options(1024, 3, MEDIA_HBM, {0, 1, 2}, 0);
AllocRequest request("my_key", options, GenerateOperateId(0));

// 序列化
NetMsgPacker packer;
request.Serialize(packer);
std::string data = packer.String();

// 发送数据...
```

### 解析接收的消息

```cpp
// 接收数据...
std::string receivedData = ...;

// 反序列化
AllocResponse response;
NetMsgUnpacker unpacker(receivedData);
response.Deserialize(unpacker);

// 处理响应
if (response.result_ == MMC_OK) {
    for (const auto &blob : response.blobs_) {
        MMC_LOG_INFO("Allocated blob: rank=" << blob.rank_
                     << ", gva=" << blob.gva_);
    }
}
```

---

## 依赖关系

### 模块依赖

```
proto 模块
    ├── mmc_common_includes.h (公共头文件)
    ├── mmc_mem_blob.h (Blob 描述符)
    └── mmc_blob_common.h (Blob 公共定义)
```

### 被依赖

```
proto 模块
    ├── 客户端服务层
    ├── 元服务层
    └── 本地服务层
```

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有协议定义
}
}
```
