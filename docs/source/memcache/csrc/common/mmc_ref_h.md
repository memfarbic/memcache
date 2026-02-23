# mmc_ref.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/common/mmc_ref.h`
- **文件用途**: 提供引用计数智能指针实现，类似于 `std::shared_ptr`
- **依赖项**: `<cstdint>`, `<utility>`, `<new>`

---

## 类定义

### MmcReferable

```cpp
class MmcReferable {
public:
    MmcReferable() = default;
    virtual ~MmcReferable() = default;

    inline void IncreaseRef();
    inline void DecreaseRef();

protected:
    int32_t mRefCount = 0;
};
```

**声明位置**: 行 23-43

**功能描述**: 可引用计数基类，所有需要使用 `MmcRef` 管理的对象必须继承此类

---

#### MmcReferable::MmcReferable()

```cpp
MmcReferable() = default;
```

**声明位置**: 行 25

**功能描述**: 默认构造函数，初始化引用计数为 0

---

#### MmcReferable::~MmcReferable()

```cpp
virtual ~MmcReferable() = default;
```

**声明位置**: 行 26

**功能描述**: 虚析构函数，确保通过基类指针正确析构派生类对象

---

#### MmcReferable::IncreaseRef()

```cpp
inline void IncreaseRef()
{
    __sync_fetch_and_add(&mRefCount, 1);
}
```

**声明位置**: 行 28-31

**功能描述**: 增加引用计数

**代码逻辑**: 使用 GCC 内置原子操作 `__sync_fetch_and_add` 原子地增加引用计数

**原子性**: 保证多线程环境下引用计数操作的原子性

---

#### MmcReferable::DecreaseRef()

```cpp
inline void DecreaseRef()
{
    // delete itself if reference count equal to 0
    if (__sync_sub_and_fetch(&mRefCount, 1) == 0) {
        delete this;
    }
}
```

**声明位置**: 行 33-39

**功能描述**: 减少引用计数，如果引用计数变为 0 则自动删除对象

**代码逻辑**:
1. 使用原子操作减少引用计数
2. 如果结果为 0，调用 `delete this` 销毁对象

**注意事项**:
- 这是自删除（suicide）模式
- 调用后不应再访问对象

---

#### MmcReferable 成员变量

```cpp
protected:
    int32_t mRefCount = 0;
```

**声明位置**: 行 42

**功能描述**: 引用计数，记录当前有多少个 `MmcRef` 指向此对象

---

### MmcRef<T>

```cpp
template<typename T>
class MmcRef {
public:
    MmcRef() noexcept = default;
    MmcRef(T *newObj) noexcept;
    MmcRef(const MmcRef<T> &other) noexcept;
    MmcRef(MmcRef<T> &&other) noexcept;
    ~MmcRef();

    MmcRef<T> &operator=(T *newObj);
    MmcRef<T> &operator=(const MmcRef<T> &other);
    MmcRef<T> &operator=(MmcRef<T> &&other) noexcept;

    bool operator==(const MmcRef<T> &other) const;
    bool operator==(T *other) const;
    bool operator!=(const MmcRef<T> &other) const;
    bool operator!=(T *other) const;

    T *operator->() const;
    T *Get() const;
    void Set(T *newObj);

private:
    T *mObj = nullptr;
};
```

**声明位置**: 行 45-164

**功能描述**: 引用计数智能指针模板类，类似于标准库的 `std::shared_ptr`

---

#### MmcRef<T>::MmcRef() [默认构造]

```cpp
MmcRef() noexcept = default;
```

**声明位置**: 行 49

**功能描述**: 默认构造函数，创建一个空的智能指针

**初始状态**: `mObj = nullptr`

---

#### MmcRef<T>::MmcRef(T*) [指针构造]

```cpp
MmcRef(T *newObj) noexcept
{
    if (newObj != nullptr) {
        newObj->IncreaseRef();
        mObj = newObj;
    }
}
```

**声明位置**: 行 52-60

**功能描述**: 从裸指针构造智能指针

**参数**:
- `newObj`: 要管理的对象指针

**代码逻辑**:
1. 如果指针非空，调用对象的 `IncreaseRef()` 增加引用计数
2. 保存指针到成员变量

**注意事项**: 注释指出不能使用 `explicit`，以支持隐式转换

---

#### MmcRef<T>::MmcRef(const MmcRef<T>&) [拷贝构造]

```cpp
MmcRef(const MmcRef<T> &other) noexcept
{
    if (other.mObj != nullptr) {
        other.mObj->IncreaseRef();
        mObj = other.mObj;
    }
}
```

**声明位置**: 行 62-70

**功能描述**: 拷贝构造函数，共享对象所有权

**参数**:
- `other`: 另一个智能指针的引用

**代码逻辑**:
1. 如果 `other` 持有对象，增加对象的引用计数
2. 两个智能指针指向同一对象

---

#### MmcRef<T>::MmcRef(MmcRef<T>&&) [移动构造]

```cpp
MmcRef(MmcRef<T> &&other) noexcept : mObj(std::__exchange(other.mObj, nullptr))
{
    // move constructor
    // since this mObj is null, just exchange
}
```

**声明位置**: 行 72-76

**功能描述**: 移动构造函数，转移对象所有权

**参数**:
- `other`: 要移动的智能指针

**代码逻辑**:
1. 使用 `std::__exchange` 将 `other` 的指针交换出来
2. 将 `other` 的指针置为 `nullptr`
3. 不修改引用计数（只是转移所有权）

---

#### MmcRef<T>::~MmcRef()

```cpp
~MmcRef()
{
    if (mObj != nullptr) {
        mObj->DecreaseRef();
    }
}
```

**声明位置**: 行 78-84

**功能描述**: 析构函数，减少引用计数

**代码逻辑**:
1. 如果持有对象，调用 `DecreaseRef()`
2. 如果这是最后一个引用，对象会被自动删除

---

#### MmcRef<T>::operator=(T*)

```cpp
inline MmcRef<T> &operator=(T *newObj)
{
    this->Set(newObj);
    return *this;
}
```

**声明位置**: 行 87-91

**功能描述**: 赋值运算符，从裸指针赋值

**参数**:
- `newObj`: 新的对象指针

**返回值**: 自身引用

---

#### MmcRef<T>::operator=(const MmcRef<T>&)

```cpp
inline MmcRef<T> &operator=(const MmcRef<T> &other)
{
    if (this != &other) {
        this->Set(other.mObj);
    }
    return *this;
}
```

**声明位置**: 行 93-99

**功能描述**: 拷贝赋值运算符

**参数**:
- `other`: 另一个智能指针的引用

**返回值**: 自身引用

**代码逻辑**:
1. 检查自赋值
2. 调用 `Set()` 重新设置对象

---

#### MmcRef<T>::operator=(MmcRef<T>&&)

```cpp
MmcRef<T> &operator=(MmcRef<T> &&other) noexcept
{
    if (this != &other) {
        auto tmp = mObj;
        mObj = std::__exchange(other.mObj, nullptr);
        if (tmp != nullptr) {
            tmp->DecreaseRef();
        }
    }
    return *this;
}
```

**声明位置**: 行 101-111

**功能描述**: 移动赋值运算符

**参数**:
- `other`: 要移动的智能指针

**返回值**: 自身引用

**代码逻辑**:
1. 保存旧指针
2. 交换新指针
3. 减少旧对象的引用计数

---

#### MmcRef<T>::operator==

```cpp
inline bool operator==(const MmcRef<T> &other) const
{
    return mObj == other.mObj;
}

inline bool operator==(T *other) const
{
    return mObj == other;
}
```

**声明位置**: 行 114-122

**功能描述**: 相等比较运算符

**参数**:
- `other`: 另一个智能指针或裸指针

**返回值**: 是否指向同一对象

---

#### MmcRef<T>::operator!=

```cpp
inline bool operator!=(const MmcRef<T> &other) const
{
    return mObj != other.mObj;
}

inline bool operator!=(T *other) const
{
    return mObj != other;
}
```

**声明位置**: 行 124-132

**功能描述**: 不等比较运算符

**参数**:
- `other`: 另一个智能指针或裸指针

**返回值**: 是否指向不同对象

---

#### MmcRef<T>::operator->

```cpp
inline T *operator->() const
{
    return mObj;
}
```

**声明位置**: 行 135-138

**功能描述**: 箭头运算符，用于访问对象成员

**返回值**: 对象指针

**使用示例**:
```cpp
MmcRef<MyClass> ref = new MyClass();
ref->SomeMethod();  // 调用 MyClass::SomeMethod()
```

---

#### MmcRef<T>::Get()

```cpp
inline T *Get() const
{
    return mObj;
}
```

**声明位置**: 行 140-143

**功能描述**: 获取裸指针

**返回值**: 管理的对象指针（可能为 nullptr）

---

#### MmcRef<T>::Set()

```cpp
inline void Set(T *newObj)
{
    if (newObj == mObj) {
        return;
    }

    if (newObj != nullptr) {
        newObj->IncreaseRef();
    }

    if (mObj != nullptr) {
        mObj->DecreaseRef();
    }

    mObj = newObj;
}
```

**声明位置**: 行 145-160

**功能描述**: 设置新对象，正确处理引用计数

**参数**:
- `newObj`: 新的对象指针

**代码逻辑**:
1. 检查自赋值（相同对象直接返回）
2. 增加新对象的引用计数
3. 减少旧对象的引用计数（可能导致删除）
4. 更新指针

**注意事项**: 先增加后减少，确保自赋值时不会错误删除对象

---

#### MmcRef<T> 成员变量

```cpp
private:
    T *mObj = nullptr;
```

**声明位置**: 行 163

**功能描述**: 管理的对象指针

---

## 全局函数

### Convert<Src, Des>()

```cpp
template<class Src, class Des>
static MmcRef<Des> Convert(const MmcRef<Src> &child)
{
    if (child.Get() != nullptr) {
        return MmcRef<Des>(static_cast<Des *>(child.Get()));
    }
    return nullptr;
}
```

**声明位置**: 行 166-173

**功能描述**: 派生类到基类的智能指针转换

**参数**:
- `child`: 源类型的智能指针

**返回值**: 目标类型的智能指针

**代码逻辑**:
1. 如果源指针非空，使用 `static_cast` 转换并创建新智能指针
2. 引用计数会自动增加

**使用场景**: 用于类型安全的向下转换

---

### MmcMakeRef<C, ARGS...>()

```cpp
template<typename C, typename... ARGS>
inline MmcRef<C> MmcMakeRef(ARGS... args)
{
    return new (std::nothrow) C(args...);
}
```

**声明位置**: 行 175-179

**功能描述**: 创建对象并返回管理它的智能指针（类似 `std::make_shared`）

**参数**:
- `args`: 传递给对象构造函数的参数

**返回值**: 管理新对象的智能指针

**特性**: 使用 `std::nothrow` 版本的 `new`，分配失败时返回 nullptr 而非抛出异常

**使用示例**:
```cpp
auto obj = MmcMakeRef<MyClass>(arg1, arg2);
```

---

## 文件级别的关系图

```
mmc_ref.h
    |
    +-- MmcReferable (可引用基类)
    |   +-- mRefCount   [引用计数]
    |   +-- IncreaseRef() [增加引用计数]
    |   +-- DecreaseRef() [减少引用计数，为0时自删除]
    |
    +-- MmcRef<T> (智能指针)
    |   +-- mObj        [管理的对象指针]
    |   +-- 构造函数    [增加引用计数]
    |   +-- 析构函数    [减少引用计数]
    |   +-- 拷贝/移动   [正确处理引用计数]
    |   +-- operator->  [访问成员]
    |   +-- Get()/Set() [获取/设置对象]
    |
    +-- Convert()   [类型转换]
    +-- MmcMakeRef() [创建智能指针]
```

---

## 使用示例

### 基本用法

```cpp
// 定义一个可引用的类
class MyData : public MmcReferable {
public:
    MyData(int value) : value_(value) {}
    int value_;
};

// 使用智能指针
MmcRef<MyData> data1 = new MyData(42);
{
    MmcRef<MyData> data2 = data1;  // 共享所有权，引用计数 = 2
    // data2 离开作用域，引用计数 = 1
}
// data1 离开作用域，引用计数 = 0，对象自动删除
```

### 使用 MmcMakeRef

```cpp
auto obj = MmcMakeRef<MyClass>(arg1, arg2);
if (obj.Get() == nullptr) {
    // 创建失败
}
```

### 类型转换

```cpp
class Base : public MmcReferable {};
class Derived : public Base {};

MmcRef<Derived> derived = new Derived();
MmcRef<Base> base = Convert<Derived, Base>(derived);
```

### 与线程池结合

```cpp
class ThreadPoolTask : public MmcReferable {
    // 任务实现
};

MmcRef<ThreadPoolTask> task = new ThreadPoolTask();
threadPool.Enqueue([task]() {
    task->Execute();  // 引用计数保证任务对象在使用期间有效
});
```

---

## 注意事项

1. **继承要求**: 所有被管理的对象必须继承 `MmcReferable`
2. **循环引用**: 与 `std::shared_ptr` 一样，需要注意循环引用问题
3. **线程安全**: 引用计数操作是原子的，但对象本身不是自动线程安全的
4. **空指针**: `MmcRef` 可以持有 nullptr，使用前应检查
5. **移动语义**: 移动操作不修改引用计数，只是转移所有权
