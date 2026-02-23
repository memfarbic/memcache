# mmc_lock.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_lock.h`
- **文件用途**: 提供互斥锁的封装和 RAII 风格的锁管理类
- **依赖项**: `<mutex>`

---

## 类定义

### Lock

```cpp
class Lock {
public:
    Lock() = default;
    ~Lock() = default;

    Lock(const Lock &) = delete;
    Lock &operator=(const Lock &) = delete;
    Lock(Lock &&) = delete;
    Lock &operator=(Lock &&) = delete;

    inline void DoLock()
    {
        mLock.lock();
    }

    inline void Unlock()
    {
        mLock.unlock();
    }

private:
    std::mutex mLock;
};
```

**声明位置**: 行 19-41

**功能描述**: 对 `std::mutex` 的简单封装类

---

#### Lock::Lock()

```cpp
Lock() = default;
```

**声明位置**: 行 21

**功能描述**: 默认构造函数

**说明**: 使用 `std::mutex` 的默认构造

---

#### Lock::~Lock()

```cpp
~Lock() = default;
```

**声明位置**: 行 22

**功能描述**: 默认析构函数

**注意事项**: 析构时不自动释放锁，需要手动调用 `Unlock()`

---

#### Lock::DoLock()

```cpp
inline void DoLock()
{
    mLock.lock();
}
```

**声明位置**: 行 29-32

**功能描述**: 获取锁

**代码逻辑**: 调用底层 `std::mutex::lock()` 阻塞直到获取锁

**注意事项**: 如果锁已被占用，当前线程会阻塞

---

#### Lock::Unlock()

```cpp
inline void Unlock()
{
    mLock.unlock();
}
```

**声明位置**: 行 34-37

**功能描述**: 释放锁

**代码逻辑**: 调用底层 `std::mutex::unlock()` 释放锁

**注意事项**: 必须在已获取锁的情况下调用，否则行为未定义

---

#### Lock 禁止的操作

```cpp
Lock(const Lock &) = delete;              // 禁止拷贝构造
Lock &operator=(const Lock &) = delete;   // 禁止拷贝赋值
Lock(Lock &&) = delete;                   // 禁止移动构造
Lock &operator=(Lock &&) = delete;        // 禁止移动赋值
```

**声明位置**: 行 24-27

**功能描述**: 禁止拷贝和移动操作

**原因**: 互斥锁不可复制或移动

---

#### Lock 成员变量

```cpp
private:
    std::mutex mLock;
```

**声明位置**: 行 40

**功能描述**: 底层的标准库互斥锁对象

---

### Locker<T>

```cpp
template<class T>
class Locker {
public:
    explicit Locker(T *lock) : mLock(lock)
    {
        if (mLock != nullptr) {
            mLock->DoLock();
        }
    }

    ~Locker()
    {
        if (mLock != nullptr) {
            mLock->Unlock();
        }
    }

    Locker(const Locker &) = delete;
    Locker &operator=(const Locker &) = delete;
    Locker(Locker &&) = delete;
    Locker &operator=(Locker &&) = delete;

private:
    T *mLock;
};
```

**声明位置**: 行 43-67

**功能描述**: RAII 风格的锁管理模板类（Scoped Lock）

**用途**: 在作用域内自动管理锁的生命周期，构造时获取锁，析构时释放锁

---

#### Locker<T>::Locker()

```cpp
explicit Locker(T *lock) : mLock(lock)
{
    if (mLock != nullptr) {
        mLock->DoLock();
    }
}
```

**声明位置**: 行 46-51

**功能描述**: 构造函数，获取锁

**参数**:
- `lock`: 指向 Lock 对象的指针

**代码逻辑**:
1. 保存锁指针
2. 如果指针非空，调用 `DoLock()` 获取锁

**注意事项**: 使用 `explicit` 防止隐式转换

---

#### Locker<T>::~Locker()

```cpp
~Locker()
{
    if (mLock != nullptr) {
        mLock->Unlock();
    }
}
```

**声明位置**: 行 53-58

**功能描述**: 析构函数，释放锁

**代码逻辑**: 如果锁指针非空，调用 `Unlock()` 释放锁

**RAII 特性**: 确保异常发生时也能正确释放锁

---

#### Locker 禁止的操作

```cpp
Locker(const Locker &) = delete;              // 禁止拷贝构造
Locker &operator=(const Locker &) = delete;   // 禁止拷贝赋值
Locker(Locker &&) = delete;                   // 禁止移动构造
Locker &operator=(Locker &&) = delete;        // 禁止移动赋值
```

**声明位置**: 行 60-63

**功能描述**: 禁止拷贝和移动操作

**原因**: 锁的管理权不可转移

---

#### Locker 成员变量

```cpp
private:
    T *mLock;
```

**声明位置**: 行 66

**功能描述**: 指向被管理的锁对象的指针

---

## 宏定义

### GUARD

```cpp
#define GUARD(lLock, alias) Locker<Lock> __l##alias(lLock)
```

**声明位置**: 行 69

**功能描述**: 创建一个作用域锁守卫的便捷宏

**参数**:
- `lLock`: Lock 对象的指针
- `alias`: 变量名后缀（用于创建唯一变量名）

**代码逻辑**: 创建一个名为 `__l<alias>` 的 `Locker<Lock>` 局部变量

**使用示例**:
```cpp
Lock myLock;
{
    GUARD(&myLock, guard1);  // 创建变量 __lguard1
    // 临界区代码
    // 作用域结束时自动释放锁
}
```

---

## 文件级别的关系图

```
mmc_lock.h
    |
    +-- Lock (互斥锁封装)
    |   +-- DoLock()   [获取锁]
    |   +-- Unlock()   [释放锁]
    |   +-- mLock      [std::mutex]
    |
    +-- Locker<T> (RAII 锁管理)
    |   +-- 构造时自动获取锁
    |   +-- 析构时自动释放锁
    |
    +-- GUARD 宏 (便捷创建守卫)
```

---

## 使用示例

### 使用 Locker 手动管理

```cpp
Lock myLock;

{
    Locker<Lock> guard(&myLock);
    // 临界区：锁已获取
    sharedData++;
    // 离开作用域：自动释放锁
}
```

### 使用 GUARD 宏

```cpp
Lock myLock;

void CriticalSection() {
    GUARD(&myLock, critical);
    // 临界区代码
    // 自动管理锁
}
```

### 线程安全示例

```cpp
class ThreadSafeCounter {
public:
    void Increment() {
        GUARD(&lock_, inc);
        counter_++;
    }

    int Get() {
        GUARD(&lock_, get);
        return counter_;
    }

private:
    Lock lock_;
    int counter_ = 0;
};
```

---

## 注意事项

1. **禁止拷贝**: `Lock` 和 `Locker` 都禁止拷贝和移动操作
2. **RAII 保证**: `Locker` 确保在任何情况下（包括异常）都能释放锁
3. **空指针安全**: `Locker` 会检查空指针，不会对 nullptr 调用锁操作
4. **命名规范**: `GUARD` 宏生成的变量名以 `__l` 开头，避免命名冲突
