# 概述
##### 为方便后续维护和脚本编写，本页面将所有数据字段归档说明。涵盖字段含义、触发逻辑与数据类型。

## 1.顶部元数据（Metadata）
位于每个作品块的顶层，定义作品的基础背景。
|字段名|类型  |说明  |触发逻辑|
| :--- | :--- | :--- | :--- |
|year|string|公映/开播年份|匹配YYYYMM的前四位
|season|string|季度|匹配YYYYMM的后二位
|title|string|作品正式标题|匹配##标题
|producution_model|string|制作模式|默认为”製作委員会“，根据行首特征识别
|committee_name|string|委员会名称|当制作模式为“製作委員会"时显示，提取行首内容
---
## 2.製作委員会成员（Committee）
存储于committee列表，解析製作委員会内的公司详情。
|字段名 |类型 |说明  |示例|
| :--- | :--- | :--- | :--- |
|company|string|参与的公司名称|（KADOKAWA，...）中的成员
|functions|list|职能|默认为[]
|window_rights|list|窗口权|默认为[]
|episodes|string|参与集数|默认为"all"
---
## 3.职员条目（CreditEntry）
核心数据对象，描述具体的职位与人员关系。

### A.人员标识
|字段名 |类型   |说明   |触发逻辑|
| :--- | :--- | :--- | :--- |
|person|string|署名，当前作品中官方给出的名称。|括号外的文本
|person_id|string|唯一标识|。用于实体关联|匹配#后的内容
|person_uncertain|bool|人名不确定标记|匹配姓名？（仅在为true时显示）
### B.机构归属
|字段名  |类型    |说明    |触发逻辑|
| :--- | :--- | :--- | :--- |
|company|string|主归属，核心雇主或所属会社|第一个括号的内容
|department|string|细分部门/工作室/编辑部|匹配括号内的\|后缀
|company_undertain|bool|机构归属不确定标记|匹配（公司？）(仅为true时显示）
|affiliations|list|关联机构、马甲、挂靠或外聘/派遣|仅在存在两个或以上括号时显示
#### C.履历与时间轴
|字段名  |类型    |说明    |触发逻辑|
| :--- | :--- | :--- | :--- |
|role|string|职位名称|匹配<职位>或职位：
|episodes|string|参与的具体集数|匹配「...」默认为"all"
|former_company|string|跳槽来源/前东家|匹配<-公司
|former_company_uncertain|bool|不确定跳槽标记|匹配?<-公司
|succession|list|继承链。职位更替、中途接手。|匹配*A->B

----
 #  开发者备忘录
## 1. 脚本聚合逻辑说明
### 在编写脚本处理JSON季度数据时，你的get_uid函数应该遵循以下优先级：
* ##### 主名优先(Identity-First)：如果person_id以id:开头，弱化该person署名，以该ID作为统计主键。
* ##### 重名隔离(Discriminator-Second)：如果person_id以org:开头（或纯数字），则将person+person_id组合成一个唯一主键。
* ##### 默认处理:如果没有ID，则以person署名为准。
## 2. 结构简化规则
##### 零冗余：所有的_uncertain字段在信息确定时（无 ?）不会出现在JSON中，以减小体积。
### 括号优先级：
 ##### \* 姓名(A) →company:A
#####  \* 姓名(A)(B)→company:A,affiliations:[{company: A}, {company: B}] (A为Primary)
## 数据导向指引
* Normalization: 全角转半角，清理冗余空格。
* Greedy Extraction: 优先切除「集数」和<- 来源。
* Parentheses Splitting: 解析括号并决定company与affiliations。
* Final Cleanup: 剩下的干净文本作为person并解析#ID。
