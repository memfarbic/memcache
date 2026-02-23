# mmc_http_server.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_http_server.cpp`
- **文件用途**: MmcHttpServer 类的实现文件
- **依赖项**: `mmc_http_server.h`, `mmc_logger.h`, `mmc_meta_metric_manager.h`

---

## 常量定义

```cpp
constexpr int HTTP_INIT_WAIT_MILLISECONDS = 100;
```

**声明位置**: 行 27

**说明**: HTTP 服务器初始化等待时间（毫秒）

---

## 函数实现

### 析构函数

```cpp
MmcHttpServer::~MmcHttpServer()
{
    Stop();
}
```

**声明位置**: 行 32-35

**功能描述**: 析构函数，自动停止服务器

---

### RegisterUrls

```cpp
void MmcHttpServer::RegisterUrls()
{
    RegisterHealthCheckEndpoint();
    RegisterDataManagementEndpoints();
    RegisterSegmentManagementEndpoints();
    RegisterMetricsEndpoint();
}
```

**声明位置**: 行 37-43

**功能描述**: 注册所有 URL 路由

---

### RegisterHealthCheckEndpoint

```cpp
void MmcHttpServer::RegisterHealthCheckEndpoint()
{
    server_.Get("/health", [](const httplib::Request &, httplib::Response &res) {
        res.status = httplib::OK_200;
        res.set_content("OK\n", "text/plain");
    });
}
```

**声明位置**: 行 45-51

**功能描述**: 注册健康检查端点

**响应**: 固定返回 "OK\n"

---

### RegisterDataManagementEndpoints

```cpp
void MmcHttpServer::RegisterDataManagementEndpoints()
```

**声明位置**: 行 53-112

**功能描述**: 注册数据管理相关的 HTTP 端点

#### GET /get_all_keys

**功能**: 获取所有存储的 key 列表

**响应格式**: 纯文本，每行一个 key

**错误处理**:
- 500: 元管理器为空或获取失败

#### GET /query_key?key=<key>

**功能**: 查询指定 key 的详细信息

**参数**:
- `key`: 要查询的 key

**响应格式**: JSON 格式的 Blob 信息

**错误处理**:
- 400: 缺少 key 参数
- 404: Key 不存在
- 500: 其他内部错误

---

### RegisterSegmentManagementEndpoints

```cpp
void MmcHttpServer::RegisterSegmentManagementEndpoints()
{
    server_.Get("/get_all_segments", [this](const httplib::Request &, httplib::Response &res) {
        if (metaMetaManager_ == nullptr) {
            MMC_LOG_ERROR("metaMetaManager_ is nullptr");
            res.status = httplib::InternalServerError_500;
            res.set_content("Internal server error", "text/plain");
            return;
        }

        const auto result = metaMetaManager_->GetAllSegmentInfo();

        res.status = httplib::OK_200;
        res.set_content(result.dump(4UL), "application/json");
    });
}
```

**声明位置**: 行 114-129

**功能描述**: 注册分段管理端点

#### GET /get_all_segments

**功能**: 获取所有内存分段信息

**响应格式**: JSON 数组，包含每个分段的 rank、介质类型、容量、使用量等信息

---

### RegisterMetricsEndpoint

```cpp
void MmcHttpServer::RegisterMetricsEndpoint()
```

**声明位置**: 行 131-159

**功能描述**: 注册监控指标端点

#### GET /metrics

**功能**: 获取 Prometheus 格式的监控指标

**响应格式**: Prometheus 文本格式

#### GET /metrics/summary

**功能**: 获取人类可读的监控指标摘要

**响应格式**: 纯文本摘要

#### GET /metrics/ptracer

**功能**: 获取性能追踪信息

**响应格式**: ptracer 输出的文本格式

**错误处理**:
- 500: ptracer 获取失败

---

### Start

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

**声明位置**: 行 161-174

**功能描述**: 启动 HTTP 服务器

**代码逻辑**:
1. 如果已运行，直接返回
2. 创建新线程并在其中启动 httplib 监听
3. 等待 100ms 确保服务器初始化完成
4. 设置运行标志

---

### Stop

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

**声明位置**: 行 176-188

**功能描述**: 停止 HTTP 服务器

**代码逻辑**:
1. 如果未运行，直接返回
2. 停止 httplib 服务器
3. 等待服务器线程结束
4. 清除运行标志

---

## 总结

此文件实现了 HTTP 服务器的功能，提供以下能力：

1. **健康检查**: `/health` 端点用于服务状态检查
2. **数据查询**:
   - 获取所有 key
   - 查询指定 key 的详细信息
3. **分段管理**: 获取所有内存分段信息
4. **监控指标**:
   - Prometheus 格式指标
   - 人类可读摘要
   - 性能追踪信息

**线程模型**:
- 主线程处理 HTTP 请求
- 使用单独的线程运行 httplib 服务器
- 通过 RAII 确保资源正确释放
