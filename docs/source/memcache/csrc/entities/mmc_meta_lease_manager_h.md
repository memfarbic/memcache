# mmc_meta_lease_manager.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/entities/mmc_meta_lease_manager.h`
- **文件用途**: 定义元数据租约管理器，用于管理 Blob 的并发访问租约
- **依赖项**:
  - `unordered_set` - 哈希集合
  - `mmc_logger.h` - 日志系统
  - `mmc_montotonic.h` - 单调时间
  - `mmc_ref.h` - 引用计数智能指针
  - `mmc_types.h` - 类型定义

---

## 常量定义

### RANK_ID_BIT_SHIFT

**声明位置**: 行 26
**完整签名**:
```cpp
constexpr int RANK_ID_BIT_SHIFT = 32;
```
**功能描述**: Rank ID 在 client ID 中的位移量
**说明**: 用于将 rankId 和 requestId 组合成一个 64 位的 client ID

---

## 类逐个解读

### MmcMetaLeaseManager

**声明位置**: 行 27-51
**完整签名**:
```cpp
class MmcMetaLeaseManager : public MmcReferable
```
**功能描述**: 元数据租约管理器，管理 Blob 的访问租约
**继承关系**:
```
MmcReferable
    ▲
    │
MmcMetaLeaseManager
```

---

#### Add

**声明位置**: 行 29
**完整签名**:
```cpp
Result Add(uint32_t id, uint32_t requestId, uint64_t ttl)
```
**功能描述**: 添加一个租约记录
**参数**:
- `id` - Rank ID
- `requestId` - 请求 ID
- `ttl` - 租约存活时间（毫秒）
**返回值**: `Result` - 操作结果
**代码逻辑** (实现位于 cpp):
1. 获取当前时间
2. 更新租约到期时间为 max(当前时间, 当前时间 + ttl)
3. 将 (rankId, requestId) 组成的 client ID 添加到集合中

---

#### Remove

**声明位置**: 行 30
**完整签名**:
```cpp
Result Remove(uint32_t id, uint32_t requestId)
```
**功能描述**: 移除一个租约记录
**参数**:
- `id` - Rank ID
- `requestId` - 请求 ID
**返回值**: `Result` - 操作结果
**代码逻辑** (实现位于 cpp):
1. 从集合中移除指定的 client ID

---

#### Extend

**声明位置**: 行 31
**完整签名**:
```cpp
Result Extend(uint64_t ttl)
```
**功能描述**: 延长所有租约的有效期
**参数**:
- `ttl` - 延长的时长（毫秒）
**返回值**: `Result` - 操作结果
**代码逻辑** (实现位于 cpp):
1. 获取当前时间
2. 更新租约到期时间为 max(当前时间, 当前时间 + ttl)

---

#### Wait

**声明位置**: 行 32
**完整签名**:
```cpp
void Wait()
```
**功能描述**: 等待所有租约过期或释放
**代码逻辑** (实现位于 cpp):
1. 循环检查租约集合
2. 如果集合为空，直接返回
3. 如果当前时间超过租约到期时间，返回
4. 否则休眠一段时间后再检查

---

#### UseCount

**声明位置**: 行 33, 55-58 (内联实现)
**完整签名**:
```cpp
inline uint32_t UseCount() { return useClient.size(); }
```
**功能描述**: 获取当前活跃租约数量
**返回值**: 当前租约数量

---

#### GenerateClientId

**声明位置**: 行 34, 60-63 (内联实现)
**完整签名**:
```cpp
inline uint64_t GenerateClientId(uint32_t rankId, uint32_t requestId)
{
    return (static_cast<uint64_t>(rankId) << RANK_ID_BIT_SHIFT) | requestId;
}
```
**功能描述**: 生成唯一的客户端 ID
**参数**:
- `rankId` - Rank ID
- `requestId` - 请求 ID
**返回值**: 64 位客户端 ID
**编码格式**:
```
┌─────────────────────┬───────────────────────┐
│   Rank ID (高32位)   │   Request ID (低32位) │
└─────────────────────┴───────────────────────┘
```

---

#### RankId

**声明位置**: 行 35, 64-67 (内联实现)
**完整签名**:
```cpp
inline uint32_t RankId(uint64_t clientId)
{
    return static_cast<uint32_t>(clientId >> RANK_ID_BIT_SHIFT);
}
```
**功能描述**: 从客户端 ID 解析出 Rank ID
**参数**:
- `clientId` - 客户端 ID
**返回值**: Rank ID（高 32 位）

---

#### RequestId

**声明位置**: 行 36, 68-71 (内联实现)
**完整签名**:
```cpp
inline uint32_t RequestId(uint64_t clientId)
{
    return static_cast<uint32_t>(clientId & 0xFFFFFFFF);
}
```
**功能描述**: 从客户端 ID 解析出请求 ID
**参数**:
- `clientId` - 客户端 ID
**返回值**: 请求 ID（低 32 位）

---

#### 成员变量

**声明位置**: 行 49-50
```cpp
private:
    uint64_t lease_{0};                       /* lease of the memory object */
    std::unordered_set<uint64_t> useClient;   /* 客户端 ID 集合 */
```

**成员变量说明**:
| 变量名 | 类型 | 说明 |
|--------|------|------|
| `lease_` | `uint64_t` | 租约到期时间戳（毫秒） |
| `useClient` | `std::unordered_set<uint64_t>` | 持有租约的客户端 ID 集合 |

---

#### 流输出运算符

**声明位置**: 行 38-46
**完整签名**:
```cpp
friend std::ostream &operator<<(std::ostream &os, const MmcMetaLeaseManager &leaseMgr)
```
**功能描述**: 输出租约管理器信息到流
**输出格式**:
```
lease={xxx,client:id1,id2,...}
```

---

## 类型别名

```cpp
using MmcMetaLeaseManagerPtr = MmcRef<MmcMetaLeaseManager>;
```

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      MmcMetaLeaseManager                                │
├─────────────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────────────────┐ │
│ │ 成员变量                                                             │ │
│ │ + lease_: uint64_t                  (租约到期时间，毫秒)              │ │
│ │ + useClient: unordered_set<uint64_t> (客户端 ID 集合)                 │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────────┤
│ 公共方法                                                                │
│ + Add(id, requestId, ttl): Result          (添加租约)                   │
│ + Remove(id, requestId): Result            (移除租约)                   │
│ + Extend(ttl): Result                      (延长租约)                   │
│ + Wait(): void                             (等待租约过期)               │
│ + UseCount(): uint32_t                     (获取租约数量)               │
│                                                                          │
│ 静态工具方法                                                              │
│ + GenerateClientId(rankId, requestId): uint64_t  (生成客户端 ID)        │
│ + RankId(clientId): uint32_t                    (解析 Rank ID)         │
│ + RequestId(clientId): uint32_t                  (解析 Request ID)      │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Client ID 编码/解码示意图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Client ID 编码                                  │
└─────────────────────────────────────────────────────────────────────────┘

  生成 Client ID:
  ┌─────────────────────┐
  │ GenerateClientId()  │
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Rank ID (32 bit)    │    Request ID (32 bit)                   │
  │        << 32         │              |                            │
  │                     │              │                            │
  │                     └──────────────┴──────────────┐              │
  │                                                │              │
  │                           (OR operation)         │              │
  │                                                │              │
  │                                   ┌─────────────┴───────────┐  │
  │                                   │                         │  │
  │                                   ▼                         │  │
  │                         ┌──────────────────┐                │  │
  │                         │   64-bit Client  │                │  │
  │                         │       ID         │                │  │
  │                         └──────────────────┘                │  │
  └─────────────────────────────────────────────────────────────┘  │
                                                                 │
  解码 Client ID:                                                │
  ┌─────────────────────────────────────────────────────────────┘
             │
             ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  64-bit Client ID                                               │
  │        │                                                         │
  │        ├─ RankId() ──▶ 高32位 ──▶ Rank ID                       │
  │        │                                                         │
  │        └─ RequestId() ─▶ 低32位 ──▶ Request ID                  │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 租约生命周期

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          租约生命周期                                   │
└─────────────────────────────────────────────────────────────────────────┘

  1. 创建租约
  ┌─────────────────┐
  │   Add()         │
  │ rankId=0        │
  │ requestId=100   │
  │ ttl=2000ms      │
  └────────┬────────┘
           │
           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  useClient = {0x0000000000000064}  (0 << 32 | 100)              │
  │  lease_ = current_time + 2000                                  │
  └─────────────────────────────────────────────────────────────────┘

  2. 延长租约
  ┌─────────────────┐
  │   Extend()      │
  │   ttl=1000ms    │
  └────────┬────────┘
           │
           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  lease_ = max(lease_, current_time + 1000)                      │
  └─────────────────────────────────────────────────────────────────┘

  3. 移除租约
  ┌─────────────────┐
  │   Remove()      │
  │ rankId=0        │
  │ requestId=100   │
  └────────┬────────┘
           │
           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  useClient.erase(0x0000000000000064)                            │
  │  useClient = {}                                                 │
  └─────────────────────────────────────────────────────────────────┘

  4. 等待租约过期
  ┌─────────────────┐
  │   Wait()        │
  └────────┬────────┘
           │
           ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  while (!useClient.empty()) {                                   │
  │      if (current_time >= lease_) return;  // 租约已过期          │
  │      sleep(MMC_DATA_TTL_MS / 10);                              │
  │  }                                                              │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 使用示例

### 示例 1: 基本租约操作

```cpp
#include "mmc_meta_lease_manager.h"

using namespace ock::mmc;

MmcMetaLeaseManager leaseMgr;

// 添加租约
Result ret = leaseMgr.Add(0, 100, 2000);
if (ret == MMC_OK) {
    std::cout << "Lease added" << std::endl;
    std::cout << "Use count: " << leaseMgr.UseCount() << std::endl;
}

// 延长租约
ret = leaseMgr.Extend(1000);

// 移除租约
ret = leaseMgr.Remove(0, 100);
std::cout << "Use count after remove: " << leaseMgr.UseCount() << std::endl;
```

### 示例 2: 等待租约过期

```cpp
// 有多个客户端持有租约
leaseMgr.Add(0, 100, 2000);
leaseMgr.Add(1, 101, 2000);
leaseMgr.Add(2, 102, 2000);

std::cout << "Current leases: " << leaseMgr.UseCount() << std::endl;

// 等待所有租约过期
leaseMgr.Wait();

std::cout << "All leases expired" << std::endl;
```

### 示例 3: Client ID 编码/解码

```cpp
// 生成 Client ID
uint32_t rankId = 5;
uint32_t requestId = 12345;
uint64_t clientId = MmcMetaLeaseManager::GenerateClientId(rankId, requestId);
// clientId = 0x0000000500003039

// 解码
uint32_t decodedRankId = MmcMetaLeaseManager::RankId(clientId);
uint32_t decodedRequestId = MmcMetaLeaseManager::RequestId(clientId);

std::cout << "Rank ID: " << decodedRankId << std::endl;       // 5
std::cout << "Request ID: " << decodedRequestId << std::endl; // 12345
```

### 示例 4: 多客户端并发访问

```cpp
MmcMetaLeaseManager leaseMgr;

// 客户端 1 开始读取
leaseMgr.Add(0, 100, 2000);
std::cout << "Client 1 reading, leases: " << leaseMgr.UseCount() << std::endl;

// 客户端 2 开始读取
leaseMgr.Add(1, 101, 2000);
std::cout << "Client 2 reading, leases: " << leaseMgr.UseCount() << std::endl;

// 客户端 1 完成读取
leaseMgr.Remove(0, 100);
std::cout << "Client 1 done, leases: " << leaseMgr.UseCount() << std::endl;

// 客户端 2 完成读取
leaseMgr.Remove(1, 101);
std::cout << "Client 2 done, leases: " << leaseMgr.UseCount() << std::endl;

// 等待租约清空
leaseMgr.Wait();
```

### 示例 5: 流输出

```cpp
MmcMetaLeaseManager leaseMgr;

leaseMgr.Add(0, 100, 2000);
leaseMgr.Add(1, 101, 2000);

std::cout << leaseMgr << std::endl;
// 输出: lease={xxx,client:0x0000000000000064,0x0000000100000065,}
```
