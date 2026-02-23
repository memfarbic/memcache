# config 模块文档

## 模块概述

`config` 模块负责 MemCache_Hybrid 的配置管理，包括配置文件解析、验证、转换和存储。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/config/`

## 文件列表

### 头文件 (.h)
- `mmc_config_const.h` - 配置常量定义
- `mmc_kv_parser.h` - 键值对解析器
- `mmc_config_validator.h` - 配置验证器
- `mmc_config_convertor.h` - 配置转换器
- `mmc_configuration.h` - 配置管理类
- `mmc_functions.h` - 配置工具函数

### 源文件 (.cpp)
- `mmc_kv_parser.cpp` - KV 解析器实现
- `mmc_configuration.cpp` - 配置管理实现
- `mmc_functions.cpp` - 工具函数实现

---

## 详细文档

### mmc_config_const.h

**功能**: 定义所有配置项的键名和默认值。

**命名空间**: `ock::mmc::ConfConstant`

**逐函数解读**: [mmc_config_const_h.md](mmc_config_const_h.md)

**元服务配置常量**:
```cpp
// 服务地址
constexpr auto OCK_MMC_META_SERVICE_URL = std::make_pair("ock.mmc.meta_service_url", "tcp://127.0.0.1:5000");
constexpr auto OCK_MMC_META_SERVICE_CONFIG_STORE_URL = std::make_pair("ock.mmc.meta_service.config_store_url", "tcp://127.0.0.1:6000");
constexpr auto OCK_MMC_META_SERVICE_HTTP_URL = std::make_pair("ock.mmc.meta_service.metrics_url", "127.0.0.1:8000");

// 高可用
constexpr auto OCK_MMC_META_HA_ENABLE = std::make_pair("ock.mmc.meta.ha.enable", false);

// 驱逐阈值
constexpr auto OKC_MMC_EVICT_THRESHOLD_HIGH = std::make_pair("ock.mmc.evict_threshold_high", 70);
constexpr auto OKC_MMC_EVICT_THRESHOLD_LOW = std::make_pair("ock.mmc.evict_threshold_low", 60);

// 日志配置
constexpr auto OCK_MMC_LOG_LEVEL = std::make_pair("ock.mmc.log_level", "info");
constexpr auto OCK_MMC_LOG_PATH = std::make_pair("ock.mmc.log_path", "/var/log/memcache_hybrid");
constexpr auto OCK_MMC_LOG_ROTATION_FILE_SIZE = std::make_pair("ock.mmc.log_rotation_file_size", 20);
constexpr auto OCK_MMC_LOG_ROTATION_FILE_COUNT = std::make_pair("ock.mmc.log_rotation_file_count", 50);
```

**TLS 配置常量**:
```cpp
constexpr auto OCK_MMC_TLS_ENABLE = std::make_pair("ock.mmc.tls.enable", false);
constexpr auto OCK_MMC_TLS_CA_PATH = std::make_pair("ock.mmc.tls.ca.path", "");
constexpr auto OCK_MMC_TLS_CRL_PATH = std::make_pair("ock.mmc.tls.ca.crl.path", "");
constexpr auto OCK_MMC_TLS_CERT_PATH = std::make_pair("ock.mmc.tls.cert.path", "");
constexpr auto OCK_MMC_TLS_KEY_PATH = std::make_pair("ock.mmc.tls.key.path", "");
// ... 更多 TLS 配置
```

**本地服务配置常量**:
```cpp
constexpr auto OKC_MMC_LOCAL_SERVICE_WORLD_SIZE = std::make_pair("ock.mmc.local_service.world_size", 16);
constexpr auto OKC_MMC_LOCAL_SERVICE_PROTOCOL = std::make_pair("ock.mmc.local_service.protocol", "host_rdma");
constexpr auto OKC_MMC_LOCAL_SERVICE_DRAM_SIZE = std::make_pair("ock.mmc.local_service.dram.size", "128MB");
constexpr auto OKC_MMC_LOCAL_SERVICE_MAX_DRAM_SIZE = std::make_pair("ock.mmc.local_service.max.dram.size", "64GB");
constexpr auto OKC_MMC_LOCAL_SERVICE_HBM_SIZE = std::make_pair("ock.mmc.local_service.hbm.size", "0");
constexpr auto OKC_MMC_LOCAL_SERVICE_MAX_HBM_SIZE = std::make_pair("ock.mmc.local_service.max.hbm.size", "0");
```

**客户端配置常量**:
```cpp
constexpr auto OKC_MMC_CLIENT_RETRY_MILLISECONDS = std::make_pair("ock.mmc.client.retry_milliseconds", 0);
constexpr auto OCK_MMC_CLIENT_TIMEOUT_SECONDS = std::make_pair("ock.mmc.client.timeout.seconds", 60);
constexpr auto OCK_MMC_CLIENT_READ_THREAD_POOL_SIZE = std::make_pair("ock.mmc.client.read_thread_pool.size", 32);
constexpr auto OCK_MMC_CLIENT_WRITE_THREAD_POOL_SIZE = std::make_pair("ock.mmc.client.write_thread_pool.size", 4);
constexpr auto OCK_MMC_CLIENT_AGGREGATE_IO = std::make_pair("ock.mmc.client.aggregate.io", true);
constexpr auto OCK_MMC_CLIENT_AGGREGATE_NUM = std::make_pair("ock.mmc.client.aggregate.num", 122);
```

**范围限制常量**:
```cpp
// 日志轮转
constexpr int MIN_LOG_ROTATION_FILE_SIZE = 1;
constexpr int MAX_LOG_ROTATION_FILE_SIZE = 500;
constexpr int MIN_LOG_ROTATION_FILE_COUNT = 1;
constexpr int MAX_LOG_ROTATION_FILE_COUNT = 50;

// 设备配置
constexpr int MIN_DEVICE_ID = 0;
constexpr int MAX_DEVICE_ID = 383;
constexpr int MIN_WORLD_SIZE = 1;
constexpr int MAX_WORLD_SIZE = 1024;

// 驱逐阈值
constexpr int MIN_EVICT_THRESHOLD = 1;
constexpr int MAX_EVICT_THRESHOLD = 100;

// 超时和重试
constexpr int MIN_RETRY_MS = 0;
constexpr int MAX_RETRY_MS = 600000;
constexpr int MIN_TIMEOUT_SEC = 1;
constexpr int MAX_TIMEOUT_SEC = 600;

// 线程池
constexpr int MIN_THREAD_POOL_SIZE = 1;
constexpr int MAX_THREAD_POOL_SIZE = 64;
constexpr int MAX_AGGREGATE_NUM = 131072; // 128K

// 内存大小
constexpr uint64_t MAX_DRAM_SIZE = 1TB;
constexpr uint64_t MAX_HBM_SIZE = 1TB;
```

**内存单位常量**:
```cpp
constexpr uint64_t KB_MEM_BYTES = 1024ULL;
constexpr uint64_t MB_MEM_BYTES = 1024ULL * 1024ULL;
constexpr uint64_t GB_MEM_BYTES = 1024ULL * 1024ULL * 1024ULL;
constexpr uint64_t TB_MEM_BYTES = 1024ULL * 1024ULL * 1024ULL * 1024ULL;
constexpr uint64_t MEM_2MB_BYTES = 2ULL * 1024ULL * 1024ULL;
constexpr uint64_t MEM_128MB_BYTES = 128ULL * 1024ULL * 1024ULL;
```

---

### mmc_kv_parser.h / mmc_kv_parser.cpp

**功能**: 键值对配置文件解析器。

**逐函数解读**:
- [mmc_kv_parser_h.md](mmc_kv_parser_h.md)
- [mmc_kv_parser_cpp.md](mmc_kv_parser_cpp.md)

**数据结构**:
```cpp
struct KvPairs {
    std::string name;   // 键名
    std::string value;  // 键值
};
```

**KVParser 类**:
```cpp
class KVParser {
public:
    KVParser();
    ~KVParser();

    // 从文件加载配置
    Result FromFile(const std::string &filePath);

    // 获取配置项
    Result GetItem(const std::string &key, std::string &outValue);

    // 设置配置项
    Result SetItem(const std::string &key, const std::string &value);

    // 获取配置项数量
    uint32_t Size();

    // 按索引获取配置项
    void GetI(const uint32_t index, std::string &outKey, std::string &outValue);

    // 打印所有配置项
    void Dump();

    // 检查必需的配置项是否已设置
    bool CheckSet(const std::vector<std::string> &keys);

private:
    Result ParseLine(std::string &strLine);  // 解析单行

    std::map<std::string, uint32_t> mItemsIndex;     // 键到索引的映射
    std::vector<KvPair *> mItems;                    // 配置项列表
    std::unordered_map<std::string, bool> mGotKeys;  // 已获取的键
    Lock mLock;                                      // 线程安全锁
};
```

**配置文件格式**:
```ini
# 这是注释
ock.mmc.meta_service_url = tcp://127.0.0.1:5000
ock.mmc.log_level = info
ock.mmc.local_service.dram.size = 128MB
```

**实现细节** (`mmc_kv_parser.cpp`):
- `FromFile()`: 打开文件，逐行解析，限制文件大小 10MB，最大行数 10000
- `ParseLine()`: 跳过空行和注释行(#开头)，按 `=` 分割键值对
- `SetItem()`: 检查键重复，动态分配 KvPair
- `CheckSet()`: 验证必需的配置项是否已设置

---

### mmc_config_validator.h

**功能**: 配置验证器，用于验证配置值的合法性。

**逐函数解读**: [mmc_config_validator_h.md](mmc_config_validator_h.md)

**Validator 基类**:
```cpp
class Validator : public MmcReferable {
public:
    virtual bool Initialize() = 0;  // 初始化验证器
    virtual bool Validate(const std::string &value);  // 验证字符串
    virtual bool Validate(int value);                 // 验证整数
    virtual bool Validate(float value);               // 验证浮点数
    virtual bool Validate(long unsigned value);       // 验证无符号长整数

    const std::string &ErrorMessage();  // 获取错误消息

protected:
    explicit Validator(std::string name);
    std::string mName;      // 验证项名称
    std::string mErrMsg;    // 错误消息
};
```

**VNoCheck - 不检查验证器**:
```cpp
class VNoCheck : public Validator {
public:
    static ValidatorPtr Create(const std::string &name = "");
    bool Initialize() override;  // 直接返回 true
};
```

**VStrEnum - 枚举字符串验证器**:
```cpp
class VStrEnum : public Validator {
public:
    static ValidatorPtr Create(const std::string &name, const std::string &enumStr,
                                const bool caseSensitive = false);
    bool Initialize() override;  // 解析枚举字符串
    bool Validate(const std::string &value) override;  // 验证值是否在枚举中

private:
    std::string mEnumString;  // 枚举字符串，格式 "val1||val2||val3"
    bool caseSensitive;       // 是否区分大小写
};
```

**使用示例**:
```cpp
// 验证日志级别：debug||info||warn||error
VStrEnum::Create("log_level", "debug||info||warn||error", false);
```

**VStrInSet - 集合字符串验证器**:
```cpp
class VStrInSet : public Validator {
public:
    static ValidatorPtr Create(const std::string &name, const std::string &enumStr);
    bool Validate(const std::string &value) override;  // 验证值(支持 | 分隔的多值)

private:
    std::set<std::string> validEnumSet;  // 有效值集合
};
```

**VStrNotNull - 非空字符串验证器**:
```cpp
class VStrNotNull : public Validator {
public:
    static ValidatorPtr Create(const std::string &name);
    bool Validate(const std::string &value) override;  // 验证字符串非空
};
```

**VStrLength - 字符串长度验证器**:
```cpp
class VStrLength : public Validator {
public:
    static ValidatorPtr Create(const std::string &name, const unsigned long lenLimit);
    bool Validate(const std::string &value) override;  // 验证长度不超过限制

private:
    unsigned long mLengthLimit;  // 长度限制
};
```

**VIntRange - 整数范围验证器**:
```cpp
class VIntRange : public Validator {
public:
    static ValidatorPtr Create(const std::string &name, const int &start, const int &end);
    bool Initialize() override;  // 验证 start < end
    bool Validate(int value) override;  // 验证值在 [start, end] 范围内

private:
    int mStart;  // 起始值
    int mEnd;    // 结束值
};
```

**VUInt64Range - 无符号64位整数范围验证器**:
```cpp
class VUInt64Range : public Validator {
public:
    static ValidatorPtr Create(const std::string &name, const uint64_t &start, const uint64_t &end);
    bool Validate(const uint64_t value) override;

private:
    uint64_t mStart;
    uint64_t mEnd;
};
```

**VPathAccess - 路径访问验证器**:
```cpp
class VPathAccess : public Validator {
public:
    static ValidatorPtr Create(const std::string &name, int flag);
    bool Validate(const std::string &path) override;  // 验证路径可访问

private:
    int mFlag;  // access() 的标志 (R_OK, W_OK, X_OK, F_OK)
    const std::vector<std::string> forbiddenWords{".."};  // 禁止的路径词
};
```

---

### mmc_config_convertor.h

**功能**: 配置值转换器。

**逐函数解读**: [mmc_config_convertor_h.md](mmc_config_convertor_h.md)

**Converter 基类**:
```cpp
class Converter : public MmcReferable {
public:
    virtual std::string Convert(const std::string &str);  // 默认不转换
};
```

**说明**: 可扩展用于实现自定义的配置值转换逻辑。

---

### mmc_configuration.h / mmc_configuration.cpp

**功能**: 配置管理类，提供配置的加载、验证、获取和设置。

**逐函数解读**:
- [mmc_configuration_h.md](mmc_configuration_h.md)
- [mmc_configuration_cpp.md](mmc_configuration_cpp.md)

**枚举类型**:
```cpp
// 内存单位
enum class MemUnit { B, KB, MB, GB, TB, UNKNOWN };

// 配置值类型
enum class ConfValueType {
    VINT = 0,     // 整数
    VFLOAT = 1,   // 浮点数
    VSTRING = 2,  // 字符串
    VBOOL = 3,    // 布尔值
    VUINT64 = 4,  // 无符号64位整数
};
```

**Configuration 类**:
```cpp
class Configuration : public MmcReferable {
public:
    ~Configuration();

    // 从文件加载配置
    bool LoadFromFile(const std::string &filePath);

    // 获取配置值(带默认值)
    int32_t GetInt(const std::pair<const char *, int32_t> &item);
    float GetFloat(const std::pair<const char *, float> &item);
    std::string GetString(const std::pair<const char *, const char *> &item);
    bool GetBool(const std::pair<const char *, bool> &item);
    uint64_t GetUInt64(const std::pair<const char *, uint64_t> &item);
    uint64_t GetUInt64(const char *key, uint64_t defaultValue);

    // 设置配置值
    void Set(const std::string &key, int32_t value);
    void Set(const std::string &key, float value);
    void Set(const std::string &key, const std::string &value);
    void Set(const std::string &key, bool value);
    void Set(const std::string &key, uint64_t value);

    // 自动类型转换设置
    bool SetWithTypeAutoConvert(const std::string &key, const std::string &value);

    // 添加配置项定义
    void AddIntConf(const std::pair<std::string, int> &pair,
                    const ValidatorPtr &validator = nullptr, uint32_t flag = CONF_MUST);
    void AddStrConf(const std::pair<std::string, std::string> &pair,
                    const ValidatorPtr &validator = nullptr, uint32_t flag = CONF_MUST);
    void AddBoolConf(const std::pair<std::string, bool> &pair,
                     const ValidatorPtr &validator = nullptr, uint32_t flag = CONF_MUST);
    void AddUInt64Conf(const std::pair<std::string, uint64_t> &pair,
                       const ValidatorPtr &validator = nullptr, uint32_t flag = CONF_MUST);
    void AddConverter(const std::string &key, const ConverterPtr &converter);
    void AddPathConf(const std::pair<std::string, std::string> &pair,
                     const ValidatorPtr &validator = nullptr, uint32_t flag = CONF_MUST);

    // 验证配置
    std::vector<std::string> ValidateConf();

    // 获取 TLS 配置
    void GetAccTlsConfig(mmc_tls_config &tlsConfig);
    void GetHcomTlsConfig(mmc_tls_config &tlsConfig);
    void GetConfigStoreTlsConfig(mmc_tls_config &tlsConfig);

    // TLS 配置验证
    static int ValidateTLSConfig(const mmc_tls_config &tlsConfig);

    // 获取二进制目录和日志路径
    const std::string GetBinDir();
    const std::string GetLogPath(const std::string &logPath);
    static int ValidateLogPathConfig(const std::string &logPath);

    bool Initialized() const;  // 是否已初始化

protected:
    virtual void LoadDefault() {}  // 子类重写，加载默认配置

private:
    bool SetWithStrAutoConvert(const std::string &key, const std::string &value);
    uint64_t ParseMemSize(const std::string &memStr);  // 解析内存大小字符串
    MemUnit ParseMemUnit(const std::string &unit);     // 解析内存单位

    // 内部存储
    std::map<std::string, int32_t> mIntItems;
    std::map<std::string, float> mFloatItems;
    std::map<std::string, std::string> mStrItems;
    std::map<std::string, bool> mBoolItems;
    std::map<std::string, uint64_t> mUInt64Items;
    std::map<std::string, ConfValueType> mValueTypes;
    std::map<std::string, ValidatorPtr> mValueValidator;
    std::map<std::string, ConverterPtr> mValueConverter;
};
```

**MetaServiceConfig - 元服务配置类**:
```cpp
class MetaServiceConfig final : public Configuration {
public:
    void LoadDefault() override;  // 加载元服务默认配置项
    void GetMetaServiceConfig(mmc_meta_service_config_t &config);  // 获取配置结构
};
```

**ClientConfig - 客户端/本地服务配置类**:
```cpp
class ClientConfig final : public Configuration {
public:
    void LoadDefault() override;  // 加载客户端默认配置项
    void GetLocalServiceConfig(mmc_local_service_config_t &config);  // 获取本地服务配置
    void GetClientConfig(mmc_client_config_t &config);  // 获取客户端配置

    static Result ValidateLocalServiceConfig(mmc_local_service_config_t &config);
};
```

**配置加载流程**:
```
LoadFromFile()
    → KVParser::FromFile()  (解析配置文件)
    → LoadDefault()         (注册默认配置项)
    → LoadConfigurations()  (加载配置值)
    → ValidateConf()        (验证配置)
```

---

### mmc_functions.h / mmc_functions.cpp

**功能**: 配置相关的工具函数。

**逐函数解读**:
- [mmc_functions_h.md](mmc_functions_h.md)
- [mmc_functions_cpp.md](mmc_functions_cpp.md)

**字符串处理**:
```cpp
void OckTrimString(std::string &str);  // 去除首尾空白
void SplitStr(const std::string &str, const std::string &separator,
              std::set<std::string> &result);  // 分割字符串到集合
void SplitStr(const std::string &str, const std::string &separator,
              std::vector<std::string> &result);  // 分割字符串到向量
```

**类型转换**:
```cpp
bool OckStol(const std::string &str, long &value);  // 字符串转长整型
bool OckStoULL(const std::string &str, uint64_t &value);  // 字符串转无符号64位
bool OckStof(const std::string &str, float &value);  // 字符串转浮点数
bool IsBool(const std::string &str, bool &value);   // 字符串转布尔值
```

**布尔值映射**:
```cpp
const std::unordered_map<std::string, bool> Str2Bool = {
    {"0", false}, {"1", true},
    {"false", false}, {"true", true}
};
```

**路径处理**:
```cpp
bool GetRealPath(std::string &path);  // 获取真实路径(解析符号链接)
```

---

## 配置文件示例

```ini
# MemCache Hybrid 配置文件示例

# 元服务配置
ock.mmc.meta_service_url = tcp://127.0.0.1:5000
ock.mmc.meta_service.config_store_url = tcp://127.0.0.1:6000
ock.mmc.meta_service.metrics_url = 127.0.0.1:8000
ock.mmc.meta.ha.enable = false

# 日志配置
ock.mmc.log_level = info
ock.mmc.log_path = /var/log/memcache_hybrid
ock.mmc.log_rotation_file_size = 20
ock.mmc.log_rotation_file_count = 50

# 驱逐阈值
ock.mmc.evict_threshold_high = 70
ock.mmc.evict_threshold_low = 60

# TLS 配置
ock.mmc.tls.enable = false
ock.mmc.tls.ca.path = /path/to/ca.crt
ock.mmc.tls.cert.path = /path/to/cert.pem
ock.mmc.tls.key.path = /path/to/key.pem

# 本地服务配置
ock.mmc.local_service.world_size = 16
ock.mmc.local_service.protocol = host_rdma
ock.mmc.local_service.dram.size = 128MB
ock.mmc.local_service.max.dram.size = 64GB
ock.mmc.local_service.hbm.size = 0
ock.mmc.local_service.max.hbm.size = 0

# 客户端配置
ock.mmc.client.retry_milliseconds = 0
ock.mmc.client.timeout.seconds = 60
ock.mmc.client.read_thread_pool.size = 32
ock.mmc.client.write_thread_pool.size = 4
ock.mmc.client.aggregate.io = true
ock.mmc.client.aggregate.num = 122
```

---

## 使用示例

```cpp
#include "mmc_configuration.h"

using namespace ock::mmc;

// 创建配置对象
auto config = MmcMakeRef<MetaServiceConfig>();

// 从文件加载
if (!config->LoadFromFile("/path/to/config.ini")) {
    // 处理错误
}

// 验证配置
std::vector<std::string> errors = config->ValidateConf();
if (!errors.empty()) {
    for (const auto &err : errors) {
        MMC_LOG_ERROR(err);
    }
}

// 获取配置值
std::string url = config->GetString(ConfConstant::OCK_MMC_META_SERVICE_URL);
int32_t highThreshold = config->GetInt(ConfConstant::OKC_MMC_EVICT_THRESHOLD_HIGH);

// 转换为 C 结构
mmc_meta_service_config_t metaConfig{};
config->GetMetaServiceConfig(metaConfig);
```

---

## 数据流图

```
配置文件 (config.ini)
        ↓
KVParser::FromFile()
        ↓
键值对解析 (ParseLine)
        ↓
mItems (存储)
        ↓
Configuration::LoadFromFile()
        ↓
LoadDefault() (注册配置项)
        ↓
LoadConfigurations() (类型转换+验证)
        ↓
ValidateConf() (验证)
        ↓
mIntItems / mStrItems / ...
        ↓
GetInt() / GetString() / ...
```
