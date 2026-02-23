# mmc_config_convertor.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/config/mmc_config_convertor.h`
- **文件用途**: 定义配置值转换器基类，用于将配置值从字符串转换为特定类型
- **依赖项**:
  - `<string>` - 字符串处理
  - `mmc_ref.h` - 引用计数智能指针

---

## 类型别名

### ConverterPtr

**声明位置**: 行 31
**完整签名**:
```cpp
using ConverterPtr = MmcRef<Converter>;
```
**功能描述**: 转换器智能指针类型别名

---

## 基类：Converter

### 类概述

**声明位置**: 行 21-29
**完整签名**:
```cpp
class Converter : public MmcReferable
```
**功能描述**: 配置值转换器抽象基类，用于将配置值从字符串转换为其他类型

**继承关系**:
```
MmcReferable
    ▲
    │
Converter (基类)
```

---

### Converter::~Converter()

**声明位置**: 行 23
**完整签名**:
```cpp
~Converter() override = default;
```
**功能描述**: 虚析构函数，确保派生类正确析构

**注意事项**:
- 使用 `= default` 使用编译器默认实现
- 声明为 `virtual` 确保通过基类指针删除派生类对象时正确调用派生类析构函数

---

### Converter::Convert()

**声明位置**: 行 25-28
**完整签名**:
```cpp
virtual std::string Convert(const std::string &str)
{
    return str;
}
```
**功能描述**: 将配置值从一种形式转换为另一种（虚函数，默认不转换）
**参数**:
- `str`: 输入的配置值字符串
**返回值**: 转换后的字符串（默认返回原字符串）

**设计说明**:
- 默认实现是恒等转换（返回输入值）
- 派生类可以重写此方法实现特定的转换逻辑
- 返回值使用 `std::string` 保持通用性

---

## 使用场景

Converter 类为配置系统提供扩展点，允许自定义配置值的转换逻辑。典型的使用场景包括：

### 1. 环境变量替换
```cpp
class EnvVariableConverter : public Converter {
public:
    std::string Convert(const std::string &str) override {
        if (str.find("${") == 0 && str.back() == '}') {
            // ${HOME} -> /home/user
            std::string envName = str.substr(2, str.length() - 3);
            return std::getenv(envName.c_str());
        }
        return str;
    }
};
```

### 2. 路径展开
```cpp
class PathExpanderConverter : public Converter {
public:
    std::string Convert(const std::string &str) override {
        if (str.front() == '~') {
            // ~/config -> /home/user/config
            std::string home = std::getenv("HOME");
            return home + str.substr(1);
        }
        return str;
    }
};
```

### 3. 单位转换
```cpp
class SizeConverter : public Converter {
public:
    std::string Convert(const std::string &str) override {
        // "128MB" -> "134217728"
        // 在 Configuration 类中有专门的实现
        return str;
    }
};
```

---

## 与 Configuration 的集成

Converter 类与 `Configuration` 类配合使用：

```cpp
#include "mmc_configuration.h"

using namespace ock::mmc;

class MyConfig : public Configuration {
    void LoadDefault() override {
        // 添加带转换器的配置项
        ConverterPtr converter = MmcMakeRef<MyCustomConverter>();

        AddConverter("ock.mmc.custom_path", converter);
        AddStrConf(
            {"ock.mmc.custom_path", "/default/path"},
            VNoCheck::Create(),
            0
        );
    }
};
```

---

## 成员变量

Converter 基类没有定义成员变量，所有状态由派生类管理。

---

## 类型转换流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    Configuration 配置加载流程                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  配置文件: "size = 128MB"                                       │
│       │                                                          │
│       ▼                                                          │
│  KVParser::FromFile()                                           │
│       │                                                          │
│       ▼                                                          │
│  KVParser::ParseLine() → key="size", value="128MB"              │
│       │                                                          │
│       ▼                                                          │
│  Configuration::SetWithTypeAutoConvert()                        │
│       │                                                          │
│       ├──▶ Configuration::SetWithStrAutoConvert()               │
│       │         │                                                │
│       │         ▼                                                │
│       │    Converter::Convert("128MB") [如果配置了转换器]        │
│       │         │                                                │
│       │         ▼                                                │
│       │    ParseMemSize("128MB") → 134217728 字节               │
│       │                                                          │
│       ▼                                                          │
│  mUInt64Items["size"] = 134217728                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 扩展指南

### 实现自定义转换器

要实现自定义的配置值转换器，需要：

1. **继承 Converter 基类**
2. **重写 Convert() 方法**
3. **在 Configuration::LoadDefault() 中注册**

```cpp
// 1. 定义转换器
class UpperCaseConverter : public Converter {
public:
    std::string Convert(const std::string &str) override {
        std::string result = str;
        std::transform(result.begin(), result.end(),
                       result.begin(), ::toupper);
        return result;
    }
};

// 2. 在配置类中使用
class MyConfig : public Configuration {
    void LoadDefault() override {
        // 添加转换器
        AddConverter("ock.mmc.protocol",
                     MmcMakeRef<UpperCaseConverter>());

        // 添加配置项
        AddStrConf({"ock.mmc.protocol", "tcp"},
                   VNoCheck::Create(), 0);
    }
};

// 使用: 配置文件 "protocol = tcp" 会被转换为 "TCP"
```

---

## 注意事项

1. **线程安全**: Converter 本身不提供线程安全保证，调用者需要确保线程安全

2. **异常处理**: Convert() 方法不应抛出异常，错误情况应该返回原值或特定错误值

3. **性能考虑**: 转换器在配置加载时被调用，应该避免耗时操作

4. **幂等性**: 理想的 Convert() 方法应该是幂等的，即 `Convert(Convert(x)) == Convert(x)`

---

## 实际应用示例

### 在 Configuration 中的应用

在 `mmc_configuration.cpp` 中，内存大小的转换是通过专门的方法实现的，而不是使用 Converter：

```cpp
// Configuration::SetWithStrAutoConvert() 中的实现
if (key == ConfConstant::OKC_MMC_LOCAL_SERVICE_DRAM_SIZE.first ||
    key == ConfConstant::OKC_MMC_LOCAL_SERVICE_MAX_DRAM_SIZE.first ||
    key == ConfConstant::OKC_MMC_LOCAL_SERVICE_HBM_SIZE.first ||
    key == ConfConstant::OKC_MMC_LOCAL_SERVICE_MAX_HBM_SIZE.first) {
    auto memSize = ParseMemSize(tempValue);  // "128MB" → 字节数
    if (memSize == UINT64_MAX) {
        return false;  // 转换失败
    }
    mUInt64Items.insert(std::make_pair(key, memSize));
}
```

这种设计表明：
- 对于简单情况，Converter 基类提供了扩展点
- 对于复杂转换（如带单位的内存大小），在 Configuration 类中直接实现更合适

---

## 类图

```
┌─────────────────────────────────────────────────────────────────┐
│                      MmcReferable                                │
│  - IncreaseRef()                                                 │
│  - DecreaseRef()                                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Converter                                  │
│  + Convert(str: string): string (virtual)                        │
│  ~Converter() (virtual)                                          │
└─────────────────────────────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ (自定义转换器) │ │ EnvVariable  │ │ PathExpander │
│              │ │ Converter    │ │ Converter    │
└──────────────┘ └──────────────┘ └──────────────┘
```

---

## 类型别名定义

```cpp
using ConverterPtr = MmcRef<Converter>;
```

这个类型别名提供了：
- 引用计数管理
- 自动内存释放
- 异常安全的指针传递

---

## 与 Validator 的区别

| 特性 | Converter | Validator |
|-----|-----------|-----------|
| **用途** | 转换配置值格式 | 验证配置值合法性 |
| **时机** | 在配置值存储前 | 在配置值存储后 |
| **修改值** | 是 | 否 |
| **返回错误** | 通常返回转换后的值 | 返回 bool 表示是否通过 |

```cpp
// 转换器示例
ConverterPtr converter = ...;
std::string converted = converter->Convert("128MB");  // "134217728"

// 验证器示例
ValidatorPtr validator = ...;
bool valid = validator->Validate(134217728);  // true/false
```
