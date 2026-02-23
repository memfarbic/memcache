# mmc_net_ctx_store.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/acc_links_impl/mmc_net_ctx_store.h`
- **文件用途**: 实现网络上下文存储，支持扁平数组和哈希表两级存储结构
- **依赖项**:
  - `mmc_common_includes.h` - 公共头文件
  - `mmc_net_common_acc.h` - ACC 网络通用定义

---

## 常量

### TRY_GET_FLAT_TIME

```cpp
constexpr uint32_t TRY_GET_FLAT_TIME = 3; // Try to get empty flat bucket 3 times
```

**说明**: 尝试获取空闲扁平数组槽位的次数

---

## 类定义

### NetContextStore 类

网络上下文存储类，用于管理 RPC 调用的上下文

```cpp
class NetContextStore : public MmcReferable {
public:
    explicit NetContextStore(uint32_t flatCapacity) : mFlatCapacity(flatCapacity) {}

    ~NetContextStore() override;

    Result Initialize() noexcept;
    void UnInitialize();

    template<typename T>
    Result PutAndGetSeqNo(T *ctx, uint32_t &output);

    template<typename T>
    Result GetSeqNoAndRemove(uint32_t seqNo, T *&out, bool decreaseRef = true);

    template<typename T>
    inline void RemoveSeqNo(uint32_t seqNo);

private:
    static constexpr uint32_t gVersionMask = 0x3F;
    static constexpr uint32_t gVersionBitWidth = 6L;
    static constexpr uint32_t gHashCount = 4L;
    static constexpr uint64_t gPtrMask = 0x03FFFFFFFFFFFFFF;

private:
    uint32_t mSeqNoAndVersionIndex = 1;
    uint32_t mSeqNoAndVersionMask = 0;
    uint32_t mSeqNoMask = 0;
    uint32_t mVersionShift = 0;
    uint32_t mFlatCapacity = N8192;
    uint64_t *mFlatCtxBucks = nullptr;

    std::mutex mHashCtxMutex[gHashCount];
    std::unordered_map<uint32_t, uint64_t> mHashCtxMap[gHashCount];
};
```

**设计说明**:
- 采用两级存储结构：扁平数组（快速访问）+ 哈希表（溢出处理）
- 使用版本号机制，支持槽位复用
- 使用无锁操作（CAS）提高并发性能

---

## 方法

### NetContextStore::~NetContextStore()

```cpp
~NetContextStore() override
{
    UnInitialize();
}
```

**声明位置**: 行 27-30

**功能描述**: 析构函数，自动调用 UnInitialize 清理资源

---

### NetContextStore::Initialize()

```cpp
Result Initialize() noexcept
{
    /* validate the capacity */
    if (mFlatCapacity < UN128) {
        mFlatCapacity = UN128;
    } else if (mFlatCapacity > UN16777216) {
        mFlatCapacity = UN16777216; /* each bucket is an uint64_t, 128MB is occupied */
    }

    /* get aligned capacity */
    mFlatCapacity = 1 << (UN32 - __builtin_clz(mFlatCapacity) - 1);
    /* get seqNo mask */
    mSeqNoMask = mFlatCapacity - 1;
    /* get version shift for move right */
    mVersionShift = __builtin_popcount(mSeqNoMask);
    /* get version and seqNo mask, as version occupied 6 bits */
    mSeqNoAndVersionMask = (1 << (mVersionShift + gVersionBitWidth)) - 1;

    mFlatCtxBucks = new (std::nothrow) uint64_t[mFlatCapacity];
    if (mFlatCtxBucks == nullptr) {
        MMC_LOG_ERROR("Failed to new service flat context buckets, probably out of memory");
        return MMC_NEW_OBJECT_FAILED;
    }

    /* make physical memory allocated and set them to 0 */
    bzero(mFlatCtxBucks, sizeof(uint64_t) * mFlatCapacity);

    /* reserved hash bucket for unordered map */
    for (auto &i : mHashCtxMap) {
        i.reserve(N1024);
    }
    MMC_LOG_INFO("Initialized context store, flatten capacity "
                 << mFlatCapacity << ", versionAndSeqMask " << mSeqNoAndVersionMask << ", seqNoMask " << mSeqNoMask
                 << ", seqNoAndVersionIndex " << mSeqNoAndVersionIndex);

    return MMC_OK;
}
```

**声明位置**: 行 37-73

**功能描述**: 初始化上下文存储

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 验证容量范围 (128 ~ 16777216)
2. 将容量调整为 2 的幂次方
3. 计算各种掩码和位移量
4. 分配扁平数组内存并清零
5. 预留哈希表空间

**计算说明**:
- `mSeqNoMask`: 序列号掩码，用于提取序列号部分
- `mVersionShift`: 版本号右移量
- `mSeqNoAndVersionMask`: 序列号和版本号组合掩码

---

### NetContextStore::UnInitialize()

```cpp
void NetContextStore::UnInitialize()
{
    if (mFlatCtxBucks != nullptr) {
        delete[] mFlatCtxBucks;
        mFlatCtxBucks = nullptr;
    }
}
```

**声明位置**: 行 75-81

**功能描述**: 释放上下文存储资源

**代码逻辑**:
1. 释放扁平数组内存
2. 将指针置空

---

### NetContextStore::PutAndGetSeqNo()

```cpp
template<typename T>
Result PutAndGetSeqNo(T *ctx, uint32_t &output)
{
    if (UNLIKELY(ctx == nullptr)) {
        return MMC_INVALID_PARAM;
    }

    auto value = reinterpret_cast<uint64_t>(ctx);
    NetSeqNo sn(0);
    uint32_t mapIndex = 0;

    uint32_t newSeqAndVersion = 0;
    uint32_t seqNo = 0;
    uint64_t version = 0;
    for (uint32_t i = 0; i < TRY_GET_FLAT_TIME; i++) {
        /* get the seqNo with increasing and mask, if the seqNo is 0, increase again */
        newSeqAndVersion = __sync_fetch_and_add(&mSeqNoAndVersionIndex, 1);
        if (UNLIKELY(newSeqAndVersion & mSeqNoMask) == 0) {
            newSeqAndVersion = __sync_fetch_and_add(&mSeqNoAndVersionIndex, 1);
        }

        /* get seqNo and version, and mixed value with version and ctx ptr for CAS */
        seqNo = newSeqAndVersion & mSeqNoMask;
        version = (newSeqAndVersion >> mVersionShift) & gVersionMask;
        value = (version << UN58) | value; /* high 6 bits store version */
        if (__sync_bool_compare_and_swap(&mFlatCtxBucks[seqNo], 0, value)) {
            sn.SetValue(1, static_cast<uint32_t>(version), seqNo);
            output = sn.wholeSeq;
            /* increase ref count */
            ctx->IncreaseRef();
            return MMC_OK;
        }
    }

    /* tried 3 times no luck to get an empty bucket, store in hash map */
    mapIndex = seqNo % gHashCount;
    sn.SetValue(0, static_cast<uint32_t>(version), seqNo);
    output = sn.wholeSeq;
    {
        std::lock_guard<std::mutex> guard(mHashCtxMutex[mapIndex]);
        bool inserted = mHashCtxMap[mapIndex].emplace(sn.wholeSeq, value).second;
        if (inserted) {
            /* increase ref count */
            ctx->IncreaseRef();
        }
        return inserted ? MMC_OK : MMC_NET_SEQ_DUP;
    }
}
```

**声明位置**: 行 93-151

**功能描述**: 存储上下文并获取序列号

**参数**:
- `ctx`: 要存储的上下文指针
- `output`: 输出参数，生成的序列号

**返回值**: `Result` - MMC_OK 表示成功

**代码逻辑**:
1. 验证输入参数
2. 尝试 3 次在扁平数组中寻找空闲槽位
3. 如果成功，使用 CAS 操作存储指针并返回序列号
4. 如果 3 次都失败，存储到哈希表中

**无锁设计**:
- 使用 `__sync_fetch_and_add` 原子递增序列号
- 使用 `__sync_bool_compare_and_swap` 原子交换

**使用示例**:
```cpp
auto waiter = MmcMakeRef<NetWaitHandler>(ctxStore);
uint32_t seqNo = 0;
result = ctxStore->PutAndGetSeqNo<NetWaitHandler>(waiter.Get(), seqNo);
```

---

### NetContextStore::GetSeqNoAndRemove()

```cpp
template<typename T>
Result GetSeqNoAndRemove(uint32_t seqNo, T *&out, bool decreaseRef = true)
{
    NetSeqNo no(0);
    no.wholeSeq = seqNo;

    if (LIKELY(no.fromFlat == 1)) {
        /* create the old pointer and */
        uint64_t value = mFlatCtxBucks[no.realSeq] & gPtrMask;
        uint64_t tmpVersion = no.version;

        /* if timeout thread already get seq no, next time will
           1、CAS OK, but get value is 0
           2、CAS ERR by version++ */
        if (__sync_bool_compare_and_swap(&mFlatCtxBucks[no.realSeq], (tmpVersion << UN58) | value, 0)) {
            if (UNLIKELY(value == 0)) {
                return MMC_NET_SEQ_NO_FOUND;
            }

            out = reinterpret_cast<T *>(value);
            /* decrease ref count */
            if (decreaseRef) {
                out->DecreaseRef();
            }
            return MMC_OK;
        }

        return MMC_NET_SEQ_NO_FOUND;
    }

    uint32_t mapIndex = no.realSeq % gHashCount;
    no.isResp = 0;
    {
        std::lock_guard<std::mutex> guard(mHashCtxMutex[mapIndex]);
        auto iter = mHashCtxMap[mapIndex].find(no.wholeSeq);
        if (LIKELY(iter != mHashCtxMap[mapIndex].end())) {
            out = reinterpret_cast<T *>(iter->second & gPtrMask);
            /* decrease ref count */
            if (decreaseRef) {
                out->DecreaseRef();
            }
            mHashCtxMap[mapIndex].erase(iter);
            return MMC_OK;
        }
    }

    return MMC_NET_SEQ_NO_FOUND;
}
```

**声明位置**: 行 164-211

**功能描述**: 根据序列号获取并移除上下文

**参数**:
- `seqNo`: 序列号
- `out`: 输出参数，获取的上下文指针
- `decreaseRef`: 是否减少引用计数，默认 true

**返回值**: `Result` - MMC_OK 表示成功，MMC_NET_SEQ_NO_FOUND 表示未找到

**代码逻辑**:
1. 解析序列号，判断存储位置
2. 如果在扁平数组中，使用 CAS 操作移除
3. 如果在哈希表中，加锁查找并移除

**版本号验证**:
- CAS 操作时验证版本号，防止 ABA 问题
- 如果版本不匹配，说明已被其他线程修改

---

### NetContextStore::RemoveSeqNo()

```cpp
template<typename T>
inline void RemoveSeqNo(uint32_t seqNo)
{
    T *out = nullptr;
    if (UNLIKELY(GetSeqNoAndRemove<T>(seqNo, out) != MMC_OK)) {
        NetSeqNo dumpSeq(seqNo);
        MMC_LOG_ERROR("Failed to remove ctx with seqNo " << dumpSeq.ToString() << " as not found");
        return;
    }
}
```

**声明位置**: 行 214-222

**功能描述**: 移除指定序列号的上下文（不获取指针）

**参数**: `seqNo` - 要移除的序列号

---

## 成员变量

### 私有静态常量

```cpp
static constexpr uint32_t gVersionMask = 0x3F;           /* mask to reverse version */
static constexpr uint32_t gVersionBitWidth = 6L;         /* mask to reverse version */
static constexpr uint32_t gHashCount = 4L;               /* hash map count */
static constexpr uint64_t gPtrMask = 0x03FFFFFFFFFFFFFF; /* ptr mask */
```

- `gVersionMask`: 版本号掩码 (0x3F = 6 位)
- `gVersionBitWidth`: 版本号位宽度
- `gHashCount`: 哈希表数量
- `gPtrMask`: 指针掩码（去掉高 6 位版本号）

---

### 私有成员变量

```cpp
uint32_t mSeqNoAndVersionIndex = 1; /* atomic increase seqNo and version */
uint32_t mSeqNoAndVersionMask = 0;  /* mask to reverse the seqNo and version */
uint32_t mSeqNoMask = 0;            /* mask to reverse the seqNo */
uint32_t mVersionShift = 0;         /* move right shift num to get version */
uint32_t mFlatCapacity = N8192;     /* flat array capacity */
uint64_t *mFlatCtxBucks = nullptr;  /* actually array to store the ptr */

std::mutex mHashCtxMutex[gHashCount];                           /* mutex to guard unordered_map */
std::unordered_map<uint32_t, uint64_t> mHashCtxMap[gHashCount]; /* unordered_map to store un-flat */
```

**布局优化**:
- 频繁访问的变量放在前面
- 确保变量对齐
- 总大小不超过一个缓存行（64 字节）

---

## 文件级别的关系图

```
mmc_net_ctx_store.h (上下文存储)
    |
    +-- NetContextStore -> 上下文存储类
            |
            +-- Initialize() -> 初始化存储
            +-- UnInitialize() -> 清理资源
            +-- PutAndGetSeqNo() -> 存储并获取序列号
            +-- GetSeqNoAndRemove() -> 获取并移除
            +-- RemoveSeqNo() -> 仅移除
            |
            +-> 两级存储: 扁平数组 + 哈希表
            +-> 版本号机制: 防止 ABA 问题
```

---

## 依赖关系

**依赖以下文件**:
- `mmc_common_includes.h` - 公共头文件
- `mmc_net_common_acc.h` - ACC 网络通用定义

**被以下文件依赖**:
- `mmc_net_wait_handle.h` - 等待处理器
- `mmc_net_engine_acc.cpp` - ACC 网络引擎实现

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有上下文存储定义都在 ock::mmc 命名空间内
}
}
```

---

## 性能设计要点

1. **扁平数组优先**: 大多数请求使用扁平数组，O(1) 访问
2. **无锁 CAS 操作**: 使用原子操作避免锁竞争
3. **版本号机制**: 6 位版本号支持 64 次槽位复用
4. **溢出哈希表**: 当扁平数组满时，使用哈希表作为后备
5. **分片哈希**: 4 个哈希表减少锁竞争
