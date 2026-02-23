# daemon 模块文档

## 模块概述

`daemon` 模块实现元服务的守护进程功能，提供独立进程运行的元服务入口。

**目录路径**: `/home/xuruiyuan/project/mini-proj/memcache/src/memcache/csrc/daemon/`

## 文件列表

### 源文件 (.cpp)
- `mmc_daemon.cpp` - 守护进程主入口

### 构建文件
- `CMakeLists.txt` - CMake 构建配置

---

## 详细文档

### mmc_daemon.cpp

**功能**: 守护进程的主入口文件，负责启动元服务进程。

**main 函数**

```cpp
int main(int argc, char *argv[])
```

**功能**: 程序主入口

**参数**:
- `argc`: 参数数量
- `argv`: 参数列表

**返回值**: `int` - 程序退出码

**代码逻辑**:
```cpp
int main(int argc, char *argv[])
{
    return MmcMetaServiceProcess::getInstance().MainForExecutable();
}
```

**说明**:
1. 直接调用 `MmcMetaServiceProcess` 单例的 `MainForExecutable()` 方法
2. 启动元服务进程，该调用是阻塞的
3. 进程会持续运行直到收到停止信号

---

## 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      mmc_daemon                              │
│                    (守护进程入口)                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ getInstance().MainForExecutable()
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              MmcMetaServiceProcess                           │
│                    (元服务进程)                               │
└───────────────────────────┬─────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
┌──────────────────┐ ┌─────────────┐ ┌──────────────┐
│ Meta Network     │ │ Meta        │ │ Leader       │
│ Server           │ │ Manager     │ │ Election     │
│ (网络服务器)       │ │ (元管理器)    │ │ (Leader选举)  │
└──────────────────┘ └─────────────┘ └──────────────┘
```

---

## 构建和部署

### CMake 配置

```cmake
# daemon/CMakeLists.txt 示例
add_executable(mmc_daemon mmc_daemon.cpp)
target_link_libraries(mmc_daemon
    PRIVATE
        mmcache
        mf_meta_service
)
```

### 编译

```bash
cd build
cmake ..
make mmc_daemon
```

### 运行

```bash
# 直接运行
./mmc_daemon

# 或通过 systemd 管理
systemctl start mmc-daemon
```

---

## 环境变量

守护进程依赖以下环境变量：

| 环境变量 | 说明 | 必需 |
|---------|------|------|
| `MMC_META_CONFIG_PATH` | 元服务配置文件路径 | 是 |
| `META_POD_NAME` | Meta Pod 名称 | 是 |
| `META_NAMESPACE` | Meta 命名空间 | 是 |
| `META_LEASE_NAME` | Meta Lease 名称 | 是 |

---

## 配置文件

元服务配置文件示例（JSON 格式）：

```json
{
  "discoveryURL": "tcp://192.168.1.1:12345",
  "logLevel": 1,
  "logPath": "/var/log/mmc/meta.log",
  "bmIpPort": "tcp://0.0.0.0:12346",
  "bmHcomUrl": "",
  "worldSize": 4,
  "deviceId": 0,
  "localDRAMSize": 10737418240,
  "localHBMSize": 0
}
```

---

## 信号处理

守护进程响应以下信号：

| 信号 | 说明 |
|------|------|
| `SIGTERM` | 优雅停止 |
| `SIGINT` | 中断停止 (Ctrl+C) |
| `SIGHUP` | 重新加载配置 |

---

## 使用示例

### 直接运行

```bash
# 设置环境变量
export MMC_META_CONFIG_PATH=/etc/mmc/meta_config.json
export META_POD_NAME=meta-pod-0
export META_NAMESPACE=default
export META_LEASE_NAME=meta-lease

# 运行守护进程
./mmc_daemon
```

### 通过 systemd 运行

创建 `/etc/systemd/system/mmc-daemon.service`:

```ini
[Unit]
Description=MemCache Meta Service Daemon
After=network.target

[Service]
Type=simple
User=mmc
Group=mmc
Environment="MMC_META_CONFIG_PATH=/etc/mmc/meta_config.json"
Environment="META_POD_NAME=meta-pod-0"
Environment="META_NAMESPACE=default"
Environment="META_LEASE_NAME=meta-lease"
ExecStart=/usr/local/bin/mmc_daemon
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启动服务：

```bash
systemctl daemon-reload
systemctl enable mmc-daemon
systemctl start mmc-daemon
systemctl status mmc-daemon
```

### 查看日志

```bash
# 查看服务日志
journalctl -u mmc-daemon -f

# 查看元服务日志
tail -f /var/log/mmc/meta.log
```

---

## Docker 部署

Dockerfile 示例：

```dockerfile
FROM ubuntu:22.04

# 安装依赖
RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# 复制程序
COPY build/mmc_daemon /usr/local/bin/
COPY config/ /etc/mmc/

# 设置环境变量
ENV MMC_META_CONFIG_PATH=/etc/mmc/meta_config.json
ENV META_POD_NAME=meta-pod-0
ENV META_NAMESPACE=default
ENV META_LEASE_NAME=meta-lease

# 暴露端口
EXPOSE 12345 12346

# 启动守护进程
CMD ["mmc_daemon"]
```

---

## 故障排查

### 常见问题

1. **启动失败**
   - 检查配置文件路径是否正确
   - 检查环境变量是否设置
   - 查看日志文件

2. **无法连接到其他节点**
   - 检查网络连接
   - 检查防火墙规则
   - 验证 discoveryURL 配置

3. **Leader 选举失败**
   - 检查 Kubernetes Lease 对象是否存在
   - 验证 RBAC 权限配置

---

## 日志位置

| 组件 | 日志路径 |
|------|---------|
| 元服务 | `/var/log/mmc/meta.log` |
| 系统日志 | `journalctl -u mmc-daemon` |
