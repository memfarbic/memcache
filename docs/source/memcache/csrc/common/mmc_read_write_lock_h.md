# mmc_read_write_lock.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_read_write_lock.h`
- **文件用途**: 提供读写锁（ReadWriteLock）实现，支持多个读者或单个写者
- **依赖项**: `<condition_variable>`, `<memory>`, `<mutex>`

---

## 类定义

### ReadWriteLock

```cpp
class ReadWriteLock {
public:
    ReadWriteLock() = default;

    void LockRead();
    void UnlockRead();
    void LockWrite();
    void UnlockWrite();

private:
    std::mutex mutex_;
    std::condition_variable cv_;
    uint16_t numReaders_{0};  // 当前读线程数量
    bool isWriting_{false};   // 是否有写线程
};
```

**声明位置**: 行 21-67

**功能描述**: 读写锁实现，支持多读单写模式

**特性**:
- 多个线程可以同时持有读锁
- 写锁是独占的，获取写锁时不能有读锁或其他写锁
- 使用条件变量实现线程同步

---

#### ReadWriteLock::ReadWriteLock()

```cpp
ReadWriteLock() = default;
```

**声明位置**: 行 23

**功能描述**: 默认构造函数

**初始状态**:
- `numReaders_ = 0`: 无读者
- `isWriting_ = false`: 无写者

---

#### ReadWriteLock::LockRead()

```cpp
void LockRead()
{
    std::unique_lock<std::mutex> lock(mutex_);
    while (isWriting_ || numReaders_ == std::numeric_limits<uint16_t>::max()) {
        cv_.wait(lock);
    }
    numReaders_++;
}
```

**声明位置**: 行 25-32

**功能描述**: 获取读锁

**代码逻辑**:
1. 获取互斥锁
2. 等待条件满足（没有写线程 且 读者数未溢出）
3. 增加读者计数
4. 释放互斥锁（通过 unique_lock 析构）

**阻塞条件**:
- 正在有写线程在执行 (`isWriting_ == true`)
- 读者数量达到最大值（防止溢出）

**注意事项**: 使用 while 循环防止虚假唤醒

---

#### ReadWriteLock::UnlockRead()

```cpp
void UnlockRead()
{
    std::unique_lock<std::mutex> lock(mutex_);
    if (numReaders_ == 0) {
        return;
    }
    numReaders_--;
    if (numReaders_ == 0) {
        cv_.notify_one(); // 唤醒等待的写线程
    }
}
```

**声明位置**: 行 34-44

**功能描述**: 释放读锁

**代码逻辑**:
1. 获取互斥锁
2. 如果读者数为 0，直接返回（防御性编程）
3. 减少读者计数
4. 如果这是最后一个读者，通知一个等待的写线程

**优化**: 只在最后一个读者离开时才通知写线程

---

#### ReadWriteLock::LockWrite()

```cpp
void LockWrite()
{
    std::unique_lock<std::mutex> lock(mutex_);
    while (isWriting_ || numReaders_ > 0) {
        cv_.wait(lock);
    }
    isWriting_ = true;
}
```

**声明位置**: 行 46-53

**功能描述**: 获取写锁

**代码逻辑**:
1. 获取互斥锁
2. 等待条件满足（没有写线程 且 没有读线程）
3. 设置写线程标志

**阻塞条件**:
- 正有其他写线程在执行
- 有读线程在执行

**排他性**: 写锁是绝对排他的，不允许任何读线程或其他写线程同时存在

---

#### ReadWriteLock::UnlockWrite()

```cpp
void UnlockWrite()
{
    std::unique_lock<std::mutex> lock(mutex_);
    isWriting_ = false;
    cv_.notify_all(); // 唤醒所有等待的读线程
}
```

**声明位置**: 行 55-60

**功能描述**: 释放写锁

**代码逻辑**:
1. 获取互斥锁
2. 清除写线程标志
3. 通知所有等待的线程（包括读线程和写线程）

**唤醒策略**: 使用 `notify_all()` 因为可能有多个读线程在等待

---

#### ReadWriteLock 成员变量

```cpp
private:
    std::mutex mutex_;                  // 保护内部状态的互斥锁
    std::condition_variable cv_;        // 条件变量用于线程同步
    uint16_t numReaders_{0};            // 当前读线程数量
    bool isWriting_{false};             // 是否有写线程
```

**说明**:
- `mutex_`: 保护 `numReaders_` 和 `isWriting_` 的互斥锁
- `cv_`: 用于线程等待/通知的条件变量
- `numReaders_`: 当前持有读锁的线程数（最大 65535）
- `isWriting_`: 是否有线程持有写锁

---

### ReadLock

```cpp
class ReadLock {
public:
    explicit ReadLock(ReadWriteLock &rwLock) : rwLock_(rwLock)
    {
        rwLock_.LockRead();
    }
    ~ReadLock()
    {
        rwLock_.UnlockRead();
    }

    ReadLock(const ReadLock &) = delete;
    ReadLock &operator=(const ReadLock &) = delete;

private:
    ReadWriteLock &rwLock_;
};
```

**声明位置**: 行 69-86

**功能描述**: RAII 风格的读锁管理类

**用途**: 在作用域内自动管理读锁的生命周期

---

#### ReadLock::ReadLock()

```cpp
explicit ReadLock(ReadWriteLock &rwLock) : rwLock_(rwLock)
{
    rwLock_.LockRead();
}
```

**声明位置**: 行 71-74

**功能描述**: 构造函数，自动获取读锁

**参数**:
- `rwLock`: 要管理的读写锁引用

---

#### ReadLock::~ReadLock()

```cpp
~ReadLock()
{
    rwLock_.UnlockRead();
}
```

**声明位置**: 行 75-77

**功能描述**: 析构函数，自动释放读锁

**RAII 保证**: 即使发生异常也会释放锁

---

#### ReadLock 禁止的操作

```cpp
ReadLock(const ReadLock &) = delete;              // 禁止拷贝构造
ReadLock &operator=(const ReadLock &) = delete;   // 禁止拷贝赋值
```

**原因**: 锁的所有权不可复制

---

#### ReadLock 成员变量

```cpp
private:
    ReadWriteLock &rwLock_;
```

**说明**: 被管理的读写锁的引用

---

### WriteLock

```cpp
class WriteLock {
public:
    explicit WriteLock(ReadWriteLock &rwLock) : rwLock_(rwLock)
    {
        rwLock_.LockWrite();
    }
    ~WriteLock()
    {
        rwLock_.UnlockWrite();
    }

    WriteLock(const ReadLock &) = delete;
    WriteLock &operator=(const ReadLock &) = delete;

private:
    ReadWriteLock &rwLock_;
};
```

**声明位置**: 行 88-105

**功能描述**: RAII 风格的写锁管理类

**用途**: 在作用域内自动管理写锁的生命周期

---

#### WriteLock::WriteLock()

```cpp
explicit WriteLock(ReadWriteLock &rwLock) : rwLock_(rwLock)
{
    rwLock_.LockWrite();
}
```

**声明位置**: 行 90-93

**功能描述**: 构造函数，自动获取写锁

**参数**:
- `rwLock`: 要管理的读写锁引用

---

#### WriteLock::~WriteLock()

```cpp
~WriteLock()
{
    rwLock_.UnlockWrite();
}
```

**声明位置**: 行 94-96

**功能描述**: 析构函数，自动释放写锁

**RAII 保证**: 即使发生异常也会释放锁

---

#### WriteLock 禁止的操作

```cpp
WriteLock(const ReadLock &) = delete;              // 禁止拷贝构造（注：参数类型可能有误）
WriteLock &operator=(const ReadLock &) = delete;   // 禁止拷贝赋值
```

**注意**: 原代码中删除声明使用的是 `ReadLock` 类型，实际应该是 `WriteLock`，这可能是一个代码缺陷

---

#### WriteLock 成员变量

```cpp
private:
    ReadWriteLock &rwLock_;
```

**说明**: 被管理的读写锁的引用

---

## 文件级别的关系图

```
mmc_read_write_lock.h
    |
    +-- ReadWriteLock (读写锁核心实现)
    |   +-- LockRead()    [获取读锁]
    |   +-- UnlockRead()  [释放读锁]
    |   +-- LockWrite()   [获取写锁]
    |   +-- UnlockWrite() [释放写锁]
    |   +-- mutex_        [内部互斥锁]
    |   +-- cv_           [条件变量]
    |   +-- numReaders_   [读者计数]
    |   +-- isWriting_    [写者标志]
    |
    +-- ReadLock (RAII 读锁)
    |   +-- 构造时自动获取读锁
    |   +-- 析构时自动释放读锁
    |
    +-- WriteLock (RAII 写锁)
    |   +-- 构造时自动获取写锁
    |   +-- 析构时自动释放写锁
```

---

## 使用示例

### 基本用法（使用 RAII 类）

```cpp
ReadWriteLock rwLock;

// 读操作
{
    ReadLock readGuard(rwLock);
    // 可以同时有多个读线程
    value = sharedData;
} // 自动释放读锁

// 写操作
{
    WriteLock writeGuard(rwLock);
    // 写操作是排他的
    sharedData = newValue;
} // 自动释放写锁
```

### 线程安全的缓存示例

```cpp
template<typename K, typename V>
class ThreadSafeCache {
public:
    bool Get(const K& key, V& value) {
        ReadLock lock(rwLock_);
        auto it = cache_.find(key);
        if (it != cache_.end()) {
            value = it->second;
            return true;
        }
        return false;
    }

    void Put(const K& key, const V& value) {
        WriteLock lock(rwLock_);
        cache_[key] = value;
    }

private:
    ReadWriteLock rwLock_;
    std::map<K, V> cache_;
};
```

---

## 锁的获取规则

```
读锁 vs 读锁: 兼容（可以同时持有）
读锁 vs 写锁: 互斥
写锁 vs 读锁: 互斥
写锁 vs 写锁: 互斥

状态转换:
    无锁  -->  读锁1 -->  读锁N -->  无锁
    无锁  -->  写锁1 --------->  无锁
```

---

## 注意事项

1. **不要混用**: 同一线程中不要混用 `ReadLock` 和 `WriteLock`
2. **避免死锁**: 不要在持有锁的情况下尝试获取同一把锁
3. **读者溢出**: `LockRead()` 会检查读者数量是否超过 `uint16_t` 最大值
4. **RAII 优先**: 优先使用 `ReadLock` 和 `WriteLock` 而不是手动调用锁操作
5. **虚假唤醒**: 使用 `while` 循环而不是 `if` 来处理条件变量等待
