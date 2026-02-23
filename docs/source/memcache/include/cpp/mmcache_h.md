# mmcache.h 文档

## 文件概述

`mmcache.h` 是 MemCache_Hybrid 项目的 C++ API 头文件，提供了面向对象的 C++ 接口，简化了客户端的使用。

**文件路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/include/cpp/mmcache.h`

## 依赖/包含

```c
#include <string>
#include <vector>
#include <iostream>
#include <memory>
```

## 命名空间

**第20-21行**:
```cpp
namespace ock {
namespace mmc {
```

**说明**: 所有类定义在 `ock::mmc` 命名空间中。

---

## 类定义

### KeyInfo 类 (第23-86行)

#### 类概述
`KeyInfo` 类用于存储和访问数据对象的元信息。

#### 成员变量 (第81-85行)
```cpp
private:
    uint64_t size_{};              // 数据大小
    uint32_t blobNum_{};           // Blob数量
    std::vector<int> loc_{};       // Blob的位置列表
    std::vector<int> type_{};      // Blob的介质类型列表
```

#### 构造函数 (第25行)
```cpp
KeyInfo(uint64_t size, uint32_t blobNum) : size_(size), blobNum_(blobNum) {};
```

#### 析构函数 (第27行)
```cpp
~KeyInfo() = default;
```

#### Size() (第29-32行)
```cpp
uint64_t Size()
{
    return size_;
}
```
**功能**: 获取数据大小。

#### GetBlobNum() (第34-37行)
```cpp
uint32_t GetBlobNum()
{
    return blobNum_;
}
```
**功能**: 获取 Blob 数量。

#### GetLocs() (第39-42行)
```cpp
std::vector<int> &GetLocs()
{
    return loc_;
}
```
**功能**: 获取 Blob 位置列表。

#### GetTypes() (第44-47行)
```cpp
std::vector<int> &GetTypes()
{
    return type_;
}
```
**功能**: 获取 Blob 介质类型列表。

#### AddType() (第49-52行)
```cpp
void AddType(int type)
{
    type_.emplace_back(type);
}
```
**功能**: 添加一个 Blob 介质类型。

#### AddLoc() (第54-57行)
```cpp
void AddLoc(int loc)
{
    loc_.emplace_back(loc);
}
```
**功能**: 添加一个 Blob 位置。

#### ToString() (第59-73行)
```cpp
std::string ToString() const
{
    std::stringstream desc;
    desc << "loc:";
    for (auto loc : loc_) {
        desc << std::to_string(loc) << ",";
    }
    desc << " type:";
    for (auto type : type_) {
        desc << std::to_string(type) << ",";
    }
    desc << " blobNum:" << blobNum_;
    desc << ", size:" << size_;
    return desc.str();
}
```
**功能**: 将 KeyInfo 转换为字符串描述。

#### operator<< (第75-79行)
```cpp
friend std::ostream &operator<<(std::ostream &os, const KeyInfo &keyInfo)
{
    os << "keyinfo{" << keyInfo.ToString() << "}";
    return os;
}
```
**功能**: 重载输出流运算符，支持直接输出。

---

### ReplicateConfig 类 (第88-92行)

#### 类概述
`ReplicateConfig` 类用于配置数据副本策略。

#### 成员变量 (第89-91行)
```cpp
public:
    uint16_t replicaNum{1u};                      // 副本数量，默认为1，最大不超过8
    std::vector<int32_t> preferredLocalServiceIDs; // 首选本地服务ID列表
```

**说明**: 用于控制数据存储时的副本数量和位置偏好。

---

### ObjectStore 类 (第94-305行)

#### 类概述
`ObjectStore` 是核心的抽象接口类，定义了对象存储的所有操作。

##### 构造/析构函数 (第96-97行)
```cpp
ObjectStore() = default;
virtual ~ObjectStore() = 0;
```

**说明**: 纯虚析构函数，使 `ObjectStore` 成为抽象类。

##### 拷贝控制 (第99-105行)
```cpp
// 禁止拷贝
ObjectStore(const ObjectStore &) = delete;
ObjectStore &operator=(const ObjectStore &) = delete;

// 允许移动
ObjectStore(ObjectStore &&) noexcept = default;
ObjectStore &operator=(ObjectStore &&) noexcept = default;
```

##### CreateObjectStore() (第111行) - 静态方法
```cpp
static std::shared_ptr<ObjectStore> CreateObjectStore();
```
**功能**: 创建默认的对象存储实例。

##### Init() (第119行)
```cpp
virtual int Init(const uint32_t deviceId, bool initBm = true) = 0;
```
**功能**: 初始化对象存储。

**参数**:
- `deviceId`: 设备 ID
- `initBm`: 是否初始化 Blob Manager，默认为 true

**返回值**: 成功返回 0，失败返回其他值。

##### TearDown() (第125行)
```cpp
virtual int TearDown() = 0;
```
**功能**: 反初始化对象存储。

##### RegisterBuffer() (第133行)
```cpp
virtual int RegisterBuffer(void *buffer, size_t size) = 0;
```
**功能**: 为零拷贝操作注册缓冲区。

##### UnRegisterBuffer() (第141行)
```cpp
virtual int UnRegisterBuffer(void *buffer, size_t size) = 0;
```
**功能**: 注销缓冲区。

##### GetInto() (第151行)
```cpp
virtual int GetInto(const std::string &key, void *buffer, size_t size, const int32_t direct = 2) = 0;
```
**功能**: 直接将对象数据获取到预分配的缓冲区。

**参数**:
- `key`: 对象的键
- `buffer`: 预分配的缓冲区指针
- `size`: 缓冲区大小
- `direct`: 数据位置指示器，参考 `smem_bm_copy_type`

##### BatchGetInto() (第163-164行)
```cpp
virtual std::vector<int> BatchGetInto(const std::vector<std::string> &keys,
                                      const std::vector<void *> &buffers,
                                      const std::vector<size_t> &sizes,
                                      const int32_t direct = 2) = 0;
```
**功能**: 批量获取多个对象数据到预分配的缓冲区。

##### GetIntoLayers() (第174-175行)
```cpp
virtual int GetIntoLayers(const std::string &key,
                          const std::vector<void *> &buffers,
                          const std::vector<size_t> &sizes,
                          const int32_t direct = 2) = 0;
```
**功能**: 获取分层对象数据（如模型层）到预分配的缓冲区。

##### BatchGetIntoLayers() (第187-190行)
```cpp
virtual std::vector<int> BatchGetIntoLayers(const std::vector<std::string> &keys,
                                            const std::vector<std::vector<void *>> &buffers,
                                            const std::vector<std::vector<size_t>> &sizes,
                                            const int32_t direct = 2) = 0;
```
**功能**: 批量获取多个分层对象数据。

##### PutFrom() (第201-202行)
```cpp
virtual int PutFrom(const std::string &key, void *buffer, size_t size,
                    const int32_t direct = 3,
                    const ReplicateConfig &replicateConfig = {}) = 0;
```
**功能**: 直接从预分配的缓冲区存储对象数据。

**参数**:
- `key`: 对象的键
- `buffer`: 包含数据的缓冲区指针
- `size`: 缓冲区大小
- `direct`: 数据位置指示器
- `replicateConfig`: 副本配置

##### GetLocalServiceId() (第209行)
```cpp
virtual int GetLocalServiceId(uint32_t &localServiceId) = 0;
```
**功能**: 获取当前服务实例 ID。

##### BatchPutFrom() (第222-224行)
```cpp
virtual std::vector<int> BatchPutFrom(const std::vector<std::string> &keys,
                                      const std::vector<void *> &buffers,
                                      const std::vector<size_t> &sizes,
                                      const int32_t direct = 3,
                                      const ReplicateConfig &replicateConfig = {}) = 0;
```
**功能**: 批量存储多个对象数据。

##### PutFromLayers() (第235-237行)
```cpp
virtual int PutFromLayers(const std::string &key,
                          const std::vector<void *> &buffers,
                          const std::vector<size_t> &sizes,
                          const int32_t direct = 3,
                          const ReplicateConfig &replicateConfig = {}) = 0;
```
**功能**: 存储分层对象数据。

##### BatchPutFromLayers() (第250-253行)
```cpp
virtual std::vector<int> BatchPutFromLayers(const std::vector<std::string> &keys,
                                            const std::vector<std::vector<void *>> &buffers,
                                            const std::vector<std::vector<size_t>> &sizes,
                                            const int32_t direct = 3,
                                            const ReplicateConfig &replicateConfig = {}) = 0;
```
**功能**: 批量存储多个分层对象数据。

##### Remove() (第260行)
```cpp
virtual int Remove(const std::string &key) = 0;
```
**功能**: 移除对象。

##### BatchRemove() (第268行)
```cpp
virtual std::vector<int> BatchRemove(const std::vector<std::string> &keys) = 0;
```
**功能**: 批量移除多个对象。

##### RemoveAll() (第275行)
```cpp
virtual int RemoveAll() = 0;
```
**功能**: 移除所有对象。

##### IsExist() (第282行)
```cpp
virtual int IsExist(const std::string &key) = 0;
```
**功能**: 检查对象是否存在。

**返回值**:
- `1`: 存在
- `0`: 不存在
- `-1`: 错误

##### BatchIsExist() (第290行)
```cpp
virtual std::vector<int> BatchIsExist(const std::vector<std::string> &keys) = 0;
```
**功能**: 批量检查多个对象是否存在。

##### GetKeyInfo() (第297行)
```cpp
virtual KeyInfo GetKeyInfo(const std::string &key) = 0;
```
**功能**: 获取对象信息。

##### BatchGetKeyInfo() (第304行)
```cpp
virtual std::vector<KeyInfo> BatchGetKeyInfo(const std::vector<std::string> &keys) = 0;
```
**功能**: 批量获取多个对象信息。

---

## 命名空间结束

**第307-308行**:
```cpp
} // namespace mmc
} // namespace ock
```

---

## 宏定义

**第12行**: 头文件保护宏
```c
#ifndef __MM_CACHE_H__
#define __MM_CACHE_H__
```

**第309行**: 头文件保护宏结束
```c
#endif
```
