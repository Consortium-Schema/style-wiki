>💡**该文档正持续更新中···**
# Consortium-Schema  ASCH 录入格式规范V1.07
>处理复杂系统数据的正确姿势永远是保持输入端是Clean Text，通过逻辑层生成Rich Data。基于此，人力扁平文本的录入做法是必要的，避免了在录入时写复杂的嵌套JSON，实现录入成本最小化与数据价值最大化。
## 目录
---
# 什么是ASCH？

ASCH是我们设计的一套面向动画作品资料录入的轻量DSL，这套格式的原则是**录入尽量简洁，解析尽量稳定，默认值尽量友好**。

---

## 1. 设计目标

ASCH旨在用**接近人类可读的纯文本**描述动画作品资料，并支持：

- 作品级元数据
- 制作委员会（committee）
- 角色分组（role）
- 人员 credit
- 允许局部注入和嵌套数据（`$key:value`）
- 嵌套结构（数组 / 对象）
- 跳槽 / 继承关系（`->`）
- 前任来源关系（`<-`）
- 基本转义

该格式的设计目标是：

1. 便于手工编辑；
2. 便于脚本稳定解析；
3. 尽量避免依赖严格 JSON；
4. 支持局部覆盖与嵌套补丁。

---

## 2. 文件总体结构

一个ASCH文件由若干非空行组成。忽略空行。常见顺序如下：

```TEXT
日期行
作品标题行
类型行                                                $注入行
製作委員会/其他製作模式标题行
公司成员行
credit行
```
作品以标题行开始：
```TEXT
## 黒猫と魔女の教室
```
遇到下一条`标题行`时，前一作品结束。

---

## 3. 词法约定

### 3.1 空白

- 行首行尾空白会被忽略。
- 空行会被忽略。
- 多个连续空格通常不会影响语义。

### 3.2 标准化

解析器通常会做以下归一化：

- Unicode NFKC 归一化；
- 全角标点转换为半角；
- 一些字符会进行统一映射，例如：
  - `（` → `(`
  - `）` → `)`
  - `＜` → `<`
  - `＞` → `>`
  - `＄` → `$`
  - `＼` → `\`
  - ···

### 3.3 符号映射表

**普通 credit 行直接写在 `<role>` 段下，不加前缀；`*` 前缀保留给特殊节点（见 §4.x）。通过下列符号映射可表达复杂的权属关系。**
| 符号 |语义  | 录入示例 | 对应输出字段 |
| :--- | :--- | :--- | :--- |
| ？ / ? | 不确定性 | 夏目公一朗（KADOKAWA？） | `company_uncertain` / `person_uncertain` |
| `<-`   | 跳槽来源 | 安倍孝二（GOOD Smile China）<-bilibili | `former_company` |
| `?<-`  | 跳槽来源不确定 | 岡﨑剛之（CBC電視台）?<-bilibili | `former_company_uncertain` |
| 「」   | 集数赋予 | 沢辺伸政（小学馆）「1-13话」 | `episodes` |
| `->`   | 后继链 | 沢辺伸政（小学馆）「1-13话」->備前島幹人（小学馆）「14话－24话」 |`succession` |
| `\|`   | 隶属分隔（部门） | 大西恒平（集英社\|周刊少年JUMP编辑部） | `department` |
| `@`    | 母公司归属 | 木下直哉（HERO'S）@木下Group；小野朗（SPEEDSTAR RECORDS@JVC建伍胜利娱乐） | `parent_company` |
| `／` / `/` | 人员本名分隔 | ワンミシェル／Michelle Wang（KADOKAWA） | `person_realname` |
| `#`    | 消歧义 | 鈴木健太#org:Aniplex（Aniplex）；アリエル・リー#id:Ariel Li（Crunchyroll） |`person_id` |


**`@` 的两种位置**：
- 放在括号外：`person（company）@parent_company`
- 放在括号内：`person（company@parent_company）`

两者语义等价，均生成 `company` + `parent_company`。

**`／` / `/` 分隔的方向**：左侧为在 credit 中出现的署名（`person`），右侧为本名 / 罗马字原名（`person_realname`）。例如 `赤崎薫／赤﨑薫（TBS）` 会输出 `person="赤崎薫"`、`person_realname="赤﨑薫"`。

---

## 4. 行类型定义

### 4.1 日期行

日期行用于给当前作品设置时间基准。

示例：

```text
202604
20260303
202604～202612
```

语义上通常表示：

- `year`：年份
- `season`：季节/月份
- 或 `release_date`：电影/特别条目日期。

>检测到`DD`时由Season转变，他会改变type值为剧场版（如果你没有在类型行显式定义）

具体映射由实现决定，但一般遵循：

- `YYYYMM` → `year=YYYY`, `season=MM`
- `YYYYMMDD` → 作为更精细日期
- `YYYYMM～YYYYMM` → 区间信息

---

### 4.2 作品标题行

标题行以 `##` 开头：

```text
## 夜櫻家大作戰 第二季	
```

其语义是：

- 新作品开始；
- 标题为 `##` 后内容；
- 新作品的 `metadata.title` 来源于此行。

---

### 4.3 类型行

类型行是一个紧凑的定义串，用于一次性表达：

- 作品类型；
- 原作类型；
- production mode；
- Netflix 关联标记；
- 可选时长。

示例：

```text
ni
4mngt24:12
b
t24:12
```

### 4.3.1 基本字符含义

- `1` → TV
- `2` → 剧场版
- `3` → OVA
- `4` → ONA
- `5` → Special
- `6` → Short

原作类型：

- `m` → 漫画
- `n` → 小说
- `g` → 游戏
- `o` → 原创动画

production mode：

- `f` → `Netflix Mode`
- `s` → `solo`
- `t` → `製作/共同製作`
- `b` → `製作委員會`

 ***"什么？！网飞的大手？！"***

- `i` → `association_netflix = true`

---

### 4.3.2 production_mode 约束

`production_mode` 只能取以下四者之一：

- `Netflix Mode`
- `solo`
- `製作/共同製作`
- `製作委員會`

对应输入字符只能单独出现一个，不允许组合，例如：

- `b` ✅
- `t` ✅
- `f` ✅
- `s` ✅
- `bt` ❌
- `fs` ❌

默认值为：

```json
"production_mode": "製作委員會"
```

---

### 4.3.3 association_netflix

若类型行中包含 `i`，则（规范预期）：

```json
"association_netflix": true
```

若未出现 `i`，则该字段省略，不建议输出为 `false`。

>多次迭代后解析器已实现动态字段嗅探功能，该功能仅作为特殊情况保留
---

### 4.3.4 时长附加

类型行可以直接附加时长：

```text
4mngt24:12
```

其中 `24:12` 表示：

- 单集 24 分钟
- 共 12 集

---


###  继承与前任来源

#### `->`

表示后继、继任或继承链：

```text
A -> B -> C
```

通常解析为：

- 主条目：`A`
- `succession`: `[B, C]`

#### `<-`

表示前任来源：

```text
岡﨑剛之 ( CBC電視台) ?<-bilibili
```

通常解析为：

- `person = 岡﨑剛之`
- `former_company = bilibili`
- `former_company_uncertain = true`

---

##  注入系统

> **现状提示**：注入的字段处理权重仅次于[]{},请谨慎使用。

ASCH 的注入系统使用 `$` 分隔，允许在任意节点追加结构化字段。

###  基本语法

```text
base$key:value
```

示例：

```text
$command:test
```

若一行以 `$` 开头，则它是**纯注入行**，不应当被当作普通文本。

---

###  递归注入

注入值可以是：

- 字符串
- 数字
- 布尔
- 数组
- 对象

示例：

```text
$command:[recommand:"这是嵌套注入",emoji:"👿“]
```

---

###  注入可出现的位置

注入可以出现在：

- 作品级 metadata 行
- committee 行
- credit 行
- role块内部段

---


##  值类型规则

###  字符串

```text
name:"abc"
name:'abc'
```

###  数字

```text
count:12
ratio:3.14
```

###  布尔

```text
flag:true
flag:false
```

###  空值

```text
value:null
value:none
```

###  数组

```text
functions:[商品, 原作, 出版社]
```

###  对象

```text
command:{foo:1, bar:"x"}
```

### 复合语法的注入与动态字段嗅探实例：
>🚥仅作为示范，请勿过度解读
```asch
185810
## 某动画
$title:靠注入改名字，可以但不犯法。
*欢乐斗幕府
《企画》
冲田总司（\?新选组）【职能1,职能2,职能3】「1-13话」//for_role动态字段的functions若为职能123代表注入失败。 $functions:[我，在，哪，？] 
《制片人》
吉田松阴（传马町牢屋敷\@）@1,2,3
<宣传·制作/授权、海外宣传>
Shown Kochi（夏田書店）@冬田書店,春田書店
// In association with Metplix 「11-12话」$add:海外流媒体合作方 
damn south (Metplix @MPLX) //\$已注入role $role:role
*尊国攘夷
//Produced by WniQlexDmyamic,Blanning Inc.  「12」
```
JSON侧输出：

````JSON
[
  {
    "metadata": {
      "year": "1858",
      "season": "10",
      "title": "靠注入改名字,可以但不犯法。",
      "production_mode": "製作委員会",
      "in_association_with": [
        {
          "value": "Metplix",
          "episodes": "11-12话"
        }
      ],
      "production": "尊国攘夷",
      "produced_by": [
        {
          "value": "WniQlexDmyamic",
          "episodes": "12"
        },
        {
          "value": "Blanning Inc.",
          "episodes": "12"
        }
      ],
      "committee_name": "欢乐斗幕府"
    },
    "committee": [
      {
        "from_role": true,
        "company": "?新选组",
        "episodes": "1-13话",
        "functions": [
          "职能1",
          "职能2",
          "职能3"
        ]
      }
    ],
    "CreditEntry": [
      {
        "role": "企画",
        "episodes": "1-13话",
        "tips": "for_role动态字段的functions若为职能123代表注入失败。",
        "company": "?新选组",
        "person": "冲田总司",
        "functions": [
          "我",
          "在",
          "哪",
          "?"
        ]
      },
      {
        "role": "制片人",
        "episodes": "all",
        "parent_company": [
          "1",
          "2",
          "3"
        ],
        "company": "传马町牢屋敷@",
        "person": "吉田松阴"
      },
      {
        "role": [
          "宣传",
          "制作",
          "授权",
          "海外宣传"
        ],
        "episodes": "all",
        "parent_company": [
          "冬田書店",
          "春田書店"
        ],
        "company": "夏田書店",
        "person": "Shown Kochi"
      },
      {
        "role": [
          "宣传",
          "制作",
          "授权",
          "海外宣传"
        ],
        "episodes": "11-12话",
        "add": "海外流媒体合作方",
        "tips": "In association with Metplix"
      },
      {
        "role": "role",
        "episodes": "all",
        "tips": "$已注入role",
        "company": "Metplix",
        "person": "damn south",
        "parent_company": "MPLX"
      },
      {
        "role": [
          "宣传",
          "制作",
          "授权",
          "海外宣传"
        ],
        "episodes": "12",
        "tips": "Produced by WniQlexDmyamic,Blanning Inc."
      }
    ]
  }
]
````



---

##  转义规则

反斜杠 `\` 用于转义特殊字符。

###  常见转义对象

- `\$`
- `\:`
- `\,`
- `\|`
- `\#`
- `\?`
- `\->`
- `\<-`
- `\(`
- `\)`
- `\{`
- `\}`
- `\[`
- `\]`
- `\\`
- `\"`
- `\'`

###  语义

转义用于保证这些字符作为字面量出现，而不参与语法切分。

示例：

```text
* \->（？）
```

语义上应尽量保留 `->` 字面量，而不是当作跳转关系。

---

##  窗口权 window_rights

###  语法

窗口权通常写在 `{}` 中：

```text
Good Smile Company{商品权:12,影视改编权:"亚洲.ex日本"}
```

###  字段

| 字段 | 含义 |
|---|---|
| `rights_name` | 权利名称 |
| `rights_area` | 权利地区 |

###  地区映射

- `1` → 日本
- `2` → 亚洲
- `3` → 欧美
- `4` → 中东
- `5` → 除日本以外的全世界

若原始地区串无法拆成纯数字组合，则会作为原始文本保留。

---

##  标签与部门

###  角色标签

支持：

```text
< 企劃 >
< 製片人 >
企劃:
```

### 部门字段

组织内可通过 `|` 分出部门：

```text
* 新居祐介 ( 電通|动画企划部 )
```

解析为：

```json
{
  "company": "電通",
  "department": "动画企划部"
}
```

---

##  不确定性

`?` 通常表示不确定性。

###  人名不确定

```text
* 加藤かさい?
```

输出：

```json
{
  "person": "加藤かさい",
  "person_uncertain": true
}
```

### 公司不确定

```text
* 宇佐義大 ( Good Smile Company？)
```

输出：

```json
{
  "company": "Good Smile Company",
  "company_uncertain": true
}
```

###  前任来源不确定

```text
?<-bilibili
```

输出：

```json
{
  "former_company": "bilibili",
  "former_company_uncertain": true
}
```

---


## 7. 推荐示例
> ⚠️  *仅为本页面展示使用，与实际表记有较大出入。*
### 7.1 製作委員會 模式

```text
202604
##淡岛百景
*淡島百景製作委員会
KADOKAWA
MADHOUSE
富士电视台
樂天
BS富士
<企画>
工藤大丈（KADOKAWA）
<执行制片人>
田中翔（KADOKAWA）
田代早苗（MADHOUSE）
```

> 以 `*` 开头的行是**委员会标题行**（填入 `committee_name`）。标题之后的组织行输出到 `committee[]`，角色段落下的人名行不带 `*` 前缀。

### 7.2 solo / 製作/共同製作 模式

```text
202604
##BESTARS最終季第二部
s24:12
东宝
<製作>
大田圭二（东宝）
<执行制片人>
山中一孝（东宝）
<主制片人>
高橋敦司（东宝）
```

---

## 约定规范
- 普通 credit 行**不带** `*` 前缀；`*` 保留给以下特殊节点：
  - 作品起始处的委员会标题行（会填入 `metadata.committee_name`）
  - Credit 区由`*`声明的动画制作公司（会填入 `metadata.production`）
- 在 Credit 最后一行声明动画制作公司（非强制）
- 禁止无意义地使用符号映射功能造成歧义。
