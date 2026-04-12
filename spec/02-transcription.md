## 1. 录入规范 (Notation Syntax)

### 1.1 基础人员录入
使用无序列表配合特定的括号标记：

```text
##标题
製作委員会（，，，）
< 职位名称 >
* 姓名 ( 所属公司 )
* 姓名 ( 母公司 | 部门 )
```
---
## 1.2 特殊标记（Speical Modifiers）
使用以下符号映射 uncertain、former 和 succession 等复杂字段,留白视作公司(Anchor Injection)

| 符号 |语义  | 录入示例 | 对应JSON字段
| :--- | :--- | :--- | :--- |
| ？   | 不确定性 | 夏目公一朗（KADOKAWA？）| person_uncertain, company_uncertain
| <-  | 跳槽来源 | 安倍孝二（GOOD Smile China）<-bilibili | former_company
| ->  | 继承链 | 沢辺伸政（小学馆）「1-13话」->備前島幹人（小学馆）「14话－24话」|succession: [CreditEntry]
| \|  | 隶属分隔 | 大西恒平（集英社\|周刊少年JUMP编辑部）|company, department

---
## 2. 实战案例预览 (Case Study)

>仅作参考，本片实际并未写出委员会公司

```
## 黒猫と魔女の教室

「黒猫と魔女の教室」製作委員会 ( 電通, Good Smile Company, CBC電視台, Ultra Super Pictures, 大一商會 )
< 企劃 >
* 新居祐介 ( 電通 )
* 宇佐義大 ( Good Smile Company )
* 高島祐一郎 ( 講談社 )
* 岡﨑剛之 ( CBC電視台 )
* 國枝信吾 ( Ultra Super Pictures )
* 市原寛之 ( 大一商會 )

< 執行製片人 >
* 石黒研三 ( 電通 )
* 中路亮輔 ( Good Smile Company )
* 古川慎 ( 講談社 )
* 今泉昌也 ( CBC電視台 )
* 里見哲朗 ( Ultra Super Pictures )
* 宮本和紀 ( 大一商會 )

< 製片人 >
* 菊池瑠梨子 ( 電通 )
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
