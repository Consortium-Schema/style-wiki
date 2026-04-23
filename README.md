# Consortium-Schema Style Wiki

本仓库用于规范动画制作委员会（製作委員会）数据的录入标准。

## 简介

日本商业动画的制作体制信息（制作委员会成员、制片人、授权窗口等）散布在各作品的片尾 credit 中。本项目为这些数据建立统一的结构化规范，使不同贡献者的录入成果可以互相兼容，并支持多语言使用。

## 支持语言

| 代码 | 语言 | 定位 |
|------|------|------|
| `ja` | 日语 | 原文层——credit 原始表记 |
| `zh_hant` | 繁体中文 | 翻译层 |
| `zh_hans` | 简体中文 | 翻译层 |
| `en` | English | 翻译层 |

## 文档结构

状态标记：✅ 已完成 · 🚧 撰写中 · 📋 规划中

```
style-wiki/
├── spec/                              # 核心规范
│   ├── 00-overview.md                 ✅ 项目概述、设计原则、收录范围
│   ├── 01-data-model.md               ✅ JSON 字段定义与 EBNF 语法
│   ├── 02-transcription.md            ✅ ASCH 录入格式规范
│   ├── 03-multilingual.md             📋 多语言政策
│   ├── 04-naming/                     📋 命名规则
│   │   ├── company.md                 📋   公司命名
│   │   ├── person.md                  📋   人名
│   │   ├── work-title.md              📋   作品标题
│   │   └── role.md                    📋   职位名称
│   ├── 05-role-taxonomy.md            📋 职位分类体系
│   ├── 06-company-identity.md         📋 公司识别与历史变迁
│   └── 07-validation.md               📋 数据验证规则
├── schema/                            📋 JSON Schema（机器可读）
├── refs/                              📋 参照数据（职位对照表、公司别名等）
└── examples/                          📋 数据文件示例
```

## 快速入门

- **了解项目**：阅读 [spec/00-overview.md](spec/00-overview.md)
- **开始录入**：阅读 [spec/02-transcription.md](spec/02-transcription.md)
- **字段定义**：阅读 [spec/01-data-model.md](spec/01-data-model.md)

## 设计原则

1. **表记层与分析层分离** —— credit 原文是客观事实，分析推论标记确信度
2. **时间感知** —— 公司名、人员归属、委员会成员都会随时间变化
3. **渐进式建档** —— 允许不完整数据，能确认多少就填多少
4. **保留原文** —— 正规化额外存储，不覆盖原始表记

详见 [spec/00-overview.md](spec/00-overview.md)。

## 贡献

欢迎参与规范的讨论与完善。请通过 Issue 提出建议或疑问。
