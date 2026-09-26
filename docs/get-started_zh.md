# 开始使用

可以通过 [Compiler Explorer](https://godbolt.org/z/P7h6cd6o3) 在线编译和运行 {fmt} 示例。

{fmt} 与任何构建系统兼容。下一节介绍如何在 CMake 中使用它，而
[构建系统](#build-systems)一节介绍其他构建方式。

## CMake

{fmt} 提供以下 CMake targets：`fmt::fmt` 用于标准编译库，
`fmt::fmt-header-only` 用于仅头文件库；启用 `FMT_MODULE` 选项后，
还可以使用 `fmt::fmt-module` 用于 C++ module 库。推荐使用编译库或
module 库，以获得更好的构建速度。

在 CMake 中使用 {fmt} 主要有三种方式：

* **FetchContent**：从 CMake 3.11 开始，可以使用 [`FetchContent`](
  https://cmake.org/cmake/help/v3.30/module/FetchContent.html) 在配置时自动
  下载 {fmt} 作为依赖：

        include(FetchContent)

        FetchContent_Declare(
          fmt
          GIT_REPOSITORY https://github.com/fmtlib/fmt
          GIT_TAG        e69e5f977d458f2650bb346dadf2ad30c5320281) # 10.2.1
        FetchContent_MakeAvailable(fmt)

        target_link_libraries(<your-target> fmt::fmt)

* **已安装**：可以在 `CMakeLists.txt` 中查找并使用已经[安装](#installation)
  的 {fmt}：

        find_package(fmt)
        target_link_libraries(<your-target> fmt::fmt)

* **嵌入**：可以将 {fmt} 源代码目录添加到项目中，并在 `CMakeLists.txt`
  中加入：

        add_subdirectory(fmt)
        target_link_libraries(<your-target> fmt::fmt)

### 备用 Targets

如果要使用仅头文件 target 或 module target，只需将上述步骤中的
`fmt::fmt` 分别替换为 `fmt::fmt-header-only` 或 `fmt::fmt-module`。

### 使用 C++20 Module

只有启用 `FMT_MODULE` CMake 选项后，`fmt::fmt-module` target 才可用。
可以在添加 {fmt} 之前配置项目时传入 `-DFMT_MODULE=ON` 来启用，
或者在添加 {fmt} 之前将 `CMAKE_CXX_STANDARD` 设置为至少 20；
当工具链支持时，这将自动启用 module 支持。

将你的 target 链接到 `fmt::fmt-module`，并使用 `import fmt`，
而不是包含 {fmt} 头文件：

    target_link_libraries(<your-target> PRIVATE fmt::fmt-module)

    import fmt;

    int main() {
      fmt::print("Hello, world!\n");
    }

使用 CMake 原生 C++ module 支持时，需要 CMake 3.28 或更高版本、
Ninja 1.11 或更高版本（使用 Ninja generator），以及 GCC 15 或更高版本
（使用 GCC）。{fmt} 还为其他工具链提供了备用构建路径。

## 安装

### Debian/Ubuntu

要在 Debian、Ubuntu 或其他基于 Debian 的 Linux 发行版上安装 {fmt}，
使用以下命令：

    apt install libfmt-dev

### Homebrew

在 macOS 上使用 [Homebrew](https://brew.sh/) 安装 {fmt}：

    brew install fmt

### Conda

可以在 Linux、macOS 和 Windows 上使用 [Conda](https://docs.conda.io/en/latest/)
安装 {fmt}，使用其 [conda-forge package](
https://github.com/conda-forge/fmt-feedstock)：

    conda install -c conda-forge fmt

### vcpkg

使用 vcpkg 包管理器下载并安装 {fmt}：

    git clone https://github.com/Microsoft/vcpkg.git
    cd vcpkg
    ./bootstrap-vcpkg.sh
    ./vcpkg integrate install
    ./vcpkg install fmt

<!-- vcpkg 中的 fmt package 由 Microsoft 团队成员和社区贡献者维护。
如果版本过旧，请在 [fmt vcpkg 仓库](https://github.com/Microsoft/vcpkg)
中创建 issue 或 pull request。 -->

### Conan

可以使用 [Conan](https://conan.io/) 包管理器下载并安装 {fmt}：

    conan install -r conancenter --requires="fmt/[*]" --build=missing

<!-- Conan Center 中的 {fmt} package 由 [ConanCenterIndex](https://github.com/conan-io/conan-center-index)
社区维护。如果版本过旧或 package 无法正常工作，请在 Conan Center Index
仓库中创建 issue 或 pull request。 -->

## 从 `printf` 迁移

[clang-tidy](https://clang.llvm.org/extra/clang-tidy/) v18 提供了
[modernize-use-std-print](https://clang.llvm.org/extra/clang-tidy/checks/modernize/use-std-print.html)
检查项，可以将 `printf` 和 `fprintf` 的调用转换为 `fmt::print`（如果进行了相应配置）。
默认情况下，它会转换为 `std::print`。

## 从源代码构建

CMake 的作用是生成可以在所选编译器环境中使用的原生 makefile 或项目文件。
典型流程如下：

    mkdir build  # 创建用于保存构建输出的目录。
    cd build
    cmake ..     # 生成原生构建脚本。

以上命令应在 `fmt` 仓库中运行。

如果你使用的是类 Unix 系统，现在应该可以在当前目录看到 Makefile。
然后运行 `make` 即可构建库。构建完成后，可以运行 `make test`
执行测试。

可以通过 `FMT_TEST` CMake 选项控制是否生成 make 的 `test` target。
当你将 fmt 作为子目录包含到项目中，但不希望将 fmt 的测试添加到
项目的 `test` target 时，这会非常有用。

要构建共享库，将 `BUILD_SHARED_LIBS` CMake 变量设置为 `TRUE`：

    cmake -DBUILD_SHARED_LIBS=TRUE ..

要构建带位置无关代码的静态库（例如将其链接到 Python 扩展等其他共享库中），
将 `CMAKE_POSITION_INDEPENDENT_CODE` CMake 变量设置为 `TRUE`：

    cmake -DCMAKE_POSITION_INDEPENDENT_CODE=TRUE ..

构建库后，在类 Unix 系统上可以运行 `sudo make install` 安装它。

### 构建文档

要构建文档，需要在系统中安装以下软件：

- [Python](https://www.python.org/)
- [Doxygen](http://www.stack.nl/~dimitri/doxygen/)
- [MkDocs](https://www.mkdocs.org/)，以及 `mkdocs-material`、`mkdocstrings`、
  `pymdown-extensions` 和 `mike`

首先按照上一节的说明使用 CMake 生成 makefile 或项目文件。
然后编译 `doc` target/project，例如：

    make doc

这会在 `doc/html` 中生成 HTML 文档。

## 构建系统

### build2

可以使用 [build2](https://build2.org)（一个依赖管理器和构建系统）
来使用 {fmt}。

目前可以从以下 package repositories 获取该 package：

- <https://cppget.org/fmt/>：用于已发布的版本。
- <https://github.com/build2-packaging/fmt>：用于尚未发布或自定义的版本。

**用法：**

- `build2` package 名称：`fmt`
- Library target 名称：`lib{fmt}`

要让你的 `build2` 项目依赖 `fmt`：

- 将其中一个 repository 添加到你的 configurations 中；如果尚未添加，
  也可以加入 `repositories.manifest`：

        :
        role: prerequisite
        location: https://pkg.cppget.org/1/stable

- 将该 package 作为依赖添加到 `manifest` 文件中（以版本 10 为例）：

        depends: fmt ~10.0.0

- 导入 target，并在对应的 `buildfile` 中使用 `fmt` 作为你自己的 target
  的 prerequisite：

        import fmt = fmt%lib{fmt}
        lib{mylib} : cxx{**} ... $fmt

然后像平常一样使用 `b` 或 `bdep update` 构建项目。

### Meson

[Meson WrapDB](https://mesonbuild.com/Wrapdb-projects.html) 包含 `fmt` package。

**用法：**

- 从项目根目录运行以下命令，通过 WrapDB 安装 `fmt` 子项目：

        meson wrap install fmt

- 在项目的 `meson.build` 文件中，为新的子项目添加一项：

        fmt = subproject('fmt')
        fmt_dep = fmt.get_variable('fmt_dep')

- 将新的依赖对象加入与 fmt 的链接：

        my_build_target = executable(
          'name', 'src/main.cc', dependencies: [fmt_dep])

**选项：**

如果需要，可以将 {fmt} 构建为静态库或仅头文件库。

对于静态构建，使用以下子项目定义：

    fmt = subproject('fmt', default_options: 'default_library=static')
    fmt_dep = fmt.get_variable('fmt_dep')

对于仅头文件版本，使用：

    fmt = subproject('fmt', default_options: ['header-only=true'])
    fmt_dep = fmt.get_variable('fmt_header_only_dep')

### Android NDK

{fmt} 提供了 [Android.mk 文件](
https://github.com/fmtlib/fmt/blob/master/support/Android.mk)，可以使用
[Android NDK](https://developer.android.com/tools/sdk/ndk/index.html) 构建库。

### 其他

要将 {fmt} 库用于任何其他构建系统，可以从 [release archive](
https://github.com/fmtlib/fmt/releases) 或 [git repository](
https://github.com/fmtlib/fmt) 中添加 `include/fmt/base.h`、
`include/fmt/format.h`、`include/fmt/format-inl.h`、`src/format.cc`
以及其他可选头文件到项目中，将 `include` 添加到 include directories，
并确保 `src/format.cc` 与你的代码一起编译和链接。
