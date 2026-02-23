# mmc_net_engine.cpp 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/net/mmc_net_engine.cpp`
- **文件用途**: 实现 NetEngine 工厂方法
- **依赖项**:
  - `mmc_ref.h` - 引用计数智能指针
  - `mmc_net_engine_acc.h` - ACC 网络引擎实现
  - `mmc_net_engine.h` - 网络引擎接口

---

## 函数

### NetEngine::Create()

```cpp
NetEnginePtr NetEngine::Create()
{
    return MmcMakeRef<NetEngineAcc>().Get();
}
```

**声明位置**: 行 18-21

**功能描述**: 创建网络引擎实例的工厂方法

**返回值**: `NetEnginePtr` - 网络引擎智能指针

**代码逻辑**:
1. 使用 `MmcMakeRef` 创建 `NetEngineAcc` 实例（ACC 实现）
2. 返回智能指针的裸指针（通过 `Get()` 方法）

**设计说明**:
- 这是一个工厂方法，将具体实现类 `NetEngineAcc` 与接口类 `NetEngine` 解耦
- 未来可以轻松切换到其他网络实现（如 RDMA、共享内存等）

**使用示例**:
```cpp
auto engine = NetEngine::Create();
NetEngineOptions options;
options.name = "my_engine";
options.ip = "0.0.0.0";
options.port = 5000;
engine->Start(options);
```

---

## 文件级别的关系图

```
mmc_net_engine.cpp (网络引擎工厂实现)
    |
    +-- NetEngine::Create() -> 创建网络引擎实例
            |
            +-> 返回 NetEngineAcc 实例
```

---

## 依赖关系

**依赖以下文件**:
- `mmc_ref.h` - 智能指针支持
- `mmc_net_engine_acc.h` - ACC 网络引擎实现
- `mmc_net_engine.h` - 网络引擎接口

**被以下文件依赖**:
- 应用层代码 - 通过 NetEngine::Create() 创建网络引擎

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    // 所有实现都在 ock::mmc 命名空间内
}
}
```
