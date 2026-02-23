# mmc_client.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/mmc_client.cpp`
- **文件用途**: 实现客户端数据操作 API (mmc_client.h)
- **依赖项**: `mmc_client.h`, `mmc_common_includes.h`, `mmc_client_default.h`

---

## 常量定义

### BUF_TYPE_BASE (第19行)

```cpp
constexpr int BUF_TYPE_BASE = 2;
```

**说明**: 缓冲区类型基准值，用于验证 buffer 类型的有效性

---

### MAX_KEY_LEN (第20行)

```cpp
constexpr int MAX_KEY_LEN = 256;
```

**说明**: 键的最大长度限制

---

## 函数

### mmcc_init (第23-35行)

```cpp
MMC_API int32_t mmcc_init(mmc_client_config_t *config)
```

**声明位置**: 行 23-35

**功能描述**: 初始化分布式内存缓存客户端（单例模式）

**参数**:
- `config` [in]: 客户端配置

**返回值**:
- `MMC_OK`: 成功
- `MMC_INVALID_PARAM`: config 为空
- 其他错误码: 初始化失败

**代码逻辑**:
1. **参数验证** (第25行): 验证 config 非空

2. **注册实例** (第27行):
   - 调用 `MmcClientDefault::RegisterInstance()`
   - 创建单例客户端实例

3. **启动客户端** (第28-33行):
   - 调用 `MmcClientDefault::GetInstance()->Start()`
   - 失败时反注册实例

---

### mmcc_uninit (第37-42行)

```cpp
MMC_API void mmcc_uninit(void)
```

**声明位置**: 行 37-42

**功能描述**: 反初始化客户端

**参数**: 无

**返回值**: 无

**代码逻辑**:
1. 验证客户端已初始化
2. 调用 `Stop()` 停止客户端
3. 调用 `UnregisterInstance()` 反注册

---

### mmcc_register_buffer (第44-51行)

```cpp
MMC_API int32_t mmcc_register_buffer(uint64_t addr, uint64_t size)
```

**声明位置**: 行 44-51

**功能描述**: 注册内存缓冲区到 Blob Manager，支持 SDMA

**参数**:
- `addr` [in]: 要注册的缓冲区地址
- `size` [in]: 要注册的缓冲区大小

**返回值**:
- `MMC_OK`: 成功
- `MMC_INVALID_PARAM`: 参数无效
- `MMC_CLIENT_NOT_INIT`: 客户端未初始化

**代码逻辑**:
1. 验证地址非零
2. 验证大小大于零
3. 验证客户端已初始化
4. 调用 `MmcClientDefault::GetInstance()->RegisterBuffer()`

---

### mmcc_unregister_buffer (第53-60行)

```cpp
MMC_API int32_t mmcc_unregister_buffer(uint64_t addr, uint64_t size)
```

**声明位置**: 行 53-60

**功能描述**: 从 Blob Manager 注销内存缓冲区

**参数**:
- `addr` [in]: 要注销的缓冲区地址
- `size` [in]: 要注销的缓冲区大小

**返回值**: 同 `mmcc_register_buffer`

---

### mmcc_put (第62-77行)

```cpp
MMC_API int32_t mmcc_put(const char *key, mmc_buffer *buf, mmc_put_options options, uint32_t flags)
```

**声明位置**: 行 62-77

**功能描述**: 将数据对象存入分布式内存缓存

**参数**:
- `key` [in]: 数据的键，长度小于 256
- `buf` [in]: 要存储的数据缓冲区
- `options` [in]: Put 操作选项
- `flags` [in]: 可选标志位，保留

**返回值**:
- `MMC_OK`: 成功
- `MMC_DUPLICATED_OBJECT`: 对象已存在
- 其他错误码: 失败

**代码逻辑**:
1. **参数验证** (第64-69行):
   - 验证 key 非空且长度有效
   - 验证 buf 非空
   - 验证 buf 地址非空
   - 验证客户端已初始化

2. **执行 Put 操作** (第71-74行):
   - 调用客户端的 `Put()` 方法
   - 处理重复对象情况（不视为错误）

3. **错误处理** (第72-74行):
   - 除重复对象外，其他错误需要记录日志

---

### mmcc_get (第79-91行)

```cpp
MMC_API int32_t mmcc_get(const char *key, mmc_buffer *buf, uint32_t flags)
```

**声明位置**: 行 79-91

**功能描述**: 根据键从分布式内存缓存获取数据对象

**参数**:
- `key` [in]: 数据的键，长度小于 256
- `buf` [in/out]: 用于存储获取数据的缓冲区
- `flags` [in]: 可选标志位，保留

**返回值**:
- `MMC_OK`: 成功

**代码逻辑**:
1. 验证 key、buf 参数有效性
2. 验证客户端已初始化
3. 调用客户端的 `Get()` 方法

---

### mmcc_query (第93-104行)

```cpp
MMC_API int32_t mmcc_query(const char *key, mmc_data_info *info, uint32_t flags)
```

**声明位置**: 行 93-104

**功能描述**: 根据键查询数据对象的信息（不传输数据）

**参数**:
- `key` [in]: 数据的键，长度小于 256
- `info` [out]: 输出数据信息
- `flags` [in]: 可选标志位，保留

**返回值**:
- `MMC_OK`: 成功

---

### mmcc_batch_query (第106-152行)

```cpp
MMC_API int32_t mmcc_batch_query(const char **keys, size_t keys_count, mmc_data_info *info, uint32_t flags)
```

**声明位置**: 行 106-152

**功能描述**: 批量查询多个键的数据信息

**参数**:
- `keys` [in]: 键数组
- `keys_count` [in]: 键的数量
- `info` [out]: 输出数据信息数组
- `flags` [in]: 操作标志位

**返回值**:
- `MMC_OK`: 成功

**代码逻辑**:
1. **参数验证** (第108-112行):
   - 验证 keys 非空
   - 验证 keys_count 在有效范围内
   - 验证 info 非空
   - 验证客户端已初始化

2. **处理无效键** (第121-133行):
   - 过滤掉空键或超长键
   - 记录无效键的索引

3. **执行批量查询** (第135行):
   - 调用客户端的 `BatchQuery()` 方法

4. **结果重组** (第142-150行):
   - 将结果填充到原始数组位置
   - 无效键位置填充空数据信息

---

### mmcc_remove (第154-163行)

```cpp
MMC_API int32_t mmcc_remove(const char *key, uint32_t flags)
```

**声明位置**: 行 154-163

**功能描述**: 从分布式内存缓存中移除指定键的对象

**参数**:
- `key` [in]: 要移除的数据的键，长度小于 256
- `flags` [in]: 可选标志位，保留

**返回值**:
- `MMC_OK`: 成功

---

### mmcc_batch_remove (第165-212行)

```cpp
MMC_API int32_t mmcc_batch_remove(const char **keys, const uint32_t keys_count, int32_t *remove_results, uint32_t flags)
```

**声明位置**: 行 165-212

**功能描述**: 批量移除多个键的数据对象

**参数**:
- `keys` [in]: 要移除的键列表
- `keys_count` [in]: 键的数量
- `remove_results` [out]: 每个移除操作的结果数组
- `flags` [in]: 操作标志位

**返回值**:
- `MMC_OK`: 成功
- 正值: 发生错误

**代码逻辑**: 类似 `mmcc_batch_query`，处理无效键并重组结果

---

### mmcc_exist (第214-227行)

```cpp
MMC_API int32_t mmcc_exist(const char *key, uint32_t flags)
```

**声明位置**: 行 214-227

**功能描述**: 判断指定键是否存在于 Blob Manager 中

**参数**:
- `key` [in]: 要判断的键，长度小于 256
- `flags` [in]: 可选标志位，保留

**返回值**:
- `MMC_OK`: 存在
- `MMC_UNMATCHED_KEY`: 不存在（不记录错误日志）

---

### mmcc_batch_exist (第229-276行)

```cpp
MMC_API int32_t mmcc_batch_exist(const char **keys, const uint32_t keys_count, int32_t *exist_results, uint32_t flags)
```

**声明位置**: 行 229-276

**功能描述**: 批量判断多个键是否存在于 Blob Manager 中

**参数**:
- `keys` [in]: 要判断的键列表
- `keys_count` [in]: 键的数量
- `exist_results` [out]: 每个键的存在状态列表
- `flags` [in]: 操作标志位

**返回值**:
- `MMC_OK`: 成功

---

### mmcc_batch_get (第278-311行)

```cpp
MMC_API int32_t mmcc_batch_get(const char **keys, uint32_t keys_count, mmc_buffer *bufs, uint32_t flags, int *results)
```

**声明位置**: 行 278-311

**功能描述**: 批量获取多个数据对象

**参数**:
- `keys` [in]: 数据对象的键数组
- `keys_count` [in]: 键的数量
- `bufs` [out]: 存储获取数据的缓冲区数组
- `flags` [in]: 可选标志位，保留
- `results` [out]: 每个操作的结果

**返回值**:
- `MMC_OK`: 成功

**代码逻辑**:
1. 参数验证
2. 构建 C++ vector
3. 调用客户端的 `BatchGet()` 方法
4. 填充结果数组

---

### mmcc_batch_put (第313-353行)

```cpp
MMC_API int32_t mmcc_batch_put(const char **keys, uint32_t keys_count, const mmc_buffer *bufs, mmc_put_options &options,
                               uint32_t flags, int *results)
```

**声明位置**: 行 313-353

**功能描述**: 批量将多个数据对象存入分布式内存缓存

**参数**:
- `keys` [in]: 数据对象的键数组
- `keys_count` [in]: 键的数量
- `bufs` [in]: 要存储的数据缓冲区数组
- `options` [in]: 批量 Put 操作选项
- `flags` [in]: 可选标志位，保留
- `results` [out]: 每个操作的结果

**返回值**:
- `MMC_OK`: 成功

**代码逻辑**:
1. 参数验证（包括 buffer 类型验证）
2. 构建 C++ vector
3. 调用客户端的 `BatchPut()` 方法
4. 填充结果数组

---

### mmcc_local_service_id (第355-361行)

```cpp
MMC_API int32_t mmcc_local_service_id(uint32_t *localServiceId)
```

**声明位置**: 行 355-361

**功能描述**: 查询本地服务 ID

**参数**:
- `localServiceId` [out]: 输出本地服务 ID

**返回值**:
- `MMC_OK`: 成功
- `MMC_INVALID_PARAM`: localServiceId 为空
- `MMC_CLIENT_NOT_INIT`: 客户端未初始化

**代码逻辑**:
1. 验证参数有效性
2. 从客户端获取 RankId
3. 返回成功

---

## API 调用流程

```
用户调用 mmcc_xxx()
    |
    v
参数验证
    |
    v
获取客户端单例 (MmcClientDefault::GetInstance())
    |
    v
调用客户端方法 (Put/Get/Remove/...)
    |
    v
返回结果
```

---

## 文件级别的关系图

```
mmc_client.cpp (C API 实现)
    |
    +-- 依赖: mmc_client.h (C 接口定义)
    +-- 依赖: mmc_client_default.h (C++ 实现类)
    +-- 依赖: mmc_common_includes.h (公共定义)
    |
    +-- 调用: MmcClientDefault::GetInstance()
    +-- 调用: MmcClientDefault::Put()
    +-- 调用: MmcClientDefault::Get()
    +-- 调用: MmcClientDefault::BatchPut()
    +-- 调用: MmcClientDefault::BatchGet()
    +-- ... 其他批量操作
```

---

## 批量操作处理模式

所有批量操作 (batch_xxx) 遵循相同的处理模式:

1. **参数验证**: 验证输入数组和计数有效
2. **无效键过滤**: 识别并跳过无效键
3. **向量构建**: 将 C 数组转换为 C++ vector
4. **调用实现**: 调用 C++ 客户端方法
5. **结果重组**: 将结果填充回原始数组，保持无效键位置对应

```
C 数组 [key0, key1(nullptr), key2, key3(invalid), key4]
    |
    v
过滤后 [key0, key2, key4]
    |
    v
批量操作
    |
    v
结果 [res0, res2, res4]
    |
    v
重组后 [res0, error, res2, error, res4]
```
