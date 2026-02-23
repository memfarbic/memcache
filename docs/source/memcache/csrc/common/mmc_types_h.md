# mmc_types.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_types.h`
- **文件用途**: 定义 MemCache 项目的核心类型、错误码、枚举、结构体和工具类
- **依赖项**: `mmc_def.h`, `<cstdint>`, `<atomic>`, `<sstream>`, `<vector>`

---

## 类型别名

### Result

```cpp
using Result = int32_t;
```

**声明位置**: 行 24

**功能描述**: 函数返回值类型别名，通常用于表示操作成功或失败

**使用场景**: 所有需要返回状态码的函数

---

## 枚举定义

### MmcErrorCode

```cpp
enum MmcErrorCode : int32_t {
    MMC_OK = 0,                          // 操作成功
    MMC_ERROR = -1,                      // 一般错误
    MMC_INVALID_PARAM = -3000,           // 无效参数
    MMC_MALLOC_FAILED = -3001,           // 内存分配失败
    MMC_NEW_OBJECT_FAILED = -3002,       // 对象创建失败
    MMC_NOT_STARTED = -3003,             // 未启动
    MMC_TIMEOUT = -3004,                 // 操作超时
    MMC_REPEAT_CALL = -3005,             // 重复调用
    MMC_DUPLICATED_OBJECT = -3006,       // 对象重复
    MMC_OBJECT_NOT_EXISTS = -3007,       // 对象不存在
    MMC_NOT_INITIALIZED = -3008,         // 未初始化
    MMC_NET_SEQ_DUP = -3009,             // 网络序列号重复
    MMC_NET_SEQ_NO_FOUND = -3010,        // 网络序列号未找到
    MMC_ALREADY_NOTIFIED = -3011,        // 已通知
    MMC_EXCEED_CAPACITY = -3013,         // 超出容量
    MMC_LINK_NOT_FOUND = -3014,          // 链接未找到
    MMC_NET_REQ_HANDLE_NO_FOUND = -3015, // 网络请求句柄未找到
    MMC_NOT_ENOUGH_MEMORY = -3016,       // 内存不足
    MMC_NOT_CONNET_META = -3017,         // 未连接元数据
    MMC_NOT_CONNET_LOCAL = -3018,        // 未连接本地
    MMC_CLIENT_NOT_INIT = -3019,         // 客户端未初始化
    MMC_UNMATCHED_STATE = -3101,         // 状态不匹配
    MMC_UNMATCHED_KEY = -3102,           // 键不匹配
    MMC_UNMATCHED_RET = -3103,           // 返回值不匹配
    MMC_LEASE_NOT_EXPIRED = -3104,       // 租约未过期
    MMC_META_BACKUP_ERROR = -3105,       // 元数据备份错误
};
```

**声明位置**: 行 26-53

**功能描述**: MemCache 操作的错误码定义

**错误码分类**:
- **0**: 成功
- **-1**: 一般错误
- **-3000 ~ -3019**: 通用错误（参数、内存、连接等）
- **-3101 ~ -3105**: 状态相关错误

---

### MediaType

```cpp
enum MediaType : uint8_t {
    MEDIA_HBM,   // High Bandwidth Memory（高带宽内存）
    MEDIA_DRAM,  // DRAM（动态随机存取内存）
    MEDIA_NONE,  // 无媒体类型
};
```

**声明位置**: 行 77-81

**功能描述**: 定义内存介质类型

**枚举值**:
- `MEDIA_HBM`: 高带宽内存，通常用于高性能计算
- `MEDIA_DRAM`: 常规动态随机存取内存
- `MEDIA_NONE`: 无效或未指定的介质类型

---

### EvictResult

```cpp
enum class EvictResult {
    REMOVE,    // 直接删除数据
    MOVE_DOWN, // 将数据移动到下层存储
    FAIL,      // 淘汰失败
};
```

**声明位置**: 行 128-132

**功能描述**: 数据淘汰操作的返回结果

**枚举值**:
- `REMOVE`: 数据已被直接删除
- `MOVE_DOWN`: 数据已被移动到下层存储介质
- `FAIL`: 淘汰操作失败

---

## 常量定义

### 整数常量

```cpp
constexpr int32_t N16 = 16;
constexpr int32_t N64 = 64;
constexpr int32_t N256 = 256;
constexpr int32_t N1024 = 1024;
constexpr int32_t N8192 = 8192;

constexpr uint32_t UN2 = 2;
constexpr uint32_t UN16 = 16;
constexpr uint32_t UN32 = 32;
constexpr uint32_t UN58 = 58;
constexpr uint32_t UN128 = 128;
constexpr uint32_t UN65536 = 65536;
constexpr uint32_t UN16777216 = 16777216;

constexpr uint32_t MMC_DEFAUT_WAIT_TIME = 120; // 120秒
```

**声明位置**: 行 61-75

**功能描述**: 常用的数值常量，避免在代码中直接使用魔法数字

---

## 结构体定义

### MmcLocation

```cpp
struct MmcLocation {
    uint32_t rank_;        // Rank 编号
    MediaType mediaType_;  // 介质类型

    // 默认构造函数
    MmcLocation() : rank_(UINT32_MAX), mediaType_(MediaType::MEDIA_NONE) {}

    // 带参数构造函数
    MmcLocation(uint32_t rank, MediaType mediaType) : rank_(rank), mediaType_(mediaType) {}

    // 小于运算符重载（用于排序）
    bool operator<(const MmcLocation &other) const
    {
        // 先比较 mediaType_
        if (mediaType_ != other.mediaType_) {
            return mediaType_ < other.mediaType_;
        }
        // mediaType_ 相等时比较 rank_
        return rank_ < other.rank_;
    }

    // 相等运算符重载
    bool operator==(const MmcLocation &other) const
    {
        return rank_ == other.rank_ && mediaType_ == other.mediaType_;
    }

    // 输出流运算符重载
    friend std::ostream &operator<<(std::ostream &os, const MmcLocation &loc)
    {
        os << "loc{rank=" << loc.rank_ << ",media=" << loc.mediaType_ << "}";
        return os;
    }
};
```

**声明位置**: 行 134-160

**功能描述**: 描述内存位置的物理坐标（Rank 和介质类型）

**成员变量**:
- `rank_`: Rank 编号（如 CXL 内存的 Rank ID）
- `mediaType_`: 介质类型（HBM/DRAM/NONE）

**成员函数**:
- `operator<`: 比较运算符，先比较介质类型，再比较 Rank
- `operator==`: 相等判断，两个成员都相等才返回 true
- `operator<<`: 流输出，格式化为 `loc{rank=X,media=Y}`

**使用示例**:
```cpp
MmcLocation loc(5, MediaType::MEDIA_HBM);
std::cout << loc << std::endl;  // 输出: loc{rank=5,media=HBM}
```

---

### MmcLocalMemlInitInfo

```cpp
struct MmcLocalMemlInitInfo {
    uint64_t bmAddr_;    // 位图地址
    uint64_t capacity_;  // 容量
};
```

**声明位置**: 行 162-165

**功能描述**: 本地内存初始化信息

**成员变量**:
- `bmAddr_`: 位图管理的内存地址
- `capacity_`: 内存容量（字节）

---

### MmcOperateIdUnion

```cpp
union MmcOperateIdUnion {
    uint64_t operateId_;  // 完整的操作 ID（64位）
    struct {
        uint32_t sequence_;  // 序列号
        uint32_t rankid_;    // Rank ID
    };
};
```

**声明位置**: 行 167-173

**功能描述**: 操作 ID 的联合体，可以将 64 位操作 ID 拆分为序列号和 Rank ID

**成员**:
- `operateId_`: 完整的 64 位操作 ID
- `sequence_`: 操作序列号
- `rankid_`: Rank 标识符

**使用场景**: 生成和解析全局唯一的操作标识符

---

## 函数

### MoveUp / MoveDown

```cpp
inline MediaType MoveUp(MediaType mediaType)
{
    if (mediaType == MediaType::MEDIA_HBM) {
        return MediaType::MEDIA_NONE;
    } else if (mediaType == MediaType::MEDIA_DRAM) {
        return MediaType::MEDIA_HBM;
    } else {
        return MediaType::MEDIA_NONE;
    }
}

inline MediaType MoveDown(MediaType mediaType)
{
    if (mediaType == MediaType::MEDIA_HBM) {
        return MediaType::MEDIA_DRAM;
    } else if (mediaType == MediaType::MEDIA_DRAM) {
        return MediaType::MEDIA_NONE;
    } else {
        return MediaType::MEDIA_NONE;
    }
}
```

**声明位置**: 行 83-103

**功能描述**: 介质层级移动操作

**参数**:
- `mediaType`: 当前介质类型

**返回值**: 移动后的介质类型

**代码逻辑**:
- `MoveUp`: DRAM -> HBM, HBM -> NONE
- `MoveDown`: HBM -> DRAM, DRAM -> NONE

**层级关系**:
```
     MoveUp
        ^
        |
     HBM <-----> DRAM
        |          |
        v          v
      NONE       NONE
     MoveDown   MoveDown
```

---

### operator<< (MediaType)

```cpp
inline std::ostream &operator<<(std::ostream &os, MediaType type)
{
    switch (type) {
        case MEDIA_DRAM:
            os << "DRAM";
            break;
        case MEDIA_HBM:
            os << "HBM";
            break;
        default:
            os << "UNKNOWN";
            break;
    }
    return os;
}
```

**声明位置**: 行 105-119

**功能描述**: MediaType 的流输出运算符重载

---

### MediumTypeToString

```cpp
inline std::string MediumTypeToString(const MediaType &mediaType)
{
    std::ostringstream oss;
    oss << mediaType;
    return oss.str();
}
```

**声明位置**: 行 121-126

**功能描述**: 将 MediaType 转换为字符串

---

### GenerateOperateId

```cpp
inline uint64_t GenerateOperateId(uint32_t rankid)
{
    static std::atomic<uint32_t> gRequestIdGenerator{0U};
    MmcOperateIdUnion requestUnion{};
    requestUnion.rankid_ = rankid;
    requestUnion.sequence_ = gRequestIdGenerator.fetch_add(1U);
    return requestUnion.operateId_;
}
```

**声明位置**: 行 175-182

**功能描述**: 生成唯一的操作 ID

**参数**:
- `rankid`: Rank 标识符

**返回值**: 64 位唯一操作 ID

**代码逻辑**:
1. 使用静态原子计数器生成序列号
2. 将 rankid 和序列号组合成 64 位 ID
3. 序列号自动递增，保证全局唯一性

---

### GetRankIdByOperateId / GetSequenceByOperateId

```cpp
inline uint64_t GetRankIdByOperateId(uint64_t operateId)
{
    MmcOperateIdUnion requestUnion{};
    requestUnion.operateId_ = operateId;
    return requestUnion.rankid_;
}

inline uint64_t GetSequenceByOperateId(uint64_t operateId)
{
    MmcOperateIdUnion requestUnion{};
    requestUnion.operateId_ = operateId;
    return requestUnion.sequence_;
}
```

**声明位置**: 行 184-196

**功能描述**: 从操作 ID 中提取 Rank ID 或序列号

---

### MmcBufSize

```cpp
inline uint64_t MmcBufSize(const mmc_buffer &buf)
{
    return buf.len;
}
```

**声明位置**: 行 198-201

**功能描述**: 获取缓冲区大小

---

## 类定义

### MmcBufferArray

```cpp
class MmcBufferArray {
public:
    MmcBufferArray() : totalSize_(0) {}

    explicit MmcBufferArray(const std::vector<mmc_buffer> &buffers) : buffers_(buffers)
    {
        totalSize_ = 0;
        for (const auto &buf : buffers_) {
            totalSize_ += MmcBufSize(buf);
        }
    }

    void AddBuffer(const mmc_buffer &buf)
    {
        buffers_.push_back(buf);
        totalSize_ += MmcBufSize(buf);
    }

    const std::vector<mmc_buffer> &Buffers() const
    {
        return buffers_;
    }

    size_t TotalSize() const
    {
        return totalSize_;
    }

private:
    std::vector<mmc_buffer> buffers_{};
    size_t totalSize_{0};
};
```

**声明位置**: 行 203-233

**功能描述**: 管理多个缓冲区的数组，自动计算总大小

**成员变量**:
- `buffers_`: 缓冲区向量
- `totalSize_`: 所有缓冲区的总大小

**成员函数**:
- `MmcBufferArray()`: 默认构造函数
- `MmcBufferArray(const std::vector<mmc_buffer> &)`: 从缓冲区向量构造
- `AddBuffer()`: 添加一个缓冲区
- `Buffers()`: 获取缓冲区向量
- `TotalSize()`: 获取总大小

---

### BatchCopyDesc

```cpp
class BatchCopyDesc {
public:
    std::vector<void *> srcs{};   // 源地址列表
    std::vector<void *> dsts{};   // 目标地址列表
    std::vector<uint64_t> sizes{}; // 大小列表

    void Append(const BatchCopyDesc &desc)
    {
        srcs.insert(srcs.end(), desc.srcs.begin(), desc.srcs.end());
        dsts.insert(dsts.end(), desc.dsts.begin(), desc.dsts.end());
        sizes.insert(sizes.end(), desc.sizes.begin(), desc.sizes.end());
    }

    void Clear()
    {
        srcs.clear();
        dsts.clear();
        sizes.clear();
    }
};
```

**声明位置**: 行 235-255

**功能描述**: 批量复制描述符，用于描述批量内存复制操作

**成员变量**:
- `srcs`: 源地址数组
- `dsts`: 目标地址数组
- `sizes`: 每个复制操作的大小

**成员函数**:
- `Append()`: 追加另一个 BatchCopyDesc 的内容
- `Clear()`: 清空所有内容

---

## 文件级别的关系图

```
mmc_types.h (核心类型定义)
    |
    +-- 错误码: MmcErrorCode
    +-- 介质类型: MediaType, MoveUp/MoveDown
    +-- 位置描述: MmcLocation
    +-- 操作ID: MmcOperateIdUnion, GenerateOperateId
    +-- 缓冲区: MmcBufferArray, BatchCopyDesc
```
