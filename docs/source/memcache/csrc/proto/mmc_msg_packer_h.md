# mmc_msg_packer.h 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/csrc/proto/mmc_msg_packer.h`
- **文件用途**: 提供网络消息的序列化和反序列化功能，支持多种数据类型
- **依赖项**:
  - `mmc_common_includes.h`
  - `<vector>` - STL 容器

---

## 常量定义

### MAX_CONTAINER_SIZE

```cpp
constexpr size_t MAX_CONTAINER_SIZE = 1024 * 1024;
```

**声明位置**: 行 18

**功能描述**: 容器的最大大小限制，用于防止反序列化时分配过大的内存

**用途**: 在反序列化 `vector` 和 `map` 时检查容器大小，超过此限制将记录错误并跳过反序列化

---

## 类定义

### NetMsgPacker

**声明位置**: 行 22-107

**功能描述**: 消息序列化器，将各种数据类型转换为字节流

**私有成员**:

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `outStream_` | `std::ostringstream` | 输出字符串流，用于存储序列化结果 |

**公共方法**:

---

#### Serialize(T&) - POD 类型

```cpp
template<typename T>
void Serialize(const T &val, typename std::enable_if<std::is_trivially_copyable<T>::value, int>::type = 0)
```

**声明位置**: 行 30-34

**功能描述**: 序列化 POD (Plain Old Data) 类型数据

**模板参数**:
- `T`: POD 类型，必须满足 `std::is_trivially_copyable` 条件

**参数**:
- `val`: 要序列化的值引用

**代码逻辑**:
```cpp
outStream_.write(reinterpret_cast<const char *>(&val), sizeof(T));
```
直接将内存内容写入输出流

**支持的类型**:
- 基本类型: `int`, `float`, `double`, `char` 等
- 枚举类型
- 简单结构体 (无虚函数、无复杂构造函数)

**使用示例**:
```cpp
int32_t value = 42;
packer.Serialize(value);  // 序列化为 4 字节
```

---

#### Serialize(string) - 字符串类型

```cpp
void Serialize(const std::string &val)
```

**声明位置**: 行 41-46

**功能描述**: 序列化字符串类型

**参数**:
- `val`: 要序列化的字符串引用

**代码逻辑**:
1. 首先写入字符串的长度 (uint32_t)
2. 然后写入字符串的实际内容

**序列化格式**:
```
[长度(4字节)][字符串内容]
```

**使用示例**:
```cpp
std::string str = "hello";
packer.Serialize(str);
// 序列化结果: 0x05 0x00 0x00 0x00 'h' 'e' 'l' 'l' 'o'
```

---

#### Serialize(pair<K,V>) - pair 类型

```cpp
template<typename K, typename V>
void Serialize(const std::pair<K, V> &val)
```

**声明位置**: 行 55-60

**功能描述**: 序列化 std::pair 类型

**模板参数**:
- `K`: pair 的 first 类型
- `V`: pair 的 second 类型

**参数**:
- `val`: 要序列化的 pair 引用

**代码逻辑**:
1. 递归调用 `Serialize(val.first)` 序列化 first 元素
2. 递归调用 `Serialize(val.second)` 序列化 second 元素

**使用示例**:
```cpp
std::pair<int, std::string> p = {42, "answer"};
packer.Serialize(p);
```

---

#### Serialize(vector<V>) - vector 类型

```cpp
template<typename V>
void Serialize(const std::vector<V> &container)
```

**声明位置**: 行 68-76

**功能描述**: 序列化 std::vector 容器

**模板参数**:
- `V`: vector 元素类型

**参数**:
- `container`: 要序列化的 vector 引用

**代码逻辑**:
1. 写入容器大小 (size_t)
2. 遍历容器，递归序列化每个元素

**序列化格式**:
```
[元素个数(8字节)][元素1][元素2]...[元素N]
```

**使用示例**:
```cpp
std::vector<int> nums = {1, 2, 3};
packer.Serialize(nums);
```

---

#### Serialize(map<K,V>) - map 类型

```cpp
template<typename K, typename V>
void Serialize(const std::map<K, V> &container)
```

**声明位置**: 行 85-93

**功能描述**: 序列化 std::map 容器

**模板参数**:
- `K`: map 键类型
- `V`: map 值类型

**参数**:
- `container`: 要序列化的 map 引用

**代码逻辑**:
1. 写入容器大小 (size_t)
2. 遍历容器，将每个键值对作为 pair 序列化

**序列化格式**:
```
[元素个数(8字节)][键值对1][键值对2]...[键值对N]
```

**注意事项**: map 的遍历顺序是按键排序的顺序

**使用示例**:
```cpp
std::map<std::string, int> m = {{"one", 1}, {"two", 2}};
packer.Serialize(m);
```

---

#### String()

```cpp
std::string String() const
```

**声明位置**: 行 100-103

**功能描述**: 获取序列化后的字符串结果

**返回值**: 包含序列化数据的字符串

**代码逻辑**:
```cpp
return outStream_.str();
```

**使用示例**:
```cpp
NetMsgPacker packer;
packer.Serialize(42);
std::string data = packer.String();  // 获取序列化结果
```

---

### NetMsgUnpacker

**声明位置**: 行 109-195

**功能描述**: 消息反序列化器，从字节流恢复各种数据类型

**私有成员**:

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `inStream_` | `std::istringstream` | 输入字符串流，用于读取序列化数据 |

**公共方法**:

---

#### NetMsgUnpacker() - 构造函数

```cpp
explicit NetMsgUnpacker(const std::string &value) : inStream_(value) {}
```

**声明位置**: 行 116

**功能描述**: 使用序列化数据创建解包器

**参数**:
- `value`: 包含序列化数据的字符串

**代码逻辑**: 使用传入的字符串初始化输入流

---

#### Deserialize(T&) - POD 类型

```cpp
template<typename T>
void Deserialize(T &val, typename std::enable_if<std::is_trivially_copyable<T>::value, int>::type = 0)
```

**声明位置**: 行 124-128

**功能描述**: 反序列化 POD 类型数据

**模板参数**:
- `T`: POD 类型

**参数**:
- `val`: 接收反序列化结果的引用

**代码逻辑**:
```cpp
inStream_.read(reinterpret_cast<char *>(&val), sizeof(T));
```
直接从输入流读取内存到目标变量

**使用示例**:
```cpp
int32_t value;
unpacker.Deserialize(value);  // 从流中读取 4 字节
```

---

#### Deserialize(string) - 字符串类型

```cpp
void Deserialize(std::string &val)
```

**声明位置**: 行 135-141

**功能描述**: 反序列化字符串类型

**参数**:
- `val`: 接收反序列化结果的字符串引用

**代码逻辑**:
1. 读取字符串长度 (uint32_t)
2. 调整字符串大小
3. 读取字符串内容

**使用示例**:
```cpp
std::string str;
unpacker.Deserialize(str);
```

---

#### Deserialize(vector<V>) - vector 类型

```cpp
template<typename V>
void Deserialize(std::vector<V> &container)
```

**声明位置**: 行 149-165

**功能描述**: 反序列化 std::vector 容器

**模板参数**:
- `V`: vector 元素类型

**参数**:
- `container`: 接收反序列化结果的 vector 引用

**代码逻辑**:
1. 读取容器大小
2. **安全检查**: 如果大小超过 `MAX_CONTAINER_SIZE`，记录错误并返回
3. 清空容器并预分配空间
4. 逐个反序列化元素，使用 `std::move` 优化性能

**使用示例**:
```cpp
std::vector<int> nums;
unpacker.Deserialize(nums);
```

---

#### Deserialize(map<K,V>) - map 类型

```cpp
template<typename K, typename V>
void Deserialize(std::map<K, V> &container)
```

**声明位置**: 行 174-191

**功能描述**: 反序列化 std::map 容器

**模板参数**:
- `K`: map 键类型
- `V`: map 值类型

**参数**:
- `container`: 接收反序列化结果的 map 引用

**代码逻辑**:
1. 读取容器大小
2. **安全检查**: 如果大小超过 `MAX_CONTAINER_SIZE`，记录错误并返回
3. 清空容器
4. 循环读取键值对:
   - 反序列化键
   - 反序列化值
   - 使用 `emplace` 和 `std::move` 插入

**使用示例**:
```cpp
std::map<std::string, int> m;
unpacker.Deserialize(m);
```

---

## 序列化格式规范

### 基本类型
```
直接写入内存的二进制表示
```

### 字符串
```
[uint32_t: 长度][char * N: 内容]
```

### 容器 (vector, map)
```
[size_t: 元素个数][元素1][元素2]...[元素N]
```

### Pair
```
[first元素][second元素]
```

---

## 使用示例

### 基本序列化/反序列化

```cpp
// 序列化
NetMsgPacker packer;
int32_t value = 12345;
std::string str = "hello";
packer.Serialize(value);
packer.Serialize(str);
std::string data = packer.String();

// 反序列化
NetMsgUnpacker unpacker(data);
int32_t decodedValue;
std::string decodedStr;
unpacker.Deserialize(decodedValue);
unpacker.Deserialize(decodedStr);
// decodedValue == 12345, decodedStr == "hello"
```

### 复杂类型序列化

```cpp
// 序列化 map
NetMsgPacker packer;
std::map<std::string, int> config = {{"timeout", 30}, {"retries", 3}};
packer.Serialize(config);
std::string data = packer.String();

// 反序列化 map
NetMsgUnpacker unpacker(data);
std::map<std::string, int> decodedConfig;
unpacker.Deserialize(decodedConfig);
```

### 自定义结构体序列化

```cpp
struct MyStruct {
    int id;
    std::string name;
    std::vector<int> scores;

    Result Serialize(NetMsgPacker &packer) const {
        packer.Serialize(id);
        packer.Serialize(name);
        packer.Serialize(scores);
        return MMC_OK;
    }

    Result Deserialize(NetMsgUnpacker &unpacker) {
        unpacker.Deserialize(id);
        unpacker.Deserialize(name);
        unpacker.Deserialize(scores);
        return MMC_OK;
    }
};
```

---

## 线程安全性

**NetMsgPacker**: 非线程安全，每个线程应使用独立的实例

**NetMsgUnpacker**: 非线程安全，每个线程应使用独立的实例

---

## 错误处理

反序列化时的安全机制:
1. **容器大小限制**: 超过 `MAX_CONTAINER_SIZE` 时记录错误并跳过
2. **流状态检查**: 读取失败时 `inStream_` 会设置 failbit，用户可以通过检查流状态判断是否成功

**建议使用方式**:
```cpp
std::vector<int> data;
unpacker.Deserialize(data);
if (unpacker.inStream_.fail()) {
    // 处理错误
}
```

---

## 性能考虑

1. **字符串优化**: 使用 `resize` 和直接写入避免额外拷贝
2. **容器预分配**: 反序列化容器时使用 `reserve` 预分配空间
3. **移动语义**: 反序列化时使用 `std::move` 减少拷贝

---

## 依赖关系

**被以下文件依赖**:
- `mmc_msg_base.h` - 消息基类使用序列化器
- `mmc_msg_client_meta.h` - 所有消息类型使用序列化器

**依赖的文件**:
- `mmc_common_includes.h` - 公共头文件

---

## 命名空间

```cpp
namespace ock {
namespace mmc {
    class NetMsgPacker;
    class NetMsgUnpacker;
}
}
```
