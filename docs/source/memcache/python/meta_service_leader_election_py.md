# meta_service_leader_election.py 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/python/memcache_hybrid/meta_service_leader_election.py`
- **文件用途**: 基于 Kubernetes Lease 机制实现元数据服务的 Leader 选举
- **依赖项**: `os`, `time`, `threading`, `ctypes`, `datetime`, `kubernetes`

---

## MmcLogger 类

### 类概述

简单的日志包装类，将日志输出到 C++ 日志系统。

**声明位置**: 行 23-43

### 构造函数 (第24-25行)

```python
def __init__(self):
    self.lib_ = cdll.LoadLibrary("libmf_memcache.so")
```

**功能**: 加载 C++ 扩展库，获取日志函数

---

### log() (第27-28行)

```python
def log(self, level: int, msg: str):
    self.lib_.mmc_logger(level, msg.encode('utf-8'))
```

**功能**: 调用 C++ 日志函数

**参数**:
- `level`: 日志级别 (0=DEBUG, 1=INFO, 2=WARN, 3=ERROR)
- `msg`: 日志消息

---

### 便捷方法 (第30-40行)

```python
def debug(self, msg: str):
    self.log(0, msg)

def info(self, msg: str):
    self.log(1, msg)

def warning(self, msg: str):
    self.log(2, msg)

def error(self, msg: str):
    self.log(3, msg)
```

**功能**: 提供不同级别的日志方法

---

## 全局 logger 实例

### logger (第43行)

```python
logger = MmcLogger()
```

**声明位置**: 行 43

**功能**: 全局日志记录器实例

---

## MetaServiceLeaderElection 类

### 类概述

基于 Kubernetes Lease 机制实现 Leader 选举的类。

**声明位置**: 行 46-297

### 构造函数 (第55-89行)

```python
def __init__(self, lease_name, namespace, pod_name, retry_period=3, log_level=1, log_path="/home/memcache")
```

**声明位置**: 行 55-89

**功能**: 初始化主备选举器

**参数**:
- `lease_name`: Lease 资源名称
- `namespace`: Kubernetes 命名空间
- `pod_name`: 当前 Pod 名称（用于标识竞选者）
- `retry_period`: 重试间隔（秒），最小为 1
- `log_level`: 日志级别（保留）
- `log_path`: 日志路径（保留）

**代码逻辑**:

1. **加载 Kubernetes 配置** (第64-73行):
   ```python
   try:
       config.load_incluster_config()  # 集群内使用
   except ConfigException:
       logger.error(f'Failed in loading config in k8s cluster')
       try:
           config.load_kube_config()  # 本地开发使用
       except ConfigException as e:
           logger.error(f'Failed in loading from ~/.kube/config')
   ```

2. **初始化 API 客户端** (第79-81行):
   ```python
   self.coordination_v1 = client.CoordinationV1Api()
   self.core_v1 = client.CoreV1Api()
   ```

3. **初始化状态变量** (第83-88行):
   ```python
   self.is_leader = False
   self.leader_identity = None
   self.lease_duration = 10
   self.stop_event = threading.Event()
   self.renew_thread = None
   ```

---

### start_election() (第90-106行)

```python
def start_election(self):
```

**声明位置**: 行 90-106

**功能**: 运行主备选举循环（测试用）

**代码逻辑**:

1. **启动续约线程** (第94-96行):
   ```python
   self.renew_thread = threading.Thread(target=self._renew_lease, daemon=True)
   self.renew_thread.start()
   ```

2. **主循环** (第98-103行):
   ```python
   try:
       while not self.stop_event.is_set():
           self._check_and_update_leadership()
           time.sleep(self.retry_period)
   finally:
       self.stop_event.set()
       logger.info(f'Stop election {self.pod_name=}')
       if self.renew_thread:
           self.renew_thread.join()
   ```

---

### stop_election() (第108-111行)

```python
def stop_election(self):
    """停止选举循环"""
    self.stop_event.set()
    logger.info(f'Stop in electing leader, {self.pod_name=}')
```

**声明位置**: 行 108-111

**功能**: 停止选举循环

---

### update_lease() (第113-140行)

```python
def update_lease(self, is_renew=False):
```

**声明位置**: 行 113-140

**功能**: 刷新/续约 Lease

**参数**:
- `is_renew`: 是否为续约操作

**返回值**:
- `True`: 成功获取/续约 Leadership
- `False`: 未能获取 Leadership

**代码逻辑**:

1. **正常更新** (第116-140行):
   ```python
   try:
       return self._update_lease(is_renew)
   except ApiException as e:
       if e.status == 409:
           # 多个 POD 并发更新，冲突
           for _ in range(3):
               time.sleep(1)
               ret = self._retry_update_lease(is_renew)
               if ret == 1:  # 更新成功，升主
                   return True
               elif ret == 0:  # 有主，且租约未过期
                   return False
               elif ret == 409:  # 继续冲突
                   continue
               else:
                   break
   ```

2. **错误处理** (第136-140行)

---

### check_leader_status() (第142-168行)

```python
def check_leader_status(self):
```

**声明位置**: 行 142-168

**功能**: 检查当前主节点状态

**返回值**:
- 当前 holder 的 pod 名称
- `"None"`: 无主节点或租约已过期

**代码逻辑**:

1. **读取 Lease** (第145-148行):
   ```python
   lease = self.coordination_v1.read_namespaced_lease(
       name=self.lease_name,
       namespace=self.namespace
   )
   ```

2. **检查 holder** (第150-152行):
   ```python
   holder = lease.spec.holder_identity
   if not holder:
       return "None"
   ```

3. **检查租约是否过期** (第154-160行):
   ```python
   renew_time = lease.spec.renew_time
   if renew_time:
       renew_time = datetime.fromisoformat(str(lease.spec.renew_time)).astimezone(UTC)
   current_time = datetime.now(UTC)
   if renew_time and (current_time - renew_time > timedelta(seconds=self.lease_duration)):
       return "None"
   ```

---

### update_pod_to_master() (第170-173行)

```python
def update_pod_to_master(self):
    """升主时，调用该方法设置标签为master"""
    self._update_pod_label({"role": "master"})
```

**声明位置**: 行 170-173

**功能**: 升主时更新 Pod 标签为 master

---

### update_pod_to_backup() (第175-178行)

```python
def update_pod_to_backup(self):
    """失去主节点身份时，调用该方法设置标签为backup"""
    self._update_pod_label({"role": "backup"})
```

**声明位置**: 行 175-178

**功能**: 降备时更新 Pod 标签为 backup

---

### _retry_update_lease() (第180-188行)

```python
def _retry_update_lease(self, is_renew):
    try:
        return 1 if self._update_lease(is_renew) else 0
    except ApiException as e:
        logger.error(f'Failed in retry updating lease {self.pod_name=}, ApiException: {e}')
        return e.status
    except Exception as e:
        logger.error(f'Failed in updating lease {self.pod_name=}, Exception: {e}')
        return -1
```

**声明位置**: 行 180-188

**功能**: 重试更新 Lease，返回状态码

---

### _update_lease() (第190-220行)

```python
def _update_lease(self, is_renew):
```

**声明位置**: 行 190-220

**功能**: 内部更新 Lease 实现

**返回值**:
- `True`: 成功更新
- `False`: 无需更新（已有其他主）

**代码逻辑**:

1. **读取现有 Lease** (第192-195行)

2. **检查是否需要更新** (第197-219行):
   ```python
   # 租约未被持有，或持有者是自己，或租约已过期
   if not holder or holder == self.pod_name or lease_expired:
       self._inner_update_lease(is_renew, lease, current_time)
       return True
   ```
   - 只有在无主、自己是主、或租约过期时才尝试更新

3. **调整续约间隔** (第203-208行):
   ```python
   if lease.spec.lease_duration_seconds is not None:
       self.lease_duration = lease.spec.lease_duration_seconds
       # 续约间隔为有效期的 1/3
       if self.retry_period * 3 > self.lease_duration:
           self.retry_period = max(self.lease_duration // 3, 1)
   ```

---

### _inner_update_lease() (第222-241行)

```python
def _inner_update_lease(self, is_renew, lease, current_time):
```

**声明位置**: 行 222-241

**功能**: 实际更新 Lease 对象

**代码逻辑**:

1. **更新 Lease 字段** (第224-228行):
   ```python
   lease.spec.holder_identity = self.pod_name
   lease.spec.lease_duration_seconds = self.lease_duration
   lease.spec.renew_time = current_time
   if not is_renew:  # 首次获取时更新获取时间
       lease.spec.acquire_time = current_time
   ```

2. **调用 Kubernetes API** (第230-235行):
   ```python
   try:
       self.coordination_v1.replace_namespaced_lease(
           name=self.lease_name,
           namespace=self.namespace,
           body=lease
       )
   ```

---

### _update_pod_label() (第242-263行)

```python
def _update_pod_label(self, labels):
```

**声明位置**: 行 242-263

**功能**: 更新当前 Pod 的标签

**代码逻辑**:

1. **读取 Pod** (第245-248行)

2. **更新标签** (第250-252行):
   ```python
   if pod.metadata.labels is None:
       pod.metadata.labels = {}
   pod.metadata.labels.update(labels)
   ```

3. **调用 Kubernetes API** (第254-258行)

---

### _renew_lease() (第265-277行)

```python
def _renew_lease(self):
```

**声明位置**: 行 265-277

**功能**: 定期续约租约（在独立线程中运行）

**代码逻辑**:

```python
while not self.stop_event.is_set():
    if self.is_leader:
        success = self.update_lease(is_renew=True)
        if not success:
            logger.warning(f'Failed in renewing lease and becoming backup, {self.pod_name=}')
            self.is_leader = False
            self.update_pod_to_backup()
    time.sleep(self.retry_period)
```

**行为**:
- 只有 Leader 才续约
- 续约失败则降为 Backup
- 定期休眠 `retry_period` 秒

---

### _check_and_update_leadership() (第279-296行)

```python
def _check_and_update_leadership(self):
```

**声明位置**: 行 279-296

**功能**: 检查并更新 Leadership 状态

**代码逻辑**:

```python
current_leader = self.check_leader_status()
if current_leader == self.pod_name:
    if not self.is_leader:
        logger.info(f'Pod {self.pod_name} becomes the leader')
        self.is_leader = True
        self.update_pod_to_master()
else:
    if self.is_leader:
        logger.info(f'Pod {self.pod_name} becomes backup')
        self.is_leader = False
        self.update_pod_to_backup()
    else:
        # 尝试竞选主节点
        if self.update_lease():
            logger.info(f'Pod {self.pod_name} becomes the leader')
            self.is_leader = True
            self.update_pod_to_master()
```

**行为**:
1. 检查当前 Leader
2. 如果自己是 Leader 且 `is_leader=False`，则升主
3. 如果自己是 Backup 且 `is_leader=True`，则降备
4. 如果无主或租约过期，尝试竞选

---

## 选举流程

```
                    ┌─────────────────┐
                    │  启动选举        │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  启动续约线程     │
                    └────────┬────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
         ┌──────▼──────┐          ┌──────▼──────┐
         │ 主循环线程  │          │ 续约线程    │
         │ (检查状态)  │          │ (定期续约)  │
         └──────┬──────┘          └──────┬──────┘
                │                        │
                ▼                        ▼
         检查Leader状态           如果is_leader
                │                        │
         ┌──────┴──────┐                │
         │             │                │
      是自己?       是别人?         续约成功?
         │             │                │
         ▼             ▼                ▼
     升为主节点    尝试竞选        保持主节点
         │             │                │
         ▼             ▼                ▼
    更新标签      成功→主节点      失败→备节点
                 失败→等待
```

---

## 使用示例

### 示例用法 (第300-320行)

```python
if __name__ == "__main__":
    # 从环境变量获取当前Pod信息
    POD_NAME = os.getenv("META_POD_NAME", "meta-pod-0")
    NAMESPACE = os.getenv("META_NAMESPACE", "default")
    LEASE_NAME = os.getenv("META_LEASE_NAME", "default")

    election = MetaServiceLeaderElection(
        lease_name=LEASE_NAME,
        namespace=NAMESPACE,
        pod_name=POD_NAME,
        retry_period=5,
        log_level=0,
        log_path="/home/memcache"
    )

    try:
        # 运行选举
        election.start_election()
    except KeyboardInterrupt:
        logger.error("election process aborted")
        election.stop_election()
```

---

## 环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| META_POD_NAME | meta-pod-0 | 当前 Pod 名称 |
| META_NAMESPACE | default | Kubernetes 命名空间 |
| META_LEASE_NAME | default | Lease 资源名称 |

---

## 注意事项

1. **续约间隔**: 建议设置为租约有效期的 1/3，避免网络延迟导致租约过期
2. **并发冲突**: 处理 409 冲突状态码，多个 Pod 同时竞选时的正常现象
3. **标签更新**: 升主/降备时会更新 Pod 的 role 标签
4. **线程安全**: 使用 `stop_event` 进行线程间通信
