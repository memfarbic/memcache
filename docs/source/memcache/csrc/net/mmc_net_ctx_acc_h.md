# mmc_net_ctx_acc.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/acc_links_impl/mmc_net_ctx_acc.h`
- **文件用途**: 定义 ACC 网络上下文实现类
- **依赖项**:
  - `algorithm` - 算法库
  - `mmc_net_engine.h` - 网络引擎接口
  - `mmc_net_common_acc.h` - ACC 网络通用定义

---

## 类定义

### NetContextAcc 类

ACC 网络上下文实现类，包装底层的 TCP 请求上下文

```cpp
class NetContextAcc final : public NetContext {
public:
    explicit NetContextAcc(const TcpReqContext &ctx) : realContext(ctx) {}

    int32_t Reply(int16_t responseCode, const char *respData, uint32_t &respDataLen) override;

    uint32_t SeqNo() const override;
    int16_t OpCode() const override;
    int16_t SrcRankId() const override;
    uint32_t DataLen() const override;
    void *Data() const override;

private:
    const TcpReqContext realContext;
};
```

**设计说明**:
- `final` 关键字表示此类不能被继承
- 持有底层 ACC TCP 请求上下文的引用
- 实现所有 NetContext 虚函数

---

## 类型别名

```cpp
using NetContextAccPtr = MmcRef<NetContextAcc>;
```

**说明**: NetContextAcc 的智能指针类型别名

---

## 方法

### NetContextAcc::NetContextAcc()

```cpp
explicit NetContextAcc(const TcpReqContext &ctx) : realContext(ctx) {}
```

**声明位置**: 行 23

**功能描述**: 构造函数，接收底层的 TCP 请求上下文

**参数**: `ctx` - ACC TCP 请求上下文（常量引用）

---

### NetContextAcc::Reply()

```cpp
inline int32_t NetContextAcc::Reply(int16_t responseCode, const char *respData, uint32_t &respDataLen)
{
    /* step2: copy data */
    TcpDataBufPtr dataBuf = new (std::nothrow) ock::acc::AccDataBuffer(respDataLen);
    MMC_ASSERT_RETURN(dataBuf.Get() != nullptr, MMC_NEW_OBJECT_FAILED);
    MMC_ASSERT_RETURN(dataBuf->AllocIfNeed(), MMC_NEW_OBJECT_FAILED);
    std::copy_n(respData, respDataLen, static_cast<char *>(dataBuf->DataPtrVoid()));
    dataBuf->SetDataSize(respDataLen);
    return realContext.Reply(responseCode, dataBuf);
}
```

**声明位置**: 行 42-51

**功能描述**: 响应请求，发送响应数据

**参数**:
- `responseCode`: 响应操作码
- `respData`: 响应数据指针
- `respDataLen`: 响应数据长度

**返回值**: `int32_t` - MMC_OK 表示成功

**代码逻辑**:
1. 创建 ACC 数据缓冲区
2. 分配内存
3. 拷贝响应数据到缓冲区
4. 调用底层上下文的 Reply 方法

---

### NetContextAcc::SeqNo()

```cpp
inline uint32_t NetContextAcc::SeqNo() const
{
    return realContext.Header().seqNo;
}
```

**声明位置**: 行 53-56

**功能描述**: 获取请求序列号

**返回值**: `uint32_t` - 序列号

---

### NetContextAcc::OpCode()

```cpp
inline int16_t NetContextAcc::OpCode() const
{
    return realContext.Header().result;
}
```

**声明位置**: 行 58-61

**功能描述**: 获取操作码

**返回值**: `int16_t` - 操作码

**注意**: ACC 框架使用 result 字段存储操作码

---

### NetContextAcc::SrcRankId()

```cpp
inline int16_t NetContextAcc::SrcRankId() const
{
    return 0;
}
```

**声明位置**: 行 63-66

**功能描述**: 获取源 Rank ID

**返回值**: `int16_t` - 固定返回 0

**注意**: 当前实现未从底层获取真实的 Rank ID

---

### NetContextAcc::DataLen()

```cpp
inline uint32_t NetContextAcc::DataLen() const
{
    return realContext.DataLen();
}
```

**声明位置**: 行 68-71

**功能描述**: 获取请求数据长度

**返回值**: `uint32_t` - 数据长度

---

### NetContextAcc::Data()

```cpp
inline void *NetContextAcc::Data() const
{
    return realContext.DataPtr();
}
```

**声明位置**: 行 73-76

**功能描述**: 获取请求数据指针

**返回值**: `void*` - 数据指针

---

## 文件级别的关系图

```
mmc_net_ctx_acc.h (ACC 网络上下文)
    |
    +-- NetContextAcc -> NetContext 的 ACC 实现
            |
            +-- Reply() -> 响应请求
            +-- SeqNo() -> 获取序列号
            +-- OpCode() -> 获取操作码
            +-- SrcRankId() -> 获取源 Rank ID
            +-- DataLen() -> 获取数据长度
            +-- Data() -> 获取数据指针
            |
            +-> 持有 TcpReqContext 引用
```

---

## 依赖关系

**依赖以下文件**:
- `algorithm` - STL 算法库
- `mmc_net_engine.h` - 网络引擎接口（NetContext 基类）
- `mmc_net_common_acc.h` - ACC 网络通用定义

**被以下文件依赖**:
- `mmc_net_engine_acc.cpp` - ACC 网络引擎实现

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有 ACC 网络上下文定义都在 ock::mmc 命名空间内
}
}
```

---

## 使用示例

```cpp
// 在请求处理器中使用
void RequestHandler(NetContextPtr &ctx) {
    // 获取请求数据
    uint32_t dataLen = ctx->DataLen();
    void *data = ctx->Data();

    // 处理请求...

    // 响应
    ResponseData resp;
    ctx->Reply(RESPONSE_CODE, reinterpret_cast<char*>(&resp), respLen);
}
```
