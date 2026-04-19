>💡**该文档正持续更新中···**
# 📖ASCH Docs
* > 本文档负责归纳描述一种用于动画作品条目、製作委員会、职能列表与人员 credit 的结构化文本。

* > 本文基于现有解析器实现整理，重点定义可解析语法、字段语义、转义规则与注入规则。

 * > 如需引用，建议以“.ASCH格式规范”称呼本文档所定义的文本格式。

----

## 1. 核心对象模型

一个 ASCH 文本最终通常会解析为多个作品条目。每个作品条目包含以下逻辑区域：

- `metadata`：作品元数据
- `committee` 或 `seisaku_company`：委员会 / 制作相关公司列表
- `CreditEntry`：人员 credit 列表

字段分流由 `production_mode` 决定：

| production_mode | 公司列表字段 | 原作归属字段 |
|---|---|---|
| `製作委員會` | `committee` | `original_oncommittee` |
| `製作/共同製作` | `seisaku_company` | `original_onseisaku` |
| `solo` | `seisaku_company` | `original_onseisaku` |
| `Netflix Mode` | `seisaku_company` | `original_onseisaku` |



## 2. metadata 规范

### 2.1 常见字段

| 字段 | 含义 |
|---|---|
| `year` | 年份（字符串，例如 `"2026"`） |
| `season` | 季节/月份（字符串，例如 `"04"`） |
| `release_date` | 发行日期（电影/特殊情况） |
| `title` | 作品标题 |
| `type` | 作品类型。默认为 `TV` |
| `original_type` | 原作类型列表（数组），例如 `["漫画"]`。默认 `["漫画"]` |
| `production_mode` | 制作模式。枚举值：`solo` / `製作委員會` / `製作/共同製作` / `Netflix Mode`。默认 `製作委員會` |
| `unit_duration` | 单集时长，例如 `"24min"` |
| `episodes_count` | 集数 |
| `total_duration` | 总时长，例如 `"04:48:00"` |
| `committee_name` | 委员会名称。仅在 `production_mode = 製作委員會` 时出现 |
| `production`     | 动画制作公司（读取 Credit 区最后一行自由文本） |
| `produced_by` | Produced by 列表（数组）。由 `Produced by：` role 段触发时产生 |
| `original_sources` | 原作来源信息（数组），每项含 `original_company`（出版社）、可选 `original_label`（发行 label） |
| `original_oncommittee` | 原作方是否在 `committee` 内，仅 `製作委員會` 模式且为 `true` 时输出 |
| `original_onseisaku` | 原作方是否在 `seisaku_company` 内，仅非委員會 模式且为 `true` 时输出 |
| `in_association_with` | 协作方（由 `//In association with X` 注释行触发，值为 `X`） |

---

### 2.2 committee_name 与字段分流

当 `production_mode = "製作委員會"` 时：

- 使用 `committee`
- 可输出 `committee_name`（取自以 `*` 开头的委员会标题行，如 `*淡島百景製作委員会`）
- 可输出 `original_oncommittee`

当 `production_mode != "製作委員會"`（`solo` / `製作/共同製作` / `Netflix Mode`）时：

- 使用 `seisaku_company`
- 不输出 `committee_name`
- 可输出 `original_onseisaku`

`original_oncommittee` / `original_onseisaku` 仅在为 `true` 时输出，不输出 `false`。

---

## 2.3 committee / seisaku_company 规范

### 2.4 製作委員會 模式

当 `production_mode = "製作委員會"` 时，`*` 开头的委员会标题行会填入 `committee_name`，其后的组织行解析为 `committee` 成员。

示例（ASCH 输入）：

```text
*淡島百景製作委員会
KADOKAWA
MADHOUSE
富士电视台
樂天
BS富士
```

对应输出：

```json
"metadata": { "committee_name": "淡島百景製作委員会", ... },
"committee": [
  { "company": "KADOKAWA", "episodes": "all" },
  { "company": "MADHOUSE", "episodes": "all" },
  ...
]
```

---

### 2.5 非 製作委員會 模式（solo / 製作/共同製作 / Netflix Mode）

示例：

```text
东宝
```

此时：

- `committee_name` 不输出；
- 公司行输出到 `seisaku_company`。

---

### 2.6 committee / seisaku_company 成员通用字段

| 字段 | 含义 |
|---|---|
| `company` | 公司/组织名 |
| `episodes` | 集数范围（默认 `"all"`） |
| `from_role` | 布尔标记，表示该成员由 role 段（如 `Produced by：`）间接派生，而非显式的组织行 |

---

## 3. CreditEntry 规范

### 3.1 基本形式

credit 行为当前 role 下的成员行，通常是纯文本（不需要 `*` 前缀），角色由紧邻的 `<role>` 行确定：

```text
<制片人>
柳澤俊介（东宝）
井上雄仁（东宝）
大田圭二（东宝|动画企划部）
高島祐一郎（講談社）「12-14话」->古川慎（講談社?）「14-24话」
```

> `*` 前缀在 ASCH 中保留给**特殊节点**：委员会标题行（如 `*淡島百景製作委員会`）、或 Credit 区末尾的动画制作公司 / 无公司信息的 person 行。普通 credit 行不使用 `*`。

### 3.2 通用字段

| 字段 | 含义 |
|---|---|
| `role` | 当前角色（由最近一个 `<role>` 行决定） |
| `person` | 人名 |
| `person_realname` | 人员本名（例如罗马字本名） |
| `company` | 公司 |
| `parent_company` | 母公司（当解析器可以关联到时输出） |
| `department` | 部门，数组（例如 `["ULTRA JUMP编辑部", "第4编辑部企画室"]`） |
| `episodes` | 参与集数。默认为 `"all"` |
| `person_uncertain` | 人名不确定（来自 `?`） |
| `company_uncertain` | 公司不确定（来自 `？` / `?`） |
| `former_company` | 跳槽来源（前东家，来自 `<-`） |
| `former_company_uncertain` | 跳槽来源不确定（来自 `?<-`） |
| `affiliations` | 附属机构/马甲公司/二次派遣列表 |
| `unverified` | 未查证。仅有 `person` 但无 `company` 时输出 `true` |
| `tips` | 注释性补充（由 `//` 注释行在当前 role 下产生，内容为注释原文） |

> 注：早期规范曾提及 `succession`（`->` 后继链）与 `person_id`（`#` 消歧义），当前解析器输出中暂未出现这些字段；若需使用请以 04-new.json 实际输出为准。


##  📚开发者备忘录

>一些要踩的坑与以及已经踩过的坑。

###  语义约定与脚本聚合逻辑说明

####  production_mode 与输出字段

- `production_mode = "製作委員會"` → 输出 `committee`
- 其他值 → 输出 `seisaku_company`

#### committee_name

- 仅在 `production_mode = "製作委員會"` 时输出；
- 非 委員會 模式不输出该字段。

####  原作归属字段

- 委員會 模式：`original_oncommittee`
- 非 委員會 模式：`original_onseisaku`
- 仅在为 `true` 时输出。

####  省略空字段

建议不要输出：

- 空字符串
- 空数组
- 空对象
- 默认 false 的布尔字段
####  在编写脚本处理JSON季度数据时，你的get_uid函数应该遵循以下优先级：
*  主名优先(Identity-First)：如果person_id以id:开头，弱化该person署名，以该ID作为统计主键。
*  重名隔离(Discriminator-Second)：如果person_id以org:开头（或纯数字），则将person+person_id组合成一个唯一主键。
*  默认处理:如果没有ID，则以person署名为准。
#### 括号优先级：
* 姓名(A) →company:A
* 姓名(A)(B)→company:A,affiliations:[{company: A}, {company: B}] (A为Primary)

### 实现建议

>如果你要重构ASCH解释器，遵循以下优先级:

##### 函数优先级：

1. 先做词法清洗，再做语法切分。
2. 所有顶层切分都应尊重括号与转义。
3. 纯注入行必须作为元数据处理。
4. `production_mode` 应优先决定字段布局。
5. 在最终输出阶段统一清理内部状态字段。
#####    解析顺序:


1. Unicode 归一化；
2. 处理转义；
3. 验证括号/引号平衡；
4. 提取注入；
5. 判断是否为标题行；
6. 判断是否为日期行；
7. 判断是否为类型行；
8. 判断是否为 role 行；
9. 判断是否为 credit 行；
10. 按当前 `production_mode` 将组织行写入 `committee`（委員會 模式）或 `seisaku_company`（其他模式）。



## 📄EBNF（Extended Backus–Naur Form）

以下EBNF描述ASCH的核心语法。

>为了贴近真实文本习惯，部分自由文本采用宽松定义。

### 文件级

```ebnf
file                = { line } ;

line                = blank-line
                    | date-line
                    | work-header
                    | type-line
                    | duration-line
                    | role-line
                    | credit-line
                    | committee-line
                    | injection-line
                    | comment-line
                    | text-line ;

comment-line        = "//", text-until-eol ;
(* "//In association with X" 行会向当前作品的 metadata 写入 in_association_with=X，
   并在当前 role 下生成一条 { role, episodes, tips } 的 CreditEntry *)

blank-line          = { space } ;
```

---

### 作品与元数据

```ebnf
date-line           = year, date-tail, [ date-span ] ;

year                = digit, digit, digit, digit ;
date-tail           = { digit } ;
date-span           = ( "～" | "~" ), year, date-tail ;

work-header         = "##", space*, title ;
title               = text-until-eol ;

type-line           = type-body, [ duration-tail ] ;

type-body           = { type-token | space | "+" } ;
type-token          = type-code
                    | source-type-code
                    | production-mode-code
                    | netflix-flag ;

type-code           = "1" | "2" | "3" | "4" | "5" | "6" ;

source-type-code    = "m" | "n" | "g" | "o"
                    | "M" | "N" | "G" | "O" ;

production-mode-code = "f" | "s" | "t" | "b" ;
netflix-flag        = "i" ;

duration-tail       = integer, ":", integer ;

duration-line       = integer, ":", integer ;
integer             = digit, { digit } ;
```

---

### 角色与credit

```ebnf
role-line           = role-angle | role-colon ;

role-angle          = "<", space*, role-name, space*, ">" ;
role-colon          = role-name, ":" ;
role-name           = text-until-eol ;

credit-line         = [ "*", space* ], credit-body ;
credit-body         = credit-fragment, { arrow, credit-fragment } ;

(* "*" 前缀仅用于特殊节点（委员会标题行、或 Credit 区末尾的 studio / 无公司 person 行） *)
(* 普通 credit 行不使用 "*" 前缀 *)
arrow               = "->" | "<-" ;

credit-fragment     = [ episode-mark ], [ person-part ], { member-tail } ;

person-part         = plain-person
                    | plain-person, [ person-id ]
                    | plain-person, [ organization-block ] ;

plain-person        = text-until-delimiter ;
person-id           = "#", id-text ;
id-text             = text-until-delimiter ;

member-tail         = organization-block
                    | injection
                    | uncertain-mark ;

uncertain-mark      = "?" ;
```

---

###  committee/seisaku_company

```ebnf
committee-line      = committee-item ;

committee-item      = committee-title
                    | committee-company-line
                    | committee-label-line ;

committee-title     = title-like-text ;
committee-company-line = company-name, [ rights-block ], [ episode-mark ] ;
committee-label-line = label-only-text ;

rights-block        = "{", rights-content, "}" ;
rights-content      = rights-item, { ",", rights-item } ;
rights-item         = rights-name, ":", rights-area ;

rights-name         = text-until-delimiter ;
rights-area         = text-until-delimiter ;

episode-mark        = "「", episode-text, "」" ;
episode-text        = { ? any character except "」" ? } ;

organization-block  = "(", organization-content, ")" ;
organization-content = { ? any character except ")" ? } ;
```

---

### 注入

```ebnf
injection-line      = "$", injection-pair, { "$", injection-pair } ;

injection-pair      = injection-key, ":", injection-value ;

injection-key       = key-text ;

injection-value     = scalar
                    | array
                    | object
                    | quoted-string
                    | plain-text ;

scalar              = boolean | null-value | integer | float ;

boolean             = "true" | "false" ;
null-value          = "null" | "none" ;
float               = [ sign ], digit, { digit }, ".", digit, { digit } ;
sign                = "+" | "-" ;

array               = "[", [ array-items ], "]" ;
array-items         = array-item, { ",", array-item } ;
array-item          = object
                    | array
                    | scalar
                    | quoted-string
                    | plain-text ;

object              = "{", [ object-members ], "}" ;
object-members      = object-member, { ",", object-member } ;
object-member       = object-key, ":", object-value ;

object-key          = key-text ;
object-value        = injection-value ;
```

---

### 字符串与转义

```ebnf
quoted-string       = double-quoted-string | single-quoted-string ;

double-quoted-string = '"', { escaped-char | dq-char }, '"' ;
single-quoted-string = "'", { escaped-char | sq-char }, "'" ;

escaped-char        = "\", escape-target ;
escape-target       = "\" | "$" | ":" | "," | "<" | ">" | "["
                    | "]" | "(" | ")" | "{"
                    | "}" | "|" | "#"
                    | "?" | "-" | '"' | "'" ;

dq-char             = ? any character except '"' and newline ? ;
sq-char             = ? any character except "'" and newline ? ;

plain-text          = { plain-char } ;
plain-char          = ? any character except top-level delimiters and newline ? ;

text-until-eol      = { ? any character except newline ? } ;
text-until-delimiter = { ? any character except top-level delimiter characters ? } ;
key-text            = { ? any character except ":" and top-level delimiter characters ? } ;
```
----
