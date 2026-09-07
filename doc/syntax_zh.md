# 格式字符串语法

本库中的格式化函数——最主要的是
[`fmt::format`](api.md#format) 和 [`fmt::print`](api.md#print)——接受
使用本文档所描述语法编写的格式字符串。

格式字符串是包含由花括号 `{` 和 `}` 分隔的*替换字段*的普通文本。
任何替换字段之外的字符都会原样复制到输出中。若要输出字面量花括号，
将其加倍：`{{` 输出一个 `{`，`}}` 输出一个 `}`。
替换字段的语法如下。

<a id="replacement-field"></a>
<pre><code class="language-json"
>replacement_field ::= "{" [arg_id] [":" (<a href="#format-spec"
  >format_spec</a> | <a href="#chrono-format-spec">chrono_format_spec</a>)] "}"
arg_id            ::= integer | identifier
integer           ::= digit+
digit             ::= "0"..."9"
identifier        ::= id_start id_continue*
id_start          ::= "a"..."z" | "A"..."Z" | "_"
id_continue       ::= id_start | digit</code>
</pre>

*arg_id* 用于选择要格式化的参数。它可以是非负整数（位置引用），
也可以是与通过 [`fmt::arg`](api.md#arg) 传入的参数名称匹配的标识符（命名引用）。
省略 *arg_id* 时，参数按照从左到右的顺序依次使用；这种*自动索引*
必须在整个格式字符串中统一使用——混合自动索引和显式数字 ID 会在编译期报错
（或者在运行时产生 `format_error`）。

以 `:` 开头的 *format_spec* 描述值的显示方式。其语法取决于类型；
标准内置类型所使用的形式将在下一节说明。

例如：

```c++
fmt::format("hello, {}", "world");
// Result: "hello, world"

fmt::format("{1}, {0}!", "world", "hello");
// Result: "hello, world!"

fmt::format("{greeting}, {name}!",
            fmt::arg("greeting", "hi"), fmt::arg("name", "fmt"));
// Result: "hi, fmt!"
```

*format_spec* 中的 *width* 或 *precision* 本身可以写成嵌套的替换字段
——`{}` 或 `{arg_id}`——此时它会在运行时从一个整数参数中取得值。
嵌套字段只能接受一个 *arg_id*，自身不能再包含 *format_spec*。

## 格式说明

下面的语法描述了内置类型共用的 *format_spec*，包括整数、浮点数、字符、
字符串、布尔值和指针，以及任何其 `formatter` 复用 fmt 解析器的用户自定义类型。

<a id="format-spec"></a>
<pre><code class="language-json"
>format_spec ::= [[fill]align][sign]["#"]["0"][width]["." precision]["L"][type]
fill        ::= &lt;a character other than '{' or '}'>
align       ::= "<" | ">" | "^"
sign        ::= "+" | "-" | " "
width       ::= <a href="#replacement-field">integer</a> | "{" [<a
  href="#replacement-field">arg_id</a>] "}"
precision   ::= <a href="#replacement-field">integer</a> | "{" [<a
  href="#replacement-field">arg_id</a>] "}"
type        ::= "a" | "A" | "b" | "B" | "c" | "d" | "e" | "E" | "f" | "F" |
                "g" | "G" | "o" | "p" | "s" | "x" | "X" | "?"</code>
</pre>

某个选项是否有意义取决于被格式化的值；对于不适用于该值类型的选项，
如果可能，会在编译期诊断，否则会产生 `format_error`。

### 填充与对齐

当 *width* 使字段宽度大于值的自然显示宽度时，*align* 字段用于选择填充的位置。

| 选项 | 作用 |
|--------|-------------------------------------------------------------------|
| `<` | 左对齐；在右侧填充。非数字类型的默认方式。 |
| `>` | 右对齐；在左侧填充。数字类型的默认方式。 |
| `^` | 居中对齐；如果填充无法平均分配，多出的填充字符放在右侧。 |

*fill* 字符可以是除 `{` 和 `}` 之外的任意单个 Unicode 码点，
并以与格式字符串相同的方式编码。它用于替代默认的空格作为填充字符。
只有当填充字符紧跟在 *align* 字符之后时，才会识别为自定义填充字符——
否则无法与其他位置的选项区分。因此，使用自定义填充字符时必须同时指定对齐方式。

当值的自然显示宽度已经大于或等于 *width* 时，对齐不会产生可观察的效果；
值绝不会为了适应宽度而被截断。

```c++
fmt::format("[{:<10}]", "42");   // Result: "[42        ]"
fmt::format("[{:>10}]", "42");   // Result: "[        42]"
fmt::format("[{:^10}]", "42");   // Result: "[    42    ]"
fmt::format("[{:*^10}]", "42");  // Result: "[****42****]"  - '*' as fill
```

### 符号

*sign* 字段控制数字值的符号如何输出。
它只适用于有符号整数和浮点类型。

| 选项 | 作用 |
|--------|-------------------------------------------------------------------|
| `+` | 始终输出符号（非负数为 `+`，负数为 `-`）。 |
| `-` | 仅对负数输出 `-`。这是默认方式。 |
| 空格 | 非负数前输出一个空格，负数输出 `-`；适用于对齐有符号数字的列。 |

浮点输出会保留 `-0.0` 的负号。

```c++
fmt::format("{:+d} {:+d}", 7, -7);  // Result: "+7 -7"
fmt::format("{: d} {: d}", 7, -7);  // Result: " 7 -7"
```

### 替代形式（`#`）

`#` 标志选择一种*替代形式*，其具体含义取决于表示类型：

- 对于以二进制、八进制或十六进制显示的整数，会添加相应的进制前缀
  (`0b`/`0B`、`0` 或 `0x`/`0X`)。前缀的大小写遵循类型说明符的大小写——
  `x` 使用 `0x`，`X` 使用 `0X`，依此类推。
- 对于浮点数，即使原本没有小数位，也会强制输出小数点，
  并阻止 `g`/`G` 表示类型删除有效数字末尾的零。

非数字类型不接受 `#` 标志。

### 零填充（`0`）

紧邻 *width* 之前的 `0` 会为数字类型启用感知符号的零填充。
零会插入符号（或进制前缀，如果存在）与最高位数字之间，
这样符号或 `0x` 前缀就会与数字保持相邻，而不会被空格分隔。
例如，对 `120` 使用 `{:+08d}` 会得到 `+0000120`。

零填充：

- 仅适用于数字类型；
- 对 `inf` 或 `nan` 无效；
- 如果同时显式指定了 *align*，则会被忽略。

### 宽度

*width* 是一个非负十进制整数，表示字段应占用的最小字符数。
如果格式化后的值短于 *width*，则按照 *align* 和 *fill* 进行填充；
如果更长，则完整输出。*width* 永远不会导致值被截断。

要在运行时提供 *width*，可以将字段写成 `{}` 来使用下一个参数，
或者写成 `{arg_id}` 来按位置或名称引用一个整数参数。

格式化字符串时，"width" 按显示列数计算，并使用 Unicode 感知的估算方式
（东亚宽字符、全角字符以及常见 Emoji 范围计为两列；其他字符计为一列）。
这样，在同时包含拉丁字符和 CJK 文本的等宽字体显示中，可以保持固定 *width* 的视觉一致性。

```c++
fmt::format("[{:6}]", 42);
// Result: "[    42]"  - right-aligned by default
fmt::format("[{:6}]", "hi");
// Result: "[hi    ]"  - left-aligned by default
fmt::format("[{:{}}]", 42, 6);
// Result: "[    42]"  - width from an argument
```

### 精度

*precision* 是由 `.` 引入的非负十进制整数，其含义取决于被格式化的值。
与 *width* 一样，它也可以通过嵌套替换字段在运行时提供。

| 类型 | `.precision` 的含义 |
|-------------------------------|-----------------------------------------------|
| `e`, `E`, `f`, `F` | 小数点后的输出位数。 |
| `g`, `G` | 有效数字的总位数。 |
| `a`, `A` | 十六进制有效数字的小数点后位数。如果省略，则输出足够多的位数以精确往返表示该值。 |
| 字符串（`s`、`?` 或默认） | 从值中复制的码点数量上限。 |

整数、字符、布尔值或指针类型不接受 *precision*。
当 *precision* 限制从 C 字符串中获取的字符数量时，该字符串仍必须以 null 结尾。

```c++
fmt::format("{:.2f}", 3.14159);        // Result: "3.14"
fmt::format("{:.3g}", 3.14159);        // Result: "3.14"
fmt::format("{:.4}", "hello, world");  // Result: "hell"
fmt::format("{:.{}f}", 3.14159, 4);
// Result: "3.1416"  - precision from an argument
```

### 本地化（`L`）

`L` 标志为数字类型选择与 locale 相关的格式化方式。
格式化器会检查传给格式化函数的 C++ locale（如果没有传入，则使用全局 locale），
并插入该 locale 的数字分组字符以及——对于浮点数——其小数点。
该标志对非数字类型没有影响。

```c++
auto loc = std::locale("en_US.UTF-8");
fmt::format(loc, "{:L}", 1234567890);     // Result: "1,234,567,890"
fmt::format(loc, "{:.2Lf}", 1234567.89);  // Result: "1,234,567.89"
```

### 表示类型

*type* 字段选择值的表示形式。下面按照适用的值类别对说明符进行分组。

**整数、布尔值和字符：**

| 类型 | 作用 |
|------|---------------------------------------------------------------------|
| `b` | 二进制。`#` 标志添加 `0b` 前缀。 |
| `B` | 二进制。`#` 标志添加 `0B` 前缀。 |
| `c` | 将整数按照对应码点的字符显示。不允许用于 `bool`。 |
| `d` | 十进制。整数类型的默认方式。 |
| `o` | 八进制。 |
| `x` | 十六进制，小写数字。`#` 标志添加 `0x` 前缀。 |
| `X` | 十六进制，大写数字。`#` 标志添加 `0X` 前缀。 |
| none | 对整数等同于 `d`，对字符等同于 `c`，对 `bool` 使用文本形式（`true`/`false`）。 |

```c++
fmt::format("{:d} {:#x} {:#o} {:#b}", 42, 42, 42, 42);
// Result: "42 0x2a 052 0b101010"

fmt::format("{:#06x}", 0xfe);  // # adds the prefix, 06 zero-pads to width 6
// Result: "0x00fe"
```

**浮点数：**

| 类型 | 作用 |
|------|---------------------------------------------------------------------|
| `a` | 十六进制有效数字形式（例如 `0x1.8p+1`）。使用小写数字和小写 `p` 表示二进制指数。始终输出 `0x` 前缀，与 printf 的 `%a` 一致。`#` 标志强制输出小数点（例如 `0x1.p+1`）。 |
| `A` | 与 `a` 相同，但全部使用大写形式。 |
| `e` | 科学计数法，十进制指数使用小写 `e`。 |
| `E` | 科学计数法，十进制指数使用大写 `E`。 |
| `f` | 定点表示法。 |
| `F` | 与 `f` 相同，但将 `nan` 输出为 `NAN`，将 `inf` 输出为 `INF`。 |
| `g` | 通用形式：当指数小于 −4 或不小于精度时使用科学计数法，否则使用定点表示法；除非设置 `#`，否则会删除小数部分末尾的零。精度为 `0` 时按 `1` 处理。 |
| `G` | 与 `g` 相同，但指数使用 `E`，并使用大写的 `INF`/`NAN`。 |
| none | 最短的可往返表示：格式化后的值重新解析为相同浮点类型时，能够逐位还原输入值。 |

**字符串和字符：**

| 类型 | 作用 |
|------|---------------------------------------------------------------------|
| `s` | 普通字符串输出。字符串类型的默认方式；对 `bool` 则输出 `true` 或 `false`。 |
| `c` | 字符输出。字符类型的默认方式。不允许用于 `bool`。 |
| `?` | 调试输出：字符使用单引号包裹，字符串使用双引号包裹；不可打印字符、非 ASCII 字符以及特殊字符使用 C 风格转义序列，例如 `\n`、`\t`、`\"` 和 `\u{...}`。 |
| none | 对字符串和 `bool` 等同于 `s`，对字符等同于 `c`。 |

```c++
fmt::format("{}",   "tab\there");  // Result contains a literal tab character.
fmt::format("{:?}", "tab\there");  // Result: "\"tab\\there\""
```

**指针：**

| 类型 | 作用 |
|------|---------------------------------------------------------------------|
| `p` | 添加 `0x` 前缀的十六进制地址。指针类型的默认方式。 |
| none | 与 `p` 相同。 |

C 字符串（`char*` 或 `const char*`）同时接受字符串表示类型和 `p`，
因此同一个值既可以格式化为文本，也可以格式化为地址。

## Chrono 格式说明

chrono duration、time point 类型以及 `std::tm` 的格式说明语法如下：

<a id="chrono-format-spec"></a>
<pre><code class="language-json"
>chrono_format_spec ::= [[<a href="#format-spec">fill</a>]<a href="#format-spec"
  >align</a>][<a href="#format-spec">width</a>]["." <a href="#format-spec"
  >precision</a>][chrono_specs]
chrono_specs       ::= conversion_spec |
                       chrono_specs (conversion_spec | literal_char)
conversion_spec    ::= "%" [padding_modifier] [locale_modifier] chrono_type
literal_char       ::= &lt;a character other than '{', '}' or '%'>
padding_modifier   ::= "-" | "_"
locale_modifier    ::= "E" | "O"
chrono_type        ::= "a" | "A" | "b" | "B" | "c" | "C" | "d" | "D" | "e" |
                       "F" | "g" | "G" | "h" | "H" | "I" | "j" | "m" | "M" |
                       "n" | "p" | "q" | "Q" | "r" | "R" | "S" | "t" | "T" |
                       "u" | "U" | "V" | "w" | "W" | "x" | "X" | "y" | "Y" |
                       "z" | "Z" | "%"</code>
</pre>

字面量字符会原样复制到输出。精度仅适用于表示类型为浮点数的
`std::chrono::duration` 类型。

可用的表示类型（*chrono_type*）如下：

<table>
<tr>
  <th>类型</th>
  <th>含义</th>
</tr>
<tr><td><code>'a'</code></td><td>星期几的缩写，例如 "Sat"。如果值不包含有效的星期几，则抛出 <code>format_error</code>。</td></tr>
<tr><td><code>'A'</code></td><td>星期几的完整名称，例如 "Saturday"。如果值不包含有效的星期几，则抛出 <code>format_error</code>。</td></tr>
<tr><td><code>'b'</code></td><td>月份名称的缩写，例如 "Nov"。如果值不包含有效月份，则抛出 <code>format_error</code>。</td></tr>
<tr><td><code>'B'</code></td><td>月份的完整名称，例如 "November"。如果值不包含有效月份，则抛出 <code>format_error</code>。</td></tr>
<tr><td><code>'c'</code></td><td>日期和时间表示，例如 "Sat Nov 12 22:04:00 1955"。修改后的命令 <code>%Ec</code> 产生 locale 的替代日期和时间表示。</td></tr>
<tr><td><code>'C'</code></td><td>使用向下取整除法得到的年份除以 100，例如 "19"。如果结果只有一位十进制数字，则前面补 0。修改后的命令 <code>%EC</code> 产生 locale 的世纪替代表示。</td></tr>
<tr><td><code>'d'</code></td><td>月份中的日期，以十进制数表示。如果结果只有一位十进制数字，则前面补 0。修改后的命令 <code>%Od</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'D'</code></td><td>等同于 <code>%m/%d/%y</code>，例如 "11/12/55"。</td></tr>
<tr><td><code>'e'</code></td><td>月份中的日期，以十进制数表示。如果结果只有一位十进制数字，则前面补空格。修改后的命令 <code>%Oe</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'F'</code></td><td>等同于 <code>%Y-%m-%d</code>，例如 "1955-11-12"。</td></tr>
<tr><td><code>'g'</code></td><td>ISO 周历年份的最后两位十进制数字。如果结果只有一位，则前面补 0。</td></tr>
<tr><td><code>'G'</code></td><td>ISO 周历年份，以十进制数表示。如果少于四位，则左侧补 0 至四位。</td></tr>
<tr><td><code>'h'</code></td><td>等同于 <code>%b</code>，例如 "Nov"。</td></tr>
<tr><td><code>'H'</code></td><td>小时（24 小时制），以十进制数表示。如果只有一位，则前面补 0。修改后的命令 <code>%OH</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'I'</code></td><td>小时（12 小时制），以十进制数表示。如果只有一位，则前面补 0。修改后的命令 <code>%OI</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'j'</code></td><td>如果被格式化类型是 duration 的特化类型，则表示不填充的天数十进制值；否则表示一年中的第几天。1 月 1 日为 001。如果少于三位，则左侧补 0 至三位。</td></tr>
<tr><td><code>'m'</code></td><td>月份，以十进制数表示。1 月为 01。如果只有一位，则前面补 0。修改后的命令 <code>%Om</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'M'</code></td><td>分钟，以十进制数表示。如果只有一位，则前面补 0。修改后的命令 <code>%OM</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'n'</code></td><td>换行字符。</td></tr>
<tr><td><code>'p'</code></td><td>与 12 小时制相关的 AM/PM 标识。</td></tr>
<tr><td><code>'q'</code></td><td>duration 的单位后缀。</td></tr>
<tr><td><code>'Q'</code></td><td>duration 的数值（如通过 <code>.count()</code> 提取的值）。</td></tr>
<tr><td><code>'r'</code></td><td>12 小时制时间，例如 "10:04:00 PM"。</td></tr>
<tr><td><code>'R'</code></td><td>等同于 <code>%H:%M</code>，例如 "22:04"。</td></tr>
<tr><td><code>'S'</code></td><td>秒，以十进制数表示。如果秒数小于 10，则前面补 0。如果输入精度无法用整数秒精确表示，则使用定点格式的十进制浮点数，并使精度与输入精度一致（如果无法在 18 位小数内转换为浮点十进制秒，则使用微秒精度）。修改后的命令 <code>%OS</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'t'</code></td><td>水平制表符字符。</td></tr>
<tr><td><code>'T'</code></td><td>等同于 <code>%H:%M:%S</code>。</td></tr>
<tr><td><code>'u'</code></td><td>ISO 星期几，以十进制数表示（1-7），星期一为 1。修改后的命令 <code>%Ou</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'U'</code></td><td>年份中的周数，以十进制数表示。一年中的第一个星期日是第 01 周；在此之前同一年中的日期属于第 00 周。如果结果只有一位，则前面补 0。修改后的命令 <code>%OU</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'V'</code></td><td>ISO 周历中的周数，以十进制数表示。如果结果只有一位，则前面补 0。修改后的命令 <code>%OV</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'w'</code></td><td>星期几，以十进制数表示（0-6），星期日为 0。修改后的命令 <code>%Ow</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'W'</code></td><td>年份中的周数，以十进制数表示。一年中的第一个星期一是第 01 周；在此之前同一年中的日期属于第 00 周。如果结果只有一位，则前面补 0。修改后的命令 <code>%OW</code> 产生 locale 的替代表示。</td></tr>
<tr><td><code>'x'</code></td><td>日期表示，例如 "11/12/55"。修改后的命令 <code>%Ex</code> 产生 locale 的替代日期表示。</td></tr>
<tr><td><code>'X'</code></td><td>时间表示，例如 "10:04:00"。修改后的命令 <code>%EX</code> 产生 locale 的替代时间表示。</td></tr>
<tr><td><code>'y'</code></td><td>年份最后两位十进制数字。如果只有一位，则前面补 0。修改后的命令 <code>%Oy</code> 产生 locale 的替代表示。修改后的命令 <code>%Ey</code> 产生相对于 <code>%EC</code>（仅年份）的 locale 替代偏移表示。</td></tr>
<tr><td><code>'Y'</code></td><td>年份，以十进制数表示。如果少于四位，则左侧补 0 至四位。修改后的命令 <code>%EY</code> 产生 locale 的完整年份替代表示。</td></tr>
<tr><td><code>'z'</code></td><td>ISO 8601:2004 格式的 UTC 偏移。例如 -0430 表示比 UTC 晚 4 小时 30 分钟。如果偏移为零，则使用 +0000。修改后的命令 <code>%Ez</code> 和 <code>%Oz</code> 在小时和分钟之间插入 <code>:</code>，例如 -04:30。如果偏移信息不可用，则抛出 <code>format_error</code>。</td></tr>
<tr><td><code>'Z'</code></td><td>时区缩写。如果时区缩写不可用，则抛出 <code>format_error</code>。</td></tr>
<tr><td><code>'%'</code></td><td>一个 % 字符。</td></tr>
</table>

具有日历成分的说明符，例如 `'d'`（月份中的日期），只对 `std::tm`
和时间点有效，对 duration 无效。

数值结果默认使用 0 填充。可用的填充修饰符（*padding_modifier*）如下：

| 类型 | 含义 |
|-------|-----------------------------------------|
| `'_'` | 使用空格填充数值结果。 |
| `'-'` | 不对数值结果字符串进行填充。 |

这些修饰符仅支持 `'H'`、`'I'`、`'M'`、`'S'`、`'U'`、`'V'`、`'W'`、
`'Y'`、`'d'`、`'j'` 和 `'m'` 表示类型。

示例：

```c++
#include <fmt/chrono.h>

auto t = std::tm();
t.tm_year = 2010 - 1900;
t.tm_mon = 7;
t.tm_mday = 4;
t.tm_hour = 12;
t.tm_min = 15;
t.tm_sec = 58;
fmt::print("{:%Y-%m-%d %H:%M:%S}", t);
// Prints: 2010-08-04 12:15:58
```

## Range 格式说明

Range 类型的格式说明语法如下：

<pre><code class="language-json"
>range_format_spec ::= ["n"][range_type][":" range_underlying_spec]</code>
</pre>

`'n'` 选项会在格式化 range 时省略开头和结尾的方括号。

`range_type` 可用的表示类型如下：

| 类型 | 含义 |
|--------|------------------------------------------------------------|
| none | 默认格式。 |
| `'s'` | 字符串格式。将 range 格式化为字符串。 |
| `'?s'` | 调试格式。将 range 格式化为转义字符串。 |

如果 `range_type` 是 `'s'` 或 `'?s'`，range 元素类型必须是字符类型。
`'n'` 选项和 `range_underlying_spec` 与 `'s'` 和 `'?s'` 互斥。

`range_underlying_spec` 根据 range 元素类型对应的 formatter 进行解析。

默认情况下，字符或字符串组成的 range 会以转义并加引号的形式输出。
但是，如果提供了任何 `range_underlying_spec`（即使它为空），
字符或字符串就会按照提供的说明进行输出。

示例：

```c++
fmt::print("{}", std::vector{10, 20, 30});
// Output: [10, 20, 30]
fmt::print("{::#x}", std::vector{10, 20, 30});
// Output: [0xa, 0x14, 0x1e]
fmt::print("{}", std::vector{'h', 'e', 'l', 'l', 'o'});
// Output: ['h', 'e', 'l', 'l', 'o']
fmt::print("{:n}", std::vector{'h', 'e', 'l', 'l', 'o'});
// Output: 'h', 'e', 'l', 'l', 'o'
fmt::print("{:s}", std::vector{'h', 'e', 'l', 'l', 'o'});
// Output: "hello"
fmt::print("{:?s}", std::vector{'h', 'e', 'l', 'l', 'o', '\n'});
// Output: "hello\n"
fmt::print("{::}", std::vector{'h', 'e', 'l', 'l', 'o'});
// Output: [h, e, l, l, o]
fmt::print("{::d}", std::vector{'h', 'e', 'l', 'l', 'o'});
// Output: [104, 101, 108, 108, 111]
fmt::print("{:n:f}", std::array{std::numbers::pi, std::numbers::e});
// Output: 3.141593, 2.718282
```

## 综合示例

下面的示例将前面介绍的多个元素结合起来——嵌套替换字段、
填充字符和居中对齐——为一条消息绘制一个固定宽度的边框：

```c++
fmt::print(
    "┌{0:─^{2}}┐\n"
    "│{1: ^{2}}│\n"
    "└{0:─^{2}}┘\n", "", "Hello, world!", 20);
```

输出：

```
┌────────────────────┐
│   Hello, world!    │
└────────────────────┘
```
