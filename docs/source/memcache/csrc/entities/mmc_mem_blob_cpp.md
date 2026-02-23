# mmc_mem_blob.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/entities/mmc_mem_blob.cpp`
- **文件用途**: 实现 `MmcMemBlob` 类的状态更新、备份管理等功能
- **依赖项**:
  - `mmc_mem_blob.h` - Blob 类定义
  - `mmc_meta_backup_mgr_factory.h` - 备份管理器工厂

---

## 静态成员初始化

### stateTransTable_

**声明位置**: 行 16
**完整签名**:
```cpp
const StateTransTable MmcMemBlob::stateTransTable_ = BlobStateMachine::GetGlobalTransTable();
```
**功能描述**: 初始化全局状态转换表
**代码逻辑**:
1. 调用 `BlobStateMachine::GetGlobalTransTable()` 获取转换表
2. 将返回的转换表赋值给静态成员变量
**注意事项**:
- 在程序启动时初始化
- 所有 `MmcMemBlob` 实例共享同一个转换表

---

## 类方法实现

### MmcMemBlob::UpdateState (带租约操作版本)

**声明位置**: 行 18-48
**完整签名**:
```cpp
Result MmcMemBlob::UpdateState(const std::string &key, uint32_t rankId, uint32_t operateId, BlobActionResult ret)
```
**功能描述**: 更新 Blob 状态并执行相应的租约操作
**参数**:
- `key` - 内存对象的键
- `rankId` - 操作发起的 Rank ID
- `operateId` - 操作 ID
- `ret` - 操作结果码
**返回值**: `Result` - 操作结果
**可能返回的错误**:
- `MMC_UNMATCHED_STATE` - 当前状态不在转换表中
- `MMC_UNMATCHED_RET` - 操作结果不匹配当前状态

**代码逻辑**:
1. **查找当前状态**:
   ```cpp
   auto curStateIter = stateTransTable_.find(state_);
   if (curStateIter == stateTransTable_.end()) {
       return MMC_UNMATCHED_STATE;
   }
   ```

2. **查找操作结果对应的转换动作**:
   ```cpp
   const auto retIter = curStateIter->second.find(ret);
   if (retIter == curStateIter->second.end()) {
       return MMC_UNMATCHED_RET;
   }
   ```

3. **处理写入成功的备份操作**:
   ```cpp
   if (state_ == ALLOCATED && ret == MMC_WRITE_OK) {
       MMC_RETURN_ERROR(Backup(key), "memBlob remove use client error");
   }
   ```
   当从 `ALLOCATED` 转换到 `READABLE` 时（写入成功），执行元数据备份

4. **更新状态**:
   ```cpp
   state_ = retIter->second.state_;
   ```

5. **执行租约操作**:
   ```cpp
   if (retIter->second.action_) {
       auto res = retIter->second.action_(metaLeaseManager_, rankId, operateId);
       if (res != MMC_OK) {
           return res;
       }
   }
   ```

**流程图**:
```
         ┌─────────────────┐
         │  UpdateState()  │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 查找当前状态    │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐     ┌──────────────────┐
         │ 找到?            │────▶│ 返回 UNMATCHED    │
         └────────┬────────┘     └──────────────────┘
                  │ 是
                  ▼
         ┌─────────────────┐     ┌──────────────────┐
         │ 查找操作结果    │────▶│ 返回 UNMATCHED_RET│
         └────────┬────────┘     └──────────────────┘
                  │ 找到
                  ▼
         ┌─────────────────┐
         │ 写入成功?        │────▶│ Backup(key)      │
         │ ALLOCATED→      │     │                  │
         │ READABLE        │     │                  │
         └────────┬────────┘     └──────────────────┘
                  │ 否
                  ▼
         ┌─────────────────┐
         │ 更新状态        │
         │ state_ = next   │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 有租约操作?      │────▶│ 执行租约操作      │
         │ action_ != null │     │                  │
         └────────┬────────┘     └──────────────────┘
                  │ 否
                  ▼
         ┌─────────────────┐
         │ 返回 MMC_OK     │
         └─────────────────┘
```

**注意事项**:
- 状态转换遵循状态机定义，不可任意转换
- 租约操作在状态更新后执行
- 写入成功时自动触发备份操作

---

### MmcMemBlob::UpdateState (不带租约操作版本)

**声明位置**: 行 50-67
**完整签名**:
```cpp
Result MmcMemBlob::UpdateState(const BlobActionResult ret)
```
**功能描述**: 更新 Blob 状态（不执行租约操作）
**参数**:
- `ret` - 操作结果码
**返回值**: `Result` - 操作结果
**可能返回的错误**:
- `MMC_UNMATCHED_STATE` - 当前状态不在转换表中
- `MMC_UNMATCHED_RET` - 操作结果不匹配当前状态

**代码逻辑**:
1. 查找当前状态
2. 查找操作结果对应的转换动作
3. 更新状态，不执行租约操作
4. 返回 `MMC_OK`

**与前者的区别**:
- 不执行备份操作
- 不执行租约操作
- 仅更新状态

**使用场景**:
- 不需要租约管理的场景
- 内部状态同步

---

### MmcMemBlob::Backup

**声明位置**: 行 69-79
**完整签名**:
```cpp
Result MmcMemBlob::Backup(const std::string &key)
```
**功能描述**: 将 Blob 元数据备份到备份管理器
**参数**:
- `key` - 内存对象的键
**返回值**: `Result` - 操作结果
**可能返回的错误**:
- `MMC_META_BACKUP_ERROR` - 备份管理器获取失败

**代码逻辑**:
1. 获取备份管理器实例:
   ```cpp
   MMCMetaBackUpMgrPtr mmcBackupPtr = MMCMetaBackUpMgrFactory::GetInstance("DefaultMetaBackup");
   if (mmcBackupPtr == nullptr) {
       return MMC_META_BACKUP_ERROR;
   }
   ```

2. 获取 Blob 描述符:
   ```cpp
   MmcMemBlobDesc desc = GetDesc();
   ```

3. 添加备份:
   ```cpp
   return mmcBackupPtr->Add(key, desc);
   ```

**流程图**:
```
         ┌─────────────────┐
         │    Backup()     │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐     ┌──────────────────┐
         │ 获取备份管理器   │────▶│ 返回 BACKUP_     │
         │                 │     │     ERROR        │
         └────────┬────────┘     └──────────────────┘
                  │ 成功
                  ▼
         ┌─────────────────┐
         │ 获取 Blob 描述符│
         │ GetDesc()       │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 添加到备份       │
         │ Add(key, desc)  │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ 返回结果         │
         └─────────────────┘
```

**注意事项**:
- 使用工厂模式获取备份管理器
- "DefaultMetaBackup" 是默认备份管理器名称
- 备份成功后可在元服务重启时恢复

---

### MmcMemBlob::BackupRemove

**声明位置**: 行 81-91
**完整签名**:
```cpp
Result MmcMemBlob::BackupRemove(const std::string &key)
```
**功能描述**: 从备份管理器中移除 Blob 元数据
**参数**:
- `key` - 内存对象的键
**返回值**: `Result` - 操作结果
**可能返回的错误**:
- `MMC_META_BACKUP_ERROR` - 备份管理器获取失败

**代码逻辑**:
1. 获取备份管理器实例
2. 获取 Blob 描述符
3. 移除备份:
   ```cpp
   return mmcBackupPtr->Remove(key, desc);
   ```

**注意事项**:
- 通常在 Blob 被释放时调用
- 确保备份管理器中不会残留无效数据

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MmcMemBlob                                  │
├─────────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 静态成员                                                         │ │
│ │ + stateTransTable_: const StateTransTable                        │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ 成员方法实现                                                     │ │
│ │ + UpdateState(key, rankId, operateId, ret): Result               │ │
│ │ + UpdateState(ret): Result                                       │ │
│ │ + Backup(key): Result                                            │ │
│ │ + BackupRemove(key): Result                                      │ │
│ └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                    │                    │
                    │ 使用               │ 使用
                    ▼                    ▼
┌─────────────────────────────┐  ┌────────────────────────────────────┐
│   BlobStateMachine          │  │ MMCMetaBackUpMgrFactory            │
│   GetGlobalTransTable()     │  │ GetInstance("DefaultMetaBackup")   │
└─────────────────────────────┘  └────────────────────────────────────┘
                                         │
                                         │ 返回
                                         ▼
                              ┌────────────────────────────────────┐
                              │      MMCMetaBackUpMgr              │
                              │  Add(key, desc) / Remove()         │
                              └────────────────────────────────────┘
```

---

## 状态转换流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Blob 状态转换完整流程                            │
└─────────────────────────────────────────────────────────────────────────┘

  1. 分配阶段
  ┌─────────┐    MMC_ALLOCATED_OK    ┌─────────────┐
  │ ALLOCATE│ ──────────────────────▶│  ALLOCATED  │
  └─────────┘                        └──────┬──────┘
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    │                       │                       │
                    │ MMC_WRITE_OK          │ MMC_WRITE_FAIL        │ MMC_REMOVE_START
                    ▼                       ▼                       ▼
            ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
            │   READABLE    │       │   REMOVING    │       │   REMOVING    │
            │  + Backup()   │       │               │       │               │
            └───────┬───────┘       └───────────────┘       └───────────────┘
                    │
    ┌───────────────┼───────────────┐
    │ MMC_READ_START│               │
    ▼               │               │ MMC_READ_FINISH
┌─────────┐         │               ▼
│ READABLE│◀────────┘       ┌───────────────┐
│+LeaseAdd│                 │   READABLE    │
└─────────┘                 │  +LeaseRemove │
                            └───────────────┘
                                    │
                                    │ MMC_REMOVE_START
                                    ▼
                            ┌───────────────┐
                            │   REMOVING    │
                            │   +LeaseWait  │
                            └───────────────┘
```

---

## 使用示例

### 示例 1: 完整的 Blob 生命周期

```cpp
#include "mmc_mem_blob.h"

using namespace ock::mmc;

// 1. 创建 Blob
MmcMemBlobPtr blob = MmcMakeRef<MmcMemBlob>(
    0,           // rank
    0x100000,    // gva
    4096,        // size
    MEDIA_HBM,   // mediaType
    ALLOCATED    // initialState
);

std::string key = "my_object";

// 2. 分配完成，添加租约
blob->UpdateState(key, 0, 1001, MMC_ALLOCATED_OK);
// 状态: ALLOCATED, 租约已添加

// 3. 写入成功，触发备份并移除租约
blob->UpdateState(key, 0, 1001, MMC_WRITE_OK);
// 状态: READABLE, 备份已完成

// 4. 开始读取，添加租约
blob->UpdateState(key, 1, 1002, MMC_READ_START);
// 状态: READABLE, 租约已添加

// 5. 读取完成，移除租约
blob->UpdateState(key, 1, 1002, MMC_READ_FINISH);
// 状态: READABLE, 租约已移除

// 6. 准备删除
blob->UpdateState(key, 0, 0, MMC_REMOVE_START);
// 状态: REMOVING, 等待租约释放
```

### 示例 2: 备份操作

```cpp
// 写入成功时自动备份
Result ret = blob->UpdateState(key, 0, 1001, MMC_WRITE_OK);
if (ret == MMC_OK) {
    MMC_LOG_INFO("Blob backed up successfully");
}

// 手动备份
ret = blob->Backup(key);
if (ret != MMC_OK) {
    MMC_LOG_ERROR("Backup failed: " << ret);
}

// 移除备份
ret = blob->BackupRemove(key);
if (ret != MMC_OK) {
    MMC_LOG_ERROR("Backup remove failed: " << ret);
}
```

### 示例 3: 错误处理

```cpp
Result ret = blob->UpdateState(key, 0, 1001, MMC_WRITE_OK);
if (ret == MMC_UNMATCHED_STATE) {
    MMC_LOG_ERROR("Current state not in transition table");
} else if (ret == MMC_UNMATCHED_RET) {
    MMC_LOG_ERROR("Operation result not valid for current state");
} else if (ret != MMC_OK) {
    MMC_LOG_ERROR("State update failed: " << ret);
}
```

### 示例 4: 不带租约的状态更新

```cpp
// 仅更新状态，不执行租约操作
Result ret = blob->UpdateState(MMC_READ_START);
if (ret == MMC_OK) {
    BlobState state = blob->State();
    MMC_LOG_DEBUG("State updated to: " << state);
}
```
