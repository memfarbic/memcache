# mmc_msg_base.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/proto/mmc_msg_base.h`
- **文件用途**: 定义 MemCache 网络通信协议的基础消息结构和操作码枚举
- **依赖项**:
  - `mmc_common_includes.h`
  - `mmc_msg_packer.h`

---

## 结构体

### MsgBase

**声明位置**: 行 20-31

**功能描述**: 所有网络消息的基类，定义了消息的基本结构和序列化接口

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `msgVer` | `int16_t` | `0` | 消息版本号，用于协议兼容性控制 |
| `msgId` | `int16_t` | `-1` | 消息ID，对应操作码枚举值 |
| `destRankId` | `uint32_t` | `0` | 目标 Rank ID，标识消息接收者 |

**成员函数**:

#### MsgBase() [默认构造函数]

```cpp
MsgBase() = default;
```

**功能**: 默认构造函数，所有成员变量使用默认值初始化

---

#### MsgBase(int16_t ver, int16_t op, uint32_t dst) [参数化构造函数]

```cpp
MsgBase(int16_t ver, int16_t op, uint32_t dst) : msgVer(ver), msgId(op), destRankId(dst) {}
```

**参数**:
- `ver`: 消息版本号
- `op`: 操作码（消息ID）
- `dst`: 目标 Rank ID

**功能**: 使用指定值初始化消息基类

---

#### Serialize()

```cpp
virtual Result Serialize(NetMsgPacker &packer) const = 0;
```

**声明位置**: 行 27

**功能描述**: 纯虚函数，将消息序列化为字节流

**参数**:
- `packer`: 消息打包器引用

**返回值**: `Result` - 操作结果码

**说明**: 派生类必须实现此方法，定义如何将自身序列化

---

#### Deserialize()

```cpp
virtual Result Deserialize(NetMsgUnpacker &packer) = 0;
```

**声明位置**: 行 28

**功能描述**: 纯虚函数，从字节流反序列化消息

**参数**:
- `packer`: 消息解包器引用

**返回值**: `Result` - 操作结果码

**说明**: 派生类必须实现此方法，定义如何从字节流恢复自身

---

#### ~MsgBase() [虚析构函数]

```cpp
virtual ~MsgBase() = default;
```

**声明位置**: 行 30

**功能**: 虚析构函数，确保派生类正确析构

---

## 枚举类型

### LOCAL_META_OPCODE_REQ

**声明位置**: 行 33-53

**功能描述**: 客户端/服务端到元服务的请求操作码枚举

**枚举值**:

| 枚举值 | 数值 | 说明 |
|--------|------|------|
| `ML_PING_REQ` | `0` | Ping 请求，用于客户端与服务端、服务端与服务端之间的心跳检测 |
| `ML_ALLOC_REQ` | `1` | 分配请求，根据 key 和 size 分配对象 |
| `ML_UPDATE_REQ` | `2` | 更新请求，更新已存在的对象 |
| `ML_GET_REQ` | `3` | 获取请求，根据 key 获取对象信息 |
| `ML_REMOVE_REQ` | `4` | 删除请求，根据 key 删除对象 |
| `ML_BM_REGISTER_REQ` | `5` | Blob Manager 注册请求，向元服务注册本地 BM |
| `LM_PING_REQ` | `6` | Ping 请求（重复定义） |
| `LM_META_REPLICATE_REQ` | `7` | 元复制请求，获取对象的副本列表 |
| `ML_IS_EXIST_REQ` | `8` | 存在性检查请求，检查对象是否存在 |
| `ML_BATCH_IS_EXIST_REQ` | `9` | 批量存在性检查请求 |
| `ML_BATCH_REMOVE_REQ` | `10` | 批量删除请求 |
| `ML_BM_UNREGISTER_REQ` | `11` | Blob Manager 注销请求 |
| `ML_BATCH_GET_REQ` | `12` | 批量获取请求 |
| `ML_QUERY_REQ` | `13` | 查询请求，向元服务查询 key 的 blob 信息 |
| `ML_BATCH_QUERY_REQ` | `14` | 批量查询请求 |
| `ML_BATCH_ALLOC_REQ` | `15` | 批量分配请求 |
| `ML_BATCH_UPDATE_REQ` | `16` | 批量更新请求 |
| `LM_BLOB_COPY_REQ` | `17` | Blob 拷贝请求，为其他 rank 拷贝 blob |
| `LM_REMOVE_ALL_REQ` | `18` | 删除所有请求 |

**命名规则说明**:
- `ML_*`: Meta Local 前缀，表示与本地元服务相关的操作
- `LM_*`: Local Meta 前缀，含义与 ML 类似

---

### LOCAL_META_OPCODE_RESP

**声明位置**: 行 55-68

**功能描述**: 元服务对请求的响应操作码枚举

**枚举值**:

| 枚举值 | 数值 | 说明 |
|--------|------|------|
| `ML_PING_RESP` | `0` | Ping 响应 |
| `ML_ALLOC_RESP` | `1` | 分配响应 |
| `ML_UPDATE_RESP` | `2` | 更新响应 |
| `ML_BM_REGISTER_RESP` | `3` | Blob Manager 注册响应 |
| `ML_IS_EXIST_RESP` | `4` | 存在性检查响应（未使用） |
| `ML_BATCH_IS_EXIST_RESP` | `5` | 批量存在性检查响应 |
| `ML_BATCH_REMOVE_RESP` | `6` | 批量删除响应 |
| `ML_BM_UNREGISTER_RESP` | `7` | Blob Manager 注销响应（未使用） |
| `ML_BATCH_ALLOC_RESP` | `8` | 批量分配响应 |
| `ML_QUERY_RESP` | `9` | 查询响应 |
| `ML_BATCH_QUERY_RESP` | `10` | 批量查询响应 |
| `ML_BATCH_UPDATE_RESP` | `11` | 批量更新响应 |

**说明**: 响应操作码与请求操作码并非一一对应，某些操作可能没有专门的响应类型

---

## 消息处理流程

```
┌─────────────┐                    ┌─────────────┐
│   Client    │                    │   Meta      │
│   Service   │                    │   Service   │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
       │  1. Create Request Msg          │
       │     (e.g., AllocRequest)        │
       │                                  │
       │  2. Serialize(packer)           │
       │     ──────────────────────────> │
       │                                  │ 3. Deserialize(unpacker)
       │                                  │
       │                                  │ 4. Process Request
       │                                  │
       │  5. Serialize Response          │ 6. Create Response Msg
       │     <────────────────────────── │     (e.g., AllocResponse)
       │                                  │
       │  7. Deserialize Response        │
       │                                  │
       │  8. Handle Result               │
       │                                  │
```

---

## 使用示例

### 基本消息结构定义

```cpp
// 派生自 MsgBase 的自定义消息
struct MyCustomRequest : public MsgBase {
    std::string data_;

    MyCustomRequest() : MsgBase(0, ML_ALLOC_REQ, 0) {}

    Result Serialize(NetMsgPacker &packer) const override {
        packer.Serialize(msgVer);
        packer.Serialize(msgId);
        packer.Serialize(destRankId);
        packer.Serialize(data_);
        return MMC_OK;
    }

    Result Deserialize(NetMsgUnpacker &packer) override {
        packer.Deserialize(msgVer);
        packer.Deserialize(msgId);
        packer.Deserialize(destRankId);
        packer.Deserialize(data_);
        return MMC_OK;
    }
};
```

---

## 依赖关系

**被以下文件依赖**:
- `mmc_msg_client_meta.h` - 所有具体消息类型继承自 MsgBase

**依赖的文件**:
- `mmc_common_includes.h` - 公共头文件
- `mmc_msg_packer.h` - 序列化/反序列化工具

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // MsgBase 和操作码枚举定义
}
}
```
