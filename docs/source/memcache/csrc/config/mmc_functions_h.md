# mmc_functions.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/config/mmc_functions.h`
- **文件用途**: 定义配置模块的工具函数，包括字符串处理、类型转换和路径验证
- **依赖项**:
  - `<climits>` - 整数类型限制
  - `<cmath>` - 数学函数
  - `<cstring>` - 字符串处理
  - `<fstream>` - 文件流
  - `<set>` - 集合容器
  - `<string>` - 字符串类
  - `<random>` - 随机数生成
  - `<vector>` - 动态数组
  - `<unordered_map>` - 哈希映射
  - `<algorithm>` - 算法

---

## 常量定义

**声明位置**: 行 28-29
```cpp
constexpr float EPSINON = 0.000001;
constexpr int DECIMAL_DIGITS = 10;
```
**功能描述**:
- `EPSINON`: 浮点数比较的误差范围
- `DECIMAL_DIGITS`: 字符串转数字的十进制位数

---

## 字符串处理函数

### OckTrimString()

**声明位置**: 行 31
**完整签名**:
```cpp
void OckTrimString(std::string &str);
```
**功能描述**: 去除字符串首尾的空白字符（空格、制表符、换行符）
**参数**:
- `str`: 引用传递，将被原地修改
**返回值**: 无
**注意事项**:
- 修改原字符串
- 空字符串不受影响
- 实现在 `mmc_functions.cpp` 中

**使用示例**:
```cpp
std::string s = "  hello world  \n";
OckTrimString(s);  // s 变为 "hello world"
```

---

### SplitStr() - set 重载

**声明位置**: 行 34
**完整签名**:
```cpp
void SplitStr(const std::string &str, const std::string &separator, std::set<std::string> &result);
```
**功能描述**: 使用指定分隔符分割字符串，结果存储到有序集合中
**参数**:
- `str`: 待分割的字符串
- `separator`: 分隔符字符串
- `result`: 输出参数，存储分割后的结果（使用 set 自动去重和排序）
**返回值**: 无

**使用示例**:
```cpp
std::string str = "read||write||execute";
std::set<std::string> result;
SplitStr(str, "||", result);
// result = {"execute", "read", "write"} (按字母排序)
```

---

### SplitStr() - vector 重载

**声明位置**: 行 35
**完整签名**:
```cpp
void SplitStr(const std::string &str, const std::string &separator, std::vector<std::string> &result);
```
**功能描述**: 使用指定分隔符分割字符串，结果存储到向量中
**参数**:
- `str`: 待分割的字符串
- `separator`: 分隔符字符串
- `result`: 输出参数，存储分割后的结果（保持原顺序）
**返回值**: 无

**使用示例**:
```cpp
std::string str = "read|write|execute";
std::vector<std::string> result;
SplitStr(str, "|", result);
// result = ["read", "write", "execute"]
```

---

## 类型转换函数

### OckStol()

**声明位置**: 行 37-49
**完整签名**:
```cpp
inline bool OckStol(const std::string &str, long &value)
```
**功能描述**: 安全地将字符串转换为 long 类型整数
**参数**:
- `str`: 输入字符串
- `value`: 输出参数，存储转换结果
**返回值**:
- `true` - 转换成功
- `false` - 转换失败

**代码逻辑**:
1. 使用 `std::strtol()` 进行转换（行 41）
2. 检查转换后的剩余字符（行 42）
3. 检查是否溢出（行 43）
4. 检查特殊情况：值为0但字符串不是"0"（行 45-46）

**错误条件**:
- 字符串包含非数字字符
- 转换结果溢出（`LONG_MAX` 或 `LONG_MIN`）
- 值为0但输入不是 "0"

**使用示例**:
```cpp
long value;
if (OckStol("12345", value)) {
    std::cout << "Value: " << value << std::endl;  // 12345
}
if (!OckStol("abc", value)) {
    std::cout << "Conversion failed" << std::endl;
}
```

---

### OckStoULL()

**声明位置**: 行 51-63
**完整签名**:
```cpp
inline bool OckStoULL(const std::string &str, uint64_t &value)
```
**功能描述**: 安全地将字符串转换为 uint64_t 类型整数
**参数**:
- `str`: 输入字符串
- `value`: 输出参数，存储转换结果
**返回值**:
- `true` - 转换成功
- `false` - 转换失败

**代码逻辑**:
1. 使用 `std::strtoull()` 进行转换（行 55）
2. 检查转换后的剩余字符（行 56）
3. 检查是否溢出（行 56）
4. 检查特殊情况：值为0但字符串不是"0"（行 59-60）

**注意事项**:
- 使用 `DECIMAL_DIGITS` (10) 作为进制基数
- 不接受负数（uint64_t 是无符号类型）

---

### OckStof()

**声明位置**: 行 65-75
**完整签名**:
```cpp
inline bool OckStof(const std::string &str, float &value)
```
**功能描述**: 安全地将字符串转换为 float 类型浮点数
**参数**:
- `str`: 输入字符串
- `value`: 输出参数，存储转换结果
**返回值**:
- `true` - 转换成功
- `false` - 转换失败

**代码逻辑**:
1. 使用 `std::strtof()` 进行转换（行 68）
2. 检查是否溢出为 `HUGE_VALF`（行 69）
3. 检查值为0但输入不是 "0.0"（行 71-72）

**注意事项**:
- 使用 `EPSINON` 进行浮点数比较

---

## 布尔类型处理

### Str2Bool

**声明位置**: 行 77
**完整签名**:
```cpp
const std::unordered_map<std::string, bool> Str2Bool{{"0", false}, {"1", true}, {"false", false}, {"true", true}};
```
**功能描述**: 全局常量映射，定义字符串到布尔值的转换规则
**支持的字符串**:
- `"0"` → `false`
- `"1"` → `true`
- `"false"` → `false`
- `"true"` → `true`

---

### IsBool()

**声明位置**: 行 79-88
**完整签名**:
```cpp
inline bool IsBool(const std::string &str, bool &value)
```
**功能描述**: 判断字符串是否为布尔值并转换
**参数**:
- `str`: 输入字符串
- `value`: 输出参数，存储转换后的布尔值
**返回值**:
- `true` - 是有效的布尔字符串
- `false` - 不是有效的布尔字符串

**代码逻辑**:
1. 将输入字符串转为小写（行 81-82）
2. 在 `Str2Bool` 映射中查找（行 83）
3. 如果找到，设置输出值并返回 true（行 84-86）
4. 如果未找到，返回 false（行 84-85）

**使用示例**:
```cpp
bool value;
if (IsBool("True", value)) {  // 大小写不敏感
    std::cout << "Value: " << std::boolalpha << value << std::endl;  // true
}
if (IsBool("yes", value)) {
    // 不会执行，"yes" 不是有效的布尔字符串
}
```

---

## 路径处理函数

### GetRealPath()

**声明位置**: 行 90-98
**完整签名**:
```cpp
inline bool GetRealPath(std::string &path)
```
**功能描述**: 将路径转换为绝对路径（解析符号链接和相对路径）
**参数**:
- `path`: 引用传递，输入原路径，输出转换后的绝对路径
**返回值**:
- `true` - 转换成功
- `false` - 转换失败

**代码逻辑**:
1. 检查路径长度不超过 `PATH_MAX`（行 93）
2. 调用 `realpath()` 获取规范化的绝对路径（行 93）
3. 检查 `realpath()` 返回值（行 93-94）
4. 更新输入路径参数（行 96）

**注意事项**:
- `realpath()` 会解析符号链接到实际路径
- 路径不存在时会失败

**使用示例**:
```cpp
std::string path = "../config/file.conf";
if (GetRealPath(path)) {
    std::cout << "Real path: " << path << std::endl;
    // 输出: /home/user/project/config/file.conf
}
```

---

## 函数使用示例

### 字符串处理组合示例

```cpp
#include "mmc_functions.h"

using namespace ock::mmc;

void ProcessConfigLine(const std::string& line) {
    // 1. 去除首尾空白
    std::string trimmed = line;
    OckTrimString(trimmed);

    // 2. 分割键值对
    size_t pos = trimmed.find('=');
    if (pos != std::string::npos) {
        std::string key = trimmed.substr(0, pos);
        std::string value = trimmed.substr(pos + 1);

        // 3. 去除键和值的空白
        OckTrimString(key);
        OckTrimString(value);

        // 4. 类型转换
        long intValue;
        if (OckStol(value, intValue)) {
            std::cout << key << " = " << intValue << " (int)" << std::endl;
        }

        bool boolValue;
        if (IsBool(value, boolValue)) {
            std::cout << key << " = " << std::boolalpha << boolValue << " (bool)" << std::endl;
        }
    }
}
```

### 枚举值验证示例

```cpp
void ValidateEnumValue(const std::string& value) {
    // 分割枚举定义
    std::string enumDef = "read||write||execute";
    std::set<std::string> validValues;
    SplitStr(enumDef, "||", validValues);
    // validValues = {"execute", "read", "write"}

    // 分割输入值（支持多值）
    std::vector<std::string> inputValues;
    SplitStr(value, "|", inputValues);

    // 验证每个输入值
    for (const auto& val : inputValues) {
        OckTrimString(const_cast<std::string&>(val));
        if (validValues.find(val) != validValues.end()) {
            std::cout << val << " is valid" << std::endl;
        } else {
            std::cout << val << " is invalid" << std::endl;
        }
    }
}
```

---

## 函数对比表

| 函数 | 输入 | 输出 | 错误处理 |
|-----|------|------|---------|
| `OckTrimString()` | string | 原地修改 | 空字符串安全 |
| `SplitStr(set)` | string, separator | set (去重/排序) | 空分隔符安全 |
| `SplitStr(vector)` | string, separator | vector (保持顺序) | 空分隔符安全 |
| `OckStol()` | string | long | 返回 bool |
| `OckStoULL()` | string | uint64_t | 返回 bool |
| `OckStof()` | string | float | 返回 bool |
| `IsBool()` | string | bool | 返回 bool |
| `GetRealPath()` | string (引用) | 原地修改为绝对路径 | 返回 bool |

---

## 类型转换支持总结

```
字符串类型转换支持:
┌─────────────────────────────────────────────────────────────────┐
│                     输入字符串                                   │
├─────────────────────────────────────────────────────────────────┤
│  "123"          ──▶  OckStol()    ──▶  long (123)               │
│  "123456"       ──▶  OckStoULL()  ──▶  uint64_t (123456)        │
│  "1.23"         ──▶  OckStof()    ──▶  float (1.23)            │
│  "true/false"   ──▶  IsBool()     ──▶  bool                     │
│  "1/0"          ──▶  IsBool()     ──▶  bool                     │
│  "/path/to"     ──▶  GetRealPath() ──▶  string (绝对路径)       │
└─────────────────────────────────────────────────────────────────┘
```
