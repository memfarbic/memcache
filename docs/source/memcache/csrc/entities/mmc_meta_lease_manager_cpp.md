# mmc_meta_lease_manager.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/entities/mmc_meta_lease_manager.cpp`
- **文件用途**: 实现 `MmcMetaLeaseManager` 类的方法
- **依赖项**:
  - `mmc_meta_lease_manager.h` - 租约管理器类定义
  - `cstdint` - 整数类型
  - `limits` - 数值限制
  - `thread` - 线程支持
  - `mmc_montotonic.h` - 单调时间
  - `mmc_types.h` - 类型定义
  - `mmc_define.h` - 宏定义

---

## 类方法实现

### MmcMetaLeaseManager::Add

**声明位置**: 行 24-34
**完整签名**:
```cpp
Result MmcMetaLeaseManager::Add(uint32_t id, uint32_t requestId, uint64_t ttl)
```
**功能描述**: 添加一个租约记录
**参数**:
- `id` - Rank ID
- `requestId` - 请求 ID
- `ttl` - 租约存活时间（毫秒）
**返回值**: `Result` - 操作结果
**可能返回的错误**:
- `MMC_INVALID_PARAM` - TTL 溢出

**代码逻辑**:
1. **记录日志**:
   ```cpp
   MMC_LOG_DEBUG("MmcMetaLeaseManager ADD " << " id " << id << " requestId " << requestId << " ttl " << ttl);
   ```

2. **获取当前时间**:
   ```cpp
   const uint64_t nowMs = ock::dagger::Monotonic::TimeUs() / 1000U;
   ```

3. **检查 TTL 溢出**:
   ```cpp
   if (ttl > std::numeric_limits<uint64_t>::max() - nowMs) {
       return MMC_INVALID_PARAM;
   }
   ```

4. **更新租约到期时间**:
   ```cpp
   lease_ = std::max(lease_, nowMs + ttl);
   ```
   - 使用 max 确保租约时间只延长不缩短

5. **添加客户端 ID**:
   ```cpp
   useClient.insert(GenerateClientId(id, requestId));
   ```

6. **返回成功**:
   ```cpp
   return MMC_OK;
   ```

**注意事项**:
- 多次调用 Add 会延长租约时间
- 同一个 (id, requestId) 组合重复添加会被 `unordered_set` 自动去重

---

### MmcMetaLeaseManager::Remove

**声明位置**: 行 36-41
**完整签名**:
```cpp
Result MmcMetaLeaseManager::Remove(uint32_t id, uint32_t requestId)
```
**功能描述**: 移除一个租约记录
**参数**:
- `id` - Rank ID
- `requestId` - 请求 ID
**返回值**: 始终返回 `MMC_OK`

**代码逻辑**:
1. **记录日志**:
   ```cpp
   MMC_LOG_DEBUG("MmcMetaLeaseManager Remove id " << id << " requestId " << requestId);
   ```

2. **移除客户端 ID**:
   ```cpp
   useClient.erase(GenerateClientId(id, requestId));
   ```

3. **返回成功**:
   ```cpp
   return MMC_OK;
   ```

**注意事项**:
- 如果客户端 ID 不存在，erase 不会报错
- Remove 操作不更新 lease_ 时间

---

### MmcMetaLeaseManager::Extend

**声明位置**: 行 43-52
**完整签名**:
```cpp
Result MmcMetaLeaseManager::Extend(uint64_t ttl)
```
**功能描述**: 延长所有租约的有效期
**参数**:
- `ttl` - 延长的时长（毫秒）
**返回值**: `Result` - 操作结果
**可能返回的错误**:
- `MMC_INVALID_PARAM` - TTL 溢出

**代码逻辑**:
1. **记录日志**:
   ```cpp
   MMC_LOG_DEBUG("MmcMetaLeaseManager Extend " << " ttl " << ttl);
   ```

2. **获取当前时间**:
   ```cpp
   const uint64_t nowMs = ock::dagger::Monotonic::TimeUs() / 1000U;
   ```

3. **检查 TTL 溢出**:
   ```cpp
   if (ttl > std::numeric_limits<uint64_t>::max() - nowMs) {
       return MMC_INVALID_PARAM;
   }
   ```

4. **更新租约到期时间**:
   ```cpp
   lease_ = std::max(lease_, nowMs + ttl);
   ```

5. **返回成功**:
   ```cpp
   return MMC_OK;
   ```

**注意事项**:
- Extend 延长所有租约的到期时间
- 使用 max 确保租约时间只延长不缩短
- 不会添加新的客户端 ID

---

### MmcMetaLeaseManager::Wait

**声明位置**: 行 54-62
**完整签名**:
```cpp
void MmcMetaLeaseManager::Wait()
```
**功能描述**: 等待所有租约过期或释放
**返回值**: 无

**代码逻辑**:
1. **循环检查**:
   ```cpp
   while (!useClient.empty()) {
   ```

2. **检查是否已过期**:
   ```cpp
   if ((ock::dagger::Monotonic::TimeUs() / 1000ULL) >= lease_) {
       return;
   }
   ```
   - 如果当前时间超过租约到期时间，直接返回

3. **休眠等待**:
   ```cpp
   std::this_thread::sleep_for(std::chrono::milliseconds(MMC_DATA_TTL_MS / 10ULL));
   ```
   - 休眠 TTL 的 1/10 时间（默认 200ms）
   - 然后再次循环检查

4. **循环结束**:
   - 当 useClient 为空时自动退出

**流程图**:
```
         ┌─────────────────┐
         │     Wait()      │
         └────────┬────────┘
                  │
                  ▼
         ┌─────────────────┐
         │ useClient.empty?│────▶│ 返回            │
         └────────┬────────┘     └──────────────────┘
                  │ 否
                  ▼
         ┌─────────────────┐     ┌──────────────────┐
         │ 当前时间 >=      │────▶│ 返回 (租约已过期) │
         │ lease_?         │     └──────────────────┘
         └────────┬────────┘
                  │ 否
                  ▼
         ┌─────────────────┐
         │ 休眠 TTL/10     │
         │ (默认 200ms)    │
         └────────┬────────┘
                  │
                  └────────────────────┘
                          │
                          ▼
                    (回到开始)
```

**注意事项**:
- 如果 useClient 为空，立即返回
- 如果租约已过期，立即返回
- 否则休眠等待，避免忙等待
- 典型用于删除操作前等待所有读取完成

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MmcMetaLeaseManager 实现                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────┐      ┌────────────────┐      ┌────────────────┐   │
│  │     Add()      │      │   Remove()     │      │   Extend()     │   │
│  │  - 获取时间     │      │  - 移除 client │      │  - 获取时间     │   │
│  │  - 检查溢出     │      │    ID         │      │  - 检查溢出     │   │
│  │  - 更新 lease_  │      │                │      │  - 更新 lease_  │   │
│  │  - 插入 client  │      │                │      │                │   │
│  │    ID          │      │                │      │                │   │
│  └────────────────┘      └────────────────┘      └────────────────┘   │
│                                                                          │
│  ┌────────────────┐                                                      │
│  │     Wait()     │                                                      │
│  │  - 检查集合     │                                                      │
│  │  - 检查时间     │                                                      │
│  │  - 休眠等待     │                                                      │
│  └────────────────┘                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                    │
                    │ 使用
                    ▼
         ┌──────────────────────────────────────────────────────────────┐
         │                    ock::dagger::Monotonic                    │
         │                    TimeUs() - 获取单调时间                    │
         └──────────────────────────────────────────────────────────────┘
```

---

## 租约管理流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        完整租约流程                                     │
└─────────────────────────────────────────────────────────────────────────┘

  客户端 A 读取:                  客户端 B 读取:
  ┌─────────────┐                ┌─────────────┐
  │ Add(0, 100) │                │ Add(1, 101) │
  └──────┬──────┘                └──────┬──────┘
         │                               │
         ▼                               ▼
  useClient = {A}                 useClient = {A, B}
  lease_ = now + 2000             lease_ = now + 2000

         ┌─────────────────────────────────────┐
         │   服务器收到删除请求                  │
         └──────────────┬──────────────────────┘
                      │
                      ▼
              ┌───────────────┐
              │ Wait() 被调用 │
              └───────┬───────┘
                      │
         ┌────────────┴────────────┐
         │                         │
         ▼                         ▼
  useClient = {A, B}          检查时间
         │                         │
         │                 ┌───────┴───────┐
         │                 │ now < lease_? │
         │                 └───────┬───────┘
         │                         │ 是
         │                         ▼
         │                 ┌───────────────┐
         │                 │ sleep(200ms)  │
         │                 └───────┬───────┘
         │                         │
         └─────────────────────────┘

  客户端 A 完成:
  ┌─────────────┐
  │Remove(0,100)│
  └──────┬──────┘
         │
         ▼
  useClient = {B}

  客户端 B 完成:
  ┌─────────────┐
  │Remove(1,101)│
  └──────┬──────┘
         │
         ▼
  useClient = {}

         ┌─────────────────┐
         │ Wait() 返回     │
         └─────────────────┘
```

---

## 使用示例

### 示例 1: 基本租约管理

```cpp
#include "mmc_meta_lease_manager.h"

using namespace ock::mmc;

MmcMetaLeaseManager leaseMgr;

// 客户端 1 添加租约
Result ret = leaseMgr.Add(0, 100, 2000);
if (ret == MMC_OK) {
    std::cout << "Lease added, use count: " << leaseMgr.UseCount() << std::endl;
}

// 客户端 2 添加租约
ret = leaseMgr.Add(1, 101, 2000);
std::cout << "Use count: " << leaseMgr.UseCount() << std::endl;

// 客户端 1 移除租约
leaseMgr.Remove(0, 100);
std::cout << "After remove, use count: " << leaseMgr.UseCount() << std::endl;
```

### 示例 2: 延长租约

```cpp
MmcMetaLeaseManager leaseMgr;

// 添加 2 秒租约
leaseMgr.Add(0, 100, 2000);

// 延长 1 秒
Result ret = leaseMgr.Extend(1000);
if (ret == MMC_OK) {
    std::cout << "Lease extended" << std::endl;
}
```

### 示例 3: 等待租约清空

```cpp
MmcMetaLeaseManager leaseMgr;

// 模拟多个客户端持有租约
leaseMgr.Add(0, 100, 2000);
leaseMgr.Add(1, 101, 2000);
leaseMgr.Add(2, 102, 2000);

std::cout << "Active leases: " << leaseMgr.UseCount() << std::endl;

// 模拟客户端逐步释放
leaseMgr.Remove(0, 100);
std::cout << "After client 0: " << leaseMgr.UseCount() << std::endl;

leaseMgr.Remove(1, 101);
std::cout << "After client 1: " << leaseMgr.UseCount() << std::endl;

// 等待租约过期（或手动移除）
leaseMgr.Wait();
std::cout << "All leases cleared" << std::endl;
```

### 示例 4: 错误处理

```cpp
MmcMetaLeaseManager leaseMgr;

// 正常添加
Result ret = leaseMgr.Add(0, 100, 2000);

// TTL 溢出（极端情况）
uint64_t hugeTTL = std::numeric_limits<uint64_t>::max();
ret = leaseMgr.Add(1, 101, hugeTTL);
if (ret == MMC_INVALID_PARAM) {
    std::cout << "Invalid TTL: overflow" << std::endl;
}
```

### 示例 5: 实际使用场景

```cpp
// 场景：Blob 删除前等待所有读取完成
class BlobManager {
    MmcMetaLeaseManager leaseMgr_;

    Result StartRead(uint32_t rankId, uint32_t requestId) {
        // 读取开始，添加租约
        return leaseMgr_.Add(rankId, requestId, MMC_DATA_TTL_MS);
    }

    Result EndRead(uint32_t rankId, uint32_t requestId) {
        // 读取结束，移除租约
        return leaseMgr_.Remove(rankId, requestId);
    }

    Result DeleteBlob() {
        // 删除前等待所有读取完成
        leaseMgr_.Wait();

        // 执行删除操作
        return DoDelete();
    }

    Result ExtendRead(uint32_t rankId, uint32_t requestId, uint64_t ttl) {
        // 读取超时，延长租约
        return leaseMgr_.Extend(ttl);
    }
};
```

### 示例 6: 并发读取场景

```cpp
MmcMetaLeaseManager leaseMgr;

// 客户端 0 开始读取
leaseMgr.Add(0, 100, 2000);
MMC_LOG_INFO("Client 0 start reading, active: " << leaseMgr.UseCount());

// 客户端 1 开始读取
leaseMgr.Add(1, 101, 2000);
MMC_LOG_INFO("Client 1 start reading, active: " << leaseMgr.UseCount());

// 客户端 0 完成读取
leaseMgr.Remove(0, 100);
MMC_LOG_INFO("Client 0 done, active: " << leaseMgr.UseCount());

// 客户端 1 完成读取
leaseMgr.Remove(1, 101);
MMC_LOG_INFO("Client 1 done, active: " << leaseMgr.UseCount());

// 等待（应该立即返回，因为集合已空）
leaseMgr.Wait();
```
