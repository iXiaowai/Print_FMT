# API 参考

{fmt} 库 API 由以下组件组成：

- [`fmt/base.h`](#base-api)：基础 API，提供主要的 `char`/UTF-8 格式化函数，
  支持 C++20 编译期检查，并具有最少的依赖
- [`fmt/format.h`](#format-api)：`fmt::format` 以及其他格式化函数，同时提供 locale 支持
- [`fmt/ranges.h`](#ranges-api)：range 和 tuple 的格式化
- [`fmt/chrono.h`](#chrono-api)：日期和时间格式化
- [`fmt/std.h`](#std-api)：标准库类型的 formatter
- [`fmt/enum.h`](#enum-api)：带注解枚举的格式化
- [`fmt/compile.h`](#compile-api)：格式字符串编译
- [`fmt/color.h`](#color-api)：终端颜色和文本样式
- [`fmt/os.h`](#os-api)：系统 API
- [`fmt/ostream.h`](#ostream-api)：`std::ostream` 支持
- [`fmt/args.h`](#args-api)：动态参数列表
- [`fmt/printf.h`](#printf-api)：安全的 `printf`
- [`fmt/xchar.h`](#xchar-api)：可选的 `wchar_t` 支持

库提供的所有函数和类型都位于 `fmt` 命名空间中，宏均以 `FMT_` 为前缀。

## C++ Module API

使用 C++ module API 时，不需要包含上面列出的头文件，可以直接使用
`import fmt;`。下面列出的其他功能保持不变。

## Base API

`fmt/base.h` 定义了基础 API，提供主要的 `char`/UTF-8 格式化函数以及
C++20 编译期检查。它的包含依赖很少，可以改善编译时间。该头文件只有在
将 {fmt} 作为库使用（默认方式）而不是仅头文件模式时才特别有用。

它还为以下类型提供 `formatter` 特化：

- `int`、`long long`
- `unsigned`、`unsigned long long`
- `float`、`double`、`long double`
- `bool`
- `char`
- `const char*`、[`fmt::string_view`](#basic_string_view)
- `const void*`

以下函数使用与 Python 中 [str.format](https://docs.python.org/3/library/stdtypes.html#str.format)
类似的[格式字符串语法](syntax.md)，并接受 *fmt* 和 *args* 作为参数。

*fmt* 是一个格式字符串，其中包含字面文本以及由 `{}` 包围的替换字段。
这些字段会在结果字符串中被格式化后的参数替换。
[`fmt::format_string`](#format_string) 是一种格式字符串，可以从字符串字面量
或 `constexpr` 字符串隐式构造，并在 C++20 中进行编译期检查。
要传入运行时格式字符串，可以使用 [`fmt::runtime`](#runtime) 包装它。

*args* 是表示待格式化对象的参数列表。

除非另有说明，I/O 错误会以 [`std::system_error`](
https://en.cppreference.com/w/cpp/error/system_error) 异常报告。

::: print(format_string<T...>, T&&...)

::: print(FILE*, format_string<T...>, T&&...)

::: println(format_string<T...>, T&&...)

::: println(FILE*, format_string<T...>, T&&...)

::: format_to(OutputIt&&, format_string<T...>, T&&...)

::: format_to_n(OutputIt, size_t, format_string<T...>, T&&...)

::: format_to_n_result

::: formatted_size(format_string<T...>, T&&...)

<a id="udt"></a>
### 格式化用户自定义类型

{fmt} 为许多标准 C++ 类型提供 formatter。关于 range 和 tuple（包括
`std::vector` 等标准容器），请参见 [`fmt/ranges.h`](#ranges-api)；
关于日期和时间格式化，请参见 [`fmt/chrono.h`](#chrono-api)；
关于其他标准库类型，请参见 [`fmt/std.h`](#std-api)。

让用户自定义类型支持格式化有两种方式：提供 `format_as` 函数，
或者特化 `formatter` 结构体模板。

非 `void` 指针类型被有意禁止格式化，也不能通过这两个扩展 API
使其支持格式化。

如果希望让自己的类型使用另一个类型的格式说明符进行格式化，
可以使用 `format_as`。`format_as` 应接受一个自身类型的对象，
并返回一个支持格式化的类型对象。它应该定义在与自身类型相同的命名空间中。

当某个类型同时匹配另一个 `formatter` 特化（例如 range `formatter`）时，
不能使用 `format_as`，因为这些特化会产生歧义。如果可能，应禁用冲突的特化，
否则应提供显式的 `formatter` 特化。

示例（[运行](https://godbolt.org/z/nvME4arz)）：

    #include <fmt/format.h>

    namespace kevin_namespacy {

    enum class film {
      house_of_cards, american_beauty, se7en = 7
    };

    auto format_as(film f) { return fmt::underlying(f); }

    }

    int main() {
      fmt::print("{}\n", kevin_namespacy::film::se7en); // Output: 7
    }

使用 `formatter` 特化更加复杂，但可以完全控制解析和格式化。
使用这种方法时，应为自己的类型特化 `formatter` 结构体模板，并实现
`parse` 和 `format` 方法。

推荐通过继承或组合的方式复用已有 formatter 来定义 formatter。
这样可以在不自行实现的情况下支持标准格式说明符。例如：

```c++
// color.h:
#include <fmt/base.h>

enum class color {red, green, blue};
template <> struct fmt::formatter<color>: formatter<string_view> {
  // parse is inherited from formatter<string_view>.

  auto format(color c, format_context& ctx) const
    -> format_context::iterator;
};
```

```c++
// color.cc:
#include "color.h"
#include <fmt/format.h>
auto fmt::formatter<color>::format(color c, format_context& ctx) const
    -> format_context::iterator {
  string_view name = "unknown";
  switch (c) {
  case color::red:   name = "red"; break;
  case color::green: name = "green"; break;
  case color::blue:  name = "blue"; break;
  }
  return formatter<string_view>::format(name, ctx);
}
```

注意，`formatter<string_view>::format` 定义在 `fmt/format.h` 中，
因此必须在源文件中包含该头文件。由于 `parse` 继承自
`formatter<string_view>`，它会识别所有字符串格式说明符，例如：

```c++
fmt::format("{:>10}", color::blue)
```

会返回：

```text
"      blue"
```

一般来说，formatter 具有以下形式：

    template <> struct fmt::formatter<T> {
      // 解析格式说明符并将其保存到 formatter 中。
      //
      // [ctx.begin(), ctx.end()) 是一个可能为空的字符范围，
      // 包含从待解析格式说明开始的部分格式字符串，例如：
      //
      //   fmt::format("{:f} continued", ...);
      //
      // 此时范围将包含 "f} continued"。
      // formatter 应解析格式说明符直到 '}' 或范围结束。
      // 在此例中，formatter 应解析 'f' 说明符，并返回指向 '}' 的迭代器。
      constexpr auto parse(format_parse_context& ctx)
        -> format_parse_context::iterator;
      // 使用存储在 formatter 中的已解析格式说明来格式化 value，
      // 并将输出写入 ctx.out()。
      auto format(const T& value, format_context& ctx) const
        -> format_context::iterator;
    };

建议至少支持应用于整个对象的 fill、align 和 width，并使其语义与标准
formatter 保持一致。

也可以为类层次结构编写 formatter：

```c++
// demo.h:
#include <type_traits>
#include <fmt/format.h>
struct A {
  virtual ~A() {}
  virtual std::string name() const { return "A"; }
};

struct B : A {
  virtual std::string name() const { return "B"; }
};

template <typename T>
struct fmt::formatter<T, std::enable_if_t<std::is_base_of_v<A, T>, char>> :
    fmt::formatter<std::string> {
  auto format(const A& a, format_context& ctx) const {
    return formatter<std::string>::format(a.name(), ctx);
  }
};
```

```c++
// demo.cc:
#include "demo.h"
#include <fmt/format.h>
int main() {
  B b;
  A& a = b;
  fmt::print("{}", a); // Output: B
}
```

同时提供 `formatter` 特化和 `format_as` 重载是不允许的。

::: basic_format_parse_context

::: context

::: format_context

### 编译期检查

对于支持 C++20 `consteval` 的编译器，编译期格式字符串检查默认启用。
对于较旧的编译器，可以使用 `fmt/format.h` 中定义的
[FMT_STRING](#legacy-checks) 宏。

未使用的参数是允许的，与 Python 的 `str.format` 和普通函数相同。

参见[类型擦除](#type-erasure)，其中展示了如何在自己的函数中使用
`fmt::format_string` 启用编译期检查，同时避免模板代码膨胀。

::: fstring
::: format_string
::: runtime(string_view)

### 类型擦除

可以创建同时具有编译期检查和较小二进制体积的自定义格式化函数，例如
（[运行](https://godbolt.org/z/b9Pbasvzc)）：

```c++
#include <fmt/format.h>

void vlog(const char* file, int line,
          fmt::string_view fmt, fmt::format_args args) {
  fmt::print("{}: {}: {}", file, line, fmt::vformat(fmt, args));
}
template <typename... T>
void log(const char* file, int line,
         fmt::format_string<T...> fmt, T&&... args) {
  vlog(file, line, fmt, fmt::make_format_args(args...));
}
#define MY_LOG(fmt, ...) log(__FILE__, __LINE__, fmt, __VA_ARGS__)

MY_LOG("invalid squishiness: {}", 42);
```

注意，`vlog` 不对参数类型进行模板化。与完全模板化的版本相比，
这样可以改善编译时间并减小二进制代码体积。

::: make_format_args(T&...)
::: basic_format_args
::: format_args
::: basic_format_arg

### 命名参数

::: arg(const char*, const T&)

### 兼容性

::: basic_string_view
::: string_view

<a id="format-api"></a>
## Format API

`fmt/format.h` 定义了完整的格式化 API，并提供额外的格式化函数和 locale 支持。

<a id="format"></a>
::: format(format_string<T...>, T&&...)
::: vformat(string_view, format_args)
::: operator""_a()

### 工具函数

::: ptr(T)
::: underlying(Enum)
::: to_string(const T&)
::: group_digits(T)
::: detail::buffer
::: basic_memory_buffer

### 系统错误

{fmt} 不使用 `errno` 向用户传递错误，但它可能调用会设置 `errno` 的系统函数。
用户不应对库函数执行过程中 `errno` 的值是否保持不变做任何假设。

::: system_error
::: format_system_error

### 自定义分配器

{fmt} 支持自定义动态内存分配器。可以将自定义 allocator 类作为模板参数
传递给 [`fmt::basic_memory_buffer`](#basic_memory_buffer)：

    using custom_memory_buffer =
      fmt::basic_memory_buffer<char, fmt::inline_buffer_size, custom_allocator>;

也可以编写使用自定义 allocator 的格式化函数：

    using custom_string =
      std::basic_string<char, std::char_traits<char>, custom_allocator>;
    auto vformat(custom_allocator alloc, fmt::string_view fmt,
                 fmt::format_args args) -> custom_string {
      auto buf = custom_memory_buffer(alloc);
      fmt::vformat_to(std::back_inserter(buf), fmt, args);
      return custom_string(buf.data(), buf.size(), alloc);
    }
    template <typename ...Args>
    auto format(custom_allocator alloc, fmt::string_view fmt,
                const Args& ... args) -> custom_string {
      return vformat(alloc, fmt, fmt::make_format_args(args...));
    }

allocator 只会用于输出容器。对于内置类型和字符串类型，
格式化函数通常不会进行内存分配；但对于非默认浮点格式化，
偶尔可能会回退到 `sprintf`。

### Locale

默认情况下，所有格式化都与 locale 无关。使用 `'L'` 格式说明符，
可以从 locale 中插入适当的数字分隔符：

    #include <fmt/format.h>
    #include <locale>

    std::locale::global(std::locale("en_US.UTF-8"));
    auto s = fmt::format("{:L}", 1000000);  // s == "1,000,000"

`fmt/format.h` 提供了以下接受 `std::locale` 参数的格式化函数重载。
locale 类型使用模板参数，以避免代价较高的 `<locale>` 头文件包含。

::: format(locale_ref, format_string<T...>, T&&...)
::: format_to(OutputIt, locale_ref, format_string<T...>, T&&...)
::: formatted_size(locale_ref, format_string<T...>, T&&...)

<a id="legacy-checks"></a>
### 传统编译期检查

`FMT_STRING` 可以在较旧的编译器上启用编译期检查。
它要求 C++14 或更高版本，在 C++11 中则为空操作。

::: FMT_STRING

如果希望强制使用传统编译期检查，可以定义预处理变量
`FMT_ENFORCE_COMPILE_STRING`。设置后，接受 `FMT_STRING` 的函数
将无法使用普通字符串编译。

<a id="ranges-api"></a>
## Range 和 Tuple 格式化

`fmt/ranges.h` 提供 range 和 tuple 的格式化支持：

    #include <fmt/ranges.h>
    fmt::print("{}", std::tuple<char, int>{'a', 42});
    // Output: ('a', 42)

使用 `fmt::join` 可以使用自定义分隔符分隔 tuple 元素：

    #include <fmt/ranges.h>
    auto t = std::tuple<int, char>{1, 'a'};
    fmt::print("{}", fmt::join(t, ", "));
    // Output: 1, a

::: join(Range&&, string_view)
::: join(It, Sentinel, string_view)
::: join(std::initializer_list<T>, string_view)

<a id="chrono-api"></a>
## 日期和时间格式化

`fmt/chrono.h` 为以下类型提供 formatter：

- [`std::chrono::duration`](https://en.cppreference.com/w/cpp/chrono/duration)
- [`std::chrono::time_point`](https://en.cppreference.com/w/cpp/chrono/time_point)
- [`std::tm`](https://en.cppreference.com/w/cpp/chrono/c/tm)

格式语法参见 [Chrono 格式说明](syntax.md#chrono-format-spec)。

**示例：**

    #include <fmt/chrono.h>

    int main() {
      auto now = std::chrono::system_clock::now();

      fmt::print("The date is {:%Y-%m-%d}.\n", now);
      // Output: The date is 2020-11-07.
      // (with 2020-11-07 replaced by the current date)

      using namespace std::literals::chrono_literals;

      fmt::print("Default format: {} {}\n", 42s, 100ms);
      // Output: Default format: 42s 100ms

      fmt::print("strftime-like format: {:%H:%M:%S}\n", 3h + 15min + 30s);
      // Output: strftime-like format: 03:15:30
    }

::: gmtime(std::time_t)

<a id="std-api"></a>
## 标准库类型格式化

`fmt/std.h` 为以下类型提供 formatter：

- [`std::atomic`](https://en.cppreference.com/w/cpp/atomic/atomic)
- [`std::atomic_flag`](https://en.cppreference.com/w/cpp/atomic/atomic_flag)
- [`std::bitset`](https://en.cppreference.com/w/cpp/utility/bitset)
- [`std::error_code`](https://en.cppreference.com/w/cpp/error/error_code)
- [`std::exception`](https://en.cppreference.com/w/cpp/error/exception)
- [`std::filesystem::path`](https://en.cppreference.com/w/cpp/filesystem/path)
- [`std::monostate`](https://en.cppreference.com/w/cpp/utility/variant/monostate)
- [`std::optional`](https://en.cppreference.com/w/cpp/utility/optional)
- [`std::source_location`](https://en.cppreference.com/w/cpp/utility/source_location)
- [`std::thread::id`](https://en.cppreference.com/w/cpp/thread/thread/id)
- [`std::variant`](https://en.cppreference.com/w/cpp/utility/variant/variant)

::: ptr(const std::unique_ptr<T, Deleter>&)
::: ptr(const std::shared_ptr<T>&)

### Variant

只有当 `std::variant` 的所有候选类型都支持格式化时，
它才能被格式化，并且要求存在 `__cpp_lib_variant` [库特性](
https://en.cppreference.com/w/cpp/feature_test)。

**示例：**

    #include <fmt/std.h>

    fmt::print("{}", std::variant<char, float>('x'));
    // Output: variant('x')

    fmt::print("{}", std::variant<std::monostate, char>());
    // Output: variant(monostate)

## 位域和紧凑结构体

要格式化位域，或者格式化应用了 `__attribute__((packed))` 的结构体字段，
需要通过强制类型转换或一元 `+` 将其转换为底层类型或兼容类型
（[godbolt](https://www.godbolt.org/z/3qKKs6T5Y)）：

```c++
struct smol {
  int bit : 1;
};

auto s = smol();
fmt::print("{}", +s.bit);
```

这是 C++ “完美转发（perfect forwarding）”的一个已知限制。

<a id="enum-api"></a>
## Enum 格式化

`fmt/enum.h` 为使用 `fmt::as_identifiers` 注解的枚举提供格式化支持。
这种枚举会被格式化为与其值匹配的枚举成员标识符：

    #include <fmt/enum.h>

    enum class [[=fmt::as_identifiers]] color { red, green, blue };

    fmt::print("{}", color::green);
    // Output: green

这种枚举使用 [Format Specification](syntax.md#format-spec) 中描述的字符串
格式说明，例如：

    fmt::print("[{:>7}]", color::red);
    // Output: [    red]

标识符只提供 `char` 字符串形式，因此带注解的枚举不能使用其他字符类型进行格式化。

如果多个枚举成员具有相同的值，则使用声明顺序中最先出现的成员。
如果一个值与任何枚举成员都不匹配，则在应用字符串格式化之前，
先将其底层值以十进制表示：

    fmt::print("{}", static_cast<color>(42));
    // Output: 42

没有该注解的枚举不受影响，仍按照之前的方式进行格式化；也就是说，
有作用域枚举需要使用 `format_as` 或 `formatter` 特化，请参见
[格式化用户自定义类型](#udt)。

此功能使用两个 C++26 特性：
[reflection](https://en.cppreference.com/w/cpp/language/operator_reflection)
用于获取枚举成员标识符，[annotations](https://en.cppreference.com/w/cpp/language/annotations)
用于通过 `fmt::as_identifiers` 选择启用枚举。

因此需要支持 reflection 的编译器，在 GCC 中可能还需要额外的
`-freflection` 等选项。若 reflection 可用，宏 `FMT_USE_REFLECTION`
会被设置为 1，否则为 0。用户也可以自行定义该宏以禁用 reflection。

当 {fmt} 作为 module 构建时，reflection 支持会在 module 本身编译时检测，
因此只有当 module 构建时启用了 reflection，该 API 才可供导入者使用。

<a id="compile-api"></a>
## 编译期支持

`fmt/compile.h` 提供格式字符串编译和编译期（`constexpr`）格式化，
可以通过 `FMT_COMPILE` 宏或 `fmt::literals` 命名空间中定义的 `_cf`
用户定义字面量启用。

使用 `FMT_COMPILE` 或 `_cf` 标记的格式字符串会在编译期进行解析、
检查并转换为高效的格式化代码。

它支持内置类型和字符串类型的参数，也支持用户自定义类型，
前提是其 `formatter` 特化中包含以格式上下文类型为模板参数的
`format` 方法。例如（[运行](https://www.godbolt.org/z/3c13erEoq)）：

    struct point {
      double x;
      double y;
    };

    template <> struct fmt::formatter<point> {
      constexpr auto parse(format_parse_context& ctx) { return ctx.begin(); }

      template <typename FormatContext>
      auto format(const point& p, FormatContext& ctx) const {
        return format_to(ctx.out(), "({}, {})"_cf, p.x, p.y);
      }
    };

    using namespace fmt::literals;
    std::string s = fmt::format("{}"_cf, point(4, 2));

与默认 API 相比，格式字符串编译可能生成更多二进制代码，
因此只建议在格式化属于性能瓶颈的位置使用。

相同的 API 也支持在编译期进行格式化，例如在 `constexpr` 和 `consteval`
函数中。此外，还有实验性的 `FMT_STATIC_FORMAT`，可以在编译期将内容
格式化为精确所需大小的字符串。

编译期格式化支持具有 `constexpr` `format` 方法的内置和用户自定义 formatter。

示例：

    template <> struct fmt::formatter<point> {
      constexpr auto parse(format_parse_context& ctx) { return ctx.begin(); }
      template <typename FormatContext>
      constexpr auto format(const point& p, FormatContext& ctx) const {
        return format_to(ctx.out(), "({}, {})"_cf, p.x, p.y);
      }
    };

    constexpr auto s = FMT_STATIC_FORMAT("{}", point(4, 2));
    const char* cstr = s.c_str(); // Points the static string "(4, 2)".

::: operator""_cf
::: FMT_COMPILE
::: FMT_STATIC_FORMAT

<a id="color-api"></a>
## 终端颜色和文本样式

`fmt/color.h` 提供终端颜色和文本样式输出支持。

::: print(text_style, format_string<T...>, T&&...)
::: fg(detail::color_type)
::: bg(detail::color_type)
::: styled(const T&, text_style)

<a id="os-api"></a>
## 系统 API

::: ostream
::: output_file(cstring_view, T...)
::: windows_error

<a id="ostream-api"></a>
## `std::ostream` 支持

`fmt/ostream.h` 提供 `std::ostream` 支持，包括对具有重载插入运算符
（`operator<<`）的用户自定义类型进行格式化。

要让类型能够通过 `std::ostream` 进行格式化，应提供继承自
`ostream_formatter` 的 `formatter` 特化：

    #include <fmt/ostream.h>

    struct date {
      int year, month, day;
      friend std::ostream& operator<<(std::ostream& os, const date& d) {
        return os << d.year << '-' << d.month << '-' << d.day;
      }
    };

    template <> struct fmt::formatter<date> : ostream_formatter {};

    std::string s = fmt::format("The date is {}", date{2012, 12, 9});
    // s == "The date is 2012-12-9"

::: streamed(const T&)
::: print(std::ostream&, format_string<T...>, T&&...)

<a id="args-api"></a>
## 动态参数列表

头文件 `fmt/args.h` 提供 `dynamic_format_arg_store`，
它是一种类似构建器的 API，可用于动态构造格式参数列表。

::: dynamic_format_arg_store

<a id="printf-api"></a>
## 安全的 `printf`

头文件 `fmt/printf.h` 提供类似 `printf` 的格式化功能。
以下函数使用带有 POSIX 位置参数扩展的
[printf 格式字符串语法](
https://pubs.opengroup.org/onlinepubs/009695399/functions/fprintf.html)。

与标准对应函数不同，`fmt` 函数具有类型安全性；如果参数类型与其
格式说明不匹配，则会抛出异常。

::: printf(string_view, const T&...)
::: fprintf(std::FILE*, string_view, const T&...)
::: sprintf(string_view, const T&...)

<a id="xchar-api"></a>
## 宽字符串

可选头文件 `fmt/xchar.h` 提供对 `wchar_t` 和其他特殊字符类型的支持。

::: wstring_view
::: wformat_context
::: to_wstring(const T&)

## 与 C++20 `std::format` 的兼容性

{fmt} 实现了几乎全部的 [C++20 格式化库](
https://en.cppreference.com/w/cpp/utility/format)，但存在以下差异：

- 名称定义在 `fmt` 命名空间而不是 `std` 命名空间中，以避免与标准库实现冲突。
- 宽度计算不使用 grapheme clusterization（字素簇分组）。相关功能已经
  在独立分支中实现，但尚未合并。
- {fmt} 默认的浮点表示使用能够保证往返转换的最小精度，
  与 Java、Python 等语言类似。`std::format` 目前基于 `std::to_chars`
  进行规定；后者尝试生成最少数量的字符（忽略指数中的多余数字和符号），
  因此可能生成比必要数量更多的小数位。

## 配置选项

{fmt} 通过 CMake 选项和预处理宏提供配置功能，可以启用或禁用特性，
并针对二进制体积进行优化。例如，在配置 CMake 时可以通过
`-DFMT_OS=OFF` 禁用 `fmt/os.h` 中定义的操作系统相关 API。

### CMake 选项

- **`FMT_OS`**：设置为 `OFF` 时，禁用操作系统相关 API（`fmt/os.h`）。
- **`FMT_UNICODE`**：设置为 `OFF` 时，在 Windows/MSVC 上禁用 Unicode 支持。
  在其他平台上 Unicode 支持始终启用。

### 宏

- **`FMT_HEADER_ONLY`**：定义后启用仅头文件模式。
  它相当于使用 `fmt::fmt-header-only` CMake target。
  默认：未定义。
- **`FMT_USE_EXCEPTIONS`**：设置为 `0` 时禁用异常。
  默认值：`1`（如果使用 `-fno-exceptions` 编译，则为 `0`）。
- **`FMT_USE_LOCALE`**：设置为 `0` 时禁用 locale 支持。
  默认值：`1`（当 `FMT_OPTIMIZE_SIZE > 1` 时为 `0`）。
- **`FMT_CUSTOM_ASSERT_FAIL`**：设置为 `1` 时，允许用户提供自定义的
  `fmt::assert_fail` 函数。该函数会在断言失败时调用，并且在禁用异常时
  也会在运行时错误发生时调用。默认值：`0`。
- **`FMT_BUILTIN_TYPES`**：设置为 `0` 时，禁用除 `int` 之外的算术类型和
  字符串类型的内置处理。这样可以减小库体积，但会增加每次调用的开销。
  默认值：`1`。
- **`FMT_OPTIMIZE_SIZE`**：控制二进制体积优化：
    - `0` - 关闭（默认）
    - `1` - 禁用 locale 支持并应用部分优化
    - `2` - 禁用部分 Unicode 功能和命名参数，并应用更激进的优化

### 二进制体积优化

如果希望尽可能减小 {fmt} 的二进制体积，可以牺牲部分功能，
使用以下配置：

- CMake 选项：
    - `FMT_OS=OFF`
- 宏：
    - `FMT_BUILTIN_TYPES=0`
    - `FMT_OPTIMIZE_SIZE=2`
