# mmc_net_link_acc.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/acc_links_impl/mmc_net_link_acc.h`
- **文件用途**: 定义 ACC 网络链接实现类
- **依赖项**:
  - `mmc_net_engine.h` - 网络引擎接口
  - `mmc_net_common_acc.h` - ACC 网络通用定义

---

## 类定义

### NetLinkAcc 类

ACC 网络链接实现类，包装底层的 TCP 链接

```cpp
class NetLinkAcc final : public NetLink {
public:
    NetLinkAcc(int32_t id, const TcpLinkPtr &tcpLink) : id_(id), tcpLink_(tcpLink) {}
    ~NetLinkAcc() override = default;

    int32_t Id() const override;
    const TcpLinkPtr &RealLink() const;

private:
    const int32_t id_;
    const TcpLinkPtr tcpLink_;
};
```

**设计说明**:
- `final` 关键字表示此类不能被继承
- 持有链接 ID 和底层 TCP 链接的智能指针
- 成员变量使用 `const` 修饰，确保不可修改

---

## 方法

### NetLinkAcc::NetLinkAcc()

```cpp
NetLinkAcc(int32_t id, const TcpLinkPtr &tcpLink) : id_(id), tcpLink_(tcpLink) {}
```

**声明位置**: 行 22

**功能描述**: 构造函数，初始化链接

**参数**:
- `id`: 链接 ID（通常是 peerId）
- `tcpLink`: 底层 TCP 链接智能指针

---

### NetLinkAcc::~NetLinkAcc()

```cpp
~NetLinkAcc() override = default;
```

**声明位置**: 行 23

**功能描述**: 析构函数，使用默认实现

---

### NetLinkAcc::Id()

```cpp
inline int32_t NetLinkAcc::Id() const
{
    return id_;
}
```

**声明位置**: 行 33-36

**功能描述**: 获取链接 ID

**返回值**: `int32_t` - 链接 ID

---

### NetLinkAcc::RealLink()

```cpp
inline const TcpLinkPtr &NetLinkAcc::RealLink() const
{
    return tcpLink_;
}
```

**声明位置**: 行 37-40

**功能描述**: 获取底层 TCP 链接

**返回值**: `const TcpLinkPtr&` - TCP 链接智能指针的常量引用

---

## 成员变量

```cpp
private:
    const int32_t id_;
    const TcpLinkPtr tcpLink_;
```

- `id_`: 链接 ID，存储对端的节点 ID
- `tcpLink_`: 底层 ACC TCP 链接智能指针

---

## 文件级别的关系图

```
mmc_net_link_acc.h (ACC 网络链接)
    |
    +-- NetLinkAcc -> NetLink 的 ACC 实现
            |
            +-- Id() -> 获取链接 ID
            +-- RealLink() -> 获取底层 TCP 链接
            +-- UpCtx()/UpCtx(c) -> 继承自 NetLink
            |
            +-> 持有 TcpLinkPtr 智能指针
```

---

## 依赖关系

**依赖以下文件**:
- `mmc_net_engine.h` - 网络引擎接口（NetLink 基类）
- `mmc_net_common_acc.h` - ACC 网络通用定义

**被以下文件依赖**:
- `mmc_net_engine_acc.cpp` - ACC 网络引擎实现

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有 ACC 网络链接定义都在 ock::mmc 命名空间内
}
}
```

---

## 使用示例

```cpp
// 创建链接
TcpLinkPtr tcpLink = /* ... 获取 TCP 链接 ... */;
auto netLink = MmcMakeRef<NetLinkAcc>(peerId, tcpLink);

// 获取链接 ID
int32_t linkId = netLink->Id();

// 设置/获取上下文值
netLink->UpCtx(peerId);  // 设置
uint64_t ctx = netLink->UpCtx();  // 获取

// 发送数据
netLink->RealLink()->NonBlockSend(MSG_TYPE_DATA, opCode, seqNo, dataBuf, nullptr);
```
