# mmc_meta_net_server.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_net_server.h`
- **文件用途**: 定义元数据网络服务器类，处理来自客户端的网络请求
- **依赖项**: `mmc_meta_common.h`, `mmc_net_engine.h`

---

## 类定义

### MetaNetServer

元数据网络服务器类，处理客户端的网络请求并调用元服务处理。

---

#### 构造函数

```cpp
explicit MetaNetServer(const MmcMetaServicePtr &metaService, const std::string inputName = "");
```

**声明位置**: 行 22

**功能描述**: 构造网络服务器

**参数**:
- `metaService`: 元服务指针
- `inputName`: 服务器名称

---

#### 析构函数

```cpp
~MetaNetServer() override;
```

**声明位置**: 行 24

**功能描述**: 析构函数

---

### Start

```cpp
Result Start(NetEngineOptions &options);
```

**声明位置**: 行 26

**功能描述**: 启动网络服务器

**参数**:
- `options`: 网络引擎选项

**返回值**: 成功返回 MMC_OK

---

### Stop

```cpp
void Stop();
```

**声明位置**: 行 28

**功能描述**: 停止网络服务器

---

### SyncCall

```cpp
template<typename REQ, typename RESP>
Result SyncCall(uint32_t rankId, const REQ &req, RESP &resp, int32_t timeoutInSecond)
{
    return engine_->Call(rankId, req.msgId, req, resp, timeoutInSecond);
}
```

**声明位置**: 行 30-34

**功能描述**: 同步调用远程节点

**参数**:
- `rankId`: 目标 Rank ID
- `req`: 请求对象
- `resp`: 响应对象
- `timeoutInSecond`: 超时时间（秒）

**返回值**: 成功返回 MMC_OK

---

## 私有方法

### HandleNewLink

```cpp
Result HandleNewLink(const NetLinkPtr &link);
```

**声明位置**: 行 37

**功能描述**: 处理新连接建立事件

**参数**:
- `link`: 网络连接指针

---

### HandleLinkBroken

```cpp
Result HandleLinkBroken(const NetLinkPtr &link);
```

**声明位置**: 行 39

**功能描述**: 处理连接断开事件

**参数**:
- `link`: 网络连接指针

---

### 消息处理函数

```cpp
Result HandleAlloc(const NetContextPtr &context);
Result HandleBatchAlloc(const NetContextPtr &context);
Result HandleBmRegister(const NetContextPtr &context);
Result HandleBmUnregister(const NetContextPtr &context);
Result HandlePing(const NetContextPtr &context);
Result HandleUpdate(const NetContextPtr &context);
Result HandleBatchUpdate(const NetContextPtr &context);
Result HandleGet(const NetContextPtr &context);
Result HandleBatchGet(const NetContextPtr &context);
Result HandleRemove(const NetContextPtr &context);
Result HandleBatchRemove(const NetContextPtr &context);
Result HandleRemoveAll(const NetContextPtr &context);
Result HandleIsExist(const NetContextPtr &context);
Result HandleBatchIsExist(const NetContextPtr &context);
Result HandleQuery(const NetContextPtr &context);
Result HandleBatchQuery(const NetContextPtr &context);
```

**声明位置**: 行 41-71

**功能描述**: 各种消息类型的处理函数

---

## 成员变量

```cpp
private:
    NetEnginePtr engine_;
    MmcMetaServicePtr metaService_;

    /* not hot used variables */
    std::mutex mutex_;
    bool started_ = false;
    std::string name_;
```

**说明**:
- `engine_`: 网络引擎指针
- `metaService_`: 元服务指针
- `mutex_`: 互斥锁（非热路径）
- `started_`: 启动标志（非热路径）
- `name_`: 服务器名称（非热路径）

---

## 类型别名

```cpp
using MetaNetServerPtr = MmcRef<MetaNetServer>;
```

**声明位置**: 行 82

**说明**: 网络服务器智能指针类型

---

## 文件级别的关系图

```
mmc_meta_net_server.h (元网络服务器)
    |
    +-- 使用 NetEngine (网络引擎)
    |
    +-- 使用 MmcMetaService (元服务)
    |
    +-- 被 MmcMetaMgrProxy 使用
    |
    +-- 被 MMCMetaBackUpMgrDefault 使用
    |
    +-- 功能:
    |   +-- 消息处理路由
    |   +-- 连接管理
    |   +-- 同步 RPC 调用
```

---

## 消息类型

| 消息 | 处理函数 | 描述 |
|------|----------|------|
| ML_ALLOC_REQ | HandleAlloc | 分配请求 |
| ML_BATCH_ALLOC_REQ | HandleBatchAlloc | 批量分配请求 |
| ML_BM_REGISTER_REQ | HandleBmRegister | BM 注册请求 |
| ML_BM_UNREGISTER_REQ | HandleBmUnregister | BM 注销请求 |
| ML_PING_REQ | HandlePing | Ping 请求 |
| ML_UPDATE_REQ | HandleUpdate | 更新请求 |
| ML_BATCH_UPDATE_REQ | HandleBatchUpdate | 批量更新请求 |
| ML_GET_REQ | HandleGet | 获取请求 |
| ML_BATCH_GET_REQ | HandleBatchGet | 批量获取请求 |
| ML_REMOVE_REQ | HandleRemove | 删除请求 |
| ML_BATCH_REMOVE_REQ | HandleBatchRemove | 批量删除请求 |
| LM_REMOVE_ALL_REQ | HandleRemoveAll | 删除所有请求 |
| ML_IS_EXIST_REQ | HandleIsExist | 检查存在请求 |
| ML_BATCH_IS_EXIST_REQ | HandleBatchIsExist | 批量检查存在请求 |
| ML_QUERY_REQ | HandleQuery | 查询请求 |
| ML_BATCH_QUERY_REQ | HandleBatchQuery | 批量查询请求 |
