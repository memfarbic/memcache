# mmc_meta_common.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_common.h`
- **文件用途**: 元服务公共头文件，定义元服务类型前向声明
- **依赖项**: `mmc_common_includes.h`

---

## 前向声明

```cpp
namespace ock {
namespace mmc {
class MmcMetaService;
using MmcMetaServicePtr = MmcRef<MmcMetaService>;
} // namespace mmc
} // namespace ock
```

**声明位置**: 行 19-21

**说明**:
- 前向声明 MmcMetaService 类
- 定义 MmcMetaServicePtr 智能指针类型

**用途**:
- 避免头文件循环依赖
- 提供元服务类型的公共声明

---

## 文件级别的关系图

```
mmc_meta_common.h (元服务公共头文件)
    |
    +-- 包含 mmc_common_includes.h
    |
    +-- 前向声明 MmcMetaService
    |
    +-- 被其他元服务头文件包含
```
