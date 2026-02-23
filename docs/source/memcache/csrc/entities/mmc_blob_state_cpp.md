# mmc_blob_state.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/entities/mmc_blob_state.cpp`
- **文件用途**: 实现 Blob 状态机的全局状态转换表和租约操作函数
- **依赖项**:
  - `mmc_blob_state.h` - Blob 状态定义
  - `mmc_mem_blob.h` - Blob 类定义

---

## 文件包含的头文件

```cpp
#include "mmc_blob_state.h"
#include "mmc_mem_blob.h"
```

---

## 辅助结构体

### StateTransitionItem

**声明位置**: 行 20-25
**完整签名**:
```cpp
struct StateTransitionItem {
    BlobState curState;
    Result retCode;
    BlobState nextState;
    BlobLeaseFunction function;
};
```
**功能描述**: 状态转换表的元组结构，用于构建转换表

**成员变量**:
| 变量名 | 类型 | 说明 |
|--------|------|------|
| `curState` | `BlobState` | 当前状态 |
| `retCode` | `Result` | 操作结果码 |
| `nextState` | `BlobState` | 下一状态 |
| `function` | `BlobLeaseFunction` | 状态转换时执行的租约函数 |

---

## 租约操作函数

### LeaseAdd

**声明位置**: 行 27-30
**完整签名**:
```cpp
Result LeaseAdd(MmcMetaLeaseManager &leaseMgr, uint32_t rankId, uint32_t requestId)
{
    return leaseMgr.Add(rankId, requestId, MMC_DATA_TTL_MS);
}
```
**功能描述**: 向租约管理器添加租约记录
**参数**:
- `leaseMgr` - 租约管理器引用
- `rankId` - Rank ID
- `requestId` - 请求 ID
**返回值**: 操作结果 `Result`
**代码逻辑**:
1. 调用 `leaseMgr.Add()` 添加租约
2. 租约 TTL 使用 `MMC_DATA_TTL_MS` (2000ms)
**使用场景**:
- `ALLOCATED + MMC_ALLOCATED_OK` - 准备写入时添加租约
- `READABLE + MMC_READ_START` - 准备读取时添加租约

### LeaseRemove

**声明位置**: 行 32-35
**完整签名**:
```cpp
Result LeaseRemove(MmcMetaLeaseManager &leaseMgr, uint32_t rankId, uint32_t requestId)
{
    return leaseMgr.Remove(rankId, requestId);
}
```
**功能描述**: 从租约管理器移除租约记录
**参数**:
- `leaseMgr` - 租约管理器引用
- `rankId` - Rank ID
- `requestId` - 请求 ID
**返回值**: 操作结果 `Result`
**代码逻辑**:
1. 调用 `leaseMgr.Remove()` 移除指定客户端的租约
**使用场景**:
- `ALLOCATED + MMC_WRITE_OK` - 写入成功后移除租约
- `ALLOCATED + MMC_WRITE_FAIL` - 写入失败后移除租约
- `READABLE + MMC_READ_FINISH` - 读取完成后移除租约

### LeaseWait

**声明位置**: 行 37-44
**完整签名**:
```cpp
Result LeaseWait(MmcMetaLeaseManager &leaseMgr, uint32_t rankId, uint32_t requestId)
{
    if (leaseMgr.UseCount() == 0) {
        return MMC_OK;
    }
    leaseMgr.Wait();
    return MMC_OK;
}
```
**功能描述**: 等待所有租约过期或释放
**参数**:
- `leaseMgr` - 租约管理器引用
- `rankId` - Rank ID (未使用)
- `requestId` - 请求 ID (未使用)
**返回值**: 始终返回 `MMC_OK`
**代码逻辑**:
1. 检查当前租约使用计数
2. 如果没有活跃租约 (`UseCount() == 0`)，直接返回成功
3. 否则调用 `leaseMgr.Wait()` 等待租约过期
**使用场景**:
- `READABLE + MMC_REMOVE_START` - 移除 Blob 前等待所有读租约释放

### LeaseExtend

**声明位置**: 行 46-49
**完整签名**:
```cpp
Result LeaseExtend(MmcMetaLeaseManager &leaseMgr, uint32_t rankId, uint32_t requestId)
{
    return leaseMgr.Extend(MMC_DATA_TTL_MS);
}
```
**功能描述**: 延长租约有效期
**参数**:
- `leaseMgr` - 租约管理器引用
- `rankId` - Rank ID (未使用)
- `requestId` - 请求 ID (未使用)
**返回值**: 操作结果 `Result`
**代码逻辑**:
1. 调用 `leaseMgr.Extend()` 延长租约
2. 延长时长为 `MMC_DATA_TTL_MS` (2000ms)
**使用场景**: 当前状态转换表中未使用，预留给未来功能

---

## 类方法实现

### BlobStateMachine::GetGlobalTransTable

**声明位置**: 行 51-73
**完整签名**:
```cpp
StateTransTable BlobStateMachine::GetGlobalTransTable()
```
**功能描述**: 构建并返回全局状态转换表
**返回值**: 状态转换表 `StateTransTable`

**代码逻辑**:
1. 创建空的 `StateTransTable` 表
2. 定义状态转换规则数组 `g_metaStateTransItemTable`:
   ```cpp
   StateTransitionItem g_metaStateTransItemTable[]{
       {ALLOCATED, MMC_ALLOCATED_OK, ALLOCATED, LeaseAdd},
       {ALLOCATED, MMC_WRITE_OK, READABLE, LeaseRemove},
       {ALLOCATED, MMC_WRITE_FAIL, REMOVING, LeaseRemove},
       {ALLOCATED, MMC_REMOVE_START, REMOVING, nullptr},
       {READABLE, MMC_READ_START, READABLE, LeaseAdd},
       {READABLE, MMC_READ_FINISH, READABLE, LeaseRemove},
       {READABLE, MMC_REMOVE_START, REMOVING, LeaseWait},
   };
   ```
3. 遍历数组，填充转换表:
   - 创建 `BlobStateAction` 对象，包含下一状态和租约函数
   - 填充到 `table[curState][retCode]`
4. 返回构建好的转换表

**状态转换规则详解**:

| 当前状态 | 结果码 | 下一状态 | 租约操作 | 说明 |
|----------|--------|----------|----------|------|
| `ALLOCATED` | `MMC_ALLOCATED_OK` | `ALLOCATED` | `LeaseAdd` | 准备写入，添加租约保护 |
| `ALLOCATED` | `MMC_WRITE_OK` | `READABLE` | `LeaseRemove` | 写入成功，变为可读，移除租约 |
| `ALLOCATED` | `MMC_WRITE_FAIL` | `REMOVING` | `LeaseRemove` | 写入失败，准备移除 |
| `ALLOCATED` | `MMC_REMOVE_START` | `REMOVING` | `nullptr` | 直接移除 |
| `READABLE` | `MMC_READ_START` | `READABLE` | `LeaseAdd` | 准备读取，添加租约 |
| `READABLE` | `MMC_READ_FINISH` | `READABLE` | `LeaseRemove` | 读取完成，移除租约 |
| `READABLE` | `MMC_REMOVE_START` | `REMOVING` | `LeaseWait` | 移除前等待租约释放 |

**注意事项**:
- 转换表在程序中只构建一次（首次调用时）
- 转换表是静态的，运行时不可修改
- `nullptr` 表示状态转换时不执行租约操作

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────┐
│              StateTransitionItem (构建辅助)                     │
├─────────────────────────────────────────────────────────────────┤
│ + curState: BlobState                                          │
│ + retCode: Result                                              │
│ + nextState: BlobState                                         │
│ + function: BlobLeaseFunction                                  │
└─────────────────────────────────────────────────────────────────┘
         │
         │ 填充到
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  StateTransTable                                │
│    unordered_map<BlobState, unordered_map<Result, Action>>      │
└─────────────────────────────────────────────────────────────────┘
         │
         │ 包含
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   BlobStateAction                               │
│  + state_: BlobState            + action_: BlobLeaseFunction    │
└─────────────────────────────────────────────────────────────────┘
         ▲                                    ▲
         │                                    │
         │ 调用                               │
         │                                    │
┌──────────────────────┐      ┌──────────────────────────────────┐
│   LeaseAdd           │      │   LeaseRemove                   │
│   LeaseWait          │      │   LeaseExtend                   │
└──────────────────────┘      └──────────────────────────────────┘
```

---

## 租约操作流程图

```
                    ┌─────────────────────┐
                    │   ALLOCATED         │
                    └──────────┬──────────┘
                               │ MMC_ALLOCATED_OK
                               ▼
                    ┌─────────────────────┐
                    │ LeaseAdd(rankId,    │
                    │   requestId, 2000)  │
                    └─────────────────────┘

                    ┌─────────────────────┐
                    │   ALLOCATED         │
                    └──────────┬──────────┘
                               │ MMC_WRITE_OK
                               ▼
                    ┌─────────────────────┐
                    │ LeaseRemove(rankId, │
                    │   requestId)        │
                    └─────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   READABLE          │
                    └─────────────────────┘

                    ┌─────────────────────┐
                    │   READABLE          │
                    └──────────┬──────────┘
                               │ MMC_REMOVE_START
                               ▼
                    ┌─────────────────────┐
                    │ if (UseCount() == 0)│
                    │   return OK         │
                    │ else                │
                    │   Wait()            │
                    └─────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   REMOVING          │
                    └─────────────────────┘
```

---

## 使用示例

### 示例 1: 获取并使用状态转换表

```cpp
#include "mmc_blob_state.h"

using namespace ock::mmc;

// 获取全局转换表
StateTransTable table = BlobStateMachine::GetGlobalTransTable();

// 查询 ALLOCATED + MMC_WRITE_OK 的转换
BlobState currentState = ALLOCATED;
BlobActionResult result = MMC_WRITE_OK;

auto &stateMap = table[currentState];
auto it = stateMap.find(result);

if (it != stateMap.end()) {
    BlobState nextState = it->second.state_;
    auto leaseFunc = it->second.action_;

    // nextState = READABLE
    // leaseFunc = LeaseRemove

    // 执行租约操作
    MmcMetaLeaseManager leaseMgr;
    if (leaseFunc) {
        Result res = leaseFunc(leaseMgr, 0, 123);
    }
}
```

### 示例 2: 自定义租约函数

```cpp
// 可以自定义租约函数并在状态转换中使用
Result MyLeaseFunc(MmcMetaLeaseManager &leaseMgr, uint32_t rankId, uint32_t requestId) {
    // 自定义租约逻辑
    MMC_LOG_INFO("Custom lease operation for rank " << rankId);
    return leaseMgr.Add(rankId, requestId, 5000);
}

// 使用自定义函数
BlobStateAction action(READABLE, MyLeaseFunc);
```

### 示例 3: 状态转换模拟

```cpp
// 模拟 Blob 写入流程
MmcMetaLeaseManager leaseMgr;
uint32_t rankId = 0;
uint32_t requestId = 100;

// 1. 分配完成
auto table = BlobStateMachine::GetGlobalTransTable();
BlobState state = ALLOCATED;

auto &allocMap = table[state];
auto it = allocMap.find(MMC_ALLOCATED_OK);
if (it != allocMap.end() && it->second.action_) {
    it->second.action_(leaseMgr, rankId, requestId); // LeaseAdd
    state = it->second.state_;
}

// 2. 写入成功
auto &writeMap = table[state];
it = writeMap.find(MMC_WRITE_OK);
if (it != writeMap.end() && it->second.action_) {
    it->second.action_(leaseMgr, rankId, requestId); // LeaseRemove
    state = it->second.state_;
}

// 现在 state = READABLE
```
