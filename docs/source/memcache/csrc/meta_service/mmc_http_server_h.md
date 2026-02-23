# mmc_http_server.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_http_server.h`
- **文件用途**: 定义 HTTP 服务器类，提供元数据的 HTTP 接口
- **依赖项**: `httplib.h`, `mmc_meta_manager.h`

---

## 类定义

### MmcHttpServer

HTTP 服务器类，用于提供元数据的 RESTful API。

#### 成员变量

```cpp
private:
    bool running_;
    std::string host_;
    uint16_t port_;
    MmcMetaManagerPtr metaMetaManager_;
    httplib::Server server_{};
    std::thread serverThread_;
```

**说明**:
- `running_`: 服务器运行状态
- `host_`: 监听主机地址
- `port_`: 监听端口
- `metaMetaManager_`: 元管理器指针
- `server_`: httplib 服务器实例
- `serverThread_`: 服务器线程

---

#### 构造函数

```cpp
MmcHttpServer(const std::string &host, const uint16_t port, const MmcMetaManagerPtr &metaMetaManager)
    : running_(false), host_(host), port_(port), metaMetaManager_(metaMetaManager)
{
    RegisterUrls();
}
```

**声明位置**: 行 27-31

**功能描述**: 构造 HTTP 服务器并注册 URL

**参数**:
- `host`: 监听主机地址
- `port`: 监听端口
- `metaMetaManager`: 元管理器指针

---

#### 析构函数

```cpp
~MmcHttpServer();
```

**声明位置**: 行 33

**功能描述**: 析构函数，自动停止服务器

---

### Start

```cpp
bool Start();
```

**声明位置**: 行 35

**功能描述**: 启动 HTTP 服务器

**返回值**: 成功返回 true

**实现** (`mmc_http_server.cpp`):
```cpp
bool MmcHttpServer::Start()
{
    if (running_) {
        return true;
    }

    serverThread_ = std::thread([this]() { server_.listen(host_, port_); });

    std::this_thread::sleep_for(std::chrono::milliseconds(HTTP_INIT_WAIT_MILLISECONDS));
    running_ = true;
    MMC_LOG_INFO("HTTP server started on " << host_ << ":" << port_);

    return true;
}
```

**代码逻辑**:
1. 如果已运行则直接返回
2. 创建服务器线程并启动监听
3. 等待一段时间确保服务器初始化完成
4. 设置运行标志

---

### Stop

```cpp
void Stop();
```

**声明位置**: 行 37

**功能描述**: 停止 HTTP 服务器

**实现** (`mmc_http_server.cpp`):
```cpp
void MmcHttpServer::Stop()
{
    if (!running_) {
        return;
    }

    server_.stop();
    if (serverThread_.joinable()) {
        serverThread_.join();
    }
    running_ = false;
    MMC_LOG_INFO("HTTP server stopped");
}
```

**代码逻辑**:
1. 如果未运行则直接返回
2. 停止 httplib 服务器
3. 等待服务器线程结束
4. 清除运行标志

---

### IsRunning

```cpp
bool IsRunning() const
{
    return running_;
}
```

**声明位置**: 行 39-42

**功能描述**: 检查服务器是否正在运行

---

### 私有方法

#### RegisterUrls

```cpp
void RegisterUrls();
```

**声明位置**: 行 48

**功能描述**: 注册所有 URL 路由

**实现** (`mmc_http_server.cpp`):
```cpp
void MmcHttpServer::RegisterUrls()
{
    RegisterHealthCheckEndpoint();
    RegisterDataManagementEndpoints();
    RegisterSegmentManagementEndpoints();
    RegisterMetricsEndpoint();
}
```

---

#### RegisterHealthCheckEndpoint

```cpp
void RegisterHealthCheckEndpoint();
```

**声明位置**: 行 49

**功能描述**: 注册健康检查端点 `/health`

**实现** (`mmc_http_server.cpp`):
```cpp
void MmcHttpServer::RegisterHealthCheckEndpoint()
{
    server_.Get("/health", [](const httplib::Request &, httplib::Response &res) {
        res.status = httplib::OK_200;
        res.set_content("OK\n", "text/plain");
    });
}
```

---

#### RegisterDataManagementEndpoints

```cpp
void RegisterDataManagementEndpoints();
```

**声明位置**: 行 50

**功能描述**: 注册数据管理端点

**端点**:
- `GET /get_all_keys`: 获取所有 key
- `GET /query_key?key=<key>`: 查询指定 key 的信息

**实现** (`mmc_http_server.cpp`):
```cpp
void MmcHttpServer::RegisterDataManagementEndpoints()
{
    server_.Get("/get_all_keys", [this](const httplib::Request &, httplib::Response &res) {
        if (metaMetaManager_ == nullptr) {
            MMC_LOG_ERROR("metaMetaManager_ is nullptr");
            res.status = httplib::InternalServerError_500;
            res.set_content("Internal server error", "text/plain");
            return;
        }

        std::vector<std::string> keys;
        if (const auto ret = metaMetaManager_->GetAllKeys(keys); ret != MMC_OK) {
            MMC_LOG_ERROR("Failed to get all keys from meta manager, ret=" << ret);
            res.status = httplib::InternalServerError_500;
            res.set_content("Internal server error", "text/plain");
            return;
        }

        std::ostringstream oss;
        for (const auto &key : keys) {
            oss << key << "\n";
        }

        res.status = httplib::OK_200;
        res.set_content(oss.str(), "text/plain");
    });

    server_.Get("/query_key", [this](const httplib::Request &req, httplib::Response &res) {
        const auto key_it = req.params.find("key");
        if (key_it == req.params.end()) {
            res.status = httplib::BadRequest_400;
            res.set_content("Missing 'key' parameter", "text/plain");
            return;
        }
        const std::string &key = key_it->second;

        if (metaMetaManager_ == nullptr) {
            MMC_LOG_ERROR("metaMetaManager_ is nullptr");
            res.status = httplib::InternalServerError_500;
            res.set_content("Internal server error", "text/plain");
            return;
        }

        MemObjQueryInfo queryInfo;
        if (const Result ret = metaMetaManager_->Query(key, queryInfo); ret != MMC_OK) {
            MMC_LOG_ERROR("Failed to query key: " << key << ", ret=" << ret);
            if (ret == MMC_UNMATCHED_KEY) {
                res.status = httplib::NotFound_404;
                res.set_content("Key not found", "text/plain");
            } else {
                res.status = httplib::InternalServerError_500;
                res.set_content("Internal server error", "text/plain");
            }
            return;
        }

        res.status = httplib::OK_200;
        res.set_content(queryInfo.toJson(key).dump(4UL), "application/json");
    });
}
```

---

#### RegisterSegmentManagementEndpoints

```cpp
void RegisterSegmentManagementEndpoints();
```

**声明位置**: 行 51

**功能描述**: 注册分段管理端点

**端点**:
- `GET /get_all_segments`: 获取所有分段信息

---

#### RegisterMetricsEndpoint

```cpp
void RegisterMetricsEndpoint();
```

**声明位置**: 行 52

**功能描述**: 注册监控指标端点

**端点**:
- `GET /metrics`: Prometheus 格式的监控指标
- `GET /metrics/summary`: 人类可读的监控指标摘要
- `GET /metrics/ptracer`: 性能追踪信息

**实现** (`mmc_http_server.cpp`):
```cpp
void MmcHttpServer::RegisterMetricsEndpoint()
{
    server_.Get("/metrics", [this](const httplib::Request &, httplib::Response &res) {
        const auto result = MmcMetaMetricManager::GetPrometheusSummary();

        res.status = httplib::OK_200;
        res.set_content(result, "text/plain");
    });

    server_.Get("/metrics/summary", [this](const httplib::Request &, httplib::Response &res) {
        const auto result = MmcMetaMetricManager::GetInstance().GetSummary();

        res.status = httplib::OK_200;
        res.set_content(result, "text/plain");
    });

    server_.Get("/metrics/ptracer", [this](const httplib::Request &, httplib::Response &res) {
        const char* str = ptracer_get_all_tp_string();
        if (str == nullptr) {
            MMC_LOG_ERROR("ptracer_get_all_tp_string failed");
            res.status = httplib::InternalServerError_500;
            res.set_content("Internal server error", "text/plain");
            return;
        }

        res.status = httplib::OK_200;
        res.set_content(str, "text/plain");
    });
}
```

---

## 禁用的方法

```cpp
MmcHttpServer(const MmcHttpServer &) = delete;
MmcHttpServer &operator=(const MmcHttpServer &) = delete;
```

**声明位置**: 行 44-45

**说明**: 禁止拷贝构造和赋值操作

---

## API 端点总结

| 端点 | 方法 | 描述 |
|------|------|------|
| `/health` | GET | 健康检查 |
| `/get_all_keys` | GET | 获取所有 key |
| `/query_key?key=<key>` | GET | 查询 key 信息 |
| `/get_all_segments` | GET | 获取所有分段信息 |
| `/metrics` | GET | Prometheus 格式指标 |
| `/metrics/summary` | GET | 人类可读指标摘要 |
| `/metrics/ptracer` | GET | 性能追踪信息 |

---

## 文件级别的关系图

```
mmc_http_server.h (HTTP 服务器)
    |
    +-- 依赖 MmcMetaManager (元管理器)
    |
    +-- 使用 httplib (HTTP 库)
    |
    +-- 提供 RESTful API
    +-- 监控指标导出
```
