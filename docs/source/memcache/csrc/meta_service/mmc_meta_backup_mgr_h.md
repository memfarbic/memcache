# mmc_meta_backup_mgr.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_backup_mgr.h`
- **文件用途**: 定义元数据备份管理器接口类，用于元数据的高可用备份
- **依赖项**: `mmc_ref.h`, `mmc_types.h`, `mmc_blob_common.h`

---

## 数据结构

### MMCMetaBackUpConf

```cpp
struct MMCMetaBackUpConf : public MmcReferable {};
```

**声明位置**: 行 24

**说明**: 元数据备份配置基类（空结构，用于类型擦除）

---

### MMCMetaBackUpConfPtr

```cpp
using MMCMetaBackUpConfPtr = MmcRef<MMCMetaBackUpConf>;
```

**声明位置**: 行 25

**说明**: 备份配置智能指针类型

---

## 类定义

### MMCMetaBackUpMgr

元数据备份管理器抽象基类，定义备份操作的接口。

#### Start

```cpp
virtual Result Start(MMCMetaBackUpConfPtr &confPtr) = 0;
```

**声明位置**: 行 29

**功能描述**: 启动备份管理器

**参数**:
- `confPtr`: 备份配置指针

**返回值**: 成功返回 MMC_OK

---

#### Stop

```cpp
virtual void Stop() = 0;
```

**声明位置**: 行 31

**功能描述**: 停止备份管理器

---

#### Add

```cpp
virtual Result Add(const std::string &key, MmcMemBlobDesc &blobDesc) = 0;
```

**声明位置**: 行 33

**功能描述**: 添加一个元数据到备份队列

**参数**:
- `key`: 元数据的 key
- `blobDesc`: Blob 描述符

**返回值**: 成功返回 MMC_OK

---

#### Remove

```cpp
virtual Result Remove(const std::string &key, MmcMemBlobDesc &blobDesc) = 0;
```

**声明位置**: 行 35

**功能描述**: 从备份中移除一个元数据

**参数**:
- `key`: 元数据的 key
- `blobDesc`: Blob 描述符

**返回值**: 成功返回 MMC_OK

---

#### Load

```cpp
virtual Result Load(std::map<std::string, MmcMemBlobDesc> &blobMap) = 0;
```

**声明位置**: 行 37

**功能描述**: 加载备份的元数据（用于恢复）

**参数**:
- `blobMap`: 输出加载的 Blob 映射表

**返回值**: 成功返回 MMC_OK

---

### MMCMetaBackUpMgrPtr

```cpp
using MMCMetaBackUpMgrPtr = MmcRef<MMCMetaBackUpMgr>;
```

**声明位置**: 行 39

**说明**: 备份管理器智能指针类型

---

## 设计模式

**接口模式**: MMCMetaBackUpMgr 是一个抽象接口类，定义了备份管理器的通用操作。

**实现类**:
- `MMCMetaBackUpMgrDefault`: 默认实现类，使用网络进行备份

---

## 文件级别的关系图

```
mmc_meta_backup_mgr.h (备份管理器接口)
    |
    +-- 被 MMCMetaBackUpMgrDefault 实现
    |
    +-- 被 MMCMetaBackUpMgrFactory 使用
    |
    +-- 被 MmcMetaService 使用
    |
    +-- 定义操作:
    |   +-- Start (启动)
    |   +-- Stop (停止)
    |   +-- Add (添加备份)
    |   +-- Remove (移除备份)
    |   +-- Load (加载备份)
```

---

## 使用场景

1. **高可用场景**: 元服务主备切换时，从备机恢复元数据
2. **增量备份**: 元数据变更时自动同步到备份节点
3. **故障恢复**: 元服务重启后从备份恢复状态
