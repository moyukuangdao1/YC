---
document_id: yc-strategy-architecture-addendum-core-solution
version: 1.0
status: accepted_for_planning
updated_at: 2026-08-06
owner: YC Founder
purpose: 沉淀“稳定地基 + 多样化业务楼宇 + 人机智能协同 + 知识资产复利”的最新战略与架构共识
supersedes: null
applies_to:
  - YC战略规划/商业计划书
  - YC KOC合作经营中枢产品技术方案
  - YC Browser Service集成规范
  - 后续Agent Runtime与知识资产设计
---

# YC战略与架构增量决策：稳定内核与业务扩展 v1.0

## 0. 本文要解决的问题

经过Day 1纵向闭环和多轮讨论，YC已经从“若干业务脚本/Skill的集合”进一步收敛为：

> **一套稳定、可靠、可审计的企业AI业务运行地基；创始人以企业AI落地负责人的身份，在这套地基上持续部署一个个高价值Business Cell。**

企业之间既存在统一性，也必然存在差异性。YC不追求把企业改造成同一套流程，也不接受为每位员工永久维护一份独立代码。正确方式是：

```text
稳定YC Core
+ 可复用Solution Package
+ 企业Tenant Deployment
+ Source/Mapping/Rule/Notification配置
+ Program/Agent/Human/Edge多种执行者
+ 运行中沉淀的Ontology/Wiki/Skill/Eval
```

本文用于统一公司战略、产品结构、技术扩展方式、知识资产和下一阶段研发顺序。

---

# 1. YC的长期产品心智模型

可以把YC类比为FastAPI，但不是Web开发框架，而是“企业AI业务运行框架”。

## 1.1 YC Core：稳定地基

YC Core提供所有业务共同需要、且不应随企业差异反复修改的能力：

- Tenant与身份隔离；
- RBAC与未来ABAC；
- Business Object与Case；
- CellRun、StepRun、Job、Command、Attempt；
- ProgramTask、AgentTask、HumanTask、EdgeCommand；
- Approval、Exception、Timer、ResumeSignal；
- 幂等、租约、对账、暂停、恢复；
- Evidence、Activity、Audit、Outcome；
- Config、Revision、Deployment；
- Notification与安全临时材料；
- Knowledge Asset Registry；
- 可观测性、成本、版本和发布。

YC Core不应该知道：

- 某企业飞书列名；
- 云仓页面如何点击；
- 某品牌重复寄样周期；
- 某员工喜欢怎样写日报；
- 女装商品分析的具体规则。

它只负责：

> **一项业务正在运行、现在等待谁完成哪种有契约的步骤、如何确认完成，以及失败后如何安全继续。**

## 1.2 Solution SDK：盖楼规则

Solution开发者或未来Designer Agent使用稳定扩展接口：

```text
CellDefinition
StepHandler
Connector
ProgramHandler
AgentProvider
HumanTaskProvider
NotificationProvider
PolicyEvaluator
KnowledgeProvider
OutcomeValidator
```

SDK必须尽量小。只有被多个真实Cell共同需要的能力，才进入YC Core或SDK。

## 1.3 Solution Package：一类可复用业务楼宇

例如KOC解决方案：

```text
solutions/koc/
├─ manifest.yaml
├─ ontology.yaml
├─ cells/
│  ├─ sample_fulfillment/
│  └─ douyin_cooperation_order/
├─ skills/
├─ policies/
├─ mapping_schema/
├─ ui/
├─ evals/
└─ migrations/
```

它定义的是一类相对稳定的业务能力，而不是某家企业的全部个性化配置。

## 1.4 Tenant Deployment：企业自己的装修

同一个Solution部署到不同企业时绑定：

- Connector实例；
- 飞书Source Binding；
- Mapping Profile；
- RuleSet；
- 审批矩阵；
- Browser Profile；
- Notification Profile；
- 成本与指标口径；
- 企业专属Skill和Wiki；
- 某个团队/品牌/员工的显示偏好。

企业差异优先通过配置、规则和Workflow Variant承载，而不是复制代码。

---

# 2. 企业统一性与差异性的承载方式

## 2.1 五类差异必须分开

| 差异 | 示例 | 承载机制 |
|---|---|---|
| 数据结构 | “达人UID”与“抖音ID” | Canonical Schema + MappingProfile |
| 数据来源 | 每位员工使用不同飞书表 | SourceBinding |
| 业务规则 | 30天或45天重复寄样 | RuleSet/Policy |
| 正式流程 | 先寄样后合作单或相反 | Workflow Variant/Cell Revision |
| 个人偏好 | 通知渠道、列顺序、看板 | User/Notification Preference |

## 2.2 配置继承

建议继承顺序：

```text
YC安全底线
→ Solution默认规则
→ Tenant企业规则
→ BusinessUnit/品牌/团队规则
→ SourceBinding规则
→ 用户显示与通知偏好
```

下层配置不得绕过上层安全底线，例如：

- 幂等与重复检查；
- 金额/数量硬限制；
- 高风险审批；
- 审计；
- 跨租户隔离。

## 2.3 先兼容，再收敛

不同员工独立飞书表是现实，但不应永远固化组织混乱。

```text
适应期：适配现有表和习惯
→ 收敛期：统一核心对象、字段和状态
→ 优化期：只保留有业务理由的正式差异
```

YC的价值不仅是自动化，也包括帮助管理者逐步建立可见、可比较、可复盘的业务标准。

---

# 3. 执行者模型：谁完成下一步

YC Core不把所有东西都称为Agent。统一使用五类执行者：

| 执行者 | 用途 |
|---|---|
| Program Worker | 校验、计算、状态推进、规则和确定性API |
| Worker Agent | 文本、视频、判断、总结等智能任务 |
| Edge Node | 客户本地受控执行、Browser/Profile/本地文件 |
| Human Worker | 审批、登录、验证码、谈判、关系和复杂例外 |
| Timer/Event | 等待时间、平台回调、定时轮询 |

Cell Runtime是业务控制者。任何Worker都不能凭自己的输出直接宣布企业业务完成。

统一Step Outcome建议：

```text
Complete
RunProgram
RunAgent
DispatchEdgeCommand
WaitHuman
WaitTimer
ResultUnknown
Fail
```

增加新业务时，应主要增加Cell、Connector、Skill和配置；只有出现新的通用执行原语时，才修改YC Core。

---

# 4. 两层Agent与确定性Runtime

## 4.1 Designer Agent

Designer Agent/Cell Architect Agent用于：

- 观察业务轨迹；
- 分析业务对象和关系；
- 拆解Workflow；
- 定义节点输入输出；
- 生成Cell、Skill、Policy、Mapping、UI和Eval候选；
- 根据运行结果提出新版本。

它可以复用Codex、Kimi、WorkBuddy或未来任何Agent产品。YC不自行制造通用Agent Harness。

但Designer Agent只能产出候选：

```text
草案
→ Sandbox
→ 历史案例回放
→ 人工审核
→ Shadow Mode
→ Pilot
→ 发布Revision
```

## 4.2 Worker Agent

Worker Agent只处理边界明确的小任务：

- 异常分类；
- 沟通记录总结；
- 标准日报；
- 视频卖点覆盖；
- 评论洞察；
- 相似案例检索。

每个任务必须固定：

- Skill版本；
- 最小上下文；
- 输出Schema；
- 模型路由和成本；
- 工具白名单；
- 人工升级条件。

## 4.3 Agent Provider Abstraction

YC不造通用Harness，但拥有稳定Provider接口：

```text
start(task)
poll(execution)
cancel(execution)
collect_result(execution)
collect_artifacts(execution)
capabilities()
```

未来可以接：

- Direct LLM；
- Codex；
- Kimi；
- WorkBuddy；
- 本地模型；
- 临时容器Agent；
- 企业私有Agent。

## 4.4 Ephemeral Agent Worker

适用于长程Agent、代码、文件、视频和并行任务：

```text
任务进入队列
→ 创建临时容器
→ 下载最小上下文和Artifact
→ 运行外部Harness
→ 心跳和Checkpoint
→ 上传结果
→ 容器销毁
```

Control Plane、Edge Node、Browser Service和Browser Profile不应随任务销毁。

这一层属于后续阶段，不进入当前真实Connector之前的P0。

---

# 5. 人类交互是一等运行原语

浏览器登录、二维码、短信验证码、人工审批不能只作为“报错给管理员”。

## 5.1 HumanTask

首批类型：

```text
AUTH_QR_SCAN
SMS_OTP_REQUIRED
MANUAL_LOGIN
APPROVAL_REQUIRED
MISSING_INFORMATION
EXTERNAL_RESULT_CONFIRMATION
MANUAL_EXCEPTION_RESOLUTION
CONTENT_REVIEW
```

状态：

```text
OPEN
→ NOTIFIED
→ WAITING_USER
→ SUBMITTED
→ VERIFYING
→ RESOLVED
```

终态：

```text
EXPIRED / CANCELLED / FAILED
```

## 5.2 登录二维码流程

```text
Connector发现登录失效
→ 返回AUTH_QR_REQUIRED
→ Runtime暂停Step
→ 创建HumanTask
→ Edge截取二维码
→ 上传短时安全Artifact
→ NotificationProvider通知员工
→ 员工扫码
→ Edge重新验证账号与登录态
→ HumanTask完成
→ ResumeSignal恢复原Step
```

## 5.3 安全边界

- 二维码使用短时一次性链接；
- 完成后删除原图，永久审计只记事件；
- OTP不写普通日志、Evidence和Wiki；
- 消息渠道不是事实来源，HumanTask才是；
- 个人微信可作为早期人工通道，但不是生产关键依赖。

---

# 6. 知识资产的部署与复利

真正的护城河不是文档数量，而是与业务事实、决策、证据和结果绑定的可执行知识。

## 6.1 Ontology Registry

版本化定义：

- 对象；
- 属性；
- 关系；
- 状态；
- 事件；
- 指标。

初期采用PostgreSQL关系模型 + YAML/JSON Schema，不引入图数据库。

层级：

```text
YC通用本体
→ 电商领域本体
→ KOC Solution本体
→ Tenant扩展与别名
```

## 6.2 Decision Ledger

专家Know-how最重要的原始资产：

- 当时事实；
- AI建议；
- 人最终决策；
- 修改差异；
- 决策理由；
- 关键线索；
- 反事实条件；
- 后续结果。

专家无法从空白完整表达经验时，采用“AI先建议、专家修改、少量追问、结果回流”。

## 6.3 Business Wiki

保存经过确认的：

- 指标口径；
- SOP；
- 常见异常；
- 产品知识；
- 有效和失败案例；
- 适用条件与例外。

正式知识至少包含：

```text
scope / source / owner / validity / evidence / confidence / version
```

## 6.4 Skill Registry

正式Skill包含：

```text
适用场景
输入Schema
目标与步骤
业务规则
知识引用
允许工具
禁止事项
输出Schema
风险等级
人工升级条件
评测案例
版本与适用范围
```

## 6.5 Evaluation Library

Skill、Agent和Designer产物必须经过：

- 历史案例；
- 正常案例；
- 边界案例；
- 错误案例；
- 专家预期；
- 后续业务结果。

## 6.6 多租户知识隔离

```text
Global
Industry
Solution
Tenant
Case
```

企业专属知识不得跨租户检索。企业数据与专属规则默认属于企业；YC通用框架和通用Cell属于YC；跨客户提炼必须授权、脱敏和抽象。

---

# 7. 商业战略：AI落地负责人的“盖楼飞轮”

创始人以外部企业AI业务落地负责人进入企业：

```text
识别高价值业务
→ 在YC Core上部署Cell
→ 收取诊断/实施/托管费用
→ 连续运行和量化价值
→ 沉淀Connector、Mapping、Policy、异常和案例
→ 提炼Solution Package
→ 第二家企业主要配置化复制
→ 形成更高毛利的软件驱动服务
```

每次客户交付必须留下至少一项可复用资产：

- Cell；
- Connector；
- Mapping Schema；
- RuleSet；
- HumanTask模式；
- 异常分类；
- Wiki/Skill候选；
- Eval Case；
- 展示和实施方法。

如果客户增加导致创始人维护时间线性增长，应暂停销售，先产品化。

---

# 8. 当前代码的架构演进

Day 1把业务与Runtime集中在`services.py`中，是纵向验证的合理临时形态；不能继续无限增加`if job_type`。

在真实Connector接入前，进行有限拆分：

```text
app/
├─ domains/
│  └─ koc/
│     ├─ cases.py
│     ├─ tracks.py
│     └─ policies.py
├─ runtime/
│  ├─ runs.py
│  ├─ jobs.py
│  ├─ commands.py
│  ├─ attempts.py
│  ├─ results.py
│  ├─ human_tasks.py
│  ├─ reconciliation.py
│  └─ handler_registry.py
├─ cells/
│  ├─ fake_sample/
│  ├─ sample_fulfillment/
│  └─ douyin_cooperation_order/
├─ connectors/
│  ├─ lark/
│  ├─ warehouse/
│  └─ douyin/
└─ agent_runtime/             # 后续，不立即实现
```

拆分纪律：

- 保持所有现有测试通过；
- 不为目录漂亮进行大重构；
- 先建立Handler Registry，逐步移动逻辑；
- Cell Runtime只依赖通用Outcome；
- 真实业务逻辑不得再次写入总`services.py`。

---

# 9. 当前阶段的建设顺序

```text
1. Day 1基线（已完成）
   Fake确定性纵向闭环

2. 可靠性原语
   RESULT_UNKNOWN / Reconcile / Job Lease Generation

3. HumanTask原语
   登录、验证码、人工确认、ResumeSignal

4. 企业差异配置
   SourceBinding / MappingProfile / RuleSet / Schema Drift

5. Core与Cell代码边界
   Handler Registry + 有限拆分services.py

6. 第一个真实Connector
   优先云仓寄样履约

7. Browser Service正式集成
   能力握手、Profile、登录恢复

8. 第二个真实Cell
   抖音合作单

9. 知识资产v0
   Decision Ledger / Wiki Candidate / Skill Version / Eval Case

10. Agent Runtime v0
    先做低风险日报/异常分类

11. Ephemeral Agent Worker
    并行、Checkpoint、成本路由

12. Designer Agent
    至少2—3个真实Cell后再建设
```

---

# 10. 当前不可变决策

1. YC以稳定可靠的Cell Runtime为地基；
2. 企业业务差异主要由Solution、Deployment、Mapping、Rule和Workflow Variant承载；
3. Agent、程序、人和Edge是不同执行者；
4. 外部Agent/Harness可替换，但业务状态必须留在YC；
5. HumanTask是一等公民；
6. Browser Service保持本地、持久和低层，不直接暴露给云端；
7. 本体/Wiki/Skill必须由真实运行与结果驱动；
8. 不提前建设通用DAG、图数据库、完整Agent平台和Designer Agent；
9. 新客户应主要通过配置复制，不能复制代码分支；
10. 每个新增能力必须同时通过产品、可靠性、业务价值和创始人负担四道门。
