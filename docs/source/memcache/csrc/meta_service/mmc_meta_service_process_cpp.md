# mmc_meta_service_process.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/meta_service/mmc_meta_service_process.cpp`
- **文件用途**: MmcMetaServiceProcess 类的实现文件
- **依赖项**: `mmc_meta_service_process.h`, `mmc_configuration.h`, `spdlogger4c.h`, `pybind11`

---

## 全局变量

```cpp
static std::mutex g_exitMtx;
static std::condition_variable g_exitCv;
static bool g_processExit = false;
```

**声明位置**: 行 41-43

**说明**: 进程退出同步变量

---

## 函数实现

### MainForExecutable

```cpp
int MmcMetaServiceProcess::MainForExecutable()
{
    pybind11::scoped_interpreter guard{};
    pybind11::gil_scoped_release release;

    // Python 方式启动已经在解释器中，不能再次创建解释器
    return MainForPython();
}
```

**声明位置**: 行 45-52

**功能描述**: 可执行文件入口

**代码逻辑**:
1. 创建 Python 解释器
2. 释放 GIL
3. 调用 Python 入口

---

### MainForPython

```cpp
int MmcMetaServiceProcess::MainForPython()
{
    if (CheckIsRunning()) {
        std::cerr << "Error, meta service is already running." << std::endl;
        return -1;
    }

    if (LoadConfig() != 0) {
        std::cerr << "Error, failed to load config." << std::endl;
        return -1;
    }

    RegisterSignal();

    ptracer_config_t ptraceConfig{.tracerType = 1, .dumpFilePath = "/var/log/memfabric_hybrid"};
    const auto result = ptracer_init(&ptraceConfig);
    if (result != MMC_OK) {
        std::cout << "Warning, init ptracer module failed, result: " << result
                  << ", error msg: " << ptracer_get_last_err_msg() << std::endl;
    }

    if (InitLogger(config_) != 0) {
        std::cerr << "Error, failed to init logger." << std::endl;
        return -1;
    }

    if (config_.haEnable) {
        leaderElection_ = new (std::nothrow)
            MmcMetaServiceLeaderElection("leader_election", META_POD_NAME, META_NAMESPACE, META_LEASE_NAME);
        if (leaderElection_ == nullptr || leaderElection_->Start(config_) != MMC_OK) {
            std::cerr << "Error, failed to start meta service leader election." << std::endl;
            Exit();
            return -1;
        }
    }

    metaService_ = new (std::nothrow) MmcMetaService("meta_service");
    if (metaService_ == nullptr || metaService_->Start(config_) != MMC_OK) {
        std::cerr << "Error, failed to start MmcMetaService." << std::endl;
        Exit();
        return -1;
    }

    if (StartHttpServer() != MMC_OK) {
        std::cerr << "Error, failed to start the HTTP Service." << std::endl;
        Exit();
        return -1;
    }

    MMC_AUDIT_LOG("Meta Service launched successfully");

    std::unique_lock<std::mutex> lock(g_exitMtx);
    g_exitCv.wait(lock, []() { return g_processExit; });
    Exit();

    MMC_AUDIT_LOG("Meta Service stopped");

    return 0;
}
```

**声明位置**: 行 54-112

**功能描述**: Python 模块入口

**代码逻辑**:
1. 检查是否已运行
2. 加载配置
3. 注册信号处理
4. 初始化性能追踪
5. 初始化日志
6. 启动 Leader 选举（如果启用 HA）
7. 启动元服务
8. 启动 HTTP 服务
9. 等待退出信号
10. 清理退出

---

### CheckIsRunning

```cpp
bool MmcMetaServiceProcess::CheckIsRunning()
{
    const std::string filePath = "/tmp/mmc_meta_service";
    const std::string fileName = filePath + ".lock";
    const int fd = open(fileName.c_str(), O_WRONLY | O_CREAT, 0600);
    if (fd < 0) {
        std::cerr << "Open file " << fileName.c_str() << " failed, error message is " << strerror(errno) << "."
                  << std::endl;
        return true;
    }
    flock lock{};
    lock.l_type = F_WRLCK;
    lock.l_start = 0;
    lock.l_whence = SEEK_SET;
    lock.l_len = 0;
    const auto ret = fcntl(fd, F_SETLK, &lock);
    if (ret < 0) {
        std::cerr << "Fail to start mmc_meta_service, process lock file is locked." << std::endl;
        close(fd);
        return true;
    }
    return false;
}
```

**声明位置**: 行 114-136

**功能描述**: 检查进程是否已在运行

**实现**: 使用文件锁机制，在 `/tmp/mmc_meta_service.lock` 上加锁

---

### LoadConfig

```cpp
int MmcMetaServiceProcess::LoadConfig()
{
    MetaServiceConfig configManager;
    if (MMC_META_CONF_PATH.empty()) {
        std::cout << "[WARNING] MMC_META_CONFIG_PATH is not set. "
                  << "All configuration items use default values." << std::endl;
    } else {
        // Read configuration from config file
        const auto confPath = MMC_META_CONF_PATH;
        if (!configManager.LoadFromFile(confPath)) {
            std::cerr << "Failed to load config from file" << std::endl;
            return -1;
        }
        const std::vector<std::string> validationError = configManager.ValidateConf();
        if (!validationError.empty()) {
            std::cerr << "Wrong configuration in file <" << confPath
                      << ">, because of following mistakes:" << std::endl;
            for (auto &item : validationError) {
                std::cout << item << std::endl;
            }
            return -1;
        }
    }
    configManager.GetMetaServiceConfig(config_);
    if (MetaServiceConfig::ValidateTLSConfig(config_.accTlsConfig) != MMC_OK) {
        std::cerr << "Invalid tls config." << std::endl;
        return -1;
    }
    if (MetaServiceConfig::ValidateTLSConfig(config_.configStoreTlsConfig) != MMC_OK) {
        std::cerr << "Invalid tls config." << std::endl;
        return -1;
    }
    if (MetaServiceConfig::ValidateLogPathConfig(config_.logPath) != MMC_OK) {
        std::cerr << "Invalid log path, please check 'ock.mmc.log_path' " << std::endl;
        return -1;
    }

    return 0;
}
```

**声明位置**: 行 138-176

**功能描述**: 加载配置文件

**代码逻辑**:
1. 检查环境变量
2. 从文件加载配置
3. 验证配置
4. 验证 TLS 配置
5. 验证日志路径

---

### RegisterSignal

```cpp
void MmcMetaServiceProcess::RegisterSignal()
{
    const sighandler_t oldIntHandler = signal(SIGINT, SignalInterruptHandler);
    if (oldIntHandler == SIG_ERR) {
        std::cerr << "Register SIGINT handler failed" << std::endl;
    }

    const sighandler_t oldTermHandler = signal(SIGTERM, SignalInterruptHandler);
    if (oldTermHandler == SIG_ERR) {
        std::cerr << "Register SIGTERM handler failed" << std::endl;
    }
}
```

**声明位置**: 行 178-189

**功能描述**: 注册信号处理函数

**处理的信号**:
- SIGINT (Ctrl+C)
- SIGTERM

---

### SignalInterruptHandler

```cpp
void MmcMetaServiceProcess::SignalInterruptHandler(const int signal)
{
    std::cout << "Received exit signal[" << signal << "]" << std::endl;

    {
        std::unique_lock<std::mutex> lock(g_exitMtx);
        g_processExit = true;
    }
    g_exitCv.notify_all();
}
```

**声明位置**: 行 191-200

**功能描述**: 信号中断处理函数

**代码逻辑**:
1. 设置退出标志
2. 通知等待线程

---

### InitLogger

```cpp
int MmcMetaServiceProcess::InitLogger(const mmc_meta_service_config_t &options)
{
    const std::string logPath = std::string(options.logPath) + "/logs/mmc-meta.log";
    const std::string logAuditPath = std::string(options.logPath) + "/logs/mmc-meta-audit.log";

    std::cout << "Meta service log level " << options.logLevel << ", log path: " << logPath
              << ", audit log path: " << logAuditPath << ", log rotation file size: " << options.logRotationFileSize
              << ", log rotation file count: " << options.logRotationFileCount << std::endl;

    auto ret = MmcOutLogger::Instance().SetLogLevel(static_cast<LogLevel>(options.logLevel));
    if (ret != 0) {
        std::cerr << "Failed to set log level " << options.logLevel << std::endl;
        return -1;
    }
    mf::OutLogger::Instance().SetLogLevel(static_cast<mf::LogLevel>(options.logLevel));
    ret = SPDLOG_Init(logPath.c_str(), options.logLevel, options.logRotationFileSize, options.logRotationFileCount);
    if (ret != 0) {
        std::cerr << "Failed to init spdlog, error: " << SPDLOG_GetLastErrorMessage() << std::endl;
        return -1;
    }
    MmcOutLogger::Instance().SetExternalLogFunction(SPDLOG_LogMessage);
    mf::OutLogger::Instance().SetExternalLogFunction(SPDLOG_LogMessage);
    ret = SPDLOG_AuditInit(logAuditPath.c_str(), options.logRotationFileSize, options.logRotationFileCount);
    if (ret != 0) {
        std::cerr << "Failed to init audit spdlog, error: " << SPDLOG_GetLastErrorMessage() << std::endl;
        return -1;
    }
    MmcOutLogger::Instance().SetExternalAuditLogFunction(SPDLOG_AuditLogMessage);

    return 0;
}
```

**声明位置**: 行 202-232

**功能描述**: 初始化日志系统

**代码逻辑**:
1. 构造日志路径
2. 设置日志级别
3. 初始化 spdlog
4. 设置外部日志函数
5. 初始化审计日志

---

### StartHttpServer

```cpp
int MmcMetaServiceProcess::StartHttpServer()
{
    MMC_VALIDATE_RETURN(metaService_ != nullptr, "metaService not been initialized", MMC_ERROR);
    auto metaMgrProxyPtr = metaService_->GetMetaMgrProxy();
    MMC_VALIDATE_RETURN(metaMgrProxyPtr != nullptr, "metaMgrProxy is nullptr", MMC_ERROR);
    auto metaManagerPtr = metaMgrProxyPtr->GetMetaManager();
    MMC_VALIDATE_RETURN(metaManagerPtr != nullptr, "metaManager is nullptr", MMC_ERROR);

    std::string httpUrl = config_.httpURL;
    const size_t colonPos = httpUrl.find(':');
    if (colonPos == std::string::npos) {
        MMC_LOG_ERROR("Invalid http URL: colon not found.");
        return MMC_INVALID_PARAM;
    }
    std::string host = httpUrl.substr(0, colonPos);
    std::string portStr = httpUrl.substr(colonPos + 1);
    uint16_t port;

    try {
        port = static_cast<uint16_t>(std::stoul(portStr));
    } catch (const std::invalid_argument &) {
        MMC_LOG_ERROR("Invalid http URL: port is not a valid number.");
        return MMC_INVALID_PARAM;
    } catch (const std::out_of_range &) {
        MMC_LOG_ERROR("Invalid http URL: port out of range.");
        return MMC_INVALID_PARAM;
    }

    MMC_LOG_INFO("Starting HTTP server on " << host << ":" << port);

    try {
        httpServer_ = new MmcHttpServer(host, port, metaManagerPtr);
        httpServer_->Start();
        return MMC_OK;
    } catch (const std::exception &e) {
        MMC_LOG_ERROR("Failed to start HTTP server: " << e.what());
        return MMC_ERROR;
    }
}
```

**声明位置**: 行 234-272

**功能描述**: 启动 HTTP 服务器

**代码逻辑**:
1. 验证指针有效
2. 解析 HTTP URL
3. 创建 HTTP 服务器
4. 启动服务器

---

### Exit

```cpp
void MmcMetaServiceProcess::Exit()
{
    if (metaService_ != nullptr) {
        metaService_->Stop();
    }
    if (leaderElection_ != nullptr) {
        leaderElection_->Stop();
    }
    if (httpServer_ != nullptr) {
        httpServer_->Stop();
    }
    ptracer_uninit();
}
```

**声明位置**: 行 274-286

**功能描述**: 退出并清理资源

**代码逻辑**:
1. 停止元服务
2. 停止 Leader 选举
3. 停止 HTTP 服务
4. 反初始化性能追踪

---

## 总结

此文件实现了元服务进程管理的功能：

1. **进程管理**: 使用文件锁确保单实例运行
2. **配置加载**: 支持从文件和环境变量加载配置
3. **信号处理**: 处理 SIGINT 和 SIGTERM 信号
4. **日志初始化**: 初始化 spdlog 和审计日志
5. **服务启动**: 按顺序启动各个子服务
6. **优雅退出**: 清理所有资源后退出

**启动顺序**:
1. 检查运行状态
2. 加载配置
3. 注册信号
4. 初始化性能追踪
5. 初始化日志
6. 启动 Leader 选举
7. 启动元服务
8. 启动 HTTP 服务
9. 等待退出信号

**退出顺序**:
1. 停止元服务
2. 停止 Leader 选举
3. 停止 HTTP 服务
4. 反初始化性能追踪
