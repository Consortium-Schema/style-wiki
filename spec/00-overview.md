# 概述

## 项目定位

本项目为日本商业动画的**制作委员会（製作委員会）及制作体制**建立一套结构化数据规范。

动画片尾的 credit 中记录了制作委员会成员公司、各职务负责人及其公司归属等信息。这些信息散布在每部作品的片头/片尾字幕中，缺乏统一的结构化整理。本项目的目标是：

1. **定义统一的数据格式**（JSON Schema），使不同贡献者录入的数据可以相互兼容
2. **建立转录规范**，明确从 credit 到结构化数据的映射规则
3. **维护参照数据**（公司名对照、职位分类等），减少重复劳动
4. **支持多语言**，使数据可为日语、繁体中文、简体中文、英语使用者所用

## 收录范围

### 收录

- 日本商业动画（TV、剧场版、OVA、ONA、Special、Short）
- 制作委员会成员公司列表
- 制作面 credit：企划、制片人、宣传、授权、音乐制片人、原作协力等
- 上述 credit 中涉及的公司归属、部门归属、人员异动

### 不收录

- 创作面 staff：监督、脚本、角色设计、作画监督、原画、声优等
- 技术面 staff：摄影、美术、音响、剪辑等
- 单集 credit（仅记录系列整体的制作面 credit）
- 非日本制作的动画

## 设计原则

### 1. 表记层与分析层分离

原始转录是客观事实——credit 上写什么就记什么。分析推论（如判断某人属于哪家公司、某公司是否为幹事社）是独立的层，需标记确信度。

两层存储，互不覆盖：

| 层 | 内容 | 举例 |
|----|------|------|
| 表记层（Transcription） | credit 原文的结构化记录 | `"company": "KADOKAWA Media Factory"` |
| 分析层（Analysis） | 正规化、归类、关联 | → 指向 company_id `kadokawa`（因 2013 年合并） |

### 2. 时间感知

制作委员会数据本质上是时间序列：

- **公司**会改名（BANDAI VISUAL → BANDAI NAMCO Arts → Bandai Namco Filmworks）
- **人员**会跳槽（从 A 公司转到 B 公司）
- **委员会成员**可能中途变更（前 14 话 vs 15 话之后）
- **Credit 人员**可能仅在部分集数登场（如制片人在 #01～#12 后更换）

数据模型需要表达"在什么时间点 / 哪些集数范围内为真"。以**集数范围**作为 credit 条目的时间维度，比记录人员交接链更直观且易于录入。

### 3. 渐进式建档

允许不完整数据。并非所有字段都必填——能确认多少就填多少：

- `？` 表示不确定
- 字段留空表示尚未查证
- 确信度字段（`confirmed` / `probable` / `uncertain`）标记信息可靠程度

### 4. 保留原文

所有正规化操作都额外存储，不覆盖原始表记。即使将 `KADOKAWA Media Factory` 归入 `KADOKAWA` 的 company_id 下，原始 credit 文本仍完整保留。

这确保数据可以追溯到原始来源，也允许正规化规则在未来修正时不丢失信息。

## 多语言范围

本项目的数据支持四种语言：

| 代码 | 语言 | 定位 |
|------|------|------|
| `ja` | 日语 | **原文层**——credit 原始表记，所有数据的最终来源 |
| `zh_hant` | 繁体中文 | 翻译层——公司名、职位名、作品名的繁体译名 |
| `zh_hans` | 简体中文 | 翻译层——公司名、职位名、作品名的简体译名 |
| `en` | 英语 | 翻译层——公司官方英文名、职位英文对应、作品英文名 |

核心规则：

- **日语为唯一 source of truth**。其他语言是衍生/翻译，不能反向修改日语层
- 多语言字段使用后缀命名：`name_ja`、`name_zh_hant`、`name_zh_hans`、`name_en`
- 非日语字段可以为空（渐进式建档），日语字段原则上必填
- 公司名、人名等专有名词优先使用**官方译名**，无官方译名时参照业界惯用译名

详细的多语言政策见 [spec/03-multilingual.md](03-multilingual.md)。

## 作品数据结构

一部作品的数据由三个层次构成：

```
Work
├── 1. committee / seisaku_company（核心）—— 最终归纳的成员公司列表
├── 2. Credit 条目（证据）—— 从片尾 credit 逐条转录的原始记录
└── 3. 作品信息（辅助）—— 类型、首播日、集数等，用于关联分析
```

### 1. committee / seisaku_company（核心数据）

每部作品最终归纳出的制作 / 委员会成员公司列表。这是本项目的**主要产出**。字段名由 `production_mode` 决定：

- `製作委員會` 模式 → `committee[]`，并在 `metadata.committee_name` 记录委员会标题
- 其他模式（`solo` / `製作/共同製作` / `Netflix Mode`） → `seisaku_company[]`

每个成员公司记录：

| 字段 | 说明 |
|------|------|
| **company** | 成员公司（指向 Company 实体） |
| **排列顺序** | credit 中的列名顺序（第一位通常是幹事社） |
| **episodes** | 集数范围（默认 `"all"`；若成员资格仅限部分集数，如 前14话 / 15话之后） |
| **from_role** | 布尔标记，表示该成员由 role 段间接派生（如 `Produced by：`），而非显式组织行 |

> 早期规范曾包含 `window_rights`（窗口权）、`functions`（职能）等字段；当前解析器输出样本中尚未生成，相关数据可由 CreditEntry 归纳得出。

### 2. Credit 条目（原始证据）

从 credit 逐条转录的结构化记录，是委员会数据的来源依据。

每条 credit 记录：

| 字段 | 说明 |
|------|------|
| **职位** | credit 上的原始职位名（如「製片人」「海外授權」） |
| **人名** | 人员姓名（可选——部分条目仅有公司） |
| **公司归属** | 人员所属公司（支持多层：部门→子公司→母公司） |
| **登场集数** | 该人员出现在哪些集数的 credit 中 |
| **确信度** | 公司归属是否确定（公司名后有 `？` 即 uncertain） |

Credit 条目只记录 credit 上**实际写了什么**，不做推断。

### 3. 作品信息（辅助 metadata）

用于未来建站时的筛选、关联、展示：

| 字段 | 说明 |
|------|------|
| **title** | 作品标题（多语言将在命名规范中展开） |
| **type** | 作品类型：TV / 剧场版 / OVA / ONA / Special / Short |
| **year / season** | TV 用，例如 `year="2026"`, `season="04"` |
| **release_date** | 首播 / 上映日（电影、OVA 使用） |
| **episodes_count** | 总集数 |
| **unit_duration / total_duration** | 单集时长 / 总时长 |
| **production** | 动画制作公司（studio），取自 Credit 区最后一行 |
| **original_type / original_sources** | 原作类型与来源（`original_company`、可选 `original_label`） |
| **committee_name** | 委员会原始日文名（仅 `製作委員會` 模式输出） |
| **production_mode** | 制作模式枚举：`solo` / `製作委員會` / `製作/共同製作` / `Netflix Mode` |
| **produced_by** | Produced by 列表（由同名 role 段派生） |
| **original_oncommittee** | 原作公司是否在 `committee` 中，仅委員會 模式且为 `true` 时输出 |
| **original_onseisaku** | 原作公司是否在 `seisaku_company` 中，仅非委員會 模式且为 `true` 时输出 |
| **in_association_with** | 协作方（由 `//In association with X` 注释行触发） |

### 独立的参照实体

以下实体不从属于某一部作品，而是跨作品共用：

| 实体 | 说明 |
|------|------|
| **Company**（公司） | 多语言名称、改名历史、母子公司 / 合并 / 分拆关系 |
| **Person**（人物） | 名称变体（新旧字体、罗马字）、公司履历 |
| **Role**（职位） | 原始职位名到正规分类的映射（category → function → raw name） |
| **Department**（部门） | 公司内部组织（编辑部、事业部等） |

### 关系总览

```
Work
├── committee[] (委員會模式)  |  seisaku_company[] (其他模式)
│   ├── → Company
│   ├── episodes            集数范围（默认 "all"）
│   └── from_role           由 role 段派生时标记
├── CreditEntry[]（证据）
│   ├── role                原始职位名
│   ├── → Person (optional)
│   ├── → Company / Department / parent_company
│   ├── affiliations[]      附属 / 马甲 / 二次派遣
│   ├── former_company(_uncertain)   跳槽来源
│   ├── unverified          仅有 person 无 company 时
│   ├── tips                由 // 注释行在当前 role 下生成
│   └── episodes            登场集数
└── metadata（辅助）
    ├── title, type, year, season, release_date ...
    ├── production, original_sources, produced_by ...
    ├── in_association_with ...
    └── production_mode, committee_name, original_oncommittee / original_onseisaku ...

Company ←→ Company（改名 / 合并 / 母子公司）
Company  → Department[]
Person   → CareerEntry[] → Company（跳槽履历）
Role     → RoleFunction  → RoleCategory
```

各实体的详细字段定义见 [spec/01-data-model.md](01-data-model.md)。

## 季度定义

TV 动画按季度归类，季度代码格式为 `YYYYMM`：

| 代码 | 季度 | 大致上架期 |
|------|------|-----------|
| 01 | 冬 | 12月～2月 |
| 04 | 春 | 3月～5月 |
| 07 | 夏 | 6月～8月 |
| 10 | 秋 | 9月～11月 |

> 代码中的月份（01/04/07/10）代表该季度的惯用标识月份，而非严格的首播月。
> 实际上架日可能略早于或晚于上述范围（例如部分冬番在 12 月末开播，归入次年冬季）。
>
> 剧场版和 OVA 不使用季度，改用上映 / 发售日期。

## 规范文档索引

状态：✅ 已完成 · 📋 规划中

| 文档 | 状态 | 内容 |
|------|------|------|
| [00-overview.md](00-overview.md) | ✅ | 本文——项目概述 |
| [01-data-model.md](01-data-model.md) | ✅ | JSON 字段定义与 EBNF 语法 |
| [02-transcription.md](02-transcription.md) | ✅ | 从 credit 到数据的转录规则（ASCH 格式） |
| 03-multilingual.md | 📋 | 多语言政策 |
| 04-naming/ | 📋 | 命名规则（公司、人名、作品、职位） |
| 05-role-taxonomy.md | 📋 | 职位分类体系 |
| 06-company-identity.md | 📋 | 公司识别与历史变迁 |
| 07-validation.md | 📋 | 数据验证规则 |
