# mmc_common_includes.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_common_includes.h`
- **文件用途**: 集中包含 common 模块的常用头文件，作为其他模块的公共包含入口
- **依赖项**: 各种 STL 容器和 common 模块头文件

---

## 包含的标准库头文件

### STL 容器

```cpp
#include <map>                  // 映射容器
#include <set>                  // 集合容器
#include <string>               // 字符串类
#include <cstring>              // C 字符串操作
#include <unordered_map>        // 哈希映射
#include <unordered_set>        // 哈希集合
```

**声明位置**: 行 15-20

**说明**: 包含常用的 STL 容器和字符串处理头文件

---

## 包含的 MMC 头文件

### 核心头文件

```cpp
#include "mmc_define.h"         // 宏定义
#include "mmc_functions.h"      // 工具函数
#include "mmc_last_error.h"     // 错误处理
#include "mmc_logger.h"         // 日志系统
#include "mmc_types.h"          // 类型定义
#include "mmc_ref.h"            // 引用计数智能指针
#include "mmc_spinlock.h"       // 自旋锁
```

**声明位置**: 行 22-28

**说明**: 按依赖顺序包含 common 模块的核心头文件

---

## 文件级别的关系图

```
mmc_common_includes.h
    |
    +-- STL 容器
    |   +-- <map>
    |   +-- <set>
    |   +-- <string>
    |   +-- <cstring>
    |   +-- <unordered_map>
    |   +-- <unordered_set>
    |
    +-- MMC 公共头文件
    |   +-- mmc_define.h      [宏定义]
    |   +-- mmc_functions.h   [工具函数]
    |   +-- mmc_last_error.h  [错误处理]
    |   +-- mmc_logger.h      [日志系统]
    |   +-- mmc_types.h       [类型定义]
    |   +-- mmc_ref.h         [智能指针]
    |   +-- mmc_spinlock.h    [自旋锁]
```

---

## 使用场景

### 1. 作为模块的公共包含

```cpp
// 在某个模块的头文件中使用
#include "mmc_common_includes.h"

class MyCache {
    // 可以直接使用所有包含的类型和工具
    std::map<std::string, MmcRef<Data>> cache_;
    void Put(const std::string& key, const MmcRef<Data>& value);
};
```

### 2. 减少重复包含

```cpp
// 之前需要这样
#include <map>
#include <string>
#include "mmc_define.h"
#include "mmc_logger.h"
#include "mmc_types.h"
// ... 更多包含

// 现在只需要
#include "mmc_common_includes.h"
```

### 3. 确保包含顺序正确

```cpp
// mmc_common_includes.h 已经处理了正确的包含顺序
// 例如：mmc_define.h 必须在 mmc_logger.h 之前包含
// 使用这个文件可以避免手动处理依赖顺序
```

---

## 包含顺序说明

头文件的包含顺序是精心设计的：

1. **STL 头文件**: 先包含标准库
2. **mmc_define.h**: 基础宏定义，其他头文件可能依赖
3. **mmc_functions.h**: 工具函数
4. **mmc_last_error.h**: 错误处理（依赖 mmc_define.h）
5. **mmc_logger.h**: 日志（依赖 mmc_define.h）
6. **mmc_types.h**: 类型定义（依赖其他基础头文件）
7. **mmc_ref.h**: 智能指针（依赖基础类型）
8. **mmc_spinlock.h**: 自旋锁（可以独立）

---

## 注意事项

1. **编译时间**: 包含了较多头文件，可能增加编译时间
2. **依赖传递**: 修改任何一个被包含的头文件都会影响所有包含此文件的源文件
3. **模块化**: 对于只使用部分功能的模块，可以考虑只包含需要的头文件
4. **更新维护**: 添加新的公共头文件时应考虑是否添加到此文件
5. **命名空间**: 包含的头文件主要在 `ock::mmc` 命名空间下

---

## 推荐使用方式

### 场景 1: 模块公共头文件

```cpp
// my_module.h
#pragma once
#include "mmc_common_includes.h"

namespace mymodule {
    using namespace ock::mmc;
    // ... 模块代码
}
```

### 场景 2: 源文件

```cpp
// my_module.cpp
#include "mmc_common_includes.h"
#include "my_module.h"

// 实现
```

### 场景 3: 不推荐的情况

```cpp
// 如果只需要使用少数功能，建议只包含需要的头文件
// 例如只需要日志功能：
#include "mmc_logger.h"  // 而不是包含整个 mmc_common_includes.h
```
