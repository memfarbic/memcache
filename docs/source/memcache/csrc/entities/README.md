# entities 模块文档

## 模块概述

`entities` 模块包含 MemCache_Hybrid 项目中核心实体类的定义和实现，包括内存块（Blob）、内存对象元数据（Meta）、租约管理器（Lease Manager）和并发哈希表（Lookup Map）。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/entities/`

## 模块架构

该模块定义了 MemCache 系统的核心数据结构：

```
                    ┌─────────────────────────────────────────┐
                    │           entities 模块                  │
                    ├─────────────────────────────────────────┤
                    │                                         │
                    │  ┌─────────────┐  ┌─────────────────┐   │
                    │  │   Blob      │  │  BlobMeta       │   │
                    │  │  基础实体    │  │  元数据管理      │   │
                    │  └─────────────┘  └─────────────────┘   │
                    │                                         │
                    │  ┌─────────────┐  ┌─────────────────┐   │
                    │  │ LeaseMgr    │  │  LookupMap      │   │
                    │  │  租约管理    │  │  并发哈希表      │   │
                    │  └─────────────┘  └─────────────────┘   │
                    │                                         │
                    │  ┌───────────────────────────────────┐   │
                    │  │      StateMachine                 │   │
                    │  │      状态机                       │   │
                    │  └───────────────────────────────────┘   │
                    └─────────────────────────────────────────┘
```

---

## 文件列表

### 头文件 (.h)
- `mmc_blob_common.h` - Blob 基础数据结构
- `mmc_blob_state.h` - Blob 状态机定义
- `mmc_mem_blob.h` - Blob 核心类
- `mmc_mem_obj_meta.h` - 内存对象元数据类
- `mmc_meta_lease_manager.h` - 租约管理器类
- `mmc_lookup_map.h` - 分桶哈希表模板

### 源文件 (.cpp)
- `mmc_blob_state.cpp` - 状态机实现
- `mmc_mem_blob.cpp` - Blob 方法实现
- `mmc_mem_obj_meta.cpp` - Meta 方法实现
- `mmc_meta_lease_manager.cpp` - 租约管理器实现

---

## 模块内文件关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         entities 模块文件依赖关系                        │
└─────────────────────────────────────────────────────────────────────────┘

    mmc_blob_common.h (基础结构)
            │
            ├─────────────────────────────────────┐
            │                                     │
            ▼                                     ▼
    mmc_blob_state.h                      mmc_mem_blob.h
    (状态机定义)                            (Blob 类)
            │                                     │
            │                                     │
            ▼                                     ▼
    mmc_blob_state.cpp                    mmc_mem_blob.cpp
    (状态机实现)                           (Blob 实现)
                                                  │
                                                  │
                                                  ▼
                                    mmc_mem_obj_meta.h
                                    (元数据类)
                                          │
                                          ▼
                                    mmc_mem_obj_meta.cpp
                                    (元数据实现)

    mmc_meta_lease_manager.h             mmc_lookup_map.h
    (租约管理器)                          (哈希表)
            │                                     │
            ▼                                     │
    mmc_meta_lease_manager.cpp                  │
    (租约管理器实现)                              │
                                                  │
    ┌─────────────────────────────────────────────┘
    │
    │ 依赖
    ▼
    common/ 模块 (mmc_ref.h, mmc_types.h, mmc_logger.h 等)
```

---

## 核心数据类型关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       核心实体类关系图                                   │
└─────────────────────────────────────────────────────────────────────────┘

    MmcReferable (基类)
         ▲
         │ 继承
         │
    ┌────┴────────────────────────────────────────────────────────┐
    │                                                             │
    │                                                             │
┌───┴────┐    ┌─────────────┐    ┌──────────────────┐    ┌──────────┐
│ MmcMem │    │ MmcMemObj   │    │ MmcMetaLease     │    │MmcLookup │
│  Blob  │    │   Meta      │    │    Manager       │    │   Map    │
└───┬────┘    └──────┬──────┘    └──────────────────┘    └──────────┘
    │                │
    │ 包含 (多个)     │ 包含 (多个)
    │                │
    │                │
    ▼                ▼
┌─────────┐    ┌───────────────────────────────────┐
│MmcMem   │    │      MmcMemBlobDesc               │
│BlobDesc │    │      (Blob 描述符)                │
└─────────┘    └───────────────────────────────────┘

    ┌───────────────────────────────────┐
    │     BlobStateMachine              │
    │     (状态机，静态类)               │
    └───────────────────────────────────┘
```

---

## 详细文档链接

### mmc_blob_common.h

**功能**: 定义 Blob 的基础描述结构

**核心类型**:
- `MmcMemBlobDesc` - Blob 描述符，包含 size、gva、rank、mediaType
- `MmcMemBlobPtr` - Blob 智能指针别名

**详细文档**: [mmc_blob_common_h.md](mmc_blob_common_h.md)

---

### mmc_blob_state.h / mmc_blob_state.cpp

**功能**: 定义和管理 Blob 的状态机

**核心组件**:
- `BlobState` - Blob 状态枚举（ALLOCATED、READABLE、REMOVING、NONE）
- `BlobActionResult` - 操作结果枚举
- `BlobStateAction` - 状态转换动作
- `BlobStateMachine` - 状态机类
- 租约操作函数：LeaseAdd、LeaseRemove、LeaseWait、LeaseExtend

**详细文档**:
- [mmc_blob_state_h.md](mmc_blob_state_h.md)
- [mmc_blob_state_cpp.md](mmc_blob_state_cpp.md)

---

### mmc_mem_blob.h / mmc_mem_blob.cpp

**功能**: Blob 核心类，表示一块内存区域

**核心功能**:
- 状态更新（UpdateState）
- 租约管理（ExtendLease、IsLeaseExpired）
- 元数据备份（Backup、BackupRemove）
- 属性访问（Rank、Gva、Size、Type、State、Prot）
- 过滤器匹配（MatchFilter）

**详细文档**:
- [mmc_mem_blob_h.md](mmc_mem_blob_h.md)
- [mmc_mem_blob_cpp.md](mmc_mem_blob_cpp.md)

---

### mmc_mem_obj_meta.h / mmc_mem_obj_meta.cpp

**功能**: 内存对象元数据类，管理多个 Blob 副本

**核心功能**:
- Blob 管理（AddBlob、RemoveBlobs、FreeBlobs、GetBlobs）
- 批量状态更新（UpdateBlobsState）
- 介质层级移动（MoveTo、GetBlobType）
- 属性访问（Prot、Priority、NumBlobs、Size）
- 线程安全（Mutex）

**详细文档**:
- [mmc_mem_obj_meta_h.md](mmc_mem_obj_meta_h.md)
- [mmc_mem_obj_meta_cpp.md](mmc_mem_obj_meta_cpp.md)

---

### mmc_meta_lease_manager.h / mmc_meta_lease_manager.cpp

**功能**: 租约管理器，控制 Blob 的并发访问

**核心功能**:
- 租约管理（Add、Remove、Extend）
- 等待租约过期（Wait）
- 客户端 ID 编码/解码（GenerateClientId、RankId、RequestId）
- 租约数量查询（UseCount）

**详细文档**:
- [mmc_meta_lease_manager_h.md](mmc_meta_lease_manager_h.md)
- [mmc_meta_lease_manager_cpp.md](mmc_meta_lease_manager_cpp.md)

---

### mmc_lookup_map.h

**功能**: 分桶哈希表，支持并发访问

**核心功能**:
- 插入/查找/删除（Insert、Find、Erase）
- 迭代器遍历（begin、end、Iterator 类）
- 分桶锁机制，减少锁竞争

**详细文档**: [mmc_lookup_map_h.md](mmc_lookup_map_h.md)

---

## Blob 生命周期

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Blob 完整生命周期                                 │
└─────────────────────────────────────────────────────────────────────────┘

  1. 创建 Blob (ALLOCATED)
  ┌─────────────────┐
  │ MmcMemBlob     │
  │ rank=0, gva=... │
  │ state=ALLOCATED│
  └────────┬────────┘
           │
           │ MMC_ALLOCATED_OK
           │ LeaseAdd(rankId, requestId)
           ▼
  ┌─────────────────┐
  │ 等待数据写入     │
  │ 租约已添加       │
  └────────┬────────┘
           │
           ├─────────────────────┐
           │                     │
      MMC_WRITE_OK         MMC_WRITE_FAIL
      LeaseRemove           LeaseRemove
           │                     │
           ▼                     ▼
  ┌─────────────────┐   ┌─────────────────┐
  │ READABLE        │   │ REMOVING        │
  │ Backup()完成    │   │ 准备删除        │
  └────────┬────────┘   └─────────────────┘
           │
      MMC_READ_START
      LeaseAdd
           │
           ▼
  ┌─────────────────┐
  │ 正在读取         │
  │ 租约已添加       │
  └────────┬────────┘
           │
      MMC_READ_FINISH
      LeaseRemove
           │
           ▼
  ┌─────────────────┐
  │ READABLE        │
  │ 可继续读取       │
  └────────┬────────┘
           │
      MMC_REMOVE_START
      LeaseWait (等待所有租约释放)
           │
           ▼
  ┌─────────────────┐
  │ REMOVING        │
  │ 准备删除        │
  └────────┬────────┘
           │
           ▼
         释放内存
```

---

## 租约机制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        租约机制工作流程                                   │
└─────────────────────────────────────────────────────────────────────────┘

  场景：多个客户端同时读取 Blob

  时间线:
  ───────────────────────────────────────────────────────────────────────▶

  T1: 客户端 A 开始读取
      ┌─────────────────┐
      │ LeaseAdd(0,100) │
      └─────────────────┘
      useClient = {A}
      lease_ = now + 2000

  T2: 客户端 B 开始读取
      ┌─────────────────┐
      │ LeaseAdd(1,101) │
      └─────────────────┘
      useClient = {A, B}
      lease_ = now + 2000

  T3: 服务器收到删除请求
      ┌─────────────────┐
      │ LeaseWait()     │
      └────────┬────────┘
               │
               │ 循环检查 useClient
               ▼
      ┌─────────────────┐
      │ 等待租约释放...   │
      └─────────────────┘

  T4: 客户端 A 完成读取
      ┌─────────────────┐
      │ LeaseRemove(A)  │
      └─────────────────┘
      useClient = {B}

  T5: 客户端 B 完成读取
      ┌─────────────────┐
      │ LeaseRemove(B)  │
      └─────────────────┘
      useClient = {}

  T6: Wait() 返回，可以删除
```

---

## 快速参考

### MmcMemBlobDesc 结构

```cpp
struct MmcMemBlobDesc {
    uint64_t size_ = 0;               // Blob 数据大小
    uint64_t gva_ = UINT64_MAX;       // 全局虚拟地址
    uint32_t rank_ = UINT32_MAX;      // Blob 所在的 Rank ID
    uint16_t mediaType_ = UINT16_MAX; // 介质类型 (DRAM/HBM)
};
```

### BlobState 枚举

```cpp
enum BlobState : uint8_t {
    ALLOCATED,   // 已分配，正在写入
    READABLE,    // 可读
    REMOVING,    // 正在移除
    NONE,        // 无效状态
};
```

### BlobActionResult 枚举

```cpp
enum BlobActionResult : uint8_t {
    MMC_ALLOCATED_OK,  // 分配完成
    MMC_WRITE_OK,      // 写入成功
    MMC_WRITE_FAIL,    // 写入失败
    MMC_READ_START,    // 读取开始
    MMC_READ_FINISH,   // 读取完成
    MMC_REMOVE_START   // 移除开始
};
```

### Blob 状态转换表

| 当前状态 | 操作结果 | 下一状态 | 租约动作 |
|----------|----------|----------|----------|
| ALLOCATED | MMC_ALLOCATED_OK | ALLOCATED | LeaseAdd |
| ALLOCATED | MMC_WRITE_OK | READABLE | LeaseRemove |
| ALLOCATED | MMC_WRITE_FAIL | REMOVING | LeaseRemove |
| ALLOCATED | MMC_REMOVE_START | REMOVING | nullptr |
| READABLE | MMC_READ_START | READABLE | LeaseAdd |
| READABLE | MMC_READ_FINISH | READABLE | LeaseRemove |
| READABLE | MMC_REMOVE_START | REMOVING | LeaseWait |

### Blob 状态与可操作性

| 状态 | 可读 | 可写 | 可删除 |
|------|------|------|--------|
| NONE | × | × | × |
| ALLOCATED | × | √ | × |
| READABLE | √ | √ | √ |
| REMOVING | × | × | × |

---

## 数据关系图

```
┌─────────────────────────────────────────────┐
│          MmcMemObjMeta (内存对象)            │
│  ┌──────────────────────────────────────┐  │
│  │  std::list<MmcMemBlobPtr> blobs_     │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐│  │
│  │  │ Blob 1  │  │ Blob 2  │  │ Blob 3  ││  │
│  │  │ (Rank 0 │  │ (Rank 1 │  │ (Rank 2 ││  │
│  │  │  DRAM)  │  │  HBM)   │  │  DRAM)  ││  │
│  │  └────┬────┘  └────┬────┘  └────┬────┘│  │
│  │       │            │            │      │  │
│  │       └────────────┴────────────┘      │  │
│  │                   │                   │  │
│  │        MmcMetaLeaseManager            │  │
│  └──────────────────────────────────────┘  │
│                                             │
│  size_, prot_, priority_, numBlobs_        │
└─────────────────────────────────────────────┘
```

---

## 使用示例

### 示例 1: 创建和管理 Blob

```cpp
#include "mmc_mem_blob.h"
#include "mmc_mem_obj_meta.h"

using namespace ock::mmc;

// 创建 Blob
MmcMemBlobPtr blob = MmcMakeRef<MmcMemBlob>(
    0,           // rank
    0x100000,    // gva
    4096,        // size
    MEDIA_HBM,   // mediaType
    ALLOCATED    // initialState
);

// 创建元数据
MmcMemObjMetaPtr meta = MmcMakeRef<MmcMemObjMeta>();
meta->AddBlob(blob);

// 更新状态
blob->UpdateState("my_key", 0, 1001, MMC_ALLOCATED_OK);
blob->UpdateState("my_key", 0, 1001, MMC_WRITE_OK);

// 现在 blob 处于 READABLE 状态
```

### 示例 2: 使用租约管理

```cpp
MmcMemBlobPtr blob = ...;
MmcMetaLeaseManager leaseMgr;

// 添加租约
leaseMgr.Add(0, 100, 2000);

// 延长租约
leaseMgr.Extend(1000);

// 移除租约
leaseMgr.Remove(0, 100);

// 等待所有租约释放
leaseMgr.Wait();
```

### 示例 3: 使用查找表

```cpp
MmcLookupMap<std::string, MmcMemObjMetaPtr, 256> metaMap;

// 插入元数据
metaMap.Insert("key1", meta);

// 查找元数据
MmcMemObjMetaPtr foundMeta;
if (metaMap.Find("key1", foundMeta) == MMC_OK) {
    std::cout << "Found object, size: " << foundMeta->Size() << std::endl;
}

// 遍历所有元数据
for (const auto &pair : metaMap) {
    std::cout << "Key: " << pair.first << std::endl;
}
```

---

## 相关模块

- **common** - 公共工具和基础类型
- **meta_service** - 元数据服务，使用 entities 模块
- **client** - 客户端，使用 entities 模块进行数据传输
