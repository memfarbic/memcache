# mmc_version.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_version.h`
- **文件用途**: 定义版本信息相关的宏和全局版本字符串
- **依赖项**: 无（基础头文件）

---

## 宏定义

### CONCAT / CONCAT2

```cpp
#define CONCAT(x, y, z)  x.##y.##z
#define STR(x)           #x
#define CONCAT2(x, y, z) CONCAT(x, y, z)
#define STR2(x)          STR(x)
```

**声明位置**: 行 18-22

**功能描述**: 用于构建版本字符串的宏展开技巧

**说明**:
- `CONCAT(x, y, z)`: 将三个 token 连接成 `x.y.z` 格式
- `STR(x)`: 将 token 转换为字符串字面量
- `CONCAT2` 和 `STR2`: 用于二次展开，确保宏参数被正确展开

**宏展开示例**:
```cpp
// 假设 VERSION_MAJOR=1, VERSION_MINOR=2, VERSION_FIX=3
CONCAT2(VERSION_MAJOR, VERSION_MINOR, VERSION_FIX)
// 展开为: CONCAT(1, 2, 3)
// 再展开为: 1.2.3

STR2(CONCAT2(VERSION_MAJOR, VERSION_MINOR, VERSION_FIX))
// 最终展开为: "1.2.3"
```

---

### MMC_VERSION

```cpp
#define MMC_VERSION STR2(CONCAT2(VERSION_MAJOR, VERSION_MINOR, VERSION_FIX))
```

**声明位置**: 行 25

**功能描述**: 版本号字符串宏

**说明**:
- 依赖外部定义的 `VERSION_MAJOR`, `VERSION_MINOR`, `VERSION_FIX` 宏
- 展开为版本号字符串（如 "1.2.3"）

**注意**: 这些宏通常在编译选项中定义（如 `-DVERSION_MAJOR=1`）

---

### GIT_LAST_COMMIT

```cpp
#ifndef GIT_LAST_COMMIT
#define GIT_LAST_COMMIT empty
#endif
```

**声明位置**: 行 27-29

**功能描述**: Git 提交哈希宏

**说明**:
- 如果未定义，默认值为 "empty"
- 通常通过构建系统注入（如 `-DGIT_LAST_COMMIT="abc123"`）

---

## 全局变量

### LIB_VERSION

```cpp
static const char *LIB_VERSION =
    "library version: " MMC_VERSION ", build time: " __DATE__ " " __TIME__ ", commit: " STR2(GIT_LAST_COMMIT);
```

**声明位置**: 行 34-35

**功能描述**: 包含完整版本信息的全局字符串常量

**内容组成部分**:
- `MMC_VERSION`: 版本号（如 1.2.3）
- `__DATE__`: 编译日期（如 "Feb 24 2026"）
- `__TIME__`: 编译时间（如 "14:30:00"）
- `GIT_LAST_COMMIT`: Git 提交哈希

**输出示例**:
```
library version: 1.2.3, build time: Feb 24 2026 14:30:00, commit: abc123def
```

**说明**:
- `__DATE__` 和 `__TIME__` 是编译器预定义宏
- `static const char *` 表示内部链接，每个编译单元有独立副本
- 使用 C 链接（在 `extern "C"` 块内）

---

## 文件级别的关系图

```
mmc_version.h
    |
    +-- 宏定义
    |   +-- CONCAT / CONCAT2  [token 连接]
    |   +-- STR / STR2        [token 转字符串]
    |   +-- MMC_VERSION       [版本号字符串]
    |
    +-- 全局常量
    |   +-- LIB_VERSION  [完整版本信息字符串]
```

---

## 使用示例

### 打印版本信息

```cpp
#include "mmc_version.h"

void PrintVersion() {
    std::cout << LIB_VERSION << std::endl;
    // 输出: library version: 1.2.3, build time: Feb 24 2026 14:30:00, commit: abc123
}
```

### 从 C 代码访问

```c
#include "mmc_version.h"

extern const char *LIB_VERSION;

void show_version(void) {
    printf("%s\n", LIB_VERSION);
}
```

### 获取版本号组件

```cpp
// 假设构建时定义了这些宏
#ifdef VERSION_MAJOR
    std::cout << "Major: " << VERSION_MAJOR << std::endl;
#endif

#ifdef VERSION_MINOR
    std::cout << "Minor: " << VERSION_MINOR << std::endl;
#endif

#ifdef VERSION_FIX
    std::cout << "Fix: " << VERSION_FIX << std::endl;
#endif
```

---

## 构建系统集成

### CMake 配置示例

```cmake
# 从 Git 获取提交哈希
execute_process(
    COMMAND git rev-parse --short HEAD
    OUTPUT_VARIABLE GIT_COMMIT_HASH
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

# 定义版本宏
set(VERSION_MAJOR 1)
set(VERSION_MINOR 2)
set(VERSION_FIX 3)

# 添加编译定义
target_compile_definitions(mmc PRIVATE
    VERSION_MAJOR=${VERSION_MAJOR}
    VERSION_MINOR=${VERSION_MINOR}
    VERSION_FIX=${VERSION_FIX}
    GIT_LAST_COMMIT="${GIT_COMMIT_HASH}"
)
```

### Makefile 示例

```makefile
VERSION_MAJOR = 1
VERSION_MINOR = 2
VERSION_FIX = 3

GIT_COMMIT = $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")

CFLAGS += -DVERSION_MAJOR=$(VERSION_MAJOR)
CFLAGS += -DVERSION_MINOR=$(VERSION_MINOR)
CFLAGS += -DVERSION_FIX=$(VERSION_FIX)
CFLAGS += -DGIT_LAST_COMMIT=\"$(GIT_COMMIT)\"
```

---

## 注意事项

1. **外部依赖**: `VERSION_MAJOR`, `VERSION_MINOR`, `VERSION_FIX` 必须由构建系统定义
2. **编译时常量**: `__DATE__` 和 `__TIME__` 是编译时的值，不是运行时
3. **C 兼容**: 使用 `extern "C"` 确保可从 C 代码访问
4. **内部链接**: `static const char *` 表示每个编译单元独立副本
5. **宏展开顺序**: 使用 `STR2` 和 `CONCAT2` 确保正确的宏展开顺序

---

## 调试版本问题

### 检查宏是否正确定义

```cpp
#include <iostream>

#ifdef VERSION_MAJOR
    #warning VERSION_MAJOR is defined
#else
    #error VERSION_MAJOR is not defined
#endif

int main() {
    std::cout << MMC_VERSION << std::endl;
    return 0;
}
```
