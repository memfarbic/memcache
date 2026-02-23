# mmc_config_validator.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/config/mmc_config_validator.h`
- **文件用途**: 定义配置验证器基类和各种具体验证器，用于验证配置值的合法性
- **依赖项**:
  - `<string>` - 字符串处理
  - `<utility>` - std::pair
  - `<sys/stat.h>` - 文件状态检查
  - `mmc_ref.h` - 引用计数智能指针
  - `mmc_functions.h` - 工具函数

---

## 类型别名

### ValidatorPtr

**声明位置**: 行 62
**完整签名**:
```cpp
using ValidatorPtr = MmcRef<Validator>;
```
**功能描述**: 验证器智能指针类型别名

---

## 基类：Validator

### 类概述

**声明位置**: 行 26-61
**完整签名**:
```cpp
class Validator : public MmcReferable
```
**功能描述**: 配置验证器抽象基类，所有具体验证器继承自此类

**继承关系**:
```
MmcReferable
    ▲
    │
Validator (基类)
    ├─ VNoCheck
    ├─ VStrEnum
    ├─ VStrInSet
    ├─ VStrNotNull
    ├─ VStrLength
    ├─ VIntRange
    ├─ VUInt64Range
    └─ VPathAccess
```

---

### Validator::~Validator()

**声明位置**: 行 28
**完整签名**:
```cpp
~Validator() override = default;
```
**功能描述**: 虚析构函数，确保派生类正确析构

---

### Validator::Initialize()

**声明位置**: 行 30
**完整签名**:
```cpp
virtual bool Initialize() = 0;
```
**功能描述**: 初始化验证器（纯虚函数，必须由派生类实现）
**返回值**:
- `true` - 初始化成功
- `false` - 初始化失败

---

### Validator::Validate() - 字符串重载

**声明位置**: 行 31-34
**完整签名**:
```cpp
virtual bool Validate(const std::string &)
{
    return true;
}
```
**功能描述**: 验证字符串类型的配置值（虚函数，默认通过）
**返回值**:
- `true` - 验证通过
- `false` - 验证失败

---

### Validator::Validate() - 整数重载

**声明位置**: 行 36-39
**完整签名**:
```cpp
virtual bool Validate(int)
{
    return true;
}
```
**功能描述**: 验证整数类型的配置值（虚函数，默认通过）

---

### Validator::Validate() - 浮点数重载

**声明位置**: 行 41-44
**完整签名**:
```cpp
virtual bool Validate(float)
{
    return true;
}
```
**功能描述**: 验证浮点数类型的配置值（虚函数，默认通过）

---

### Validator::Validate() - 无符号长整重载

**声明位置**: 行 46-49
**完整签名**:
```cpp
virtual bool Validate(long unsigned)
{
    return true;
}
```
**功能描述**: 验证无符号长整类型的配置值（虚函数，默认通过）

---

### Validator::ErrorMessage()

**声明位置**: 行 51-54
**完整签名**:
```cpp
const std::string &ErrorMessage()
{
    return mErrMsg;
}
```
**功能描述**: 获取验证失败的错误信息
**返回值**: 错误信息字符串的引用

---

### Validator::mName / mErrMsg

**声明位置**: 行 56-60
```cpp
std::string mName;    // 验证器名称（配置项名称）
std::string mErrMsg;  // 错误信息
```

---

## 派生类：VNoCheck（无检查验证器）

### 类概述

**声明位置**: 行 64-79
**功能描述**: 不做任何验证的验证器，所有值都通过

---

### VNoCheck::Create()

**声明位置**: 行 66-69
**完整签名**:
```cpp
static ValidatorPtr Create(const std::string &name = "")
{
    return {new (std::nothrow) VNoCheck(name)};
}
```
**功能描述**: 创建 VNoCheck 验证器实例的工厂方法
**参数**:
- `name`: 配置项名称（可选）
**返回值**: 验证器智能指针

---

### VNoCheck::VNoCheck()

**声明位置**: 行 71
**完整签名**:
```cpp
explicit VNoCheck(const std::string &name) : Validator(name) {}
```
**功能描述**: 构造函数

---

### VNoCheck::Initialize()

**声明位置**: 行 75-78
**完整签名**:
```cpp
bool Initialize() override
{
    return true;
}
```
**功能描述**: 初始化，始终成功

---

## 派生类：VStrEnum（字符串枚举验证器）

### 类概述

**声明位置**: 行 81-138
**功能描述**: 验证字符串值是否在指定的枚举列表中（单值）

---

### VStrEnum::Create()

**声明位置**: 行 83-86
**完整签名**:
```cpp
static ValidatorPtr Create(const std::string &name, const std::string &enumStr, const bool caseSensitive = false)
```
**功能描述**: 创建字符串枚举验证器
**参数**:
- `name`: 配置项名称
- `enumStr`: 枚举值字符串，格式 `"value1||value2||value3"`
- `caseSensitive`: 是否区分大小写（默认不区分）
**返回值**: 验证器智能指针

---

### VStrEnum::VStrEnum()

**声明位置**: 行 88-94
**完整签名**:
```cpp
VStrEnum(const std::string &name, std::string enumStr, const bool caseSensitive = false)
    : Validator(name), mEnumString(std::move(enumStr)), caseSensitive(caseSensitive)
```
**功能描述**: 构造函数
**代码逻辑**:
1. 保存枚举字符串
2. 如果不区分大小写，将枚举字符串转换为小写

---

### VStrEnum::Initialize()

**声明位置**: 行 98-109
**完整签名**:
```cpp
bool Initialize() override
```
**功能描述**: 初始化并验证枚举字符串格式
**代码逻辑**:
1. 使用 `SplitStr` 按 `||` 分割枚举字符串
2. 检查分割结果不为空
**返回值**:
- `true` - 枚举字符串格式正确
- `false` - 枚举字符串为空

---

### VStrEnum::Validate()

**声明位置**: 行 111-133
**完整签名**:
```cpp
bool Validate(const std::string &value) override
```
**功能描述**: 验证输入值是否在枚举列表中
**代码逻辑**:
1. 检查值不包含 `||`（防止多值注入）
2. 如果不区分大小写，将输入值转为小写
3. 构造查找模板 `"||enumStr||"`
4. 在模板中查找 `"||value||"`
**返回值**:
- `true` - 值在枚举列表中
- `false` - 值不在列表中

**注意事项**:
- 使用 `"||"` 包裹的字符串匹配技巧，避免部分匹配

---

## 派生类：VStrInSet（字符串集合验证器）

### 类概述

**声明位置**: 行 140-177
**功能描述**: 验证字符串值（支持多值用 `|` 分隔）是否在指定的集合中

---

### VStrInSet::Create()

**声明位置**: 行 142-145
**完整签名**:
```cpp
static ValidatorPtr Create(const std::string &name, const std::string &enumStr)
```
**功能描述**: 创建字符串集合验证器
**参数**:
- `name`: 配置项名称
- `enumStr`: 枚举值字符串，格式 `"value1||value2||value3"`

---

### VStrInSet::VStrInSet()

**声明位置**: 行 147
**完整签名**:
```cpp
VStrInSet(const std::string &name, std::string enumStr) : Validator(name), mEnumString(std::move(enumStr)) {}
```

---

### VStrInSet::Initialize()

**声明位置**: 行 151-159
**完整签名**:
```cpp
bool Initialize() override
```
**功能描述**: 初始化，将枚举字符串解析到 `std::set` 中
**代码逻辑**:
1. 按 `||` 分割枚举字符串
2. 存储到 `validEnumSet` 有序集合中

---

### VStrInSet::Validate()

**声明位置**: 行 161-172
**完整签名**:
```cpp
bool Validate(const std::string &value) override
```
**功能描述**: 验证输入值（支持多值）是否都在集合中
**代码逻辑**:
1. 按 `|` 分割输入值
2. 检查每个分割后的值是否在 `validEnumSet` 中
**返回值**:
- `true` - 所有值都在集合中
- `false` - 存在不在集合中的值

**使用示例**:
```cpp
// 枚举: "read||write||execute"
// 输入: "read|write" -> 通过
// 输入: "read|delete" -> 失败
```

---

## 派生类：VStrNotNull（非空字符串验证器）

### 类概述

**声明位置**: 行 179-203
**功能描述**: 验证字符串不为空

---

### VStrNotNull::Create()

**声明位置**: 行 181-184
**完整签名**:
```cpp
static ValidatorPtr Create(const std::string &name)
```

---

### VStrNotNull::VStrNotNull()

**声明位置**: 行 186
**完整签名**:
```cpp
explicit VStrNotNull(const std::string &name) : Validator(name) {};
```

---

### VStrNotNull::Initialize()

**声明位置**: 行 190-193
**完整签名**:
```cpp
bool Initialize() override
{
    return true;
}
```

---

### VStrNotNull::Validate()

**声明位置**: 行 195-202
**完整签名**:
```cpp
bool Validate(const std::string &value) override
```
**功能描述**: 验证字符串非空
**代码逻辑**:
1. 检查 `value.empty()`
2. 如果为空，设置错误信息
**返回值**:
- `true` - 字符串非空
- `false` - 字符串为空

---

## 派生类：VStrLength（字符串长度验证器）

### 类概述

**声明位置**: 行 205-234
**功能描述**: 验证字符串长度不超过指定限制

---

### VStrLength::Create()

**声明位置**: 行 207-210
**完整签名**:
```cpp
static ValidatorPtr Create(const std::string &name, const unsigned long lenLimit)
```
**参数**:
- `name`: 配置项名称
- `lenLimit`: 最大长度限制

---

### VStrLength::VStrLength()

**声明位置**: 行 212-213
**完整签名**:
```cpp
explicit VStrLength(const std::string &name, const unsigned long lenLimit)
    : Validator(name), mLengthLimit(lenLimit) {};
```

---

### VStrLength::Validate()

**声明位置**: 行 222-230
**完整签名**:
```cpp
bool Validate(const std::string &value) override
```
**功能描述**: 验证字符串长度
**代码逻辑**:
1. 检查 `value.length() > mLengthLimit`
2. 如果超长，设置错误信息（包含限制值）
**返回值**:
- `true` - 长度符合要求
- `false` - 长度超限

---

## 派生类：VIntRange（整数范围验证器）

### 类概述

**声明位置**: 行 236-273
**功能描述**: 验证整数在指定范围内

---

### VIntRange::Create()

**声明位置**: 行 238-241
**完整签名**:
```cpp
static ValidatorPtr Create(const std::string &name, const int &start, const int &end)
```
**参数**:
- `name`: 配置项名称
- `start`: 范围起始值（包含）
- `end`: 范围结束值（包含）

---

### VIntRange::VIntRange()

**声明位置**: 行 242
**完整签名**:
```cpp
VIntRange(const std::string &name, const int &start, const int &end) : Validator(name), mStart(start), mEnd(end) {};
```

---

### VIntRange::Initialize()

**声明位置**: 行 246-253
**完整签名**:
```cpp
bool Initialize() override
```
**功能描述**: 初始化并验证范围参数
**代码逻辑**:
1. 检查 `mStart >= mEnd`
2. 如果起始大于等于结束，设置错误信息
**返回值**:
- `true` - 范围有效
- `false` - 范围无效（start >= end）

---

### VIntRange::Validate()

**声明位置**: 行 255-268
**完整签名**:
```cpp
bool Validate(int value) override
```
**功能描述**: 验证整数是否在范围内
**代码逻辑**:
1. 检查 `value < mStart || value > mEnd`
2. 如果超出范围，根据是否为最大值生成不同的错误信息
**返回值**:
- `true` - 值在范围内
- `false` - 值超出范围

**注意事项**:
- 当 `end == INT32_MAX` 时，错误信息只显示下限

---

## 派生类：VUInt64Range（无符号64位整数范围验证器）

### 类概述

**声明位置**: 行 275-309
**功能描述**: 验证 uint64_t 类型整数在指定范围内

---

### VUInt64Range::Create()

**声明位置**: 行 277-280
**完整签名**:
```cpp
static ValidatorPtr Create(const std::string &name, const uint64_t &start, const uint64_t &end)
```

---

### VUInt64Range::VUInt64Range()

**声明位置**: 行 281-282
**完整签名**:
```cpp
VUInt64Range(const std::string &name, const uint64_t &start, const uint64_t &end)
    : Validator(name), mStart(start), mEnd(end) {};
```

---

### VUInt64Range::Initialize()

**声明位置**: 行 286-293
**完整签名**:
```cpp
bool Initialize() override
```
**功能描述**: 初始化并验证范围参数
**代码逻辑**: 与 VIntRange::Initialize() 相同

---

### VUInt64Range::Validate()

**声明位置**: 行 295-304
**完整签名**:
```cpp
bool Validate(const uint64_t value) override
```
**功能描述**: 验证 uint64_t 值是否在范围内
**代码逻辑**: 与 VIntRange::Validate() 相同

---

## 派生类：VPathAccess（路径访问权限验证器）

### 类概述

**声明位置**: 行 311-397
**功能描述**: 验证路径的有效性和访问权限

---

### VPathAccess::Create()

**声明位置**: 行 313-316
**完整签名**:
```cpp
static ValidatorPtr Create(const std::string &name, int flag)
```
**参数**:
- `name`: 配置项名称
- `flag`: 访问权限标志（如 `R_OK`, `W_OK`, `X_OK`, `F_OK`）

---

### VPathAccess::VPathAccess()

**声明位置**: 行 318-321
**完整签名**:
```cpp
VPathAccess(const std::string &name, int flag) : Validator(name)
{
    mFlag = flag;
}
```

---

### VPathAccess::Initialize()

**声明位置**: 行 325-328
**完整签名**:
```cpp
bool Initialize() override
{
    return true;
}
```

---

### VPathAccess::Validate()

**声明位置**: 行 330-359
**完整签名**:
```cpp
bool Validate(const std::string &path) override
```
**功能描述**: 验证路径的有效性和权限
**代码逻辑**:
1. **路径基本检查** (行 332-335)
   - 检查路径非空
   - 确保路径以 `/` 结尾

2. **查找已存在的最深目录** (行 340-357)
   - 逐级检查路径中的每个目录
   - 找到最后一个存在的目录
   - 记录为 `existDeepestDir`

3. **调用 PathCheck** (行 358)
   - 检查已存在目录的权限
   - 检查待创建部分是否包含禁止词

**返回值**:
- `true` - 路径有效且有权限访问
- `false` - 路径无效或无权限

---

### VPathAccess::PathCheck()

**声明位置**: 行 362-393
**完整签名**:
```cpp
bool PathCheck(const std::string &existDeepestDir, const std::string &rest, const std::string &path)
```
**功能描述**: 私有方法，检查路径权限和安全性
**代码逻辑**:
1. **路径规范化** (行 365-372)
   - 使用 `realpath()` 获取真实路径
   - 失败时返回权限错误

2. **权限检查** (行 375-378)
   - 使用 `access()` 检查指定权限
   - 失败时返回权限错误

3. **安全性检查** (行 380-391)
   - 检查待创建路径部分是否包含 `..`
   - 防止路径遍历攻击

**返回值**:
- `true` - 路径安全且有权限
- `false` - 权限不足或路径不安全

**注意事项**:
- `forbiddenWords` 包含 `".."`，防止目录遍历攻击

---

## 成员变量

### VStrEnum
```cpp
std::string mEnumString;  // 枚举字符串
bool caseSensitive;       // 是否区分大小写
```

### VStrInSet
```cpp
std::string mEnumString;           // 枚举字符串
std::set<std::string> validEnumSet; // 有效值集合
```

### VStrLength
```cpp
unsigned long mLengthLimit;  // 最大长度限制
```

### VIntRange / VUInt64Range
```cpp
int/uint64_t mStart;  // 范围起始值
int/uint64_t mEnd;    // 范围结束值
```

### VPathAccess
```cpp
int mFlag;                                // 访问权限标志
const std::vector<std::string> forbiddenWords{".."};  // 禁止词
```

---

## 验证器类继承关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    MmcReferable (基类)                          │
└────────────────────────────────────┬────────────────────────────┘
                                     │
                                     ▼
                           ┌─────────────────────┐
                           │    Validator        │
                           │  - mName: string    │
                           │  - mErrMsg: string  │
                           │  + Initialize()     │
                           │  + Validate()       │
                           │  + ErrorMessage()   │
                           └─────────────────────┘
                                     │
        ┌────────────┬───────────────┼───────────────┬────────────┐
        │            │               │               │            │
        ▼            ▼               ▼               ▼            ▼
  ┌──────────┐ ┌──────────┐  ┌──────────┐   ┌──────────┐  ┌──────────┐
  │VNoCheck  │ │VStrEnum  │  │VStrInSet │   │VStrNotNull│ │VStrLength│
  │无检查    │ │枚举验证  │  │集合验证  │   │非空验证   │ │长度验证  │
  └──────────┘ └──────────┘  └──────────┘   └──────────┘  └──────────┘
        │            │               │               │
        ▼            ▼               ▼               ▼
  ┌──────────┐ ┌──────────┐  ┌──────────┐   ┌──────────┐
  │          │ │caseSensi-│  │validEnum-│   │mLength   │
  │          │ │tive      │  │Set       │   │Limit     │
  └──────────┘ └──────────┘  └──────────┘   └──────────┘

        ┌────────────┬────────────┐
        │            │            │
        ▼            ▼            ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │VIntRange │ │VUInt64   │ │VPathAccess│
  │整数范围  │ │Range     │ │路径验证  │
  │          │ │uint64    │ │          │
  │mStart    │ │mStart    │ │mFlag     │
  │mEnd      │ │mEnd      │ │forbidden │
  └──────────┘ └──────────┘ │Words     │
                             └──────────┘
```

---

## 使用示例

### 基本使用
```cpp
#include "mmc_config_validator.h"

using namespace ock::mmc;

// 1. 创建无检查验证器
ValidatorPtr noCheck = VNoCheck::Create("config_item");

// 2. 创建枚举验证器
ValidatorPtr logLevelValidator = VStrEnum::Create(
    "ock.mmc.log_level",
    "debug||info||warn||error"
);

// 3. 创建范围验证器
ValidatorPtr portValidator = VIntRange::Create(
    "ock.mmc.port",
    1024,    // 起始值
    65535    // 结束值
);

// 4. 创建路径验证器
ValidatorPtr pathValidator = VPathAccess::Create(
    "ock.mmc.log_path",
    W_OK | R_OK  // 检查读写权限
);

// 5. 使用验证器
if (!logLevelValidator->Initialize()) {
    std::cerr << "Validator init failed: "
              << logLevelValidator->ErrorMessage() << std::endl;
}

if (!logLevelValidator->Validate("info")) {
    std::cerr << "Validation failed: "
              << logLevelValidator->ErrorMessage() << std::endl;
}
```

### 与 Configuration 配合使用
```cpp
#include "mmc_configuration.h"

class MyConfig : public Configuration {
    void LoadDefault() override {
        // 添加带验证器的配置项
        AddIntConf(
            {"ock.mmc.port", 8080},
            VIntRange::Create("ock.mmc.port", 1024, 65535),
            CONF_MUST  // 必需配置
        );

        AddStrConf(
            {"ock.mmc.log_level", "info"},
            VStrEnum::Create("ock.mmc.log_level", "debug||info||warn||error"),
            0  // 非必需
        );
    }
};
```

### VStrInSet 多值验证
```cpp
// 验证支持多种操作
ValidatorPtr opsValidator = VStrInSet::Create(
    "ock.mmc.operations",
    "read||write||execute||delete"
);

// 单值验证
opsValidator->Validate("read");   // 通过
opsValidator->Validate("read|write");  // 通过
opsValidator->Validate("read|admin");  // 失败，"admin" 不在列表中
```

### 路径验证
```cpp
// 验证日志目录路径（需要读写权限）
ValidatorPtr logPathValidator = VPathAccess::Create(
    "ock.mmc.log_path",
    W_OK | R_OK  // 检查读写权限
);

// 有效路径
logPathValidator->Validate("/var/log/memcache");  // 通过（如果有权限）
logPathValidator->Validate("/tmp/../etc/passwd"); // 失败（包含 ".."）
```
