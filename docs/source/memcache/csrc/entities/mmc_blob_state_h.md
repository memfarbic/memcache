# mmc_blob_state.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/entities/mmc_blob_state.h`
- **文件用途**: 定义 Blob 状态机相关的枚举、类型和状态转换表结构
- **依赖项**:
  - `utility` - 标准工具库
  - `functional` - 函数对象封装
  - `mmc_common_includes.h` - 公共头文件
  - `mmc_meta_lease_manager.h` - 租约管理器

---

## 类型别名

### BlobLeaseFunction

**声明位置**: 行 26
**完整签名**:
```cpp
using BlobLeaseFunction = std::function<Result(MmcMetaLeaseManager &leaseMgr, uint32_t rankId, uint32_t requestId)>;
```
**功能描述**: 状态转换时执行的租约操作函数类型
**参数**:
- `leaseMgr` - 租约管理器引用
- `rankId` - Rank ID
- `requestId` - 请求 ID
**返回值**: 操作结果 `Result`

### StateTransTable

**声明位置**: 行 112
**完整签名**:
```cpp
using StateTransTable = std::unordered_map<BlobState, std::unordered_map<Result, BlobStateAction>>;
```
**功能描述**: 状态转换表类型，嵌套哈希映射结构
**结构说明**:
- 外层 `unordered_map`: 当前状态 -> 结果映射表
- 内层 `unordered_map`: 操作结果 -> 状态动作

---

## 枚举类型

### BlobState

**声明位置**: 行 36-41
**完整签名**:
```cpp
enum BlobState : uint8_t {
    ALLOCATED,  // 已分配，正在写入
    READABLE,   // 可读
    REMOVING,   // 正在移除
    NONE,       // 无效/初始状态
};
```
**功能描述**: Blob 的生命周期状态
**状态机转换图**:
```
NONE -----> ALLOCATED -----------
                |               |
                |               |
                |               |
            READABLE -----> REMOVING -----> NONE
```

**状态说明**:
| 状态 | 说明 |
|------|------|
| `NONE` | 初始状态或已被移除 |
| `ALLOCATED` | 内存已分配，等待数据写入 |
| `READABLE` | 数据已写入完成，可以读取 |
| `REMOVING` | 正在移除过程中 |

**流输出运算符** (行 43-60):
```cpp
inline std::ostream &operator<<(std::ostream &os, BlobState type)
```
将状态枚举转换为字符串输出。

---

### BlobActionResult

**声明位置**: 行 72-82
**完整签名**:
```cpp
enum BlobActionResult : uint8_t {
    MMC_ALLOCATED_OK,  // alloc complete
    MMC_WRITE_OK,      // write success
    MMC_WRITE_FAIL,    // write failed
    MMC_READ_START,    // read operation started
    MMC_READ_FINISH,   // read operation finished
    MMC_REMOVE_START   // remove operation started
};
```
**功能描述**: Blob 操作的结果码，用于驱动状态机转换

**结果码说明**:
| 结果码 | 说明 | 可能的源状态 |
|--------|------|-------------|
| `MMC_ALLOCATED_OK` | 分配完成 | `ALLOCATED` |
| `MMC_WRITE_OK` | 写入成功 | `ALLOCATED` |
| `MMC_WRITE_FAIL` | 写入失败 | `ALLOCATED` |
| `MMC_READ_START` | 开始读取 | `READABLE` |
| `MMC_READ_FINISH` | 读取完成 | `READABLE` |
| `MMC_REMOVE_START` | 开始移除 | `ALLOCATED`, `READABLE` |

**流输出运算符** (行 84-110):
```cpp
inline std::ostream &operator<<(std::ostream &os, BlobActionResult ret)
```
将结果枚举转换为字符串输出。

---

## 结构体逐个解读

### BlobStateAction

**声明位置**: 行 62-67
**完整签名**:
```cpp
struct BlobStateAction {
    BlobState state_ = NONE;
    BlobLeaseFunction action_ = nullptr;
    BlobStateAction(BlobState state, BlobLeaseFunction action) : state_(state), action_(std::move(action)) {}
    BlobStateAction() = default;
};
```
**功能描述**: 状态转换的动作描述

**成员变量**:
| 变量名 | 类型 | 说明 |
|--------|------|------|
| `state_` | `BlobState` | 转换后的目标状态 |
| `action_` | `BlobLeaseFunction` | 状态转换时执行的回调函数 |

**构造函数**:
- `BlobStateAction()` - 默认构造，目标状态为 `NONE`，无动作
- `BlobStateAction(BlobState state, BlobLeaseFunction action)` - 参数化构造

**代码逻辑**:
1. 状态转换时，先更新 `state_`
2. 如果 `action_` 不为空，则执行回调函数处理租约

**使用示例**:
```cpp
// 创建一个添加租约的动作
BlobStateAction action(READABLE, LeaseAdd);
```

---

## 类逐个解读

### BlobStateMachine

**声明位置**: 行 114-117
**完整签名**:
```cpp
class BlobStateMachine : public MmcReferable {
public:
    static StateTransTable GetGlobalTransTable();
};
```
**功能描述**: Blob 状态机类，提供全局状态转换表

**继承关系**:
```
MmcReferable
    ▲
    │
BlobStateMachine
```

#### GetGlobalTransTable

**声明位置**: 行 116
**完整签名**:
```cpp
static StateTransTable GetGlobalTransTable();
```
**功能描述**: 获取全局状态转换表
**返回值**: 状态转换表 `StateTransTable`
**代码逻辑** (实现位于 `mmc_blob_state.cpp`):
1. 创建状态转换表
2. 定义转换规则数组 `g_metaStateTransItemTable`:
   - `ALLOCATED + MMC_ALLOCATED_OK → ALLOCATED` + `LeaseAdd`
   - `ALLOCATED + MMC_WRITE_OK → READABLE` + `LeaseRemove`
   - `ALLOCATED + MMC_WRITE_FAIL → REMOVING` + `LeaseRemove`
   - `ALLOCATED + MMC_REMOVE_START → REMOVING` + 无动作
   - `READABLE + MMC_READ_START → READABLE` + `LeaseAdd`
   - `READABLE + MMC_READ_FINISH → READABLE` + `LeaseRemove`
   - `READABLE + MMC_REMOVE_START → REMOVING` + `LeaseWait`
3. 将转换规则填充到表中

**状态转换表**:

| 当前状态 | 操作结果 | 下一状态 | 租约动作 |
|----------|----------|----------|----------|
| ALLOCATED | MMC_ALLOCATED_OK | ALLOCATED | LeaseAdd |
| ALLOCATED | MMC_WRITE_OK | READABLE | LeaseRemove |
| ALLOCATED | MMC_WRITE_FAIL | REMOVING | LeaseRemove |
| ALLOCATED | MMC_REMOVE_START | REMOVING | nullptr |
| READABLE | MMC_READ_START | READABLE | LeaseAdd |
| READABLE | MMC_READ_FINISH | READABLE | LeaseRemove |
| READABLE | MMC_REMOVE_START | REMOVING | LeaseWait |

**注意事项**:
- 状态转换表是全局唯一的
- 转换表在首次调用时构建

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                     BlobStateMachine                            │
├─────────────────────────────────────────────────────────────────┤
│ + GetGlobalTransTable(): StateTransTable                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 返回
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     StateTransTable                             │
├─────────────────────────────────────────────────────────────────┤
│  unordered_map<BlobState, unordered_map<Result, BlobStateAction>>│
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 包含
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     BlobStateAction                             │
├─────────────────────────────────────────────────────────────────┤
│ + state_: BlobState              (目标状态)                      │
│ + action_: BlobLeaseFunction     (租约回调)                      │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ 使用
                              │
┌─────────────────────────────────────────────────────────────────┐
│                  BlobLeaseFunction (类型别名)                   │
│  std::function<Result(MmcMetaLeaseManager&, uint32_t, uint32_t)>│
└─────────────────────────────────────────────────────────────────┘
```

**状态机图**:
```
                     ┌─────────────┐
                     │    NONE     │
                     └──────┬──────┘
                            │ allocate
                            ▼
                     ┌─────────────┐
                     │  ALLOCATED  │◄───────────────────┐
                     └──────┬──────┘                    │
                            │                           │
         ┌──────────────────┼──────────────────┐       │
         │ MMC_WRITE_OK      │ MMC_WRITE_FAIL   │       │
         ▼                   ▼                  │       │
    ┌─────────┐        ┌─────────┐             │       │
    │ READABLE│        │ REMOVING│◄──────┐     │       │
    └────┬────┘        └────┬────┘       │     │       │
         │                  │             │     │       │
         │ MMC_REMOVE_START │             │     │       │
         └──────────────────┴─────────────┘     │       │
                                               │       │
         ┌──────────────────────────────────────┴───────┘
         │   MMC_READ_START / MMC_READ_FINISH (保持 READABLE)
         ▼
    (循环自 READABLE)
```

---

## 使用示例

### 示例 1: 获取状态转换表

```cpp
#include "mmc_blob_state.h"

using namespace ock::mmc;

// 获取全局状态转换表
StateTransTable table = BlobStateMachine::GetGlobalTransTable();

// 查询特定状态转换
BlobState currentState = ALLOCATED;
BlobActionResult result = MMC_WRITE_OK;

auto &stateMap = table[currentState];
auto it = stateMap.find(result);

if (it != stateMap.end()) {
    BlobState nextState = it->second.state_;
    BlobLeaseFunction action = it->second.action_;
    // nextState = READABLE
    // action = LeaseRemove
}
```

### 示例 2: 状态流输出

```cpp
BlobState state = READABLE;
std::cout << "Current state: " << state << std::endl;
// 输出: Current state: READABLE

BlobActionResult result = MMC_WRITE_OK;
std::cout << "Operation result: " << result << std::endl;
// 输出: Operation result: MMC_WRITE_OK
```

### 示例 3: 创建自定义状态动作

```cpp
// 定义自定义租约函数
Result MyCustomLeaseAction(MmcMetaLeaseManager &leaseMgr, uint32_t rankId, uint32_t requestId) {
    // 自定义租约处理逻辑
    return leaseMgr.Add(rankId, requestId, 5000);
}

// 创建状态动作
BlobStateAction customAction(READABLE, MyCustomLeaseAction);
```
