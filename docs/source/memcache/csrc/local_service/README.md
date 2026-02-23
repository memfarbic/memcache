# local_service 模块文档

## 模块概述

`local_service` 模块实现本地服务，负责管理本地内存资源和与 Blob Manager 交互。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/local_service/`

## 文件列表

- `mmc_local_service.h` - 本地服务接口
- `mmc_local_service_default.h` / `mmc_local_service_default.cpp` - 默认实现
- `mmc_local_common.h` - 本地服务通用定义
- `mmc_bm_proxy.h` / `mmc_bm_proxy.cpp` - Blob Manager 代理

---

## mmc_local_service.h

### MmcLocalService 接口

本地服务抽象接口:
```cpp
class MmcLocalService : public MmcReferable {
public:
    // 启动本地服务
    virtual Result Start(const mmc_local_service_config_t &config) = 0;

    // 停止本地服务
    virtual void Stop() = 0;

    // 获取服务名称
    virtual const std::string &Name() const = 0;

    // 获取配置选项
    virtual const mmc_local_service_config_t &Options() const = 0;
};
```

---

## mmc_local_service_default.h

### MmcLocalServiceDefault 类

本地服务默认实现:
```cpp
class MmcLocalServiceDefault final : public MmcLocalService {
public:
    MmcLocalServiceDefault();
    ~MmcLocalServiceDefault() override;

    // 实现接口
    Result Start(const mmc_local_service_config_t &config) override;
    void Stop() override;
    const std::string &Name() const override;
    const mmc_local_service_config_t &Options() const override;

    // 本地存储操作
    Result Put(const std::string &key, const MmcBufferArray &bufArr,
              const mmc_put_options &options, uint32_t flags);
    Result Get(const std::string &key, const MmcBufferArray &bufArr, uint32_t flags);
    Result Remove(const std::string &key, uint32_t flags);
    Result RemoveAll(uint32_t flags);
    Result IsExist(const std::string &key, uint32_t flags);

    // 获取本地服务 ID
    Result GetLocalServiceId(uint32_t &localServiceId);

private:
    MmcBmProxyPtr bmProxy_;                  // Blob Manager 代理
    MmcStorePtr store_;                      // 本地存储
    std::string name_;                       // 服务名称
    mmc_local_service_config_t config_;     // 配置
    bool started_;                           // 启动状态
    uint32_t localServiceId_;                // 本地服务 ID
};
```

---

## mmc_bm_proxy.h

### MmcBmProxy 类

Blob Manager 代理，封装对 SMEM BM 的访问:
```cpp
class MmcBmProxy : public MmcReferable {
public:
    explicit MmcBmProxy(const std::string &name);

    // Blob Manager 初始化
    Result InitBm(const mmc_bm_init_config_t &initConfig,
                  const mmc_bm_create_config_t &createConfig);
    void DestroyBm();

    // 数据操作
    Result Copy(uint64_t srcBmAddr, uint64_t dstBmAddr,
               uint64_t size, smem_bm_copy_type type);
    Result Put(const mmc_buffer *buf, uint64_t bmAddr, uint64_t size);
    Result Get(const mmc_buffer *buf, uint64_t bmAddr, uint64_t size);

    // 异步操作
    Result AsyncPut(const MmcBufferArray &bufArr, const MmcMemBlobDesc &blob);
    Result AsyncGet(const MmcBufferArray &bufArr, const MmcMemBlobDesc &blob);

    // 批量操作
    Result BatchPut(const MmcBufferArray &bufArr, const MmcMemBlobDesc &blob);
    Result BatchGet(const MmcBufferArray &bufArr, const MmcMemBlobDesc &blob);
    Result BatchDataPut(std::vector<void *> &sources,
                        std::vector<void *> &destinations,
                        const std::vector<uint64_t> &sizes, MediaType localMedia);
    Result BatchDataGet(std::vector<void *> &sources,
                        std::vector<void *> &destinations,
                        const std::vector<uint64_t> &sizes, MediaType localMedia);

    // 缓冲区管理
    Result RegisterBuffer(uint64_t addr, uint64_t size);
    Result UnRegisterBuffer(uint64_t addr);

    // 等待异步操作完成
    Result CopyWait();

    // 属性访问
    uint64_t GetGva(MediaType type) const;
    uint64_t GetCapacity(MediaType type) const;
    std::string GetDataOpType() const;
    uint32_t RankId() const;

private:
    Result InternalCreateBm(const mmc_bm_create_config_t &createConfig);

private:
    void *gvas_[MEDIA_NONE];               // 各介质类型的 GVA
    uint64_t spaces_[MEDIA_NONE];          // 各介质类型的容量
    smem_bm_t handle_;                      // SMEM BM 句柄
    std::string name_;
    bool started_;
    std::mutex mutex_;
    uint32_t bmRankId_;
    MediaType mediaType_;
    mmc_bm_create_config_t createConfig_;
};
```

### 配置结构

```cpp
// Blob Manager 初始化配置
typedef struct {
    uint32_t deviceId;
    uint32_t worldSize;
    std::string ipPort;
    std::string hcomUrl;
    int32_t logLevel;
    ExternalLog logFunc;
    uint32_t flags;
    mmc_tls_config hcomTlsConfig;
    mmc_tls_config storeTlsConfig;
} mmc_bm_init_config_t;

// Blob Manager 创建配置
typedef struct {
    uint32_t id;
    uint32_t memberSize;
    std::string dataOpType;
    uint64_t localDRAMSize;
    uint64_t localMaxDRAMSize;
    uint64_t localHBMSize;
    uint64_t localMaxHBMSize;
    uint32_t flags;
} mmc_bm_create_config_t;
```

### MmcBmProxyFactory

工厂类，管理 MmcBmProxy 单例:
```cpp
class MmcBmProxyFactory : public MmcReferable {
public:
    static MmcBmProxyPtr GetInstance(const std::string &key = "");

private:
    static std::map<std::string, MmcRef<MmcBmProxy>> instances_;
    static std::mutex instanceMutex_;
};
```

---

## mmc_local_common.h

### 本地服务通用定义

```cpp
// 本地内存初始化信息
struct MmcLocalMemlInitInfo {
    uint64_t bmAddr_;      // Blob Manager 地址
    uint64_t capacity_;    // 容量
};

// 数据拷贝类型
enum smem_bm_copy_type : int {
    SMEMB_DATA_OP_HOST_TCP = 0,
    SMEMB_DATA_OP_HOST_RDMA,
    SMEMB_DATA_OP_DEVICE_RDMA,
    SMEMB_DATA_OP_SDMA,
    SMEMB_DATA_OP_HOST_URMA,
};
```

---

## 数据流图

```
应用层
    ↓
MmcLocalServiceDefault
    ↓
    ├─→ MmcBmProxy ──→ SMEM BM (本地内存)
    │
    └─→ MmcStore ──→ 本地缓存/索引
```

---

## 使用示例

```cpp
// 创建本地服务
auto localService = MmcMakeRef<MmcLocalServiceDefault>();

// 配置
mmc_local_service_config_t config{};
config.deviceId = 0;
config.rankId = 0;
config.worldSize = 4;
// ... 配置初始化

// 启动服务
localService->Start(config);

// 获取 BM 代理
auto bmProxy = MmcBmProxyFactory::GetInstance();
bmProxy->InitBm(initConfig, createConfig);

// 数据操作
mmc_buffer buf{addr, type, offset, size};
bmProxy->Put(&buf, bmAddr, size);
bmProxy->Get(&buf, bmAddr, size);

// 批量操作
MmcBufferArray bufArr(buffers);
MmcMemBlobDesc blob{rank, gva, size, mediaType};
bmProxy->BatchPut(bufArr, blob);

// 停止服务
localService->Stop();
```
