# mmc_configuration.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/config/mmc_configuration.h`
- **文件用途**: 定义配置管理类，支持配置文件加载、类型转换、验证和获取
- **依赖项**:
  - `mmc_lock.h` - 锁机制
  - `mmc_ref.h` - 引用计数智能指针
  - `mmc_config_validator.h` - 配置验证器
  - `mmc_config_convertor.h` - 配置转换器
  - `mmc_config_const.h` - 配置常量
  - `mmc_def.h` - 配置相关结构定义
  - `mmc_logger.h` - 日志系统
  - `mmc_types.h` - 类型定义
  - `mmc_last_error.h` - 错误处理
  - `smem_bm_def.h` - SMEM Blob Manager 定义
  - `common/mmc_functions.h` - 通用工具函数

---

## 常量定义

**声明位置**: 行 34-40
```cpp
constexpr uint32_t CONF_MUST = 1;                              // 必需配置标志
constexpr uint64_t DRAM_SIZE_ALIGNMENT = 2097152;              // DRAM 2MB 对齐
constexpr uint64_t HBM_SIZE_ALIGNMENT = 2097152;               // HBM 2MB 对齐

const std::string BOOL_ENUM_STR = "false||true";               // 布尔枚举字符串
const std::string LOG_LEVEL_ENUM_STR = "debug||info||warn||error";  // 日志级别枚举
const std::string LOCAL_SERVER_PROTOCAL_ENUM_STR = "host_rdma||host_urma||host_tcp||device_rdma||device_sdma";
```

---

## 枚举类型

### MemUnit

**声明位置**: 行 43
**完整签名**:
```cpp
enum class MemUnit { B, KB, MB, GB, TB, UNKNOWN };
```
**功能描述**: 内存单位枚举，用于解析带单位的内存大小字符串

### ConfValueType

**声明位置**: 行 45-51
**完整签名**:
```cpp
enum class ConfValueType {
    VINT = 0,      // 整数类型
    VFLOAT = 1,    // 浮点数类型
    VSTRING = 2,   // 字符串类型
    VBOOL = 3,     // 布尔类型
    VUINT64 = 4,   // 无符号64位整数类型
};
```
**功能描述**: 配置值类型枚举

---

## 辅助函数

### StringToUpper()

**声明位置**: 行 53
**完整签名**:
```cpp
void StringToUpper(std::string &str);
```
**功能描述**: 将字符串转换为大写（在 .cpp 中实现）

---

## 类型别名

**声明位置**: 行 55-56
```cpp
class Configuration;
using ConfigurationPtr = MmcRef<Configuration>;
```

---

## Configuration 类

### 类概述

**声明位置**: 行 58-165
**完整签名**:
```cpp
class Configuration : public MmcReferable
```
**功能描述**: 配置管理基类，提供配置文件的加载、解析、验证和访问功能

**特性**:
- 支持多种数据类型（int, float, string, bool, uint64）
- 支持配置验证器
- 支持配置转换器
- 线程安全（使用互斥锁）
- 禁止拷贝和移动

---

### Configuration::Configuration()

**声明位置**: 行 60
**完整签名**:
```cpp
Configuration() = default;
```
**功能描述**: 默认构造函数

---

### Configuration::~Configuration()

**声明位置**: 行 61
**完整签名**:
```cpp
~Configuration() override;
```
**功能描述**: 析构函数，释放所有验证器和转换器资源

---

### Configuration 拷贝/移动删除

**声明位置**: 行 64-69
**完整签名**:
```cpp
Configuration(const Configuration &) = delete;
Configuration &operator=(const Configuration &) = delete;
Configuration(const Configuration &&) = delete;
Configuration &operator=(const Configuration &&) = delete;
```
**功能描述**: 禁止拷贝和移动操作

---

### Configuration::LoadFromFile()

**声明位置**: 行 71
**完整签名**:
```cpp
bool LoadFromFile(const std::string &filePath);
```
**功能描述**: 从文件加载配置
**参数**:
- `filePath`: 配置文件路径
**返回值**:
- `true` - 加载成功
- `false` - 加载失败

---

### Configuration::GetInt()

**声明位置**: 行 73
**完整签名**:
```cpp
int32_t GetInt(const std::pair<const char *, int32_t> &item);
```
**功能描述**: 获取整数配置值，如果不存在返回默认值
**参数**:
- `item`: pair，first 为键名，second 为默认值
**返回值**: 配置值或默认值

**使用示例**:
```cpp
int port = GetInt(ConfConstant::OCK_MMC_META_SERVICE_PORT);
// 或使用自定义默认值
int port = GetInt({"ock.mmc.port", 8080});
```

---

### Configuration::GetFloat()

**声明位置**: 行 74
**完整签名**:
```cpp
float GetFloat(const std::pair<const char *, float> &item);
```
**功能描述**: 获取浮点数配置值
**参数**:
- `item`: pair，first 为键名，second 为默认值
**返回值**: 配置值或默认值

---

### Configuration::GetString()

**声明位置**: 行 75
**完整签名**:
```cpp
std::string GetString(const std::pair<const char *, const char *> &item);
```
**功能描述**: 获取字符串配置值
**参数**:
- `item`: pair，first 为键名，second 为默认值
**返回值**: 配置值或默认值

---

### Configuration::GetBool()

**声明位置**: 行 76
**完整签名**:
```cpp
bool GetBool(const std::pair<const char *, bool> &item);
```
**功能描述**: 获取布尔配置值
**参数**:
- `item`: pair，first 为键名，second 为默认值
**返回值**: 配置值或默认值

---

### Configuration::GetUInt64() - 重载1

**声明位置**: 行 77
**完整签名**:
```cpp
uint64_t GetUInt64(const std::pair<const char *, uint64_t> &item);
```
**功能描述**: 获取 uint64_t 配置值（使用常量定义的默认值）

---

### Configuration::GetUInt64() - 重载2

**声明位置**: 行 78
**完整签名**:
```cpp
uint64_t GetUInt64(const char *key, uint64_t defaultValue);
```
**功能描述**: 获取 uint64_t 配置值（使用自定义默认值）
**参数**:
- `key`: 配置项键名
- `defaultValue`: 默认值
**返回值**: 配置值或默认值

---

### Configuration::Set() - 整数重载

**声明位置**: 行 80
**完整签名**:
```cpp
void Set(const std::string &key, int32_t value);
```
**功能描述**: 设置整数配置项的值
**参数**:
- `key`: 配置项键名
- `value`: 要设置的值

**注意事项**:
- 只能修改已注册的配置项
- 未知键会被忽略

---

### Configuration::Set() - 浮点数重载

**声明位置**: 行 81
**完整签名**:
```cpp
void Set(const std::string &key, float value);
```
**功能描述**: 设置浮点数配置项的值

---

### Configuration::Set() - 字符串重载

**声明位置**: 行 82
**完整签名**:
```cpp
void Set(const std::string &key, const std::string &value);
```
**功能描述**: 设置字符串配置项的值

---

### Configuration::Set() - 布尔重载

**声明位置**: 行 83
**完整签名**:
```cpp
void Set(const std::string &key, bool value);
```
**功能描述**: 设置布尔配置项的值

---

### Configuration::Set() - uint64 重载

**声明位置**: 行 84
**完整签名**:
```cpp
void Set(const std::string &key, uint64_t value);
```
**功能描述**: 设置 uint64_t 配置项的值

---

### Configuration::SetWithTypeAutoConvert()

**声明位置**: 行 86
**完整签名**:
```cpp
bool SetWithTypeAutoConvert(const std::string &key, const std::string &value);
```
**功能描述**: 自动类型转换后设置配置值（内部方法）
**参数**:
- `key`: 配置项键名
- `value`: 字符串形式的配置值
**返回值**:
- `true` - 设置成功
- `false` - 设置失败

---

### Configuration::AddIntConf()

**声明位置**: 行 88-89
**完整签名**:
```cpp
void AddIntConf(const std::pair<std::string, int> &pair, const ValidatorPtr &validator = nullptr,
                uint32_t flag = CONF_MUST);
```
**功能描述**: 注册整数配置项
**参数**:
- `pair`: 配置项（键名和默认值）
- `validator`: 验证器（可选）
- `flag`: 标志位（如 CONF_MUST 表示必需）

---

### Configuration::AddStrConf()

**声明位置**: 行 90-91
**完整签名**:
```cpp
void AddStrConf(const std::pair<std::string, std::string> &pair, const ValidatorPtr &validator = nullptr,
                uint32_t flag = CONF_MUST);
```
**功能描述**: 注册字符串配置项

---

### Configuration::AddBoolConf()

**声明位置**: 行 92-93
**完整签名**:
```cpp
void AddBoolConf(const std::pair<std::string, bool> &pair, const ValidatorPtr &validator = nullptr,
                 uint32_t flag = CONF_MUST);
```
**功能描述**: 注册布尔配置项

---

### Configuration::AddUInt64Conf()

**声明位置**: 行 94-95
**完整签名**:
```cpp
void AddUInt64Conf(const std::pair<std::string, uint64_t> &pair, const ValidatorPtr &validator = nullptr,
                   uint32_t flag = CONF_MUST);
```
**功能描述**: 注册 uint64_t 配置项

---

### Configuration::AddConverter()

**声明位置**: 行 96
**完整签名**:
```cpp
void AddConverter(const std::string &key, const ConverterPtr &converter);
```
**功能描述**: 为配置项添加转换器
**参数**:
- `key`: 配置项键名
- `converter`: 转换器指针

---

### Configuration::AddPathConf()

**声明位置**: 行 97-98
**完整签名**:
```cpp
void AddPathConf(const std::pair<std::string, std::string> &pair, const ValidatorPtr &validator = nullptr,
                 uint32_t flag = CONF_MUST);
```
**功能描述**: 注册路径类型配置项（会自动转换为绝对路径）

---

### Configuration::ValidateConf()

**声明位置**: 行 99
**完整签名**:
```cpp
std::vector<std::string> ValidateConf();
```
**功能描述**: 验证所有配置项
**返回值**: 错误信息列表（空列表表示全部通过）

---

### Configuration::GetAccTlsConfig()

**声明位置**: 行 100
**完整签名**:
```cpp
void GetAccTlsConfig(mmc_tls_config &tlsConfig);
```
**功能描述**: 获取 ACC 层 TLS 配置
**参数**:
- `tlsConfig`: 输出参数，存储 TLS 配置

---

### Configuration::GetHcomTlsConfig()

**声明位置**: 行 101
**完整签名**:
```cpp
void GetHcomTlsConfig(mmc_tls_config &tlsConfig);
```
**功能描述**: 获取 HCOM TLS 配置

---

### Configuration::GetConfigStoreTlsConfig()

**声明位置**: 行 102
**完整签名**:
```cpp
void GetConfigStoreTlsConfig(mmc_tls_config &tlsConfig);
```
**功能描述**: 获取配置存储 TLS 配置

---

### Configuration::ValidateTLSConfig()

**声明位置**: 行 104
**完整签名**:
```cpp
static int ValidateTLSConfig(const mmc_tls_config &tlsConfig);
```
**功能描述**: 静态方法，验证 TLS 配置的合法性
**参数**:
- `tlsConfig`: TLS 配置结构
**返回值**:
- `MMC_OK` - 验证通过
- `MMC_ERROR` - 验证失败

---

### Configuration::GetBinDir()

**声明位置**: 行 106
**完整签名**:
```cpp
const std::string GetBinDir();
```
**功能描述**: 获取可执行文件所在目录
**返回值**: 可执行文件目录路径

---

### Configuration::GetLogPath()

**声明位置**: 行 107
**完整签名**:
```cpp
const std::string GetLogPath(const std::string &logPath);
```
**功能描述**: 处理日志路径，支持相对路径转换为绝对路径
**参数**:
- `logPath`: 配置的日志路径
**返回值**: 处理后的绝对路径

---

### Configuration::ValidateLogPathConfig()

**声明位置**: 行 108
**完整签名**:
```cpp
static int ValidateLogPathConfig(const std::string &logPath);
```
**功能描述**: 静态方法，验证日志路径配置
**参数**:
- `logPath`: 日志路径
**返回值**:
- `MMC_OK` - 路径有效
- `MMC_ERROR` - 路径无效或为符号链接

---

### Configuration::Initialized()

**声明位置**: 行 110-113
**完整签名**:
```cpp
bool Initialized() const
{
    return mInitialized;
}
```
**功能描述**: 检查配置是否已初始化
**返回值**:
- `true` - 已初始化
- `false` - 未初始化

---

## 私有方法

### Configuration::SetWithStrAutoConvert()

**声明位置**: 行 116
**完整签名**:
```cpp
bool SetWithStrAutoConvert(const std::string &key, const std::string &value);
```
**功能描述**: 字符串类型配置的自动转换（处理内存大小、路径等）

---

### Configuration::ParseMemSize()

**声明位置**: 行 117
**完整签名**:
```cpp
uint64_t ParseMemSize(const std::string &memStr);
```
**功能描述**: 解析带单位的内存大小字符串（如 "128MB"）
**返回值**: 字节数，失败返回 UINT64_MAX

---

### Configuration::ParseMemUnit()

**声明位置**: 行 118
**完整签名**:
```cpp
MemUnit ParseMemUnit(const std::string &unit);
```
**功能描述**: 解析内存单位字符串
**返回值**: MemUnit 枚举值

---

### Configuration::SetValidator()

**声明位置**: 行 120
**完整签名**:
```cpp
void SetValidator(const std::string &key, const ValidatorPtr &validator, uint32_t flag);
```
**功能描述**: 设置配置项的验证器

---

### Configuration::AddValidateError()

**声明位置**: 行 122-132
**完整签名**:
```cpp
template<class T>
static void AddValidateError(const ValidatorPtr &validator, std::vector<std::string> &errors, const T &iter)
```
**功能描述**: 模板方法，添加验证错误信息

---

### Configuration::ValidateOneType()

**声明位置**: 行 133-134
**完整签名**:
```cpp
void ValidateOneType(const std::string &key, const ValidatorPtr &validator, std::vector<std::string> &errors,
                     ConfValueType &vType);
```
**功能描述**: 验证单个配置项

---

### Configuration::ValidateItem()

**声明位置**: 行 136
**完整签名**:
```cpp
void ValidateItem(const std::string &itemKey, std::vector<std::string> &errors);
```
**功能描述**: 验证指定配置项

---

### Configuration::LoadConfigurations()

**声明位置**: 行 138
**完整签名**:
```cpp
void LoadConfigurations();
```
**功能描述**: 加载默认配置并初始化

---

### Configuration::LoadDefault()

**声明位置**: 行 140
**完整签名**:
```cpp
virtual void LoadDefault() {}
```
**功能描述**: 纯虚函数，由派生类实现以注册默认配置项

---

## 成员变量

**声明位置**: 行 142-164
```cpp
std::string mConfigPath;                           // 配置文件路径

std::map<std::string, int32_t> mIntItems;          // 整数配置项
std::map<std::string, float> mFloatItems;          // 浮点数配置项
std::map<std::string, std::string> mStrItems;      // 字符串配置项
std::map<std::string, bool> mBoolItems;            // 布尔配置项
std::map<std::string, uint64_t> mUInt64Items;      // uint64配置项
std::map<std::string, std::string> mAllItems;      // 所有配置项

std::map<std::string, ConfValueType> mValueTypes;  // 配置值类型
std::map<std::string, ValidatorPtr> mValueValidator;     // 验证器映射
std::map<std::string, ConverterPtr> mValueConverter;     // 转换器映射

std::vector<std::pair<std::string, std::string>> mServiceList;  // 服务列表
std::vector<std::string> mMustKeys;               // 必需配置键列表
std::vector<std::string> mLoadDefaultErrors;      // 加载默认配置错误

std::vector<std::string> mPathConfs;              // 路径配置列表
std::vector<std::string> mExceptPrintConfs;       // 不打印的配置列表
std::vector<std::string> mInvalidSetConfs;        // 禁止设置的配置列表

bool mInitialized = false;                        // 初始化标志
Lock mLock;                                       // 互斥锁
```

---

## 派生类：MetaServiceConfig

### 类概述

**声明位置**: 行 167-236
**完整签名**:
```cpp
class MetaServiceConfig final : public Configuration
```
**功能描述**: 元服务配置类，定义元服务相关的所有配置项

---

### MetaServiceConfig::LoadDefault()

**声明位置**: 行 169-213
**完整签名**:
```cpp
void LoadDefault() override
```
**功能描述**: 注册元服务的默认配置项

**注册的配置项**:
1. **服务地址配置**
   - `OCK_MMC_META_SERVICE_URL` - 元服务 URL
   - `OCK_MMC_META_SERVICE_CONFIG_STORE_URL` - 配置存储 URL
   - `OCK_MMC_META_SERVICE_HTTP_URL` - HTTP 指标 URL

2. **HA 配置**
   - `OCK_MMC_META_HA_ENABLE` - 高可用开关

3. **日志配置**
   - `OCK_MMC_LOG_LEVEL` - 日志级别
   - `OCK_MMC_LOG_PATH` - 日志路径
   - `OCK_MMC_LOG_ROTATION_FILE_SIZE` - 日志文件大小
   - `OCK_MMC_LOG_ROTATION_FILE_COUNT` - 日志文件数量

4. **驱逐阈值**
   - `OKC_MMC_EVICT_THRESHOLD_HIGH` - 高阈值
   - `OKC_MMC_EVICT_THRESHOLD_LOW` - 低阈值

5. **TLS 配置（ACC）**
   - `OCK_MMC_TLS_ENABLE` - TLS 开关
   - `OCK_MMC_TLS_CA_PATH` - CA 路径
   - `OCK_MMC_TLS_CRL_PATH` - CRL 路径
   - `OCK_MMC_TLS_CERT_PATH` - 证书路径
   - `OCK_MMC_TLS_KEY_PATH` - 私钥路径
   - `OCK_MMC_TLS_KEY_PASS_PATH` - 密码路径
   - `OCK_MMC_TLS_PACKAGE_PATH` - OpenSSL 目录
   - `OCK_MMC_TLS_DECRYPTER_PATH` - 解密器库路径

6. **TLS 配置（Config Store）**
   - `OCK_MMC_CS_TLS_*` 系列配置

---

### MetaServiceConfig::GetMetaServiceConfig()

**声明位置**: 行 215-235
**完整签名**:
```cpp
void GetMetaServiceConfig(mmc_meta_service_config_t &config);
```
**功能描述**: 将配置填充到 C 结构体中
**参数**:
- `config`: 输出参数，元服务配置结构体

---

## 派生类：ClientConfig

### 类概述

**声明位置**: 行 238-408
**完整签名**:
```cpp
class ClientConfig final : public Configuration
```
**功能描述**: 客户端配置类，定义客户端和本地服务相关的所有配置项

---

### ClientConfig::LoadDefault()

**声明位置**: 行 240-307
**完整签名**:
```cpp
void LoadDefault() override
```
**功能描述**: 注册客户端的默认配置项

**注册的配置项**:
1. **元服务配置** - 与 MetaServiceConfig 类似
2. **本地服务配置**
   - `OKC_MMC_LOCAL_SERVICE_WORLD_SIZE` - World Size
   - `OKC_MMC_LOCAL_SERVICE_BM_IP_PORT` - BM IP:Port
   - `OKC_MMC_LOCAL_SERVICE_PROTOCOL` - 传输协议
   - `OKC_MMC_LOCAL_SERVICE_DRAM_SIZE` - DRAM 大小
   - `OKC_MMC_LOCAL_SERVICE_MAX_DRAM_SIZE` - 最大 DRAM
   - `OKC_MMC_LOCAL_SERVICE_HBM_SIZE` - HBM 大小
   - `OKC_MMC_LOCAL_SERVICE_MAX_HBM_SIZE` - 最大 HBM
   - `OKC_MMC_LOCAL_SERVICE_BM_HCOM_URL` - HCOM URL

3. **HCOM TLS 配置**
   - `OCK_MMC_HCOM_TLS_*` 系列配置

4. **客户端配置**
   - `OKC_MMC_CLIENT_RETRY_MILLISECONDS` - 重试超时
   - `OCK_MMC_CLIENT_TIMEOUT_SECONDS` - 超时时间
   - `OCK_MMC_CLIENT_READ_THREAD_POOL_SIZE` - 读线程池
   - `OCK_MMC_CLIENT_AGGREGATE_IO` - 聚合 IO 开关
   - `OCK_MMC_CLIENT_WRITE_THREAD_POOL_SIZE` - 写线程池
   - `OCK_MMC_CLIENT_AGGREGATE_NUM` - 聚合数量

---

### ClientConfig::GetLocalServiceConfig()

**声明位置**: 行 309-330
**完整签名**:
```cpp
void GetLocalServiceConfig(mmc_local_service_config_t &config);
```
**功能描述**: 获取本地服务配置

---

### ClientConfig::GetClientConfig()

**声明位置**: 行 332-346
**完整签名**:
```cpp
void GetClientConfig(mmc_client_config_t &config);
```
**功能描述**: 获取客户端配置

---

### ClientConfig::ValidateLocalServiceConfig()

**声明位置**: 行 348-407
**完整签名**:
```cpp
static Result ValidateLocalServiceConfig(mmc_local_service_config_t &config);
```
**功能描述**: 静态方法，验证本地服务配置的合法性
**参数**:
- `config`: 本地服务配置结构
**返回值**:
- `MMC_OK` - 验证通过
- `MMC_INVALID_PARAM` - 参数无效

**验证内容**:
1. 内存大小对齐（2MB）
2. 内存大小不超过 1TB
3. 最大值不小于初始值
4. DRAM 和 HBM 不能同时为 0
5. TLS 配置验证

---

## 类继承关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                      MmcReferable                                │
│  - IncreaseRef()                                                 │
│  - DecreaseRef()                                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Configuration                                 │
│  + LoadFromFile()                                               │
│  + GetInt/GetFloat/GetString/GetBool/GetUInt64()                │
│  + Set()                                                         │
│  + Add*Conf()                                                    │
│  + ValidateConf()                                               │
│  + Get*TlsConfig()                                              │
│  # LoadDefault() = 0                                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
         ▼                               ▼
┌──────────────────────┐      ┌──────────────────────┐
│  MetaServiceConfig   │      │   ClientConfig       │
│  - 元服务配置         │      │  - 客户端配置        │
│  + LoadDefault()     │      │  - 本地服务配置      │
│  + GetMetaService    │      │  + LoadDefault()     │
│    Config()          │      │  + GetLocalService   │
└──────────────────────┘      │    Config()          │
                              │  + GetClientConfig() │
                              │  + ValidateLocal     │
                              │    ServiceConfig()   │
                              └──────────────────────┘
```

---

## 配置项类型映射

```
配置文件字符串 → ConfValueType → 存储容器
     "123"      →    VINT       → mIntItems
     "1.23"     →    VFLOAT     → mFloatItems
     "hello"    →    VSTRING    → mStrItems
     "true"     →    VBOOL      → mBoolItems
     "123456"   →    VUINT64    → mUInt64Items
     "128MB"    →    VSTRING    → mUInt64Items (特殊处理)
```
