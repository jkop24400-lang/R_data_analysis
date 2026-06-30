````markdown name=readr_parse_number_and_numeric_conversion_guide.md
# readr::parse_number() 与数值转换完全指南

## 目录

1. [核心概念](#核心概念)
2. [parse_number() 详解](#parse_number-详解)
3. [parse_number() vs as.numeric()](#parse_number-vs-asnumeric)
4. [科学计数法处理](#科学计数法处理)
5. [实战：脏数据处理函数](#实战脏数据处理函数)
6. [常见问题与陷阱](#常见问题与陷阱)
7. [tidyeval 在数值转换中的应用](#tidyeval-在数值转换中的应用)

---

## 核心概念

### 什么是脏数据

实际数据中的数值列常常混杂：

- 中文符号与单位：`"1.2km"`、`"大于1.2"`、`"2,345.67人"`
- 科学计数法与单位：`"1e-3mm"`、`"1.23e4kg"`
- 千分位与小数点歧义：`"2,345.67"`（美式）vs `"2.345,67"`（欧式）
- 空白与特殊字符：`"  1.2  "`、`"\u00A0123\u00A0"`（不换行空格）
- 区间表示：`"12~13"`、`"12–13"`
- NA 与空字符串：`"NA"`（字符串）vs `NA`（真正的 NA）vs `""`（空字符串）

### 转换策略选择

| 工具                  | 主要特点                        | 适用场景              | 速度 |
| --------------------- | ------------------------------- | --------------------- | ---- |
| as.numeric()          | 严格解析，非数值字符直接返回 NA | 纯净数据或已预处理    | 快   |
| readr::parse_number() | 宽松提取，忽略单位与格式        | 混杂单位/格式的脏数据 | 快   |
| readr::parse_double() | 严格 + 支持 locale 与千分位     | 带格式的规范数值      | 快   |
| 正则 + as.numeric()   | 精细控制                        | 特殊格式或混合场景    | 中等 |

---

## parse_number() 详解

### 基础用法

```r
library(readr)

x <- c("1.2", "1.2mm", "1e-3", "1e-3mm", "$12.34", "12,345.67")
parse_number(x)
# [1]     1.2    1.2    0.001   0.001   12.34 12345.67
```
````

### 关键特性

#### 1. 自动忽略首尾空白

```r
parse_number("  1.5  ")           # 1.5
parse_number("\t\n123\n")         # 123
```

因此**不需要提前用 trimws() 处理**。

#### 2. 正确识别科学计数法（包括单位）

```r
parse_number("1e-3")              # 0.001
parse_number("1e-3mm")            # 0.001（忽略单位）
parse_number("1.23e4kg")          # 12300
parse_number("2.5E+2")            # 250
```

#### 3. 提取第一个数值片段

```r
parse_number("abc123def")         # 123
parse_number("12~13")             # 12（只取第一个，区间需人工处理）
parse_number("v1.2.3")            # 1.2（只到第一个完整数值）
```

#### 4. 支持 locale 参数（千分位与小数点规范）

```r
# 默认 locale（美式：. 为小数点，, 为千分位）
parse_number("2,345.67")          # 2345.67
parse_number("2.345,67")          # 2.345（不认出欧式格式）

# 自定义 locale（欧式：, 为小数点，. 为千分位）
parse_number("2.345,67",
  locale = locale(decimal_mark = ",", grouping_mark = "."))
# 2345.67

# 中文场景：逗号可能是中文标点
parse_number("2，345.67",         # 忽略中文逗号
  locale = locale(grouping_mark = ""))
# 2（只取 2，因为中文逗号被忽略）
```

#### 5. 对非数值返回 NA

```r
parse_number(NA)                  # NA
parse_number("")                  # NA
parse_number("abc")               # NA
parse_number("NA")                # NA（字符串 "NA" 也返回 NA）
```

---

## parse_number() vs as.numeric()

### 行为对比表

| 输入         | parse_number() | as.numeric() | 说明                            |
| ------------ | -------------- | ------------ | ------------------------------- |
| `"1.2"`      | 1.2            | 1.2          | 都正常                          |
| `"1.2mm"`    | 1.2            | NA           | parse_number 忽略单位           |
| `"1e-3"`     | 0.001          | 0.001        | 都支持科学计数                  |
| `"1e-3mm"`   | 0.001          | NA           | parse_number 忽略单位           |
| `"2,345.67"` | 2345.67        | NA           | parse_number 智能处理千分位     |
| `"2.345,67"` | 2.345          | NA           | 都无法识别欧式格式（需 locale） |
| `"  1.5  "`  | 1.5            | 1.5          | 都能处理首尾空白                |
| `"12~13"`    | 12             | NA           | parse_number 取第一个数         |
| `NA`         | NA             | NA           | 都返回 NA                       |
| `""`         | NA             | NA           | 都返回 NA                       |
| `"abc"`      | NA             | NA           | 都返回 NA                       |

### 实战选择

- **用 parse_number()**：数据来自用户输入、爬虫、或包含单位的报告。
- **用 as.numeric()**：数据已清洁或你想严格检测"原始字符串是否为纯数值"。
- **两者结合**：先用 parse_number() 提取数值，再用 as.numeric() 的 NA 结果作为"脏数据"标记。

---

## 科学计数法处理

### parse_number() 的强大之处

parse_number() **完全支持**科学计数法，即使带单位：

```r
test_cases <- c(
  "1e-3",           # 0.001
  "1e-3mm",         # 0.001（忽略 mm）
  "1.23e4",         # 12300
  "1.23e4 kg",      # 12300（忽略空格与单位）
  "2.5E+2",         # 250
  "-3.14e-1",       # -0.314
  "5e10 people"     # 5e10（即 50000000000）
)

parse_number(test_cases)
# [1] 0.001 0.001 12300.000 12300.000 250.000 -0.314 50000000000.000
```

### 为什么 parse_number() 能胜任

readr::parse_number() 的内部实现：

1. 跳过首尾非数值字符。
2. 识别可选的符号（+/-）。
3. 识别整数、小数与指数部分（包括 e/E 及其符号）。
4. 停止在第一个"不属于数值语法"的字符（如单位字母）。
5. 将提取的字符串用 as.numeric() 解析。

因此对 `"1e-3mm"` 的处理过程：

- 提取数值文本：`"1e-3"`（因为 m 不属于数值语法）
- 转为数值：as.numeric("1e-3") = 0.001

---

## 实战：脏数据处理函数

### 小而美的设计哲学

对多列处理，**不必追求一次性转换所有列**。链式调用符合 tidyverse 管道风格，更易维护：

```r
df %>% to_numeric("x") %>% to_numeric("y") %>% to_numeric("z")
```

### 完整实现

```r
library(dplyr)
library(rlang)
library(readr)
library(stringr)

#' 对单列进行数值转换并诊断
#'
#' @param dt tibble，数据框
#' @param col_var 字符串，列名
#'
#' @return 新增两列：
#'   - {col_var}_parsed: readr::parse_number() 的结果
#'   - {col_var}@strict_is_na: as.numeric() 是否返回 NA（严格诊断）
#'
#' @examples
#' df <- tibble(
#'   x = c("1.2", "1.2mm", "1e-3", "NA", NA, ""),
#'   y = c("大于1.2", "2e3", "112.213 m", "NA", NA, "")
#' )
#' df %>% to_numeric("x") %>% to_numeric("y")
to_numeric <- function(dt, col_var) {
  if (!tibble::is_tibble(dt)) stop("the data must be a tibble format!")
  if (!col_var %in% colnames(dt)) stop(sprintf("column '%s' doesn't exist!", col_var))

  dt %>%
    mutate(
      !!sym(str_c(col_var, "parsed", sep = "_")) :=
        readr::parse_number(.data[[col_var]]),
      !!sym(str_c(col_var, "strict_is_na", sep = "@")) :=
        is.na(suppressWarnings(as.numeric(.data[[col_var]])))
    )
}
```

### 使用示例

```r
library(tibble)

df <- tibble(
  nb = 1:7,
  x = c(" 1.2", "2.8", "1e-3", "2e3", "NA", NA, ""),
  y = c(" 大于1.2", "不小于2.", "1e-3mm", "2e3", "112.213 m", NA, "NA")
)

# 单列转换
df %>% to_numeric("x")
# 输出新增列：x_parsed, x@strict_is_na

# 多列链式转换
result <- df %>% to_numeric("x") %>% to_numeric("y")

# 查看诊断结果
result %>% select(x, x_parsed, `x@strict_is_na`, y, y_parsed, `y@strict_is_na`)
```

### 理解诊断列的含义

| 情形            | parse_number() | strict_is_na | 说明                                                  |
| --------------- | -------------- | ------------ | ----------------------------------------------------- |
| `"1.2"`         | 1.2            | FALSE        | 纯净数���，直接使用 parsed 列                         |
| `"1.2mm"`       | 1.2            | TRUE         | parse_number 抽出了数，但原始字符包含单位，需人工确认 |
| `"大于1.2"`     | 1.2            | TRUE         | 包含中文修饰，parse_number 仍能抽数，但需确认语义     |
| `"NA"` (字符)   | NA             | TRUE         | 字符串 "NA"，非真正缺失值，应补全上下文处理           |
| `NA` (真正缺失) | NA             | NA           | 真正的缺失值，TRUE                               |
| `""` (空字符串) | NA             | TRUE         | 空值，看业务是否应作为 0 或保留 NA                    |

### 筛查可疑数据

```r
result %>%
  filter(`x@strict_is_na` == TRUE | `y@strict_is_na` == TRUE) %>%
  select(nb, x, x_parsed, `x@strict_is_na`, y, y_parsed, `y@strict_is_na`)
```

这样能快速定位哪些行需要人工复核与数据清洁。

---

## 常见问题与陷阱

### 问题 1：为什么 parse_number("2.345,67") 返回 2.345 而不是 2345.67？

**原因**：parse_number() 的默认 locale 是美式，即 `.` 为小数点、`,` 为千分位。对 "2.345,67" 它会：

1. 识别 2.345 为完整数值（小数点后跟三位数）。
2. 遇到逗号（在美式 locale 中不属于数值）而停止。
3. 返回 2.345。

**解决**：显式指定 locale：

```r
parse_number("2.345,67",
  locale = locale(decimal_mark = ",", grouping_mark = "."))
# 2345.67
```

### 问题 2：parse_number("12~13") 返回 12，这是设计缺陷吗？

**不是**。parse_number() 的目标是"从混杂文本中快速抽出第一个数值"。对于区间/范围，这是正确行为——你应该在上游检测并标记区间，再决定如何处理（取首/尾/均值/保留原值）。

**处理方式**：

```r
detect_range <- function(x) grepl("\\d+\\s*[~\\-–—]\\s*\\d+", x)
df <- df %>% mutate(is_range = detect_range(col))
# 再根据 is_range 标记来单独处理
```

### 问题 3：为什么我的代码出现"missing value where TRUE/FALSE needed"？

**原因**：通常是在对向量的逻辑操作中混入了 NA。例如：

```r
pos <- gregexpr(",", NA)[[1]]  # 返回 -1（未找到）
if (length(pos) == 1 && pos[1] == -1) ...  # 若 pos 是 NA，pos[1] 也是 NA
                                            # if(...) 期望 TRUE/FALSE，得到 NA 则报错
```

**修正**：先检测 NA：

```r
if (length(pos) == 1 && (is.na(pos) || pos == -1)) return(-1L)
```

### 问题 4：处理多列时应该用 across() 还是链式调用？

**建议**：

- **小数据或常规流程**：用链式 `df %>% to_numeric("x") %>% to_numeric("y")`（简洁、易读）。
- **大量列或重复模式**：考虑 across() 或向量化版本（但增加复杂度）。

参考本指南"tidyeval 应用"章节了解 across() 的细节。

---

## tidyeval 在数值转换中的应用

### !! 与 !!sym() 的区别

在 dplyr mutate 中：

```r
col_var <- "x"

# 方式 1：!!sym(col_var) - 显式转为 symbol
df %>% mutate(!!sym(col_var) := col_var * 2)  # 新增列名为 x，值为 x*2

# 方式 2：!!col_var - 直接解引用字符串
df %>% mutate(!!col_var := col_var * 2)       # 在 := 的左侧等价

# 两者效果相同（:= 会自动把字符串当列名）
# 但 !!sym() 更显式，可读性更强
```

**在命名新列时**：

```r
parsed_name <- "x_parsed"

# 正确：使用 !!sym() 或 !! 将字符串转为列名
df %>% mutate(!!sym(parsed_name) := parse_number(x))

# 等价于
df %>% mutate(!!parsed_name := parse_number(x))  # := 会自动处理字符串
```

### ensym、enquo 与 sym 的选择

| 函数     | 输入       | 输出    | 使用场景                                                   |
| -------- | ---------- | ------- | ---------------------------------------------------------- |
| sym(x)   | 字符串     | symbol  | 你已有列名字符串，需转为名字                               |
| ensym(x) | 未求值参数 | symbol  | 函数参数接收列名，支持非标准评估（如 `f(x)` 而非 `f("x")`) |
| enquo(x) | 未求值参数 | quosure | 需保留参数的表达式与环境（更高阶用法）                     |

示例：

```r
# 用户想写 to_numeric(df, x) 而不是 to_numeric(df, "x")
to_numeric_nse <- function(dt, col) {
  col_sym <- ensym(col)  # 捕获 x，得到 symbol x
  col_chr <- as_string(col_sym)  # 转回字符串 "x"
  # ... 后续逻辑
}

to_numeric_nse(df, x)  # 不需要引号
```

### across() 简介

当需要对多列批量应用相同转换时：

```r
# 方法 1：for 循环 + 表达式列表（你当前的做法）
exprs <- list()
for (col in c("x", "y")) {
  exprs[[paste0(col, "_parsed")]] <- expr(parse_number(.data[[!!col]]))
}
df %>% mutate(!!!exprs)

# 方法 2：across() - 更简洁（dplyr 推荐）
df %>% mutate(
  across(
    c(x, y),
    list(parsed = ~ parse_number(.)),
    .names = "{.col}_{.fn}"
  )
)
```

across() 的优势：

- 代码更短、更易读。
- 支持 tidyselect helper：`starts_with()`、`contains()`、`where(is.character)` 等。
- .names 参数自动生成列名（{.col} 代表原列名，{.fn} 代表函数名）。

---

## 进阶技巧

### 处理多种数值格式

```r
# 结合多个策略
smart_parse <- function(x) {
  # 1. 先尝试 parse_number（宽松）
  parsed <- parse_number(x)

  # 2. 标记原始是否为"纯数值"（严格）
  strict <- suppressWarnings(as.numeric(x))

  # 3. 检测特殊格式（如区间）
  is_range <- grepl("\\d+\\s*[~\\-]\\s*\\d+", x)

  list(parsed = parsed, strict = strict, is_range = is_range)
}
```

### 批量处理与诊断报告

```r
# 生成转换诊断报告
diagnostic_report <- function(dt, col_var) {
  res <- dt %>% to_numeric(col_var)

  parsed_col <- paste0(col_var, "_parsed")
  strict_col <- paste0(col_var, "@strict_is_na")

  tibble(
    column = col_var,
    total_rows = nrow(dt),
    non_na_original = sum(!is.na(dt[[col_var]])),
    successfully_parsed = sum(!is.na(res[[parsed_col]])),
    suspect_rows = sum(res[[strict_col]] == TRUE & !is.na(res[[parsed_col]])),
    completely_failed = sum(is.na(res[[parsed_col]]))
  )
}

# 使用
df %>% to_numeric("x") %>% diagnostic_report("x")
```

---

## 总结与最佳实践

### 五条黄金法则

1. **用 parse_number() 处理混杂单位的脏数据**，它比 as.numeric() 更宽松。
2. **同时用 as.numeric() 的 NA 结果作诊断**，标记"原始不是纯数值"的行。
3. **对locale敏感的数据（千分位/小数点）明确指定 locale**。
4. **对区间/范围/含义模糊的值，标记后交人工处理**，不要盲目自动化。
5. **优先选择小、单一职责的函数，用管道链接**，避免过度抽象。

### 函数设计建议

```r
# 推荐：小而美的单列版本
to_numeric <- function(dt, col_var) { ... }

# 使用：链式调用
df %>% to_numeric("x") %>% to_numeric("y") %>% to_numeric("z")

# 而非：一次性多列版本（除非你真的经常这样用）
to_numeric_multi <- function(dt, cols) { ... }
```

### 何时需要更复杂的处理

- 大量格式变体（中文数字、多种单位、罗马数字等）→ 专业数据清洁库。
- 精密科学数据（需保留有效数字、处理不确定性）→ 专门的科学计算包。
- 时间序列或地理坐标 → 使用专门的 readr::parse_date()、sf 包等。

---

## 参考资源

- readr 官方文档：https://readr.tidyverse.org/
- dplyr across() 文档：https://dplyr.tidyverse.org/reference/across.html
- tidyselect 选择器：https://tidyselect.r-lib.org/
- rlang quasiquotation：https://rlang.r-lib.org/reference/quasiquotation.html

---

**最后的话**：数据清洁没有银弹。好的做法是写清晰、可维护的小函数，配合诊断列标记异常值，然后人工审视与修复。自动化应该处理 80% 的"标准脏值"，剩下 20% 留给专业判断。

```

这个文档涵盖了我们对话中的所有要点，并有适当延伸：
- 从概念、工具对比、实现细节
- 到常见陷阱、tidyeval 机制、进阶技巧
- 再到最佳实践与何时求助

可以直接用于学习、参考或分享给团队。
```
