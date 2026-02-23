# mmc_kv_parser.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/config/mmc_kv_parser.h`
- **文件用途**: 定义键值对配置文件解析器，用于读取和解析 Key-Value 格式的配置文件
- **依赖项**:
  - `<map>` - 映射容器
  - `<string>` - 字符串处理
  - `<vector>` - 动态数组
  - `<fstream>` - 文件流
  - `<unordered_map>` - 哈希映射
  - `mmc_lock.h` - 锁机制
  - `mmc_types.h` - 类型定义

---

## 数据结构定义

### KvPair 结构体

**声明位置**: 行 26-29
**完整签名**:
```cpp
using KvPair = struct KvPairs {
    std::string name;   // 配置项名称（键）
    std::string value;  // 配置项值
};
```
**功能描述**: 键值对结构，用于存储单个配置项
**成员变量**:
- `name`: 配置项的键名
- `value`: 配置项的值

---

## KVParser 类

### 类概述

**声明位置**: 行 31-59
**功能描述**: 键值对配置文件解析器，支持从文件读取 Key-Value 格式的配置

**特性**:
- 线程安全（使用互斥锁保护）
- 支持注释行（以 `#` 开头）
- 支持空行过滤
- 禁止拷贝和移动操作

---

### KVParser::KVParser()

**声明位置**: 行 33
**完整签名**:
```cpp
KVParser();
```
**功能描述**: 默认构造函数
**返回值**: 无
**注意事项**: 使用 `= default` 声明，使用编译器默认实现

---

### KVParser::~KVParser()

**声明位置**: 行 34
**完整签名**:
```cpp
~KVParser();
```
**功能描述**: 析构函数，释放所有动态分配的 `KvPair` 对象
**返回值**: 无
**注意事项**:
- 会遍历 `mItems` 并删除所有指针
- 清空 `mItems` 和 `mItemsIndex` 容器

---

### KVParser() - 禁止拷贝/移动

**声明位置**: 行 36-39
**完整签名**:
```cpp
KVParser(const KVParser &) = delete;
KVParser &operator=(const KVParser &) = delete;
KVParser(const KVParser &&) = delete;
KVParser &operator=(const KVParser &&) = delete;
```
**功能描述**: 禁止拷贝和移动操作
**原因**: 类内部管理动态资源，需要显式控制生命周期

---

### KVParser::FromFile()

**声明位置**: 行 41
**完整签名**:
```cpp
Result FromFile(const std::string &filePath);
```
**功能描述**: 从指定文件路径读取并解析配置文件
**参数**:
- `filePath`: 配置文件的路径
**返回值**: `Result` - `MMC_OK` 表示成功，`MMC_ERROR` 表示失败
**代码逻辑**:
1. 验证文件路径有效性和安全性（非符号链接）
2. 检查文件大小不超过限制（10MB）
3. 逐行读取文件内容
4. 对每行调用 `ParseLine()` 进行解析
5. 检查总行数不超过限制（10000 行）

**注意事项**:
- 配置文件格式：`key = value`
- 支持 `#` 开头的注释行
- 支持 Windows (`\r\n`) 和 Unix (`\n`) 换行符

**使用示例**:
```cpp
KVParser parser;
if (parser.FromFile("/etc/memcache/config.conf") != MMC_OK) {
    std::cerr << "Failed to load config file" << std::endl;
}
```

---

### KVParser::GetItem()

**声明位置**: 行 43
**完整签名**:
```cpp
Result GetItem(const std::string &key, std::string &outValue);
```
**功能描述**: 根据键名获取对应的配置值
**参数**:
- `key`: 配置项的键名
- `outValue`: 输出参数，存储找到的配置值
**返回值**:
- `MMC_OK` - 找到配置项
- `MMC_ERROR` - 未找到配置项或发生错误

**代码逻辑**:
1. 加锁保护访问
2. 在 `mItemsIndex` 中查找键名
3. 如果找到，通过索引从 `mItems` 获取值
4. 检查索引有效性防止越界

**使用示例**:
```cpp
std::string value;
if (parser.GetItem("ock.mmc.log_level", value) == MMC_OK) {
    std::cout << "Log level: " << value << std::endl;
}
```

---

### KVParser::SetItem()

**声明位置**: 行 44
**完整签名**:
```cpp
Result SetItem(const std::string &key, const std::string &value);
```
**功能描述**: 设置一个配置项（内部方法，由解析过程调用）
**参数**:
- `key`: 配置项的键名
- `value`: 配置项的值
**返回值**:
- `MMC_OK` - 设置成功
- `MMC_ERROR` - 键已重复或内存分配失败

**代码逻辑**:
1. 检查键是否已存在（禁止重复键）
2. 动态分配新的 `KvPair` 对象
3. 将对象添加到 `mItems` 并记录索引
4. 在 `mGotKeys` 中标记该键已被获取

**注意事项**:
- 键名不能重复
- 使用 `std::nothrow` 进行内存分配，失败返回空指针

---

### KVParser::Size()

**声明位置**: 行 46
**完整签名**:
```cpp
uint32_t Size();
```
**功能描述**: 获取已解析的配置项数量
**返回值**: `uint32_t` - 配置项总数

---

### KVParser::GetI()

**声明位置**: 行 47
**完整签名**:
```cpp
void GetI(const uint32_t index, std::string &outKey, std::string &outValue);
```
**功能描述**: 根据索引获取配置项的键和值
**参数**:
- `index`: 配置项索引（从 0 开始）
- `outKey`: 输出参数，存储键名
- `outValue`: 输出参数，存储配置值

**代码逻辑**:
1. 检查索引有效性
2. 从 `mItems` 获取指定索引的 `KvPair`
3. 返回键名和值

**使用示例**:
```cpp
for (uint32_t i = 0; i < parser.Size(); ++i) {
    std::string key, value;
    parser.GetI(i, key, value);
    std::cout << key << " = " << value << std::endl;
}
```

---

### KVParser::Dump()

**声明位置**: 行 49
**完整签名**:
```cpp
void Dump();
```
**功能描述**: 打印所有配置项到标准输出（用于调试）
**输出格式**: `key = value`

**使用示例**:
```cpp
parser.Dump();  // 输出所有配置项
```

---

### KVParser::CheckSet()

**声明位置**: 行 50
**完整签名**:
```cpp
bool CheckSet(const std::vector<std::string> &keys);
```
**功能描述**: 检查指定的键是否都已设置（即是否在配置文件中出现）
**参数**:
- `keys`: 需要检查的键名列表
**返回值**:
- `true` - 所有键都已设置
- `false` - 存在未设置的键

**代码逻辑**:
1. 遍历 `keys` 列表
2. 检查每个键是否存在于 `mGotKeys` 中
3. 如果有未设置的键，输出错误信息
4. 清空 `mGotKeys`（为下次检查做准备）

**注意事项**:
- 调用后会清空 `mGotKeys`，不能重复调用获取相同结果

**使用示例**:
```cpp
std::vector<std::string> requiredKeys = {
    "ock.mmc.log_level",
    "ock.mmc.log_path"
};
if (!parser.CheckSet(requiredKeys)) {
    std::cerr << "Missing required configuration items" << std::endl;
}
```

---

## 私有成员

### KVParser::ParseLine()

**声明位置**: 行 53
**完整签名**:
```cpp
Result ParseLine(std::string &strLine);
```
**功能描述**: 解析单行配置内容（私有方法）
**参数**:
- `strLine`: 待解析的行内容
**返回值**: `Result` - `MMC_OK` 表示成功，`MMC_ERROR` 表示失败

**代码逻辑**:
1. 移除行尾的 `\r` 字符
2. 去除首尾空白字符
3. 跳过空行和注释行（以 `#` 开头）
4. 检查行长度不超过限制（4096 字符）
5. 以 `=` 为分隔符分割键和值
6. 去除键和值的首尾空白
7. 调用 `SetItem()` 存储配置项

---

## 成员变量

**声明位置**: 行 55-58
```cpp
std::map<std::string, uint32_t> mItemsIndex;       // 键到索引的映射
std::vector<KvPair *> mItems;                       // 配置项数组
std::unordered_map<std::string, bool> mGotKeys;     // 已被获取的键记录
Lock mLock;                                         // 保护访问的互斥锁
```

**成员变量说明**:
- `mItemsIndex`: 使用有序映射存储键到索引的关系，支持快速查找
- `mItems`: 动态数组存储所有配置项，按插入顺序排列
- `mGotKeys`: 记录哪些键被 `GetItem()` 访问过，用于 `CheckSet()` 验证
- `mLock`: 互斥锁，保护多线程访问

---

## 数据结构关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                         KVParser 类                              │
├─────────────────────────────────────────────────────────────────┤
│  mItemsIndex (map)       mItems (vector)        mGotKeys (map)  │
│  ┌─────────────────┐     ┌──────────────────┐    ┌──────────┐   │
│  │ key      -> idx │     │ [0] KvPair*      │    │ key -> T │   │
│  │ "log_level"->0  │ ──▶ │ [1] KvPair*      │    │ "log_path"│   │
│  │ "log_path" ->1  │     │ [2] KvPair*      │    └──────────┘   │
│  └─────────────────┘     └──────────────────┘                    │
│                                  │                                │
│                                  ▼                                │
│                          ┌──────────────────┐                     │
│                          │ KvPair           │                     │
│                          │  - name: string  │                     │
│                          │  - value: string │                     │
│                          └──────────────────┘                     │
├─────────────────────────────────────────────────────────────────┤
│  mLock (Lock)                                                    │
│  └── 保护所有容器的并发访问                                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 配置文件格式示例

```
# MemCache 配置文件示例

# 日志配置
ock.mmc.log_level = info
ock.mmc.log_path = /var/log/memcache

# 网络配置
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

# TLS 配置
ock.mmc.tls.enable = false

# 内存配置（支持单位）
ock.mmc.local_service.dram.size = 128MB
ock.mmc.local_service.hbm.size = 0
```

---

## 使用示例

```cpp
#include "mmc_kv_parser.h"

using namespace ock::mmc;

// 1. 创建解析器
KVParser parser;

// 2. 从文件加载配置
if (parser.FromFile("/etc/memcache/mmc.conf") != MMC_OK) {
    std::cerr << "Failed to parse configuration file" << std::endl;
    return -1;
}

// 3. 获取特定配置项
std::string logLevel;
if (parser.GetItem("ock.mmc.log_level", logLevel) == MMC_OK) {
    std::cout << "Log Level: " << logLevel << std::endl;
}

// 4. 遍历所有配置项
for (uint32_t i = 0; i < parser.Size(); ++i) {
    std::string key, value;
    parser.GetI(i, key, value);
    std::cout << key << " = " << value << std::endl;
}

// 5. 验证必需配置项是否存在
std::vector<std::string> required = {
    "ock.mmc.log_level",
    "ock.mmc.meta_service_url"
};
if (!parser.CheckSet(required)) {
    std::cerr << "Missing required configuration" << std::endl;
}

// 6. 调试输出所有配置
parser.Dump();
```

---

## 错误处理

| 错误场景 | 返回值 | 说明 |
|---------|--------|------|
| 文件路径不存在 | `MMC_ERROR` | `FromFile()` 返回错误 |
| 文件是符号链接 | `MMC_ERROR` | 安全检查拒绝 |
| 文件超过 10MB | `MMC_ERROR` | 防止过大文件 |
| 行数超过 10000 | `MMC_ERROR` | 防止恶意文件 |
| 键名重复 | `MMC_ERROR` | `SetItem()` 拒绝重复 |
| 行长度超过 4096 | `MMC_ERROR` | `ParseLine()` 拒绝 |
| 键名为空 | `MMC_ERROR` | 无效配置项 |
