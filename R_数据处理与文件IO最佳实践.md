以下为您精心提炼的 Markdown 技术文档。本文档严格遵循 **“去伪存真”** 的原则，剔除了所有曾经探讨过的错误认知与 AI 幻觉，仅保留经过验证的、最优雅的 R 语言（特别是 `tidyverse` 生态）实战代码与底层逻辑。

建议保存为 **`R_tidyverse_Advanced_CheatSheet.md`**。

---

# R 语言 tidyverse 核心实战与避坑指南

本文档总结了在 R 语言（特别是 `tidyverse` 生态）中进行数据初始化、整洁评估、文本处理及文件系统操作时的核心技巧与底层逻辑，旨在帮助开发者写出更健壮、更优雅的代码。

---

## 一、 数据框 (tibble) 的初始化与底层类型控制

### 1. 优雅创建指定行列的“空” tibble

在 R 中，“空”有两种截然不同的含义，必须严格区分以防爆错。

#### 场景 A：初始化为缺失值 (`NA`) —— 🌟 数据分析首选

使用专属的类型常量（如 `NA_character_`），确保列类型绝对稳定。

```R
library(tibble)
library(magrittr) # 提供 %>%

col_names <- c("姓名", "部门", "职位")
n_rows <- 5
n_cols <- length(col_names)

# ✅ 终极优雅写法：利用 list 的复制与 as_tibble 的自动解析
df_na <- rep(
  list(rep(NA_character_, n_rows)), # 内层 rep 控制行数
  n_cols                            # 外层 rep 控制列数
) %>%
  set_names(col_names) %>%
  as_tibble()
```

#### 场景 B：初始化为空字符串 (`""`)

如果业务逻辑需要字面意义上的空文本，使用 `character(n)`。

```R
# 注意：character(n) 生成的是 n 个 ""，而不是 NA！
df_empty <- rep(
  list(character(n_rows)),
  n_cols
) %>%
  set_names(col_names) %>%
  as_tibble()
```

### 2. 类型锚点：带下划线的 `NA` 家族

R 底层是强类型的，普通的 `NA` 默认是逻辑型 (`logical`)。在严格场景下，必须使用带下划线的专属 `NA` 来锚定类型。

| 目标类型             | 专属缺失值常量  | 典型应用场景                                           |
| :------------------- | :-------------- | :----------------------------------------------------- |
| **double / numeric** | **`NA_real_`**  | `dplyr::if_else()` 的数值分支、预分配数值向量。        |
| **character**        | `NA_character_` | 初始化文本列、`if_else()` 的文本分支。                 |
| **integer**          | `NA_integer_`   | 初始化整数列（注意：`1L` 是 integer，`1` 是 double）。 |
| **logical**          | `NA_logical_`   | 即普通的 `NA`，初始化逻辑列。                          |

**⚠️ 核心避坑：`dplyr::if_else()` 的严格类型检查**

```R
library(dplyr)
df <- tibble(score = c(85, 40))

# ❌ 报错：true 分支是 double，false 分支的普通 NA 是 logical，类型不匹配！
# df %>% mutate(result = if_else(score >= 60, score, NA))

# ✅ 正确：使用 NA_real_ 保持类型一致
df %>% mutate(result = if_else(score >= 60, score, NA_real_))
```

---

## 二、 数据选择与整洁评估 (Tidy Evaluation)

### `select()` 传入变量时的警告与 `all_of()` / `any_of()`

当你把包含列名的字符向量直接传给 `select()` 时，会触发警告。这是因为 `tidyselect` 引擎无法确定你是在引用“名为该变量名的列”，还是“该变量里包含的列名”。

**破局法则：必须使用 `all_of()` 或 `any_of()` 显式解包变量。**

| 函数            | 语义         | 行为表现                                                        | 最佳实践场景                                               |
| :-------------- | :----------- | :-------------------------------------------------------------- | :--------------------------------------------------------- |
| **`all_of(x)`** | **严格模式** | 向量 `x` 中的列名**必须全部存在**，否则直接报错 (Fail-Fast)。   | 数据管道中，确信某些核心列必须存在，缺失即代表数据源异常。 |
| **`any_of(x)`** | **宽容模式** | 向量 `x` 中**存在多少列就选多少列**，不存在的默默忽略，不报错。 | 动态列名、合并多个结构略有差异的表、处理脏数据。           |

```R
my_cols <- c("mpg", "cyl", "Not_Exist_Col")

# 报错：Can't subset columns that don't exist.
# mtcars %>% select(all_of(my_cols))

# 成功：只选出 mpg 和 cyl，忽略 Not_Exist_Col
mtcars %>% select(any_of(my_cols))
```

---

## 三、 文本与字符串处理

### `str_glue()` 中输出字面量花括号 `{}`

在拼接 JSON、正则表达式或 LaTeX 时，需要输出原生的 `{}`，而不是将其作为变量插值。

#### 方法 1：双写转义（适用于少量花括号）

用 `{{` 代表 `{`，用 `}}` 代表 `}`。

```R
library(stringr)
name <- "Alice"
# 输出: {Alice} 的分数是 95
str_glue("{{name}} 的分数是 {95}")
```

#### 方法 2：修改插值分隔符（适用于大量花括号，如写 JSON）

使用 `.open` 和 `.close` 参数，临时将插值符号改为 `[]`。

```R
color <- "red"
# 输出: body { background-color: red; }
str_glue(
  "body { background-color: [color]; }",
  .open = "[",
  .close = "]"
)
```

---

## 四、 文件 I/O 与路径管理 (`fs` 与 `readr`)

### 1. `readr::write_lines()`：纯粹的逐行写入

**核心定位**：专门用于将**一维的字符向量**逐行写入纯文本文件。它**不处理**数据框，没有表头，没有分隔符。

```R
library(readr)

my_texts <- c("第一行", NA, "第三行")

# 基础写入（NA 默认写为 "NA" 字符串）
write_lines(my_texts, "output.txt")

# 自定义 NA 的显示，并追加写入
write_lines(c("新增行"), "output.txt", na = "缺失", append = TRUE)

# ⚠️ 避坑：不能直接传数据框！必须先用 pull() 提取单列。
# mtcars %>% pull(mpg) %>% write_lines("mpg.txt")
```

### 2. 优雅去除文件后缀

#### 🌟 首选方案：`fs::path_ext_remove()`

完美保留路径，且对 Mac/Linux 的**隐藏文件**（如 `.Rprofile`）极其友好，会原样返回。

```R
library(fs)
fs::path_ext_remove("data/report.xlsx")      # -> "data/report"
fs::path_ext_remove(".Rprofile")             # -> ".Rprofile" (安全)
```

#### ⚠️ 备选方案：`tools::file_path_sans_ext()`

基础 R 内置，速度快，但**对隐藏文件处理有坑**。

```R
tools::file_path_sans_ext(".Rprofile")       # -> "" (变成空字符串，极易引发后续 Bug！)
```

### 3. `fs` 包：R 语言文件系统的“版本答案”

`fs` 彻底解决了基础 R 路径函数跨平台表现不一、返回值难以融入管道的问题。

#### 核心优势：

1. **跨平台统一**：内部永远使用 `/` 作为路径分隔符，自动适配 Win/Mac/Linux。
2. **专属对象**：返回 `fs_path` 对象，在 RStudio 中高亮显示，且支持智能截断打印。
3. **向量化与静默执行**：支持一次操作多个文件，且修改类函数默认不打印冗余信息。

#### 核心武器库速查：

```R
library(fs)

# --- 路径解剖 ---
p <- path("/data/raw", "sales.xlsx")
path_dir(p)                 # 获取目录: "/data/raw"
path_file(p)                # 获取文件名: "sales.xlsx"
path_ext(p)                 # 获取后缀: "xlsx"
path_ext_set(p, "csv")      # 修改后缀: "/data/raw/sales.csv"

# --- 文件探查 (降维打击 list.files) ---
# dir_info() 直接返回包含所有元数据的 tibble！完美融入 dplyr
dir_info(glob = "*.csv") %>%
  filter(size > 1024 * 1024) %>%       # 筛选大于 1MB 的文件
  arrange(desc(modification_time))     # 按修改时间倒序

# --- 文件操作 ---
dir_create("data/new_folder")          # 递归创建目录
file_create("data/new_folder/log.txt") # 创建空文件
file_copy("a.csv", "backup/")          # 复制文件
file_delete("log.txt")                 # 删除文件

# --- 高阶实战：结合 purrr 批量读取 ---
library(purrr)
library(readr)

dir_ls("data/", glob = "*.csv") %>%
  map_dfr(~ read_csv(.x, show_col_types = FALSE), .id = "source_file") %>%
  mutate(source_file = path_file(source_file)) # 清洗冗长的路径
```

---

## 五、 核心口诀与最佳实践总结

1. **tibble 初始化**：缺值用 `NA_xxx_`，空串用 `character()`；动态建表用 `list + rep + set_names + as_tibble`。
2. **dplyr 选列**：裸名直接写，变量必须包；全在 `all_of`，缺漏 `any_of` 挑。
3. **类型严格匹配**：遇到 `if_else` / `case_when` 报错类型不一致，立刻检查是否漏写了 `_real_` / `_character_` 等后缀。
4. **文本拼接**：`str_glue` 遇花括号，双写 `{{}}` 可破局；模板太乱改 `.open`。
5. **文件路径**：抛弃 `file.path` 和 `basename`，拥抱 `fs` 包；去后缀认准 `path_ext_remove`，小心隐藏文件变空串。
6. **纯文本写入**：`write_lines` 只认字符向量，数据框请先 `pull()`。
