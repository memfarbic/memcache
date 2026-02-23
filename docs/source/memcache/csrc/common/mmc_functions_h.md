# mmc_functions.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_functions.h`
- **文件用途**: 提供通用的工具函数和宏定义
- **依赖项**: `mmc_types.h`, `mmc_logger.h`, `<algorithm>`, `<climits>`, `<string>`, `<sys/stat.h>`

---

## 宏定义

### DL_LOAD_SYM

```cpp
#define DL_LOAD_SYM(TARGET_FUNC_VAR, TARGET_FUNC_TYPE, FILE_HANDLE, SYMBOL_NAME)           \
    do {                                                                                   \
        TARGET_FUNC_VAR = (TARGET_FUNC_TYPE)dlsym(FILE_HANDLE, SYMBOL_NAME);               \
        if ((TARGET_FUNC_VAR) == nullptr) {                                                \
            MMC_LOG_ERROR("Failed to call dlsym to load SYMBOL_NAME, error" << dlerror()); \
            dlclose(FILE_HANDLE);                                                          \
            return MMC_ERROR;                                                              \
        }                                                                                  \
    } while (0)
```

**声明位置**: 行 26-34

**功能描述**: 动态加载符号的宏，用于从动态库中加载函数

**参数**:
- `TARGET_FUNC_VAR`: 目标函数指针变量
- `TARGET_FUNC_TYPE`: 目标函数类型
- `FILE_HANDLE`: 动态库句柄
- `SYMBOL_NAME`: 符号名称

**代码逻辑**:
1. 调用 `dlsym()` 加载符号
2. 检查加载是否成功
3. 失败时记录错误、关闭库并返回错误码

**使用示例**:
```cpp
void* handle = dlopen("libexample.so", RTLD_LAZY);
DL_LOAD_SYM(myFunc, MyFuncType, handle, "my_function");
// 现在可以调用 myFunc()
```

---

## 类定义

### Func

```cpp
class Func {
public:
    static bool Realpath(std::string &path);
    static Result LibraryRealPath(const std::string &libDirPath, const std::string &libName, std::string &realPath);
};
```

**声明位置**: 行 36-55

**功能描述**: 文件路径处理工具类，所有方法都是静态的

---

#### Func::Realpath()

```cpp
static bool Realpath(std::string &path);
```

**声明位置**: 行 44

**功能描述**: 获取路径的真实路径（解析符号链接）

**参数**:
- `path`: [输入/输出] 输入路径，输出为解析后的真实路径

**返回值**: 成功返回 true，失败返回 false

**实现**:
```cpp
inline bool Func::Realpath(std::string &path)
{
    if (path.empty() || path.size() > PATH_MAX) {
        MMC_LOG_ERROR("Failed to get realpath of [" << path << "] as path is invalid");
        return false;
    }

    /* It will allocate memory to store path */
    char *realPath = realpath(path.c_str(), nullptr);
    if (realPath == nullptr) {
        MMC_LOG_ERROR("Failed to get realpath of [" << path << "] as error " << errno);
        return false;
    }

    path = realPath;
    free(realPath);
    realPath = nullptr;
    return true;
}
```

**代码逻辑**:
1. 验证路径不为空且长度不超过 `PATH_MAX`
2. 调用 `realpath()` 解析路径（传入 nullptr 让系统分配内存）
3. 更新输入的 path 参数
4. 释放系统分配的内存

**注意事项**:
- 会解析所有符号链接
- 传入 nullptr 给 `realpath()` 需要 POSIX 兼容的系统
- 调用者需要处理内存释放（函数内部已处理）

---

#### Func::LibraryRealPath()

```cpp
static Result LibraryRealPath(const std::string &libDirPath, const std::string &libName, std::string &realPath);
```

**声明位置**: 行 54

**功能描述**: 获取库文件的真实路径并检查是否存在

**参数**:
- `libDirPath`: 库所在目录路径
- `libName`: 库文件名
- `realPath`: [输出] 库的完整真实路径

**返回值**: 成功返回 `MMC_OK`，失败返回错误码

**实现**:
```cpp
inline Result Func::LibraryRealPath(const std::string &libDirPath, const std::string &libName, std::string &realPath)
{
    std::string tmpFullPath = libDirPath;
    if (!Realpath(tmpFullPath)) {
        MMC_LOG_ERROR("directory is a symlink.");
        return MMC_INVALID_PARAM;
    }

    if (tmpFullPath.back() != '/') {
        tmpFullPath.push_back('/');
    }

    tmpFullPath.append(libName);

    if (!Realpath(tmpFullPath)) {
        MMC_LOG_ERROR("library path is a symlink.");
        return MMC_INVALID_PARAM;
    }

    auto ret = ::access(tmpFullPath.c_str(), F_OK);
    if (ret != 0) {
        MMC_LOG_ERROR(tmpFullPath << " cannot be accessed, ret: " << ret);
        return MMC_ERROR;
    }

    realPath = tmpFullPath;
    return MMC_OK;
}
```

**代码逻辑**:
1. 解析目录路径的真实路径（不能是符号链接）
2. 确保目录以 `/` 结尾
3. 拼接库文件名
4. 解析完整路径的真实路径（不能是符号链接）
5. 检查文件是否可访问
6. 输出真实路径

**注意事项**:
- 拒绝符号链接路径
- 检查文件是否存在且可访问

---

## 全局函数

### ValidatePathNotSymlink()

```cpp
inline int ValidatePathNotSymlink(const char *path)
```

**声明位置**: 行 111-139

**功能描述**: 校验路径存在且不是软链接

**参数**:
- `path`: 要校验的路径

**返回值**: 成功返回 `MMC_OK`，失败返回错误码

**实现**:
```cpp
inline int ValidatePathNotSymlink(const char *path)
{
    struct stat path_stat{};

    if (path == nullptr) {
        MMC_LOG_ERROR("null path");
        return MMC_INVALID_PARAM;
    }

    // 检查路径是否存在
    if (access(path, F_OK) != 0) {
        MMC_LOG_ERROR("path " << path << " does not exist. ");
        return MMC_ERROR;
    }

    // 使用lstat检查是否为软链接
    if (lstat(path, &path_stat) != 0) {
        MMC_LOG_ERROR("lstat failed for path " << path << ", error: " << errno);
        return MMC_ERROR;
    }

    // 检查是否为软链接
    if (S_ISLNK(path_stat.st_mode)) {
        MMC_LOG_ERROR("path " << path << " is a symlink. ");
        return MMC_ERROR;
    }

    return MMC_OK;
}
```

**代码逻辑**:
1. 检查路径是否为 nullptr
2. 使用 `access()` 检查路径是否存在
3. 使用 `lstat()` 获取文件状态（不解引用符号链接）
4. 检查是否为符号链接

**注意事项**:
- 使用 `lstat()` 而非 `stat()`，因为 `stat()` 会跟随符号链接
- `S_ISLNK()` 宏用于判断是否为符号链接

---

### SafeCopy()

```cpp
inline void SafeCopy(const std::string &src, char *dst, const size_t dstSize)
```

**声明位置**: 行 141-146

**功能描述**: 安全地将字符串复制到字符数组，防止缓冲区溢出

**参数**:
- `src`: 源字符串
- `dst`: 目标字符数组
- `dstSize`: 目标缓冲区大小

**实现**:
```cpp
inline void SafeCopy(const std::string &src, char *dst, const size_t dstSize)
{
    const size_t count = std::min(src.length(), dstSize - 1);
    std::copy_n(src.c_str(), count, dst);
    dst[count] = '\0';
}
```

**代码逻辑**:
1. 计算安全复制长度（源长度和目标容量-1的较小值）
2. 复制字符
3. 添加字符串结束符

**注意事项**:
- 总是保证字符串以 `\0` 结尾
- 不会发生缓冲区溢出

---

### SafeGetEnv()

```cpp
inline std::string SafeGetEnv(const char *name) noexcept
```

**声明位置**: 行 148-155

**功能描述**: 安全地获取环境变量

**参数**:
- `name`: 环境变量名称

**返回值**: 环境变量值，不存在时返回空字符串

**实现**:
```cpp
inline std::string SafeGetEnv(const char *name) noexcept
{
    const auto value = std::getenv(name);
    if (value == nullptr) {
        return "";
    }
    return value;
}
```

**代码逻辑**:
1. 调用 `std::getenv()` 获取环境变量
2. 如果返回 nullptr，返回空字符串
3. 否则返回环境变量值

**特性**:
- 使用 `noexcept` 保证不抛出异常
- 空指针安全

---

## 文件级别的关系图

```
mmc_functions.h
    |
    +-- 宏定义
    |   +-- DL_LOAD_SYM  [动态加载符号]
    |
    +-- Func 类
    |   +-- Realpath()           [解析真实路径]
    |   +-- LibraryRealPath()    [获取库文件真实路径]
    |
    +-- 全局函数
    |   +-- ValidatePathNotSymlink()  [验证路径非符号链接]
    |   +-- SafeCopy()                [安全字符串复制]
    |   +-- SafeGetEnv()              [安全获取环境变量]
```

---

## 使用示例

### 解析路径

```cpp
#include "mmc_functions.h"

std::string path = "/usr/local/lib";
if (ock::mmc::Func::Realpath(path)) {
    std::cout << "Real path: " << path << std::endl;
}
```

### 获取库路径

```cpp
std::string realPath;
auto ret = ock::mmc::Func::LibraryRealPath("/usr/lib", "libexample.so", realPath);
if (ret == MMC_OK) {
    std::cout << "Library: " << realPath << std::endl;
}
```

### 验证路径

```cpp
if (ValidatePathNotSymlink("/var/data") != MMC_OK) {
    std::cerr << "Invalid path" << std::endl;
}
```

### 安全复制

```cpp
char buffer[256];
std::string longString = "This is a very long string...";
SafeCopy(longString, buffer, sizeof(buffer));
// buffer 现在包含截断后以 null 结尾的字符串
```

### 获取环境变量

```cpp
std::string configPath = SafeGetEnv("MY_CONFIG_PATH");
if (!configPath.empty()) {
    LoadConfig(configPath);
}
```

---

## 注意事项

1. **Realpath 内存**: `Realpath()` 使用 `realpath(path, nullptr)`，需要 POSIX.1-2008 支持
2. **符号链接**: `LibraryRealPath()` 和 `ValidatePathNotSymlink()` 明确拒绝符号链接
3. **缓冲区安全**: `SafeCopy()` 总是添加 null 终止符
4. **异常安全**: `SafeGetEnv()` 使用 `noexcept` 不会抛出异常
5. **动态加载**: `DL_LOAD_SYM` 宏用于动态库加载，失败时会自动清理资源
