# setup.py 逐函数解读

## 文件概述

- **文件路径**: `src/memcache/python/setup.py`
- **文件用途**: Python 包的构建和安装配置，支持 CMake 构建和 wheel 打包
- **依赖项**: `os`, `sys`, `platform`, `glob`, `subprocess`, `setuptools`

---

## 环境变量读取

### check_env_flag() (第27-28行)

```python
def check_env_flag(name: str, default: str = "") -> bool:
    return os.getenv(name, default).upper() in ["ON", "1", "YES", "TRUE", "Y"]
```

**声明位置**: 行 27-28

**功能**: 检查环境变量是否为真值

**参数**:
- `name`: 环境变量名
- `default`: 默认值

**返回值**: 布尔值

---

### 环境变量列表 (第31-39行)

```python
os.environ["SOURCE_DATE_EPOCH"] = "0"  # 消除whl压缩包的时间戳差异
current_version = os.getenv("MEMCACHE_VERSION")
is_manylinux = check_env_flag("IS_MANYLINUX", "FALSE")
build_open_abi = os.getenv("BUILD_OPEN_ABI", "OFF")
build_mode = os.getenv("BUILD_MODE", "RELEASE")
enable_ptracer = os.getenv("ENABLE_PTRACER", "ON")
python3_executable = os.getenv("PYTHON3_EXECUTABLE", "/usr/local/bin/python3")
```

**声明位置**: 行 31-39

**环境变量说明**:

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| SOURCE_DATE_EPOCH | 0 | 消除 whl 时间戳差异 |
| MEMCACHE_VERSION | - | 包版本号（必须设置） |
| IS_MANYLINUX | FALSE | 是否为 manylinux 构建 |
| BUILD_OPEN_ABI | OFF | 是否构建开放 ABI |
| BUILD_MODE | RELEASE | 构建模式 (RELEASE/DEBUG) |
| ENABLE_PTRACER | ON | 是否启用性能追踪 |
| PYTHON3_EXECUTABLE | /usr/local/bin/python3 | Python3 路径 |

---

## 类定义

### BinaryDistribution (第42-46行)

```python
class BinaryDistribution(Distribution):
    """Distribution which always forces a binary package with platform name"""

    def has_ext_modules(self):
        return True
```

**声明位置**: 行 42-46

**功能**: 强制生成带平台名的二进制包

**说明**: 即使没有扩展模块，也生成平台特定的 wheel

---

### BuildWheel (第49-68行)

```python
class BuildWheel(bdist_wheel):
    def run(self):
        bdist_wheel.run(self)

        if is_manylinux:
            file = glob.glob(os.path.join(self.dist_dir, "*-linux_*.whl"))[0]
            auditwheel_cmd = [
                "auditwheel",
                "-v",
                "repair",
                "--plat",
                f"manylinux_2_27_{platform.machine()}",
                "--plat",
                f"manylinux_2_28_{platform.machine()}",
                "-w",
                self.dist_dir,
                file,
            ]
            subprocess.check_call(auditwheel_cmd)
            os.remove(file)
```

**声明位置**: 行 49-68

**功能**: 自定义 wheel 构建过程

**行为**:
1. 先运行默认的 bdist_wheel
2. 如果是 manylinux 构建，使用 auditwheel repair 处理
3. 支持多个 manylinux 版本 (2_27, 2_28)
4. 删除原始 wheel，保留 repair 后的

---

### CMakeBuildExt (第71-112行)

```python
class CMakeBuildExt(build_ext):
    def run(self):
        # 获取目录路径
        root_dir = os.path.abspath(os.path.join(os.path.dirname(__file__), "../../../"))
        build_dir = os.path.abspath(os.path.join(root_dir, "build"))
        install_dir = os.path.abspath(os.path.join(build_dir, "install"))
        os.makedirs(build_dir, exist_ok=True)
        config_mode = "Release"
        if build_mode == "DEBUG":
            config_mode = "Debug"
```

**声明位置**: 行 71-112

**功能**: 使用 CMake 构建扩展

**run() 方法逻辑**:

1. **设置路径** (第73-77行):
   ```python
   root_dir = os.path.abspath(os.path.join(os.path.dirname(__file__), "../../../"))
   build_dir = os.path.abspath(os.path.join(root_dir, "build"))
   install_dir = os.path.abspath(os.path.join(build_dir, "install"))
   ```

2. **确定构建模式** (第78-80行):
   ```python
   config_mode = "Release"
   if build_mode == "DEBUG":
       config_mode = "Debug"
   ```

3. **运行 CMake 配置** (第80-93行):
   ```python
   subprocess.check_call([
       "cmake",
       f"-S{root_dir}",
       f"-B{build_dir}",
       f"-DPython3_EXECUTABLE={python3_executable}",
       f"-DCMAKE_INSTALL_PREFIX={install_dir}",
       f"-DCMAKE_BUILD_TYPE={build_mode}",
       f"-DBUILD_OPEN_ABI={build_open_abi}",
       f"-DENABLE_PTRACER={enable_ptracer}",
       "-DBUILD_PYTHON=ON",
       "-DBUILD_TESTS=OFF",
   ])
   ```

4. **运行 CMake 构建** (第95-106行):
   ```python
   subprocess.check_call([
       "cmake",
       "--build",
       build_dir,
       "--config",
       config_mode,
       "--target",
       "install",
       "-j8",
   ])
   ```

5. **调用父类** (第108行)

---

### build_extension() (第110-111行)

```python
def build_extension(self, ext):
    super().build_extension(ext)
```

**声明位置**: 行 110-111

**功能**: 构建单个扩展（调用父类实现）

---

## setup() 配置

### setup() 调用 (第114-131行)

```python
setup(
    name="memcache_hybrid",
    version=current_version,
    author="",
    author_email="",
    description="python api for memcache_hybrid",
    packages=find_namespace_packages(exclude=("tests*",)),
    url="https://gitcode.com/Ascend/memcache",
    license="Mulan PSL v2",
    python_requires=">=3.7",
    zip_safe=False,
    package_data={"memcache_hybrid": ["_pymmc.cpython*.so", "lib/**", "VERSION"]},
    cmdclass={
        "build_ext": CMakeBuildExt,
        "bdist_wheel": BuildWheel,
    },
    distclass=BinaryDistribution,
)
```

**声明位置**: 行 114-131

**参数说明**:

| 参数 | 值 | 说明 |
|------|-----|------|
| name | memcache_hybrid | 包名称 |
| version | current_version | 版本号（从环境变量读取） |
| description | python api for memcache_hybrid | 包描述 |
| packages | find_namespace_packages | 使用命名空间包 |
| url | https://gitcode.com/Ascend/memcache | 项目主页 |
| license | Mulan PSL v2 | 许可证 |
| python_requires | >=3.7 | Python 版本要求 |
| zip_safe | False | 不使用 zip 安全模式 |
| package_data | _pymmc.so, lib/, VERSION | 包含的数据文件 |
| cmdclass | 自定义命令 | 使用自定义构建命令 |
| distclass | BinaryDistribution | 使用自定义分发类 |

---

## 构建流程

```
用户执行 pip install .
    |
    v
setup.py 被调用
    |
    +-- CMakeBuildExt.run()
    |       |
    |       +-- 创建 build/ 目录
    |       |
    |       +-- 运行 CMake 配置
    |       |   -S{root_dir} -B{build_dir}
    |       |   -DBUILD_PYTHON=ON
    |       |   -DBUILD_TESTS=OFF
    |       |
    |       +-- 运行 CMake 构建
    |       |   --build {build_dir}
    |       |   --target install
    |       |
    |       +-- 生成 _pymmc.so 和 libmf_memcache.so
    |
    +-- BuildWheel.run() (如果 bdist_wheel)
    |       |
    |       +-- 生成 wheel 文件
    |       |
    |       +-- 如果 is_manylinux:
    |           +-- 运行 auditwheel repair
    |           +-- 生成兼容多个 manylinux 版本的 wheel
    |
    v
安装完成
```

---

## 包结构

```
memcache_hybrid/
├── __init__.py              # 包入口
├── meta_service_leader_election.py  # Leader 选举
├── _pymmc.cpython-*.so      # Pybind11 生成的扩展模块
├── lib/
│   └── libmf_memcache.so    # C++ 扩展库
└── VERSION                  # 版本文件
```

---

## manylinux 构建

manylinux 是一个标准，确保 Python wheel 在多个 Linux 发行版上兼容。

**支持的 manylinux 版本**:
- manylinux_2_27_x86_64
- manylinux_2_28_x86_64

**auditwheel repair 处理**:
1. 分析 wheel 的依赖
2. 将依赖库打包到 wheel 中
3. 修改 wheel 标签为 manylinux
4. 确保在目标系统上可运行

---

## 使用示例

### 直接安装

```bash
# 设置版本
export MEMCACHE_VERSION="1.0.0"

# 安装
pip install .
```

### 构建 wheel

```bash
# 设置版本
export MEMCACHE_VERSION="1.0.0"

# 构建 wheel
python setup.py bdist_wheel

# 输出在 dist/
```

### manylinux 构建

```bash
# 在 manylinux 容器中
export IS_MANYLINUX="TRUE"
export MEMCACHE_VERSION="1.0.0"

python setup.py bdist_wheel
```

### Debug 构建

```bash
export BUILD_MODE="DEBUG"
python setup.py bdist_wheel
```

### 指定 Python

```bash
export PYTHON3_EXECUTABLE="/usr/bin/python3.10"
python setup.py bdist_wheel
```

---

## 注意事项

1. **版本号**: 必须设置 `MEMCACHE_VERSION` 环境变量
2. **Python 版本**: 需要 Python 3.7+
3. **CMake**: 系统需要安装 CMake
4. **编译器**: 需要 C++ 编译器支持 C++17
5. **manylinux**: 在容器中构建以获得最佳兼容性
6. **auditwheel**: manylinux 构建需要安装 auditwheel

---

## 依赖关系

```
setup.py
    |
    +-- 依赖: setuptools
    +-- 依赖: wheel
    +-- 依赖: CMake (外部)
    +-- 依赖: auditwheel (manylinux 构建)
    +-- 调用: CMake 构建系统
    +-- 生成: _pymmc.so (Pybind11)
    +-- 生成: libmf_memcache.so
```
