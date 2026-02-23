# mmc_meta_backup_mgr_factory.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_backup_mgr_factory.h`
- **文件用途**: 定义元数据备份管理器工厂类，实现单例模式
- **依赖项**: `mmc_meta_backup_mgr.h`, `mmc_meta_backup_mgr_default.h`

---

## 类定义

### MMCMetaBackUpMgrFactory

备份管理器工厂类，使用单例模式创建和管理备份管理器实例。

---

### GetInstance

```cpp
static MmcRef<MMCMetaBackUpMgr> GetInstance(const std::string inputName = "")
{
    std::lock_guard<std::mutex> lock(instanceMutex_);
    std::string key = inputName;
    auto it = instances_.find(key);
    if (it == instances_.end()) {
        MmcRef<MMCMetaBackUpMgrDefault> instance = new (std::nothrow) MMCMetaBackUpMgrDefault();
        if (instance == nullptr) {
            MMC_LOG_ERROR("new MetaNetClient failed, probably out of memory");
            return nullptr;
        }
        instances_[key] = instance.Get();
        return instance.Get();
    }
    return it->second;
}
```

**声明位置**: 行 21-36

**功能描述**: 获取备份管理器单例实例

**参数**:
- `inputName`: 实例名称（默认为空字符串）

**返回值**: 备份管理器智能指针

**代码逻辑**:
1. 获取互斥锁
2. 使用 inputName 作为 key 查找已存在的实例
3. 如果不存在则创建新实例
4. 如果创建失败则返回 nullptr
5. 将新实例加入实例映射表
6. 返回实例指针

**单例模式**:
- 使用名称 key 区分不同的实例
- 同一名称返回同一实例
- 空字符串作为默认 key

---

## 私有成员

```cpp
private:
    static std::map<std::string, MmcRef<MMCMetaBackUpMgr>> instances_;
    static std::mutex instanceMutex_;
```

**说明**:
- `instances_`: 实例映射表，key 为实例名称
- `instanceMutex_`: 保护实例映射表的互斥锁

---

## 文件级别的关系图

```
mmc_meta_backup_mgr_factory.h (工厂类)
    |
    +-- 创建 MMCMetaBackUpMgrDefault 实例
    |
    +-- 单例模式
    |
    +-- 线程安全
    |
    +-- 支持命名实例
```

---

## 使用示例

```cpp
// 获取默认实例
auto backupMgr = MMCMetaBackUpMgrFactory::GetInstance();

// 获取命名实例
auto namedBackupMgr = MMCMetaBackUpMgrFactory::GetInstance("MyBackup");

// 启动备份管理器
MMCMetaBackUpConfDefaultPtr conf = MmcMakeRef<MMCMetaBackUpConfDefault>(netServer);
backupMgr->Start(conf);
```
