# mmc_meta_backup_mgr_default.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_backup_mgr_default.h`
- **文件用途**: 定义元数据备份管理器的默认实现类，使用异步线程进行备份
- **依赖项**: `mmc_meta_backup_mgr.h`, `mmc_meta_net_server.h`

---

## 枚举定义

### BackUpOperate

```cpp
enum BackUpOperate { META_BACKUP_ADD = 0, META_BACKUP_REMOVE = 1 };
```

**声明位置**: 行 30

**说明**: 备份操作类型
- `META_BACKUP_ADD`: 添加元数据
- `META_BACKUP_REMOVE`: 移除元数据

---

## 数据结构

### MetaBackUpOperate

```cpp
struct MetaBackUpOperate {
    uint32_t op_;
    std::string key_;
    MmcMemBlobDesc desc_;

    MetaBackUpOperate() {}
    MetaBackUpOperate(uint32_t op, const std::string &key, MmcMemBlobDesc &desc) : op_(op), key_(key), desc_(desc) {}
};
```

**声明位置**: 行 32-38

**说明**: 备份操作描述符
- `op_`: 操作类型
- `key_`: 元数据的 key
- `desc_`: Blob 描述符

---

### MMCMetaBackUpConfDefault

```cpp
struct MMCMetaBackUpConfDefault : public MMCMetaBackUpConf {
    MetaNetServerPtr serverPtr_;

    explicit MMCMetaBackUpConfDefault(MetaNetServerPtr serverPtr) : serverPtr_(serverPtr) {}
};
```

**声明位置**: 行 40-44

**说明**: 默认备份配置，包含网络服务器指针

---

### MMCMetaBackUpConfDefaultPtr

```cpp
using MMCMetaBackUpConfDefaultPtr = MmcRef<MMCMetaBackUpConfDefault>;
```

**声明位置**: 行 45

**说明**: 默认备份配置智能指针类型

---

## 类定义

### MMCMetaBackUpMgrDefault

元数据备份管理器的默认实现，使用异步线程池进行备份。

---

#### 构造函数

```cpp
explicit MMCMetaBackUpMgrDefault() {}
```

**声明位置**: 行 49

**功能描述**: 默认构造函数

---

#### 析构函数

```cpp
~MMCMetaBackUpMgrDefault() override
{
    Stop();
}
```

**声明位置**: 行 51-54

**功能描述**: 析构函数，自动停止备份管理器

---

### Start

```cpp
Result Start(MMCMetaBackUpConfPtr &confPtr) override
```

**声明位置**: 行 56-72

**功能描述**: 启动备份管理器

**参数**:
- `confPtr`: 备份配置指针

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 获取互斥锁
2. 如果已启动则直接返回
3. 将配置转换为默认配置类型
4. 保存网络服务器指针
5. 设置启动标志
6. 创建备份线程并启动

---

### Stop

```cpp
void Stop() override
```

**声明位置**: 行 74-89

**功能描述**: 停止备份管理器

**代码逻辑**:
1. 获取互斥锁
2. 如果未启动则直接返回
3. 通知备份线程停止
4. 等待备份线程结束
5. 清理资源

---

### BackupThreadFunc

```cpp
void BackupThreadFunc();
```

**声明位置**: 行 90

**功能描述**: 备份线程的主函数

**实现** (`mmc_meta_backup_mgr_default.cpp`):
```cpp
void MMCMetaBackUpMgrDefault::BackupThreadFunc()
{
    while (true) {
        {
            std::unique_lock<std::mutex> lock(backupThreadLock_);
            backupThreadCv_.wait(lock, [this] { return backupList_.size() || !started_; });
        }
        if (!started_) {
            MMC_LOG_INFO("backup thread destroy, thread id " << pthread_self());
            break;
        }
        MMC_LOG_DEBUG("MMCMetaBackU thread will backup count " << backupList_.size() << " thread id "
                                                               << pthread_self());

        SendBackup2Local();
    }
}
```

**代码逻辑**:
1. 等待备份队列有数据或停止信号
2. 如果收到停止信号则退出循环
3. 调用 SendBackup2Local 发送备份数据

---

### Add

```cpp
Result Add(const std::string &key, MmcMemBlobDesc &blobDesc) override
```

**声明位置**: 行 92-103

**功能描述**: 添加一个元数据到备份队列

**参数**:
- `key`: 元数据的 key
- `blobDesc`: Blob 描述符

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 将操作添加到备份队列
2. 通知备份线程

---

### Remove

```cpp
Result Remove(const std::string &key, MmcMemBlobDesc &blobDesc) override
```

**声明位置**: 行 105-116

**功能描述**: 从备份中移除一个元数据

**参数**:
- `key`: 元数据的 key
- `blobDesc`: Blob 描述符

**返回值**: 成功返回 MMC_OK

**代码逻辑**:
1. 将移除操作添加到备份队列
2. 通知备份线程

---

### Load

```cpp
Result Load(std::map<std::string, MmcMemBlobDesc> &blobMap) override
```

**声明位置**: 行 118-121

**功能描述**: 加载备份的元数据

**返回值**: 固定返回 MMC_OK（当前实现为空）

---

### 私有方法

#### PopMetas2Backup

```cpp
uint32_t PopMetas2Backup(std::vector<uint32_t> &ops, std::vector<std::string> &keys,
                         std::vector<MmcMemBlobDesc> &blobs);
```

**声明位置**: 行 124-125

**功能描述**: 从备份队列中弹出元数据（最多 1024 个）

**参数**:
- `ops`: 输出操作类型列表
- `keys`: 输出 key 列表
- `blobs`: 输出 Blob 描述符列表

**返回值**: 目标 Rank ID

---

#### SendBackup2Local

```cpp
void SendBackup2Local();
```

**声明位置**: 行 126

**功能描述**: 发送备份到本地节点

---

## 成员变量

```cpp
private:
    MetaNetServerPtr metaNetServer_;
    std::mutex mutex_;
    bool started_ = false;
    std::thread backupThread_;
    std::mutex backupThreadLock_;
    std::condition_variable backupThreadCv_;
    std::mutex backupListLock_;
    std::list<MetaBackUpOperate> backupList_;
```

**说明**:
- `metaNetServer_`: 网络服务器指针，用于发送备份
- `mutex_`: 保护 started_ 标志
- `started_`: 启动标志
- `backupThread_`: 备份线程
- `backupThreadLock_`: 备份线程锁
- `backupThreadCv_`: 备份线程条件变量
- `backupListLock_`: 备份队列锁
- `backupList_`: 备份操作队列

---

## 文件级别的关系图

```
mmc_meta_backup_mgr_default.h (备份管理器实现)
    |
    +-- 继承 MMCMetaBackUpMgr (接口)
    |
    +-- 使用 MetaNetServer (网络通信)
    |
    +-- 异步备份线程模型
    +-- 批量备份 (最多 1024 个)
```
