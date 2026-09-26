# Print_FMT

基于 [`{fmt}`](https://github.com/fmtlib/fmt) 的精简版本，用于 **C/C++ 格式化库学习、代码裁剪以及嵌入式平台移植**。

本项目保留 `{fmt}` 的核心格式化功能，并对原项目的目录结构、构建系统、测试代码和文档进行精简，方便在资源受限或构建环境较简单的平台中进行移植和学习。

---

## 一、项目特点

* 基于 `{fmt}` 源码进行整理和裁剪
* 保留常用格式化功能
* 精简原项目目录结构
* 使用 CMake 管理库和测试构建
* 支持独立构建静态库
* 保留核心、可选以及嵌入式相关测试
* 提供中文 API、语法和快速入门文档
* 适合作为嵌入式平台移植的参考代码

当前项目主要关注：

```text
源码结构
    ↓
CMake 构建
    ↓
格式化功能
    ↓
测试验证
    ↓
嵌入式平台移植
```

---

## 二、项目结构

```text
Print_FMT/
├── CMakeLists.txt
├── LICENSE
├── README.md
│
├── include/
│   └── fmt/
│       ├── args.h
│       ├── base.h
│       ├── chrono.h
│       ├── color.h
│       ├── compile.h
│       ├── core.h
│       ├── enum.h
│       ├── fmt-c.h
│       ├── format.h
│       ├── format-inl.h
│       ├── os.h
│       ├── ostream.h
│       ├── printf.h
│       ├── ranges.h
│       ├── std.h
│       └── xchar.h
│
├── src/
│   ├── fmt-c.cc
│   ├── format.cc
│   └── os.cc
│
├── cases/
│   ├── CMakeLists.txt
│   ├── core/
│   ├── optional/
│   ├── embedded/
│   └── support/
│
└── docs/
    ├── api_zh.md
    ├── get-started_zh.md
    └── syntax_zh.md
```

### 目录说明

| 目录 / 文件                | 作用             |
| ---------------------- | -------------- |
| `include/fmt/`         | 格式化库头文件        |
| `src/`                 | 需要编译的源文件       |
| `cases/core/`          | 核心功能测试         |
| `cases/optional/`      | 可选功能测试         |
| `cases/embedded/`      | 嵌入式相关测试        |
| `cases/support/`       | 测试框架及测试辅助代码    |
| `docs/`                | 中文使用、API 和语法文档 |
| `CMakeLists.txt`       | 主构建脚本          |
| `cases/CMakeLists.txt` | 测试构建脚本         |

---

## 三、构建库

项目使用 CMake 构建。

### 1. 创建构建目录

```bash
cmake -S . -B build -G "MinGW Makefiles"
```

### 2. 编译

```bash
cmake --build build
```

构建完成后会生成 `fmt` 静态库。

---

## 四、构建测试

默认情况下：

```cmake
FMT_BUILD_TEST=OFF
```

只构建库，不构建测试。

如果需要构建测试：

```bash
cmake -S . -B build -G "MinGW Makefiles" -DFMT_BUILD_TEST=ON
```

然后：

```bash
cmake --build build
```

运行测试：

```bash
ctest --test-dir build --output-on-failure
```

测试代码按照功能划分：

```text
cases/
├── core/       核心功能
├── optional/   可选功能
├── embedded/   嵌入式相关
└── support/    测试支持代码
```

---

## 五、基本使用

### 格式化字符串

```cpp
#include <fmt/format.h>

int main() {
    auto text = fmt::format("value = {}", 123);
}
```

### 输出格式化内容

```cpp
#include <fmt/base.h>

int main() {
    fmt::print("Hello, {}!\n", "world");
}
```

### 多个参数

```cpp
#include <fmt/format.h>

int main() {
    auto text = fmt::format(
        "name = {}, value = {}",
        "test",
        123
    );
}
```

---

## 六、源码组成

当前静态库主要由以下源文件组成：

```text
src/
├── format.cc
├── os.cc
└── fmt-c.cc
```

对应头文件位于：

```text
include/fmt/
```

其中：

* `format.cc`：主要格式化实现
* `os.cc`：操作系统相关功能
* `fmt-c.cc`：C API 相关实现

具体 API 和功能可以参考：

* [`docs/api_zh.md`](docs/api_zh.md)
* [`docs/syntax_zh.md`](docs/syntax_zh.md)
* [`docs/get-started_zh.md`](docs/get-started_zh.md)

---

## 七、嵌入式移植

本项目的一个主要用途是作为 `{fmt}` 的**嵌入式移植参考**。

移植时主要关注：

```text
include/fmt/
        │
        ├── 头文件依赖
        │
        ▼
src/
        │
        ├── format.cc
        ├── os.cc
        └── fmt-c.cc
        │
        ▼
目标平台编译器 / SDK
```

对于资源受限的平台，可以根据实际需求进一步裁剪：

* 不需要的功能头文件
* 不需要的源文件
* 与目标平台无关的 OS 功能
* 不需要的测试代码

建议先保证核心格式化功能正常，再根据目标平台逐步进行裁剪。

---

## 八、测试与移植验证

在修改源码或进行平台移植后，可以按照以下顺序验证：

```text
1. 编译 fmt 库
       ↓
2. 编译测试程序
       ↓
3. 运行 CTest
       ↓
4. 确认核心格式化功能
       ↓
5. 再进行目标平台移植
```

这样可以将：

```text
源码问题
```

与：

```text
平台移植问题
```

尽量分开。

---

## 九、中文文档

项目目前保留以下中文文档：

### API 文档

[`docs/api_zh.md`](docs/api_zh.md)

用于了解主要 API 和接口使用方式。

### 快速入门

[`docs/get-started_zh.md`](docs/get-started_zh.md)

用于了解项目构建、使用以及不同构建方式。

### 语法

[`docs/syntax_zh.md`](docs/syntax_zh.md)

用于了解格式字符串的语法和格式说明。

---

## 十、项目定位

`Print_FMT` 并不是对上游 `{fmt}` 项目的完整复制。

本项目主要用于：

```text
学习
 ↓
理解源码
 ↓
理解构建系统
 ↓
裁剪不需要的功能
 ↓
测试
 ↓
嵌入式平台移植
```

如果需要完整的 `{fmt}` 项目、完整测试体系、完整文档系统以及最新上游功能，请参考官方项目：

https://github.com/fmtlib/fmt

---

## 十一、许可证

本项目基于 `{fmt}` 整理。

相关许可证信息请参阅：

```text
LICENSE
```

上游 `{fmt}` 项目：

https://github.com/fmtlib/fmt
