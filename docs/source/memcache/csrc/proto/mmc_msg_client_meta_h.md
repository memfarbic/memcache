# mmc_msg_client_meta.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/proto/mmc_msg_client_meta.h`
- **文件用途**: 定义客户端与元服务之间通信的具体消息类型，包括请求和响应消息
- **依赖项**:
  - `mmc_mem_blob.h` - Blob 描述符和状态定义
  - `mmc_msg_base.h` - 消息基类
  - `mmc_msg_packer.h` - 序列化/反序列化工具

**对应的源文件**: `src/memcache/csrc/proto/mmc_msg_client_meta.cpp` (仅包含版权声明)

---

## 辅助结构体

### AllocOptions

**声明位置**: 行 22-62

**功能描述**: 内存分配选项结构体，用于描述分配请求的参数

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `blobSize_` | `uint64_t` | `0` | 每个 Blob 的大小 |
| `numBlobs_` | `uint32_t` | `0` | 需要分配的 Blob 数量 |
| `mediaType_` | `uint16_t` | `0` | 介质类型 (DRAM/HBM) |
| `preferredRank_` | `vector<uint32_t>` | `{}` | 首选 Rank ID 列表 |
| `flags_` | `uint32_t` | `0` | 分配标志位 |

**成员函数**:

#### AllocOptions() [默认构造函数]

```cpp
AllocOptions() = default;
```

---

#### AllocOptions(...) [参数化构造函数]

```cpp
AllocOptions(const uint64_t blobSize, const uint32_t numBlobs, const uint16_t mediaType,
             const std::vector<uint32_t> &preferredRank, const uint32_t flags)
```

**功能**: 使用指定参数初始化分配选项

---

#### Serialize()

```cpp
Result Serialize(NetMsgPacker &packer) const
```

**声明位置**: 行 34-42

**功能描述**: 将分配选项序列化到打包器

**参数**:
- `packer`: 消息打包器引用

**返回值**: `MMC_OK` - 序列化成功

**代码逻辑**:
1. 序列化 `blobSize_`
2. 序列化 `numBlobs_`
3. 序列化 `mediaType_`
4. 序列化 `preferredRank_`
5. 序列化 `flags_`

---

#### Deserialize()

```cpp
Result Deserialize(NetMsgUnpacker &packer)
```

**声明位置**: 行 44-52

**功能描述**: 从解包器反序列化分配选项

**参数**:
- `packer`: 消息解包器引用

**返回值**: `MMC_OK` - 反序列化成功

---

#### operator<<

```cpp
friend std::ostream &operator<<(std::ostream &os, const AllocOptions &obj)
```

**声明位置**: 行 54-61

**功能描述**: 输出分配选项的字符串表示

---

## 请求消息类型

### PingMsg

**声明位置**: 行 64-85

**功能描述**: Ping 请求消息，用于心跳检测和连接保活

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `num` | `uint64_t` | `UINT64_MAX` | Ping 序列号/计数器 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_PING_REQ (0)`

**成员函数**:

#### PingMsg() [默认构造函数]

```cpp
PingMsg() : MsgBase{0, ML_PING_REQ, 0} {}
```

**功能**: 初始化 Ping 消息，版本为 0，操作码为 ML_PING_REQ

---

#### Serialize()

```cpp
Result Serialize(NetMsgPacker &packer) const override
```

**声明位置**: 行 68-75

**功能描述**: 序列化 Ping 消息

**序列化顺序**:
1. `msgVer` - 消息版本
2. `msgId` - 消息ID
3. `destRankId` - 目标 Rank ID
4. `num` - Ping 序列号

---

#### Deserialize()

```cpp
Result Deserialize(NetMsgUnpacker &packer) override
```

**声明位置**: 行 77-84

**功能描述**: 反序列化 Ping 消息

---

### AllocRequest

**声明位置**: 行 87-117

**功能描述**: 内存分配请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `operateId_` | `uint64_t` | `0` | 操作 ID，用于跟踪请求 |
| `key_` | `string` | `` | 对象的键值 |
| `options_` | `AllocOptions` | `{}` | 分配选项 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_ALLOC_REQ (1)`

**成员函数**:

#### AllocRequest(...) [参数化构造函数]

```cpp
AllocRequest(const std::string &key, const AllocOptions &prot, uint64_t operateId)
```

**功能**: 使用指定参数创建分配请求

---

#### Serialize()

```cpp
Result Serialize(NetMsgPacker &packer) const override
```

**声明位置**: 行 96-105

**序列化顺序**:
1. 基类成员 (msgVer, msgId, destRankId)
2. `key_` - 键值
3. `options_` - 分配选项
4. `operateId_` - 操作 ID

---

#### Deserialize()

```cpp
Result Deserialize(NetMsgUnpacker &packer) override
```

**声明位置**: 行 107-116

---

### GetRequest

**声明位置**: 行 119-152

**功能描述**: 获取对象请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `operateId_` | `uint64_t` | `0` | 操作 ID |
| `rankId_` | `uint32_t` | `0` | Rank ID |
| `key_` | `string` | `` | 对象键值 |
| `isGet_` | `bool` | `false` | 是否为 Get 操作 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_GET_REQ (3)`

**成员函数**:

#### GetRequest(...) [参数化构造函数]

```cpp
explicit GetRequest(const std::string &key, uint32_t rankId, uint64_t operateId, bool isGet)
```

---

#### Serialize()

```cpp
Result Serialize(NetMsgPacker &packer) const override
```

**声明位置**: 行 129-139

**序列化顺序**:
1. 基类成员
2. `operateId_`
3. `rankId_`
4. `isGet_`
5. `key_`

---

#### Deserialize()

```cpp
Result Deserialize(NetMsgUnpacker &packer) override
```

**声明位置**: 行 141-151

---

### BatchGetRequest

**声明位置**: 行 154-185

**功能描述**: 批量获取对象请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `operateId_` | `uint64_t` | `0` | 操作 ID |
| `rankId_` | `uint32_t` | `0` | Rank ID |
| `keys_` | `vector<string>` | `{}` | 键值列表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_GET_REQ (12)`

**成员函数**:

#### BatchGetRequest(...) [参数化构造函数]

```cpp
explicit BatchGetRequest(const std::vector<std::string> &keys, uint32_t rankId, uint64_t operateId)
```

---

#### Serialize()

```cpp
Result Serialize(NetMsgPacker &packer) const override
```

**声明位置**: 行 164-173

---

#### Deserialize()

```cpp
Result Deserialize(NetMsgUnpacker &packer) override
```

**声明位置**: 行 175-184

---

### BatchUpdateRequest

**声明位置**: 行 187-227

**功能描述**: 批量更新对象请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `actionResults_` | `vector<BlobActionResult>` | `{}` | Blob 操作结果列表 |
| `keys_` | `vector<string>` | `{}` | 键值列表 |
| `ranks_` | `vector<uint32_t>` | `{}` | Rank ID 列表 |
| `mediaTypes_` | `vector<uint16_t>` | `{}` | 介质类型列表 |
| `operateId_` | `uint64_t` | `0` | 操作 ID |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_UPDATE_REQ (16)`

**成员函数**:

#### BatchUpdateRequest(...) [参数化构造函数]

```cpp
BatchUpdateRequest(const std::vector<BlobActionResult> &actionResults, const std::vector<std::string> &keys,
                   const std::vector<uint32_t> &ranks, const std::vector<uint16_t> &mediaTypes, uint64_t operateId)
```

---

#### Serialize()

```cpp
Result Serialize(NetMsgPacker &packer) const override
```

**声明位置**: 行 202-213

---

#### Deserialize()

```cpp
Result Deserialize(NetMsgUnpacker &packer) override
```

**声明位置**: 行 215-226

---

### BatchAllocRequest

**声明位置**: 行 229-272

**功能描述**: 批量分配请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `keys_` | `vector<string>` | `{}` | 键值列表 |
| `options_` | `vector<AllocOptions>` | `{}` | 分配选项列表 |
| `flags_` | `uint32_t` | `0` | 标志位 |
| `operateId_` | `uint64_t` | `0` | 操作 ID |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_ALLOC_REQ (15)`

**成员函数**:

#### BatchAllocRequest(...) [参数化构造函数]

```cpp
BatchAllocRequest(const std::vector<std::string> &keys, const std::vector<AllocOptions> &options,
                  uint32_t flags, uint64_t operateId)
```

---

#### Serialize()

```cpp
Result Serialize(NetMsgPacker &packer) const override
```

**声明位置**: 行 242-255

**特殊逻辑**:
```cpp
MMC_ASSERT_RETURN(keys_.size() == options_.size(), MMC_ERROR);
```
确保键值数量与选项数量一致

---

#### Deserialize()

```cpp
Result Deserialize(NetMsgUnpacker &packer) override
```

**声明位置**: 行 257-271

**特殊逻辑**:
```cpp
options_.clear();
options_.assign(keys_.size(), AllocOptions());
```
根据键值数量初始化选项列表

---

### RemoveRequest

**声明位置**: 行 319-342

**功能描述**: 删除对象请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `key_` | `string` | `` | 要删除的键值 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_REMOVE_REQ (4)`

---

### BatchRemoveRequest

**声明位置**: 行 344-368

**功能描述**: 批量删除对象请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `keys_` | `vector<string>` | `{}` | 要删除的键值列表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_REMOVE_REQ (10)`

---

### UpdateRequest

**声明位置**: 行 436-474

**功能描述**: 更新对象请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `actionResult_` | `BlobActionResult` | `{}` | Blob 操作结果 |
| `key_` | `string` | `` | 键值 |
| `rank_` | `uint32_t` | `UINT32_MAX` | Rank ID |
| `mediaType_` | `uint16_t` | `UINT16_MAX` | 介质类型 |
| `operateId_` | `uint64_t` | `0` | 操作 ID |

**继承**: 继承自 `MsgBase`，操作码为 `ML_UPDATE_REQ (2)`

---

### BmRegisterRequest

**声明位置**: 行 529-563

**功能描述**: Blob Manager 注册请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `rank_` | `uint32_t` | `UINT32_MAX` | Rank ID |
| `mediaType_` | `vector<uint16_t>` | `{}` | 支持的介质类型列表 |
| `addr_` | `vector<uint64_t>` | `{}` | 地址列表 |
| `capacity_` | `vector<uint64_t>` | `{}` | 容量列表 |
| `blobMap_` | `map<string, MmcMemBlobDesc>` | `{}` | Blob 映射表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BM_REGISTER_REQ (5)`

---

### BmUnregisterRequest

**声明位置**: 行 565-593

**功能描述**: Blob Manager 注销请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `rank_` | `uint32_t` | `0` | Rank ID |
| `mediaType_` | `vector<uint16_t>` | `{UINT16_MAX}` | 介质类型列表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BM_UNREGISTER_REQ (11)`

---

### MetaReplicateRequest

**声明位置**: 行 595-641

**功能描述**: 元数据复制请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `ops_` | `vector<uint32_t>` | `{}` | 操作列表 |
| `keys_` | `vector<string>` | `{}` | 键值列表 |
| `blobs_` | `vector<MmcMemBlobDesc>` | `{}` | Blob 描述符列表 |

**继承**: 继承自 `MsgBase`，操作码为 `LM_META_REPLICATE_REQ (7)`

**成员函数**:

#### KeysString()

```cpp
std::string KeysString()
```

**声明位置**: 行 628-640

**功能描述**: 将键值列表转换为字符串表示

**返回值**: 格式化的键值字符串，如 `"[key1, key2, key3]"`

---

### BlobCopyRequest

**声明位置**: 行 643-671

**功能描述**: Blob 拷贝请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `srcBlob_` | `MmcMemBlobDesc` | `{}` | 源 Blob 描述符 |
| `dstBlob_` | `MmcMemBlobDesc` | `{}` | 目标 Blob 描述符 |

**继承**: 继承自 `MsgBase`，操作码为 `LM_BLOB_COPY_REQ (17)`

---

### IsExistRequest

**声明位置**: 行 673-695

**功能描述**: 检查对象是否存在请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `key_` | `string` | `` | 键值 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_IS_EXIST_REQ (8)`

---

### BatchIsExistRequest

**声明位置**: 行 722-746

**功能描述**: 批量检查对象是否存在请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `keys_` | `vector<string>` | `{}` | 键值列表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_IS_EXIST_REQ (9)`

---

### QueryRequest

**声明位置**: 行 775-798

**功能描述**: 查询对象请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `key_` | `string` | `` | 键值 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_QUERY_REQ (13)`

---

### BatchQueryRequest

**声明位置**: 行 825-848

**功能描述**: 批量查询对象请求消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `keys_` | `vector<string>` | `{}` | 键值列表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_QUERY_REQ (14)`

---

### RemoveAllRequest

**声明位置**: 行 877-895

**功能描述**: 删除所有对象请求消息

**继承**: 继承自 `MsgBase`，操作码为 `LM_REMOVE_ALL_REQ (18)`

---

## 响应消息类型

### BatchAllocResponse

**声明位置**: 行 274-317

**功能描述**: 批量分配响应消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `blobs_` | `vector<vector<MmcMemBlobDesc>>` | `{}` | 分配的 Blob 描述符二维数组 |
| `numBlobs_` | `vector<uint8_t>` | `{}` | 每个对象的 Blob 数量 |
| `prots_` | `vector<uint16_t>` | `{}` | 保护属性列表 |
| `priorities_` | `vector<uint8_t>` | `{}` | 优先级列表 |
| `leases_` | `vector<uint64_t>` | `{}` | 租约列表 |
| `results_` | `vector<Result>` | `{}` | 操作结果列表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_ALLOC_RESP (8)`

---

### BatchRemoveResponse

**声明位置**: 行 370-395

**功能描述**: 批量删除响应消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `results_` | `vector<Result>` | `{}` | 操作结果列表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_REMOVE_RESP (6)`

---

### AllocResponse

**声明位置**: 行 397-434

**功能描述**: 分配响应消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `blobs_` | `vector<MmcMemBlobDesc>` | `{}` | 分配的 Blob 描述符列表 |
| `numBlobs_` | `uint8_t` | `0` | Blob 数量（副本数） |
| `prot_` | `uint16_t` | `0` | 保护属性（访问权限） |
| `priority_` | `uint8_t` | `0` | 优先级（用于驱逐策略） |
| `result_` | `Result` | `0` | 操作结果 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_ALLOC_RESP (1)`

---

### Response

**声明位置**: 行 476-499

**功能描述**: 通用响应消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `ret_` | `Result` | `0` | 返回码 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_UPDATE_REQ (2)`

---

### BatchUpdateResponse

**声明位置**: 行 501-527

**功能描述**: 批量更新响应消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `results_` | `vector<Result>` | `{}` | 操作结果列表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_UPDATE_RESP (11)`

---

### IsExistResponse

**声明位置**: 行 697-720

**功能描述**: 存在性检查响应消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `ret_` | `Result` | `-1` | 返回码（存在/不存在） |

**继承**: 继承自 `MsgBase`，操作码为 `ML_IS_EXIST_RESP (4)`

---

### BatchIsExistResponse

**声明位置**: 行 748-773

**功能描述**: 批量存在性检查响应消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `results_` | `vector<Result>` | `{}` | 操作结果列表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_IS_EXIST_RESP (5)`

---

### QueryResponse

**声明位置**: 行 800-823

**功能描述**: 查询响应消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `queryInfo_` | `MemObjQueryInfo` | `{}` | 内存对象查询信息 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_QUERY_RESP (9)`

**MemObjQueryInfo 结构说明**:
```cpp
struct MemObjQueryInfo {
    uint64_t size_;              // 对象大小
    uint16_t prot_;              // 保护属性
    uint8_t numBlobs_;           // Blob 数量
    bool valid_;                 // 是否有效
    uint32_t blobRanks_[MAX_BLOB_COPIES];  // Blob Rank 列表
    uint16_t blobTypes_[MAX_BLOB_COPIES];  // Blob 类型列表
};
```

---

### BatchQueryResponse

**声明位置**: 行 850-875

**功能描述**: 批量查询响应消息

**成员变量**:

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `batchQueryInfos_` | `vector<MemObjQueryInfo>` | `{}` | 批量查询信息列表 |

**继承**: 继承自 `MsgBase`，操作码为 `ML_BATCH_QUERY_RESP (10)`

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

## 依赖关系

**被以下文件依赖**:
- 客户端服务层
- 元服务层
- 本地服务层

**依赖的文件**:
- `mmc_mem_blob.h` - Blob 描述符 (`MmcMemBlobDesc`)
- `mmc_msg_base.h` - 消息基类 (`MsgBase`)
- `mmc_msg_packer.h` - 序列化工具 (`NetMsgPacker`, `NetMsgUnpacker`)

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有消息类型定义
}
}
```

---

## 消息通信流程示例

### 分配对象流程

```
Client                    Meta Service
  |                           |
  |--- AllocRequest --------->|
  |    (key, options, opId)   |
  |                           |
  |                           | 1. Deserialize
  |                           | 2. Process Allocation
  |                           |
  |<-- AllocResponse ---------|
  |    (blobs, prot, result)  |
  |                           |
```

### 批量查询流程

```
Client                    Meta Service
  |                           |
  |--- BatchQueryRequest ---->|
  |    (keys[])               |
  |                           |
  |                           | 1. Deserialize
  |                           | 2. Query Each Key
  |                           |
  |<-- BatchQueryResponse ----|
  |    (batchQueryInfos[])    |
  |                           |
```
