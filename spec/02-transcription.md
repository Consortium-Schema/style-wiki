## 1. 录入规范 (Ntation Syntax)
>处理复杂系统数据的正确姿势永远是保持输入端是Clean Text，通过逻辑层生成Rich Data。基于此，人力扁平文本的录入做法是必要的，避免了在录入时写复杂的嵌套JSON，实现录入成本最小化与数据价值最大化。
### 1.1 基础人员录入
使用无序列表配合特定的括号标记：

```text
##标题
製作委員会（，，，）*若表记完整写出委員会名单
< 职位名称 >
* 姓名 ( 所属公司 )
* 姓名 ( 母公司 | 部门 )
```
---
## 1.2 特殊标记（Speical Modifiers）
使用以下符号映射 uncertain、former 和 succession 等复杂字段,留白视作公司(Anchor Injection)
| 符号 |语义  | 录入示例 | 对应JSON字段
| :--- | :--- | :--- | :--- |
| ？   | 不确定性 | 夏目公一朗（KADOKAWA？）| person_uncertain, company_uncertain, former_company_uncertain
| <-   | 跳槽来源 | 安倍孝二（GOOD Smile China）<-bilibili | former_company
| ->   | 继承链 | 沢辺伸政（小学馆）「1-13话」->備前島幹人（小学馆）「14话－24话」|succession: [CreditEntry]
| \|   | 隶属分隔 | 大西恒平（集英社\|周刊少年JUMP编辑部）|company, department
| #    | 消歧义（高录入成本）（非必要不使用）| 鈴木健太#org:Aniplex（Aniplex)或アリエル・リー#id:Ariel Li（Crunchyroll）| person_id

---
## 2.基础结构
### 三段式布局
编写时请严格遵守以下层级，每一行都代表一个独立的数据维度。
```
202604                                   <-- [维度1: 年月] 必须是6位纯数字
## 黒猫と魔女の教室              <-- [维度2: 标题] 必须以 ## 开头
```
 关于製作委員会，若为电视台/Netflix Mode/独资作品等特殊情况时，则必须将公司/In Association with Netflix/Produce by xxx置于表记首行。

---
### 核心解析语法 (Credit Entry)
##### 每一行以 * 开头的条目都是一个证据节点。通过符号映射，你可以表达极其复杂的权属关系：
* 基础归属与部门划分 ( | )使用括号标记机构。如果涉及具体部门，使用 | 分隔。
* 格式：* 姓名 ( 公司 | 部门 )
* 示例：* 川窪慎太郎 ( 講談社 | 週刊少年MAGAZINE編輯部 )
---

### 确定性标记 ?
 ? 是一个多态修饰符，它会根据放置的位置，自动在JSON中生成对应的uncertain字段。
* 人员不确定：* 姓名? ( 公司 )
* 公司不确定：* 姓名 ( 公司 )? 或 * 姓名 ( 公司? )
*  跳槽来源不确定：* 姓名 ( 公司 ) ?<- 前公司
---
### 跳槽/来源溯源 <-
##### 用于记录人才的流动背景，这对于分析CareerEntry非常重要。
* 语法：* 姓名 ( 当前公司 ) <- 来源公司
* 示例：* 安倍孝二 ( GSC ) <- bilibili
---
###  继承链与集数映射 -> 与 「」
##### 这是最强大的功能，用于处理中途换人或特定集数职位的变动。

---
### 消歧义标签 \#
##### 用于处理姓名重合与Nick Name等person字段的极端情况。
|录入场景|录入格式|清洗场景|结果
| :--- | :--- | :--- | :--- |
|重名区分|鈴木健太#org：MBS|组合键：人名+org:MBS|独立统计
|别名/真名|Yui Lin#id：林韋菱|唯一键: id:林韋菱|强聚合

---
## 2.1.高效录入:归一化处理
* >你在录入时无需担心的事情。

#### 全/半角符号透明切换
脚本会自动将全角符号映射为半角。你可以完全根据输入法状态随心所欲地输入：
* 括号：( ) 与（ ）效果一致。
* 职位：< >、＜ ＞、《 》、〈 〉效果一致。
* 分隔符：|与｜、,与，、: 与 ：效果一致。

#### 职位标签空格免疫
在输入职位标签时，标签内的空格会被自动剔除：
*  < 製作 委員會 > 会自动识别为 <製作委員會>。
*  这允许你在编辑 MD 时为了美观故意留白，而不影响后端解析。

 #### 元数据区括号自由
在作品标题下的製作委員会声明区：
* 你可以随意换行缩进来兼具可读性 例:
```
「黒猫と魔女の教室」製作委員会                                      （
    電通, 
    Good Smile Company, 
    CBC電視台, Ultra Super Pictures, 
    大一商會  


                                                                                                )
```
* 即使名单过长导致换行，只要括号未闭合，脚本会合并下一行，直到括号闭合。

---
## 2.2进阶录入技巧 (Tips)
### 处理马甲公司与多重身份
将第一个括号的公司解析为主Company字段，后续所有括号内公司录入affiliations字段。
* 输入：*高木隆行 (Christmas Holy) (Global Solutions)
* 解析：主字段company是Christmas Holy,而Global Solutions会出现在affiliations的字段列表中。
### 日期与逻辑分段
* 使用YYYYMM（如 202604）作为独立行，定义之后所有作品的季度归属。沿用到下个日期定义前（如 202607），你不需要为每个作品都写上日期
* 在不同的职位块（如<制片人>到<副制片人>）之间保留一个空行，有助于脚本清晰地重置 current_role状态。

---
## 3. 实战案例预览 (Case Study)

>仅作本页面理解参考，与真实表记存在出入。

```
202604
## 黒猫と魔女の教室

「黒猫と魔女の教室」製作委員会 ( 電通, Good Smile Company, CBC電視台, Ultra Super Pictures, 大一商會 )
< 企劃 >
* 新居祐介 ( 電通|动画企划部 )
* 宇佐義大 ( Good Smile Company？)
* 高島祐一郎 ( 講談社 )「12-14话」->古川慎（講談社）「14-24话」
* 岡﨑剛之 ( CBC電視台) ?<-bilibili
* 國枝信吾 ( Ultra Super Pictures )
* 市原寛之 ( 大一商會 )

< 執行製片人 >
* 石黒研三 ( 電通 )
* 中路亮輔 ( Good Smile Company )
* 古川慎 ( 講談社 )
* 今泉昌也 ( CBC電視台 )
* 里見哲朗 ( Ultra Super Pictures )
* 宮本和紀 ( 大一商會 )( 大二商會  )( 大三商會  )

< 製片人 >
* 菊池瑠梨子 ( 電通 )
* 菊池瑠梨子 #org:Good Smile Company （Good Smile Company）
* 冨田功一郎 ( Good Smile Company )
* 塩谷佳之 ( 講談社 )
* 柴田知宏 ( CBC電視台 )
* 西川恭平 ( Ultra Super Pictures )
* 加藤弘泰 ( 大一商會 )

< 副製片人 >
* 西康介 ( 電通 )
* 池内矩史 ( Good Smile Company )
* 尾上裕紀 ( 講談社 )
* 吉田翔平 ( CBC電視台 )
* 篠崎友美 ( 大一商會 )

< 企劃協力 >
* 荒木めぐみ ( 電通 )
* 安部正実 ( CBC電視台 )
* 市川湧 ( CBC電視台 )

< 原作協力 >
* 川窪慎太郎 ( 講談社|週刊少年MAGAZINE編輯部 )
* 菊地優斗 ( 講談社|週刊少年MAGAZINE編輯部 )
* 金子昇太 ( 講談社|週刊少年MAGAZINE編輯部 )

< 宣傳製片人 >
* 小幡敬志朗 ( Good Smile Company )

< 宣傳協力 >
* 上村真由 ( Good Smile Company )

< 授權 >
* 五十嵐桃子 ( Good Smile Company )
* 石綿春也 ( 講談社 )
* 熊谷亜希子 ( 講談社 )
* 多賀井勲 ( 講談社 )
* 根本樹 ( 講談社 )
* 杜伊 ( 講談社 )

< 海外銷售Promotion >
* 天野友里亜 ( 講談社 )
* 小田部恵流川 ( 講談社 )
* 魏思思 ( 講談社 )
* 和田光代 ( 講談社 )
* 岡部愛実里 ( 講談社 )

< 原作Promotion >
* 田幸志朗 ( 講談社 )
* 久松誠輝 ( 講談社 )
* 柴田薫 ( 講談社 )
* 秋吉正太 ( 講談社 )

< OP >
* ( SACRA MUSIC )

< 主題歌協力 >
* 大浜拓哉 ( Sony Music Entertainment )
```
清洗后的JSON大致如下:
```
[
  {
    "metadata": {
      "year": "2026",
      "season": "04",
      "title": "黒猫と魔女の教室",
      "production_model": "製作委員會",
      "committee_name": "「黒猫と魔女の教室」製作委員会"
    },
    "committee": [
      {
        "company": "電通",
        "functions": [],
        "window_rights": [],
        "episodes": "all"
      },
      {
        "company": "Good Smile Company",
        "functions": [],
        "window_rights": [],
        "episodes": "all"
      },
      {
        "company": "CBC電視台",
        "functions": [],
        "window_rights": [],
        "episodes": "all"
      },
      {
        "company": "Ultra Super Pictures",
        "functions": [],
        "window_rights": [],
        "episodes": "all"
      },
      {
        "company": "大一商會",
        "functions": [],
        "window_rights": [],
        "episodes": "all"
      }
    ],
    "CreditEntry": [
      {
        "role": "企劃",
        "episodes": "all",
        "company": "電通",
        "department": "动画企划部",
        "person": "新居祐介"
      },
      {
        "role": "企劃",
        "episodes": "all",
        "company": "Good Smile Company",
        "company_uncertain": true,
        "person": "宇佐義大"
      },
      {
        "role": "企劃",
        "episodes": "12-14话",
        "company": "講談社",
        "person": "高島祐一郎",
        "succession": [
          {
            "role": "企劃",
            "episodes": "14-24话",
            "company": "講談社",
            "person": "古川慎"
          }
        ]
      },
      {
        "role": "企劃",
        "episodes": "all",
        "former_company": "bilibili",
        "former_company_uncertain": true,
        "company": "CBC電視台",
        "person": "岡﨑剛之"
      },
      {
        "role": "企劃",
        "episodes": "all",
        "company": "Ultra Super Pictures",
        "person": "國枝信吾"
      },
      {
        "role": "企劃",
        "episodes": "all",
        "company": "大一商會",
        "person": "市原寛之"
      },
      {
        "role": "執行製片人",
        "episodes": "all",
        "company": "電通",
        "person": "石黒研三"
      },
      {
        "role": "執行製片人",
        "episodes": "all",
        "company": "Good Smile Company",
        "person": "中路亮輔"
      },
      {
        "role": "執行製片人",
        "episodes": "all",
        "company": "講談社",
        "person": "古川慎"
      },
      {
        "role": "執行製片人",
        "episodes": "all",
        "company": "CBC電視台",
        "person": "今泉昌也"
      },
      {
        "role": "執行製片人",
        "episodes": "all",
        "company": "Ultra Super Pictures",
        "person": "里見哲朗"
      },
      {
        "role": "執行製片人",
        "episodes": "all",
        "company": "大一商會",
        "affiliations": [
          {
            "company": "大二商會"
          },
          {
            "company": "大三商會"
          }
        ],
        "person": "宮本和紀"
      },
      {
        "role": "製片人",
        "episodes": "all",
        "company": "電通",
        "person": "菊池瑠梨子"
      },
      {
        "role": "製片人",
        "episodes": "all",
        "company": "Good Smile Company",
        "person": "菊池瑠梨子",
        "person_id": "org:Good Smile Company"
      },
      {
        "role": "製片人",
        "episodes": "all",
        "company": "Good Smile Company",
        "person": "冨田功一郎"
      },
      {
        "role": "製片人",
        "episodes": "all",
        "company": "講談社",
        "person": "塩谷佳之"
      },
      {
        "role": "製片人",
        "episodes": "all",
        "company": "CBC電視台",
        "person": "柴田知宏"
      },
      {
        "role": "製片人",
        "episodes": "all",
        "company": "Ultra Super Pictures",
        "person": "西川恭平"
      },
      {
        "role": "製片人",
        "episodes": "all",
        "company": "大一商會",
        "person": "加藤弘泰"
      },
      {
        "role": "副製片人",
        "episodes": "all",
        "company": "電通",
        "person": "西康介"
      },
      {
        "role": "副製片人",
        "episodes": "all",
        "company": "Good Smile Company",
        "person": "池内矩史"
      },
      {
        "role": "副製片人",
        "episodes": "all",
        "company": "講談社",
        "person": "尾上裕紀"
      },
      {
        "role": "副製片人",
        "episodes": "all",
        "company": "CBC電視台",
        "person": "吉田翔平"
      },
      {
        "role": "副製片人",
        "episodes": "all",
        "company": "大一商會",
        "person": "篠崎友美"
      },
      {
        "role": "企劃協力",
        "episodes": "all",
        "company": "電通",
        "person": "荒木めぐみ"
      },
      {
        "role": "企劃協力",
        "episodes": "all",
        "company": "CBC電視台",
        "person": "安部正実"
      },
      {
        "role": "企劃協力",
        "episodes": "all",
        "company": "CBC電視台",
        "person": "市川湧"
      },
      {
        "role": "原作協力",
        "episodes": "all",
        "company": "講談社",
        "department": "週刊少年MAGAZINE編輯部",
        "person": "川窪慎太郎"
      },
      {
        "role": "原作協力",
        "episodes": "all",
        "company": "講談社",
        "department": "週刊少年MAGAZINE編輯部",
        "person": "菊地優斗"
      },
      {
        "role": "原作協力",
        "episodes": "all",
        "company": "講談社",
        "department": "週刊少年MAGAZINE編輯部",
        "person": "金子昇太"
      },
      {
        "role": "宣傳製片人",
        "episodes": "all",
        "company": "Good Smile Company",
        "person": "小幡敬志朗"
      },
      {
        "role": "宣傳協力",
        "episodes": "all",
        "company": "Good Smile Company",
        "person": "上村真由"
      },
      {
        "role": "授權",
        "episodes": "all",
        "company": "Good Smile Company",
        "person": "五十嵐桃子"
      },
      {
        "role": "授權",
        "episodes": "all",
        "company": "講談社",
        "person": "石綿春也"
      },
      {
        "role": "授權",
        "episodes": "all",
        "company": "講談社",
        "person": "熊谷亜希子"
      },
      {
        "role": "授權",
        "episodes": "all",
        "company": "講談社",
        "person": "多賀井勲"
      },
      {
        "role": "授權",
        "episodes": "all",
        "company": "講談社",
        "person": "根本樹"
      },
      {
        "role": "授權",
        "episodes": "all",
        "company": "講談社",
        "person": "杜伊"
      },
      {
        "role": "海外銷售Promotion",
        "episodes": "all",
        "company": "講談社",
        "person": "天野友里亜"
      },
      {
        "role": "海外銷售Promotion",
        "episodes": "all",
        "company": "講談社",
        "person": "小田部恵流川"
      },
      {
        "role": "海外銷售Promotion",
        "episodes": "all",
        "company": "講談社",
        "person": "魏思思"
      },
      {
        "role": "海外銷售Promotion",
        "episodes": "all",
        "company": "講談社",
        "person": "和田光代"
      },
      {
        "role": "海外銷售Promotion",
        "episodes": "all",
        "company": "講談社",
        "person": "岡部愛実里"
      },
      {
        "role": "原作Promotion",
        "episodes": "all",
        "company": "講談社",
        "person": "田幸志朗"
      },
      {
        "role": "原作Promotion",
        "episodes": "all",
        "company": "講談社",
        "person": "久松誠輝"
      },
      {
        "role": "原作Promotion",
        "episodes": "all",
        "company": "講談社",
        "person": "柴田薫"
      },
      {
        "role": "原作Promotion",
        "episodes": "all",
        "company": "講談社",
        "person": "秋吉正太"
      },
      {
        "role": "OP",
        "episodes": "all",
        "company": "SACRA MUSIC"
      },
      {
        "role": "主題歌協力",
        "episodes": "all",
        "company": "Sony Music Entertainment",
        "person": "大浜拓哉"
      }
    ]
  }
]
```
