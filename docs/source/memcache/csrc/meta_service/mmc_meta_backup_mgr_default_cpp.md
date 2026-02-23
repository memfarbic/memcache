# mmc_meta_backup_mgr_default.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_backup_mgr_default.cpp`
- **文件用途**: MMCMetaBackUpMgrDefault 类的实现文件
- **依赖项**: `mmc_meta_backup_mgr_default.h`, `mmc_msg_client_meta.h`

---

## 静态成员变量初始化

```cpp
std::map<std::string, MmcRef<MMCMetaBackUpMgr>> MMCMetaBackUpMgrFactory::instances_;
std::mutex MMCMetaBackUpMgrFactory::instanceMutex_;
```

**声明位置**: 行 22-23

**说明**: 工厂类的静态成员变量初始化

---

## 常量定义

```cpp
constexpr uint32_t BATCH_BACKUP_SIZE = 1024;
```

**声明位置**: 行 20

**说明**: 批量备份的最大数量

---

## 函数实现

### BackupThreadFunc

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

**声明位置**: 行 25-41

**功能描述**: 备份线程的主函数

**代码逻辑**:
1. 等待备份队列有数据或停止信号
2. 如果收到停止信号，退出循环
3. 调用 SendBackup2Local 发送备份数据
4. 循环继续

**线程同步**:
- 使用条件变量等待
- 使用 lambda 表达式作为等待条件

---

### SendBackup2Local

```cpp
void MMCMetaBackUpMgrDefault::SendBackup2Local()
{
    MetaReplicateRequest request;
    Response response;
    uint32_t haveCount = 1;
    std::vector<uint32_t> ops;
    std::vector<std::string> keys;
    std::vector<MmcMemBlobDesc> blobs;
    uint32_t rank;
    while (haveCount && started_) {
        {
            std::lock_guard<std::mutex> lg(backupListLock_);
            rank = PopMetas2Backup(ops, keys, blobs);
            haveCount = backupList_.size();
            MMC_LOG_DEBUG("BackupThreadFunc bm rank=" << rank);
        }
        if (metaNetServer_ == nullptr) {
            MMC_LOG_WARN("MMCMetaBackUpMgr back up net not start");
            continue;
        }

        if (!keys.empty()) {
            request.ops_ = std::move(ops);
            request.keys_ = std::move(keys);
            request.blobs_ = std::move(blobs);
            Result ret = metaNetServer_->SyncCall(rank, request, response, 60);
            if (ret != MMC_OK) {
                MMC_LOG_ERROR("mmc meta back up failed, bm rank " << rank << ", keys: " << request.KeysString());
            }
        }
    }
}
```

**声明位置**: 行 43-74

**功能描述**: 发送备份到本地节点

**代码逻辑**:
1. 初始化请求和响应对象
2. 循环处理备份队列：
   - 从队列中弹出一批元数据
   - 如果网络服务器未启动则跳过
   - 如果有数据则发送到目标 Rank
   - 处理发送结果
3. 直到队列为空或停止

**批量发送**:
- 一次最多发送 BATCH_BACKUP_SIZE (1024) 个元数据
- 按 Rank 分组发送

---

### PopMetas2Backup

```cpp
uint32_t MMCMetaBackUpMgrDefault::PopMetas2Backup(std::vector<uint32_t> &ops, std::vector<std::string> &keys,
                                                  std::vector<MmcMemBlobDesc> &blobs)
{
    // 清空数组
    ops.clear();
    keys.clear();
    blobs.clear();

    // 获取目标bm rank
    uint32_t rank = UINT32_MAX;
    if (!backupList_.empty()) {
        rank = backupList_.front().desc_.rank_;
    }

    // 根据目标bm rank，尝试获取1024个待备份meta对象
    uint32_t count = 0;
    auto it = backupList_.begin();
    while (it != backupList_.end()) {
        const auto &opInfo = *it;
        if (opInfo.desc_.rank_ == rank) {
            ops.push_back(opInfo.op_);
            keys.push_back(opInfo.key_);
            blobs.push_back(opInfo.desc_);
            it = backupList_.erase(it);
            count++;
        } else {
            ++it;
        }
        if (count >= BATCH_BACKUP_SIZE) {
            break;
        }
    }
    return rank;
}
```

**声明位置**: 行 76-109

**功能描述**: 从备份队列中弹出元数据

**参数**:
- `ops`: 输出操作类型列表
- `keys`: 输出 key 列表
- `blobs`: 输出 Blob 描述符列表

**返回值**: 目标 Rank ID

**代码逻辑**:
1. 清空输出数组
2. 获取队列首个元素的 Rank 作为目标 Rank
3. 遍历队列，收集属于同一 Rank 的元数据
4. 最多收集 BATCH_BACKUP_SIZE 个
5. 从队列中移除已收集的元素
6. 返回目标 Rank ID

**注意事项**:
- 此函数假设已持有 backupListLock_
- 只处理同一 Rank 的元数据，实现按 Rank 分组发送

---

## 总结

此文件实现了默认备份管理器的核心功能：

1. **异步备份**: 使用单独的线程进行备份操作
2. **批量发送**: 一次最多发送 1024 个元数据
3. **按 Rank 分组**: 同一批次的元数据必须属于同一 Rank
4. **网络传输**: 通过 MetaNetServer 发送备份数据

**线程模型**:
- 主线程：添加备份操作到队列
- 备份线程：从队列取出数据并发送

**同步机制**:
- 互斥锁保护队列访问
- 条件变量通知有新数据
