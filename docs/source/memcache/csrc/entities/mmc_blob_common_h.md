# mmc_blob_common.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/entities/mmc_blob_common.h`
- **文件用途**: 定义内存块（Blob）的通用描述结构和类型别名，作为 Blob 相关数据的基础数据结构
- **依赖项**:
  - `iostream` - 标准输入输出流
  - `mmc_ref.h` - 引用计数智能指针

---

## 类型别名

### MmcMemBlobPtr

**声明位置**: 行 20
**完整签名**: `using MmcMemBlobPtr = MmcRef<MmcMemBlob>;`
**功能描述**: `MmcMemBlob` 类型的智能指针别名，用于引用计数管理
**说明**: 使用自定义的 `MmcRef` 智能指针而非 `std::shared_ptr`

---

## 结构体逐个解读

### MmcMemBlobDesc

**声明位置**: 行 22-50
**完整签名**:
```cpp
struct MmcMemBlobDesc {
    uint64_t size_ = 0;               /* data size of the blob */
    uint64_t gva_ = UINT64_MAX;       /* global virtual address */
    uint32_t rank_ = UINT32_MAX;      /* rank id of the blob located */
    uint16_t mediaType_ = UINT16_MAX; /* media type where blob located */

    MmcMemBlobDesc() = default;
    MmcMemBlobDesc(const uint32_t &rank, const uint64_t &gva, const uint64_t &size, const uint16_t &mediaType)
        : rank_(rank), size_(size), gva_(gva), mediaType_(mediaType)
    {}

    friend bool operator==(const MmcMemBlobDesc &lhs, const MmcMemBlobDesc &rhs);
    friend bool operator!=(const MmcMemBlobDesc &lhs, const MmcMemBlobDesc &rhs);
    friend std::ostream &operator<<(std::ostream &os, const MmcMemBlobDesc &blob);
};
```

**功能描述**: 内存块描述符，用于记录 Blob 的位置和大小信息

**成员变量**:
| 变量名 | 类型 | 说明 |
|--------|------|------|
| `size_` | `uint64_t` | Blob 的数据大小（字节） |
| `gva_` | `uint64_t` | 全局虚拟地址 |
| `rank_` | `uint32_t` | Blob 所在的 Rank ID |
| `mediaType_` | `uint16_t` | Blob 所在的介质类型（HBM/DRAM） |

**构造函数**:
- `MmcMemBlobDesc()` - 默认构造函数，所有成员初始化为默认值
- `MmcMemBlobDesc(const uint32_t &rank, const uint64_t &gva, const uint64_t &size, const uint16_t &mediaType)` - 参数化构造函数

**运算符重载**:
- `operator==` - 比较两个描述符是否相等（所有字段相等）
- `operator!=` - 比较两个描述符是否不相等
- `operator<<` - 流输出运算符，用于日志打印

**代码逻辑**:
1. 相等比较通过逐字段比较实现
2. 流输出格式: `blob{size=xxx,gva=xxx,rank=xxx,media=xxx}`

**注意事项**:
- `gva_` 使用 `UINT64_MAX` 作为无效值标记
- `rank_` 使用 `UINT32_MAX` 作为无效值标记
- `mediaType_` 使用 `UINT16_MAX` 作为无效值标记

**使用示例**:
```cpp
// 创建描述符
MmcMemBlobDesc desc(0, 0x1000, 4096, MEDIA_HBM);

// 比较描述符
if (desc1 == desc2) {
    // 两者完全相同
}

// 打印描述符
std::cout << desc << std::endl;
// 输出: blob{size=4096,gva=4096,rank=0,media=0}
```

---

## 数据结构关系图

```
┌─────────────────────────────────────────┐
│           MmcMemBlobDesc                │
├─────────────────────────────────────────┤
│ + size_: uint64_t        (数据大小)      │
│ + gva_: uint64_t         (全局虚拟地址)  │
│ + rank_: uint32_t        (Rank ID)      │
│ + mediaType_: uint16_t   (介质类型)      │
├─────────────────────────────────────────┤
│ + MmcMemBlobDesc()                     │
│ + MmcMemBlobDesc(rank, gva, size, type)│
│ + operator==(lhs, rhs): bool           │
│ + operator!=(lhs, rhs): bool           │
│ + operator<<(os, blob): ostream&       │
└─────────────────────────────────────────┘
                    │
                    │ 使用
                    ▼
┌─────────────────────────────────────────┐
│         MmcMemBlobPtr                   │
│      (MmcRef<MmcMemBlob>)               │
└─────────────────────────────────────────┘
```

---

## 使用示例

### 示例 1: 创建 Blob 描述符

```cpp
#include "mmc_blob_common.h"

using namespace ock::mmc;

// 使用默认构造
MmcMemBlobDesc desc1;

// 使用参数化构造
MmcMemBlobDesc desc2(0,    // rank
                     0x1000000,  // gva
                     1024,   // size
                     MEDIA_HBM);  // mediaType
```

### 示例 2: 描述符比较

```cpp
MmcMemBlobDesc desc1(0, 0x1000, 4096, MEDIA_HBM);
MmcMemBlobDesc desc2(0, 0x1000, 4096, MEDIA_HBM);
MmcMemBlobDesc desc3(1, 0x2000, 4096, MEDIA_DRAM);

if (desc1 == desc2) {
    std::cout << "desc1 and desc2 are identical" << std::endl;
}

if (desc1 != desc3) {
    std::cout << "desc1 and desc3 are different" << std::endl;
}
```

### 示例 3: 日志输出

```cpp
MmcMemBlobDesc desc(0, 0x1000, 4096, MEDIA_HBM);
MMC_LOG_INFO("Created blob: " << desc);
// 输出: blob{size=4096,gva=4096,rank=0,media=0}
```
