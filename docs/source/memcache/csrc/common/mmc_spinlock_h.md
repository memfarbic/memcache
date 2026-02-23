# mmc_spinlock.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_spinlock.h`
- **文件用途**: 提供基于原子操作的自旋锁（Spinlock）实现
- **依赖项**: `<atomic>`, `<mutex>`

---

## 类定义

### Spinlock

```cpp
class Spinlock {
public:
    void lock();
    void unlock();

private:
    std::atomic<uint32_t> lock_{0}; // 4字节原子变量
};
```

**声明位置**: 行 21-47

**功能描述**: 使用原子操作实现的自旋锁

**特性**:
- 使用 `std::atomic` 实现无锁编程
- 与标准库 `std::lock_guard` 兼容（使用小写的 `lock()`/`unlock()` 方法名）
- 自旋等待（不会让出 CPU）

---

#### Spinlock::lock()

```cpp
inline void Spinlock::lock()
{
    while (lock_.exchange(1, std::memory_order_acquire)) {}
}
```

**声明位置**: 行 39-42

**功能描述**: 获取自旋锁

**代码逻辑**:
1. 调用 `exchange()` 原子操作
2. 如果旧值为 0（锁未被占用），设置新值为 1 并返回 0，循环结束
3. 如果旧值为 1（锁已被占用），返回 1，继续循环等待

**内存序**: `memory_order_acquire` - 获取语义，确保后续操作不会被重排到锁获取之前

**自旋行为**: 线程会持续检查锁状态，不会让出 CPU

**注意事项**:
- 自旋锁适用于短时间持有锁的场景
- 长时间持有自旋锁会浪费 CPU 资源

---

#### Spinlock::unlock()

```cpp
inline void Spinlock::unlock()
{
    lock_.store(0, std::memory_order_release);
}
```

**声明位置**: 行 44-47

**功能描述**: 释放自旋锁

**代码逻辑**: 将原子变量设置为 0，表示锁已释放

**内存序**: `memory_order_release` - 释放语义，确保之前的所有操作在释放锁前完成

**注意事项**: 只有获得锁的线程才能释放锁

---

#### Spinlock 成员变量

```cpp
private:
    std::atomic<uint32_t> lock_{0}; // 4 bytes atomic variable
```

**声明位置**: 行 36

**功能描述**: 原子变量，表示锁的状态

**值含义**:
- `0`: 锁未被占用
- `1`: 锁已被占用

**类型选择**: 使用 `uint32_t` 而非 `bool`，确保与原子操作的良好兼容性

---

## 文件级别的关系图

```
mmc_spinlock.h
    |
    +-- Spinlock (自旋锁)
    |   +-- lock()   [获取锁，自旋等待]
    |   +-- unlock() [释放锁]
    |   +-- lock_    [原子变量，0=未锁定, 1=已锁定]
```

---

## 使用示例

### 基本用法

```cpp
#include "mmc_spinlock.h"
#include <mutex>

Spinlock spinLock;

void CriticalSection() {
    std::lock_guard<Spinlock> guard(spinLock);
    // 临界区代码
    sharedCounter++;
}
```

### 手动加锁

```cpp
Spinlock spinLock;

void Process() {
    spinLock.lock();
    // 临界区
    spinLock.unlock();
}
```

### 与标准库容器结合

```cpp
#include <mutex>

class SpinLockGuardedData {
public:
    void Add(int value) {
        std::lock_guard<Spinlock> lock(spinlock_);
        data_.push_back(value);
    }

    size_t Size() {
        std::lock_guard<Spinlock> lock(spinlock_);
        return data_.size();
    }

private:
    Spinlock spinlock_;
    std::vector<int> data_;
};
```

---

## 自旋锁 vs 互斥锁

| 特性 | 自旋锁 (Spinlock) | 互斥锁 (std::mutex) |
|------|-------------------|---------------------|
| 等待方式 | 忙等待（CPU 空转） | 阻塞（让出 CPU） |
| 适用场景 | 锁持有时间极短 | 锁持有时间较长 |
| 上下文切换 | 无 | 有 |
| CPU 消耗 | 高（等待时） | 低 |
| 响应速度 | 快 | 较慢（需调度） |

---

## 实现细节

### 原子操作的内存序

```cpp
lock_.exchange(1, std::memory_order_acquire)  // 获取锁
lock_.store(0, std::memory_order_release)     // 释放锁
```

**memory_order_acquire (获取语义)**:
- 确保在获取锁之后的操作不会被重排到获取锁之前
- 配合 `release` 使用，确保正确的同步关系

**memory_order_release (释放语义)**:
- 确保在释放锁之前的操作不会被重排到释放锁之后
- 保证所有写操作在释放锁前对其他线程可见

**Acquire-Release 同步模型**:
```
线程1:                      线程2:
锁定 (acquire)              ...
临界区操作                  ...
临界区操作                  锁定 (acquire)
解锁 (release)              临界区操作
                            解锁 (release)
```

---

## 注意事项

1. **使用场景**: 自旋锁适用于临界区非常小的场景（通常只有几条指令）
2. **避免死锁**: 不要在持有自旋锁时调用可能阻塞的函数
3. **不可中断**: 自旋锁期间线程无法响应信号
4. **递归问题**: 这个实现不支持递归加锁，同一线程重复加锁会死锁
5. **命名规范**: 使用小写 `lock()`/`unlock()` 是为了与 C++ 标准库的 `std::lock_guard` 兼容
