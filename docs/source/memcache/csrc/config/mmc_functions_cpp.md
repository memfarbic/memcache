# mmc_functions.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/config/mmc_functions.cpp`
- **文件用途**: 配置模块工具函数的实现
- **依赖项**:
  - `<string>` - 字符串类
  - `mmc_functions.h` - 对应的头文件

---

## 函数实现

### OckTrimString()

**声明位置**: 行 19-27
**完整签名**:
```cpp
void OckTrimString(std::string &str)
```
**功能描述**: 去除字符串首尾的空白字符
**代码逻辑**:
1. **空字符串检查** (行 21-23)
   - 如果字符串为空，直接返回

2. **去除首部空白** (行 25)
   - 使用 `find_first_not_of(" \t\n\r")` 查找第一个非空白字符
   - 使用 `erase(0, pos)` 删除首部空白

3. **去除尾部空白** (行 26)
   - 使用 `find_last_not_of(" \t\n\r")` 查找最后一个非空白字符
   - 使用 `erase(pos + 1)` 删除尾部空白

**注意事项**:
- 修改原字符串
- 处理的空白字符包括：空格、制表符、换行符、回车符

**执行示例**:
```
输入: "  \t  hello world \n\r  "
     │              │
     ▼              ▼
  首部空白        尾部空白

步骤1: find_first_not_of → 返回 3 (指向 'h')
步骤2: erase(0, 3) → "hello world \n\r  "
步骤3: find_last_not_of → 返回 11 (指向 'd')
步骤4: erase(12) → "hello world"

输出: "hello world"
```

---

### SplitStr() - set 重载

**声明位置**: 行 29-58
**完整签名**:
```cpp
void SplitStr(const std::string &str, const std::string &separator, std::set<std::string> &result)
```
**功能描述**: 使用指定分隔符分割字符串，结果存储到有序集合中
**代码逻辑**:
1. **空分隔符处理** (行 31-37)
   - 如果分隔符为空
   - 如果字符串非空，将整个字符串插入到结果集
   - 返回

2. **查找分隔符** (行 38-39)
   - `pos1`: 当前搜索起始位置
   - `pos2`: 分隔符首次出现位置

3. **循环分割** (行 42-50)
   - 当找到分隔符时（行 42）
   - 提取子串 `str.substr(pos1, pos2 - pos1)`（行 43）
   - 调用 `OckTrimString()` 去除子串首尾空白（行 45）
   - 将处理后的子串插入到 set（行 46）
   - 更新搜索位置（行 48）

4. **处理最后一段** (行 52-57)
   - 如果还有剩余字符（pos1 不是末尾）
   - 提取剩余部分并处理
   - 插入到结果集

**注意事项**:
- 使用 set 自动去重和排序
- 每个分割后的子串都会去除首尾空白

**执行示例**:
```
输入: str = "read||write||execute||read"
      separator = "||"

循环过程:
┌──────┬────────┬────────┬─────────┬────────────────────┐
│ 轮次 │ pos1   │ pos2   │ 子串    │ 结果集             │
├──────┼────────┼────────┼─────────┼────────────────────┤
│ 1    │ 0      │ 4      │ "read"  │ {"read"}           │
│ 2    │ 6      │ 12     │ "write" │ {"read", "write"}  │
│ 3    │ 14     │ 22     │ "execute"│ {"execute", "read", "write"}│
│ 4    │ 24     │ npos   │ "read"  │ 重复，自动忽略      │
└──────┴────────┴────────┴─────────┴────────────────────┘

输出: {"execute", "read", "write"} (按字母排序)
```

---

### SplitStr() - vector 重载

**声明位置**: 行 60-86
**完整签名**:
```cpp
void SplitStr(const std::string &str, const std::string &separator, std::vector<std::string> &result)
```
**功能描述**: 使用指定分隔符分割字符串，结果存储到向量中
**代码逻辑**:
1. **空分隔符处理** (行 62-68)
   - 与 set 重载类似
   - 使用 `emplace_back()` 添加元素

2. **循环分割** (行 69-79)
   - 与 set 重载逻辑相同
   - 使用 `emplace_back()` 而非 `insert()`

3. **处理最后一段** (行 81-85)
   - 与 set 重载逻辑相同

**注意事项**:
- 保持原始顺序
- 不会去重
- 每个分割后的子串都会去除首尾空白

**执行示例**:
```
输入: str = "read|write|execute"
      separator = "|"

循环过程:
┌──────┬────────┬────────┬─────────┬────────────────────────┐
│ 轮次 │ pos1   │ pos2   │ 子串    │ vector                  │
├──────┼────────┼────────┼─────────┼────────────────────────┤
│ 1    │ 0      │ 4      │ "read"  │ ["read"]                │
│ 2    │ 5      │ 10     │ "write" │ ["read", "write"]       │
│ 3    │ 11     │ npos   │ 最后部分│ ["read", "write", "execute"] │
└──────┴────────┴────────┴─────────┴────────────────────────┘

输出: ["read", "write", "execute"]
```

---

## 算法分析

### OckTrimString 时间复杂度

```
最佳情况: O(n)
- find_first_not_of() 最多遍历 n 个字符
- find_last_not_of() 最多遍历 n 个字符
- erase() 操作为 O(1) 或 O(m)，其中 m 为删除的字符数

总体时间复杂度: O(n)，其中 n 为字符串长度
```

### SplitStr 时间复杂度

```
设 n 为字符串长度，k 为分割后的段数

- find() 操作: O(n) 每次查找
- substr() 操作: O(m)，其中 m 为子串长度
- OckTrimString(): O(m)
- insert()/emplace_back(): O(log m) for set, O(1) amortized for vector

总体时间复杂度: O(n * k)
```

---

## 函数对比

| 特性 | OckTrimString | SplitStr(set) | SplitStr(vector) |
|-----|---------------|---------------|------------------|
| **输入** | string | string, separator | string, separator |
| **输出** | 原地修改 | set | vector |
| **去重** | N/A | 是 | 否 |
| **排序** | N/A | 是（按字母） | 否（保持原顺序） |
| **空白处理** | 去除首尾 | 每段去除首尾 | 每段去除首尾 |
| **空分隔符** | 不适用 | 整串作为一段 | 整串作为一段 |

---

## 使用流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                     配置字符串处理流程                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  原始配置行: "  key1 || key2 || key3  "                         │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────────┐                                           │
│  │ OckTrimString() │                                           │
│  └────────┬────────┘                                           │
│           │                                                      │
│           ▼                                                      │
│  "key1 || key2 || key3"                                        │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐                                           │
│  │  SplitStr()     │                                           │
│  │  separator="||" │                                           │
│  └────────┬────────┘                                           │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────────────────────────────┐                   │
│  │  ["key1", "key2", "key3"]               │                   │
│  │  或                                       │                   │
│  │  {"key1", "key2", "key3"} (排序后)       │                   │
│  └─────────────────────────────────────────┘                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 实际应用示例

### 配置文件解析

```cpp
#include "mmc_functions.h"

using namespace ock::mmc;

// 模拟解析配置行
void ParseConfigLine(const std::string& line) {
    // 1. 去除首尾空白
    std::string trimmed = line;
    OckTrimString(trimmed);

    // 2. 跳过注释和空行
    if (trimmed.empty() || trimmed[0] == '#') {
        return;
    }

    // 3. 分割键值对
    size_t equalPos = trimmed.find('=');
    if (equalPos != std::string::npos) {
        std::string key = trimmed.substr(0, equalPos);
        std::string value = trimmed.substr(equalPos + 1);

        // 4. 去除键和值的首尾空白
        OckTrimString(key);
        OckTrimString(value);

        std::cout << "Key: [" << key << "], Value: [" << value << "]" << std::endl;
    }
}

// 示例使用
int main() {
    std::vector<std::string> lines = {
        "  # This is a comment  ",
        "  ",
        "  log_level = info  ",
        "  allowed_ops = read||write||execute  ",
        "  servers = server1|server2|server3  "
    };

    for (const auto& line : lines) {
        ParseConfigLine(line);
    }

    // 处理枚举值
    std::string ops = "read||write||execute";
    std::set<std::string> validOps;
    SplitStr(ops, "||", validOps);
    // validOps = {"execute", "read", "write"}

    // 处理多值输入
    std::string input = "read|write";
    std::vector<std::string> inputOps;
    SplitStr(input, "|", inputOps);
    // inputOps = ["read", "write"]
}
```

---

## 边界情况处理

### OckTrimString 边界情况

| 输入 | 输出 | 说明 |
|-----|------|------|
| `""` | `""` | 空字符串 |
| `"   "` | `""` | 全是空白 |
| `"hello"` | `"hello"` | 无空白 |
| `"  hello  "` | `"hello"` | 两边有空白 |
| `"\t\n\r"` | `""` | 特殊空白字符 |

### SplitStr 边界情况

| 输入 | 分隔符 | 输出 (set) | 输出 (vector) |
|-----|--------|-----------|--------------|
| `"a||b"` | `"||"` | `{"a", "b"}` | `["a", "b"]` |
| `"a||a"` | `"||"` | `{"a"}` | `["a", "a"]` |
| `"a"` | `"||"` | `{"a"}` | `["a"]` |
| `""` | `"||"` | `{}` | `[]` |
| `"||a||"` | `"||"` | `{"a"}` | `["", "", "a", ""]` |
| `" a "` | `"||"` | `{"a"}` | `["a"]` |
| `"a || b"` | `"||"` | `{"a", "b"}` | `["a", "b"]` |

**注意事项**:
- 空字符串用 vector 版本分割时会产生多个空字符串元素
- set 版本会自动去重
- 所有分割后的子串都会去除首尾空白

---

## 性能优化建议

1. **预分配结果容器空间**
   ```cpp
   // 如果知道大致的分割数量，可以预留空间
   std::vector<std::string> result;
   result.reserve(10);  // 预留空间
   SplitStr(str, "|", result);
   ```

2. **避免不必要的拷贝**
   ```cpp
   // 对于大字符串，可以使用移动语义
   std::string largeStr = GetLargeString();
   OckTrimString(largeStr);  // 原地修改，避免拷贝
   ```

3. **选择合适的容器**
   - 需要去重和排序：使用 set 版本
   - 需要保持顺序和允许重复：使用 vector 版本
