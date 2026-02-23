# mmc_kv_parser.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/config/mmc_kv_parser.cpp`
- **文件用途**: KVParser 类的实现，提供键值对配置文件的解析功能
- **依赖项**:
  - `<linux/limits.h>` - PATH_MAX 常量
  - `mf_file_util.h` - 文件工具类
  - `mmc_functions.h` - 工具函数
  - `common/mmc_functions.h` - 通用工具函数

---

## 局部常量定义

**声明位置**: 行 24-26
```cpp
constexpr int MAX_CONF_FILE_SIZE = 10485760; // 10MB
constexpr int MAX_LINE_NUMBER = 10000;
constexpr uint32_t MAX_LINE_LENGTH = 4096;
```
**功能描述**: 配置文件解析的限制常量
- `MAX_CONF_FILE_SIZE`: 配置文件最大 10MB
- `MAX_LINE_NUMBER`: 最大行数 10000
- `MAX_LINE_LENGTH`: 单行最大长度 4096 字符

---

## 类实现

### KVParser::KVParser()

**声明位置**: 行 28
**完整签名**:
```cpp
KVParser::KVParser() = default;
```
**功能描述**: 默认构造函数实现
**注意事项**: 使用编译器默认实现，不执行特殊初始化

---

### KVParser::~KVParser()

**声明位置**: 行 29-41
**完整签名**:
```cpp
KVParser::~KVParser()
```
**功能描述**: 析构函数实现，释放所有动态分配的资源

**代码逻辑**:
1. 创建 `GUARD` 守卫获取锁（行 32）
2. 遍历 `mItems` 向量（行 33）
3. 对每个 `KvPair` 指针调用 `SAFE_DELETE` 删除并置空（行 35）
4. 清空 `mItems` 和 `mItemsIndex` 容器（行 38-39）

**注意事项**:
- 使用 RAII 锁守卫确保异常安全
- 使用 `SAFE_DELETE` 宏确保指针置空，防止悬空指针

---

### KVParser::FromFile()

**声明位置**: 行 43-82
**完整签名**:
```cpp
Result KVParser::FromFile(const std::string &filePath)
```
**功能描述**: 从文件加载并解析配置

**参数**:
- `filePath`: 配置文件的路径

**返回值**:
- `MMC_OK` - 成功加载并解析
- `MMC_ERROR` - 文件无效、过大或解析失败

**代码逻辑**:
1. **路径验证** (行 45-53)
   - 检查路径长度不超过 `PATH_MAX`
   - 使用 `realpath()` 获取规范化路径
   - 调用 `ValidatePathNotSymlink()` 确保不是符号链接

2. **文件大小检查** (行 55-58)
   - 使用 `FileUtil::CheckFileSize()` 验证文件不超过 10MB

3. **逐行解析** (行 60-76)
   - 打开文件流（行 60）
   - 使用 `getline()` 逐行读取（行 64）
   - 调用 `ParseLine()` 解析每行（行 65）
   - 检查行数不超过限制（行 71-75）

4. **清理** (行 78-79)
   - 关闭文件流
   - 清除文件流状态

**注意事项**:
- 路径必须存在且不是符号链接（安全考虑）
- 文件大小和行数限制防止资源耗尽攻击
- 部分解析失败时仍会关闭文件

---

### KVParser::GetItem()

**声明位置**: 行 84-101
**完整签名**:
```cpp
Result KVParser::GetItem(const std::string &key, std::string &outValue)
```
**功能描述**: 根据键名获取配置值

**参数**:
- `key`: 配置项键名
- `outValue`: 输出参数，存储找到的值

**返回值**:
- `MMC_OK` - 找到配置项
- `MMC_ERROR` - 未找到或索引无效

**代码逻辑**:
1. 创建 `GUARD` 获取锁（行 86）
2. 在 `mItemsIndex` 中查找键（行 87）
3. 如果找到，检查索引有效性（行 88-92）
4. 获取对应的 `KvPair` 并返回值（行 93-98）
5. 未找到返回错误（行 100）

**注意事项**:
- 线程安全：所有访问都在锁保护下
- 包含索引越界检查，防止数据不一致

---

### KVParser::SetItem()

**声明位置**: 行 103-124
**完整签名**:
```cpp
Result KVParser::SetItem(const std::string &key, const std::string &value)
```
**功能描述**: 设置一个配置项（内部方法）

**参数**:
- `key`: 配置项键名
- `value`: 配置项值

**返回值**:
- `MMC_OK` - 设置成功
- `MMC_ERROR` - 键重复或内存分配失败

**代码逻辑**:
1. 创建 `GUARD` 获取锁（行 105）
2. 检查键是否已存在（行 106-110）
   - 如果存在，输出错误并返回
3. 使用 `std::nothrow` 分配新的 `KvPair`（行 111-115）
4. 设置键值（行 116-117）
5. 添加到 `mItems` 向量并记录索引（行 118-119）
6. 在 `mGotKeys` 中标记该键（行 120-122）

**注意事项**:
- 禁止重复键，保证配置唯一性
- 内存分配失败时返回错误，不抛出异常
- 在 `mGotKeys` 中记录键，供 `CheckSet()` 使用

---

### KVParser::Size()

**声明位置**: 行 126-130
**完整签名**:
```cpp
uint32_t KVParser::Size()
```
**功能描述**: 获取配置项数量
**返回值**: `mItems` 的大小

---

### KVParser::GetI()

**声明位置**: 行 132-144
**完整签名**:
```cpp
void KVParser::GetI(const uint32_t index, std::string &outKey, std::string &outValue)
```
**功能描述**: 按索引获取键值对

**参数**:
- `index`: 配置项索引
- `outKey`: 输出键名
- `outValue`: 输出配置值

**代码逻辑**:
1. 创建 `GUARD` 获取锁（行 134）
2. 检查索引有效性（行 135-137）
3. 检查指针有效性（行 138-141）
4. 返回键和值（行 142-143）

**注意事项**:
- 索引无效时静默返回，不输出错误
- 指针为空时静默返回

---

### KVParser::Dump()

**声明位置**: 行 146-156
**完整签名**:
```cpp
void KVParser::Dump()
```
**功能描述**: 打印所有配置项到标准输出

**代码逻辑**:
1. 创建 `GUARD` 获取锁（行 148）
2. 遍历所有配置项（行 149）
3. 检查指针有效性（行 151-153）
4. 使用 `printf` 输出 `key = value` 格式（行 154）

**注意事项**:
- 跳过空指针，避免崩溃
- 输出格式：`key = value\n`

---

### KVParser::CheckSet()

**声明位置**: 行 158-169
**完整签名**:
```cpp
bool KVParser::CheckSet(const std::vector<std::string> &keys)
```
**功能描述**: 检查指定的键是否都在配置中设置过

**参数**:
- `keys`: 需要检查的键名列表

**返回值**:
- `true` - 所有键都已设置
- `false` - 存在未设置的键

**代码逻辑**:
1. 初始化检查结果为 `true`（行 160）
2. 遍历所有待检查的键（行 161-166）
   - 如果键不在 `mGotKeys` 中，输出错误并设为 `false`
3. 清空 `mGotKeys`（行 167）
4. 返回检查结果（行 168）

**注意事项**:
- 调用后会清空 `mGotKeys`，无法重复检查
- 未设置的键会输出到 `stderr`

---

### KVParser::ParseLine()

**声明位置**: 行 171-203
**完整签名**:
```cpp
Result KVParser::ParseLine(std::string &strLine)
```
**功能描述**: 解析单行配置内容（私有方法）

**参数**:
- `strLine`: 待解析的行内容（引用传递）

**返回值**:
- `MMC_OK` - 解析成功或跳过（空行/注释）
- `MMC_ERROR` - 格式错误

**代码逻辑**:
1. **预处理** (行 173-177)
   - 移除 `\r` 字符（Windows 换行符）
   - 调用 `OckTrimString()` 去除首尾空白
   - 跳过空行和以 `#` 开头的注释行

2. **长度检查** (行 178-181)
   - 检查行长度不超过 `MAX_LINE_LENGTH`

3. **解析键值对** (行 183-189)
   - 查找 `=` 分隔符
   - 如果没有 `=`，跳过该行
   - 分割键和值

4. **后处理** (行 190-200)
   - 去除键和值的首尾空白
   - 检查键非空
   - 调用 `SetItem()` 存储配置项

**注意事项**:
- 支持 Windows 和 Unix 换行符
- 注释行和空行被静默跳过
- 没有等号的行被忽略（可能用于格式分隔）

---

## 辅助函数调用

### OckTrimString()
**来源**: `mmc_functions.h`
**功能**: 去除字符串首尾的空白字符（空格、制表符、换行符）

### ValidatePathNotSymlink()
**来源**: `common/mmc_functions.h`
**功能**: 验证路径存在且不是符号链接

### FileUtil::CheckFileSize()
**来源**: `mf_file_util.h`
**功能**: 检查文件大小不超过指定值

---

## 配置文件解析流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                       FromFile(filePath)                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  路径有效性检查       │
              │  - realpath()        │
              │  - 非符号链接检查     │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  文件大小检查         │
              │  MAX_CONF_FILE_SIZE  │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  逐行读取文件         │
              │  while(getline)      │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │   ParseLine()        │
              └──────────┬───────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ 空行/   │    │ 注释行  │    │ 键值对  │
   │ 跳过    │    │ 跳过    │    │ SetItem │
   └─────────┘    └─────────┘    └────┬────┘
                                      │
                                      ▼
                              ┌──────────────┐
                              │  检查重复键  │
                              └──────┬───────┘
                                     │
                                     ▼
                              ┌──────────────┐
                              │  存储到容器  │
                              └──────────────┘
```

---

## 内存管理

```
┌─────────────────────────────────────────────────────────────────┐
│                      KVParser 内存布局                           │
├─────────────────────────────────────────────────────────────────┤
│  mItems (vector<KvPair*>)                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  [0] ────▶ KvPair { "key1", "value1" }  (堆分配)          │   │
│  │  [1] ────▶ KvPair { "key2", "value2" }  (堆分配)          │   │
│  │  [2] ────▶ KvPair { "key3", "value3" }  (堆分配)          │   │
│  │  ...                                                     │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  mItemsIndex (map<string, uint32_t>)                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  "key1" ▶ 0                                               │   │
│  │  "key2" ▶ 1                                               │   │
│  │  "key3" ▶ 2                                               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  析构时：遍历 mItems，对每个 KvPair* 调用 SAFE_DELETE             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 使用示例

```cpp
#include "mmc_kv_parser.h"

using namespace ock::mmc;

int main() {
    // 创建解析器
    KVParser parser;

    // 加载配置文件
    Result result = parser.FromFile("/etc/memcache/mmc.conf");
    if (result != MMC_OK) {
        std::cerr << "Failed to load config file" << std::endl;
        return 1;
    }

    // 获取特定配置
    std::string logLevel;
    if (parser.GetItem("ock.mmc.log_level", logLevel) == MMC_OK) {
        std::cout << "Log Level: " << logLevel << std::endl;
    }

    // 遍历所有配置
    for (uint32_t i = 0; i < parser.Size(); ++i) {
        std::string key, value;
        parser.GetI(i, key, value);
        std::cout << key << " = " << value << std::endl;
    }

    // 验证必需配置
    std::vector<std::string> required = {
        "ock.mmc.log_level",
        "ock.mmc.meta_service_url"
    };
    if (!parser.CheckSet(required)) {
        std::cerr << "Missing required config" << std::endl;
        return 1;
    }

    return 0;
}
```
