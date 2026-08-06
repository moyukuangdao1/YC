---
document_id: yc-koc-hub-technical-plan
title: YC KOC合作经营中枢 v0.1｜产品技术方案
version: 2.1
status: implementation_baseline
updated_at: 2026-08-06
owner: YC Founder
audience:
  - Founder
  - Codex
  - Future Engineering Collaborators
supersedes:
  - YC_Cell01_KOC寄样自动下单_产品技术方案_v1.0.md
source_of_truth: true
---

# YC KOC合作经营中枢 v0.1｜产品技术方案 v2.1

> **目标**：在30天内，把现有的云仓发货单脚本、抖音合作单脚本、飞书台账和浏览器持久化能力，升级为一套可演示、可部署、可审计、可恢复、可复制给第二家企业的产品雏形。  
> **实施原则**：先搭可成长的业务与运行框架，再由创始人/Codex逐个接入真实业务动作。  
> **产品策略**：一个“薄业务壳”承接完整KOC合作生命周期；两个“深自动化Cell”先做到生产可用。

---

## 0. 版本说明：v2.1为何替代v2.0与v1.0

v2.0已经将单一自动下单升级为“薄业务壳 + 两个深Cell”。v2.1继续补齐一个人落地企业产品必须具备、但v2.0表达不足的内容：范围优先级、数据同步语义、隐私、Connector契约、运维、环境发布和现实实施节奏。最初v1.0把“寄样自动下单”视为单一流程，以下三个纠正继续有效：

1. **云仓发货单**与**抖音合作订单**是两个不同业务对象、不同外部系统、不同状态机，不能混成一个`SampleOrder`。
2. KOC合作不是一条单线流程，而是达人沟通、商务条件、寄样、合作单、内容、数据投流、财务成本等多条并行轨道。
3. 第一产品既不能只是脚本，也不能一开始建设完整KOC SaaS；应采用“薄业务壳 + 两个深Cell”。

因此v2.1正式定义：

```text
YC KOC合作经营中枢（薄业务壳）
├─ Cell A：云仓寄样履约
└─ Cell B：抖音合作单创建与确认
```

达人自行下单与报销、内容审核、视频数据、投流和财务核算先进入统一业务模型，以人工任务或只读同步存在，后续按真实价值逐步自动化。

## 0.1 v2.1新增的硬性补充

1. **P0/P1/P2范围分层**：30天只承诺P0，避免一个人同时建设完整KOC SaaS。
2. **外部事实与同步语义**：明确飞书、云仓、抖音与YC谁是事实来源，禁止双向静默覆盖。
3. **字段所有权和Schema Drift**：企业改表后通过映射、Dry Run和版本发布处理，不修改核心代码。
4. **敏感数据最小化**：手机号、地址、Cookie、Token和证据截图进入明确的数据分类、加密和保留策略。
5. **Connector SDK**：低级浏览器能力不进入Cloud；平台逻辑封装在Edge Connector。
6. **结果未知与对账**：所有外部写入均必须支持`RESULT_UNKNOWN → reconcile`。
7. **环境、发布和兼容**：Cloud、Edge、Browser、Connector、Cell和配置版本都可追踪。
8. **更现实的实施节奏**：Day 1任务独立为v2.0，技术方案不再要求一天完成全部表和全部可靠性机制。

## 0.2 实施优先级

| 优先级 | 30天要求 | 内容 |
|---|---|---|
| P0 | 必须完成 | 多租户骨架、KOC案例、两深Cell、飞书映射、Edge、异常、证据、Demo、基础看板 |
| P1 | 时间允许/30—90天 | 达人自行下单报销、视频数据同步、日报、基础成本完整度、Wiki候选 |
| P2 | 外部复制后 | 投流候选、完整财务核算、BusinessUnit细粒度权限、自动更新、专家节点 |

P0没有稳定运行前，不得以“未来平台化”为理由提前建设P2。

---

# 1. 产品定义

## 1.1 面向企业的一句话

> **不替换飞书、抖音、云仓和员工现有习惯，把KOC合作中的寄样、合作单、内容、数据和成本连接成可执行、可跟进、可核算的业务闭环。**

## 1.2 第一版能够让采购者看到什么

管理者能够看到：

- 当前有多少KOC正在合作；
- 每个KOC卡在哪一条业务轨道；
- 谁应该做下一步；
- 哪些记录超时或异常；
- 云仓发货单和抖音合作单是否已完成；
- 自动化完成率、人工介入率、节省工时；
- 每次跨平台操作的时间线和证据。

媒介员工继续主要使用飞书和微信，只需：

- 维护少量标准字段；
- 处理YC推送的异常或审批；
- 完成谈判、沟通、内容协作等必须由人承担的工作。

## 1.3 第一版用户与成功定义

### 业务用户

- 媒介专员：继续在飞书和微信工作，处理沟通、审核、异常和人工判断；
- KOC负责人：看整体漏斗、审批、超时、异常和规则；
- 财务：查看标准成本项、报销和核销状态；
- 企业老板/管理者：看处理量、效率、风险和业务结果；
- YC实施人员：管理映射、Connector、版本和运行诊断，所有支持操作受审计。

### 30天产品成功

- 真实内部业务能连续运行；
- 云仓和抖音两个外部副作用具有幂等、验证和恢复；
- 员工不需要启动脚本或理解Agent；
- 管理者在YC看见全局，员工在飞书看见日常状态；
- Demo不依赖真实平台临场可用；
- 第二家企业接入所需差异主要落在配置和Connector，而不是复制系统。

## 1.4 第一版明确不做

- 不做完整达人CRM；
- 不自动操作个人微信；
- 不自动决定和调整投流预算；
- 不建设通用低代码工作流画布；
- 不建设大型本体论平台或图数据库；
- 不建设开放插件市场；
- 不集中复制企业全部原始数据；
- 不允许云端下发任意JavaScript、任意URL或任意浏览器指令；
- 不承诺绕过平台风控。

---

# 2. 产品结构：业务壳、Cell、连接器、知识

YC v0.1采用四个清晰层次，但它们不是四套产品。

```text
业务壳 Business Hub
  └─ 描述KOC合作、人员、状态、任务、成本和结果

自动化Cell
  └─ 把一项明确业务动作可靠做完

连接与执行层
  └─ 飞书、云仓、抖音、Edge、Browser Service

业务记忆层
  └─ 本体、Wiki、Skill、案例和规则版本
```

## 2.1 业务壳：KOC合作经营中枢

根对象是`CooperationCase`，代表“一名KOC围绕某个产品或活动的一次合作”。

一个合作案例拥有多条并行轨道：

| 轨道 | `track_key` | 第一版处理方式 |
|---|---|---|
| 达人发现与沟通 | `DISCOVERY_COMMUNICATION` | 人工为主，YC记录状态与下一步 |
| 商务条件 | `COMMERCIAL_TERMS` | 结构化记录 |
| 寄样与履约 | `SAMPLE_FULFILLMENT` | Cell A深自动化 |
| 抖音合作单 | `DOUYIN_COOP_ORDER` | Cell B深自动化 |
| 内容协作与发布 | `CONTENT_PUBLICATION` | 人工任务、文件和审核状态 |
| 数据与投流 | `DATA_MEDIA_BUYING` | 第一版只预留与人工记录 |
| 成本与财务 | `FINANCE_COST` | 第一版记录基础成本，后续核算 |

## 2.2 深Cell A：云仓寄样履约

目标：

```text
飞书合作记录
→ 校验收件人、商品、数量和寄样规则
→ 创建云仓发货单
→ 查询仓库发货状态
→ 回传物流单号
→ 写回飞书
→ 保存证据和异常
```

## 2.3 深Cell B：抖音合作单创建与确认

目标：

```text
飞书合作码 + KOC ID + 商务条件
→ 校验和重复检查
→ 创建抖音合作单
→ 获取合作单ID
→ 写回飞书
→ 定时查询达人确认状态
→ 超时、拒绝、失效进入异常
```

## 2.4 人工路线：KOC自行店铺下单并报销

第一版不强行自动化，先结构化：

```text
待达人下单
→ 待订单截图
→ 截图已提交
→ 待签收
→ 待报销
→ 已转账
→ 财务已核销
```

保留：

- 截图；
- 平台订单号；
- 购买金额；
- 转账金额；
- 样品成本；
- 财务核销状态；
- 与正常订单的区分标记。

---

# 3. 核心架构决策（ADR摘要）

## ADR-001：云端控制、本地执行

- YC Cloud部署于阿里云；
- YC Edge部署于企业内部长期在线Windows设备；
- Browser Service与Edge同机，只监听本机；
- 客户登录态和浏览器Profile不上传云端；
- Edge只主动连接云端，不要求企业开放入站端口。

## ADR-002：模块化单体，而非微服务

30天内采用：

- 一个FastAPI云端代码库；
- 一个独立Cloud Worker进程；
- 一个YC Edge代码库；
- 一个现有Browser Service；
- PostgreSQL作为业务事实和运行事实来源。

不拆微服务，不上Kubernetes。

## ADR-003：使用显式PostgreSQL耐久状态，不先建设通用工作流引擎

v0.1不引入通用DAG编辑器，也不复制HomeRail。

运行时只实现当前产品必须的原语：

- `CellDefinition`
- `CellDeployment`
- `CellRun`
- `StepRun`
- `Command`
- `Attempt`
- `Approval`
- `Exception`
- `Evidence`
- `ActivityEvent`
- `OutboxEvent`

工作流在代码中定义并版本化。Cloud Worker使用PostgreSQL任务表和
`FOR UPDATE SKIP LOCKED`领取工作，保证进程重启后可以继续。

保留将来迁移到Temporal/DBOS等耐久执行引擎的边界，但现在不承担其学习和运维成本。

## ADR-004：多租户从第一天进入数据模型

- 每家企业是一个`Tenant`；
- 所有企业业务表必须有`tenant_id`；
- 任何查询必须经Tenant Context；
- 唯一约束均按租户作用域定义；
- Edge、Connector、Profile、Cell部署和证据均归属于租户；
- 第一版在应用层强制隔离并建立自动化测试；
- 第二个外部客户前评估PostgreSQL RLS。

## ADR-005：飞书是员工协作面，YC是运行与管理面

- 员工继续在飞书维护标准字段、看日常状态、接收通知；
- YC负责管理看板、审批、异常、证据、配置、运行诊断和审计；
- 微信日报由YC生成文本与图片，员工一键复制发送，不自动操作个人微信。

## ADR-006：云端只能下发受限业务命令

允许的命令由Edge本地注册，例如：

- `WAREHOUSE_CREATE_SHIPMENT`
- `WAREHOUSE_QUERY_SHIPMENT`
- `DOUYIN_CREATE_COOPERATION_ORDER`
- `DOUYIN_QUERY_COOPERATION_ORDER`

禁止云端下发：

- 任意JavaScript；
- 任意URL导航；
- 任意Fetch；
- 任意Cookie读取；
- 任意本地文件路径；
- 任意Shell命令。

## ADR-007：外部系统仍是领域事实来源，YC保存运行事实和必要快照

- 飞书是员工协作字段的主要来源；
- 云仓是发货单、发货状态和物流事实来源；
- 抖音是合作单、达人确认和视频销售事实来源；
- YC保存标准化业务对象、来源引用、输入快照、状态、动作、异常、证据和结果；
- YC写回字段必须标记所有权，不能静默覆盖员工修改；
- 双向字段冲突必须生成`SYNC_CONFLICT`，不采用简单最后写入者覆盖。

## ADR-008：敏感数据最小化与按租户加密

- Cookie、Token和Browser Profile只保留Edge本地；
- 手机号、地址等PII只保存业务所需最小范围；
- Cloud必须保存时使用应用层加密/密钥引用，并在日志和UI中脱敏；
- 证据默认截取关键区域，不长期保留完整HTML和网络响应；
- 每个租户有数据保留、导出和删除策略。

## ADR-009：环境与版本隔离

至少区分：`local`、`staging`、`production`、`demo`。

每次Run固定记录：

- Cloud release；
- Edge version；
- Browser Service version；
- Chromium version；
- Connector version；
- CellDefinition version/hash；
- MappingProfile version；
- RuleSet version。

## ADR-010：金额、比例和时间使用确定性类型

- 金额使用整数分或`NUMERIC`，禁止浮点；
- 佣金率/ROI目标使用基点或`NUMERIC`；
- 数据库存UTC，界面按`Asia/Shanghai`展示；
- 外部时间同时保存来源时区和同步时间；
- 任何经营指标显示数据更新时间、口径和完整度。

---

# 4. 部署拓扑

```text
┌─────────────────────────────────────────────┐
│                  YC Cloud                   │
│                                             │
│ FastAPI API                                 │
│ Cloud Worker                                │
│ PostgreSQL                                  │
│ OSS / 本地开发Artifact Store                │
│ Tenant / RBAC                               │
│ KOC Hub / Cell Runtime                      │
│ Approval / Exception / Evidence / Dashboard │
└──────────────────────┬──────────────────────┘
                       │ HTTPS出站长轮询
                       │ 设备Token + 命令租约
┌──────────────────────▼──────────────────────┐
│             企业内部 YC Edge Node           │
│                                             │
│ Device Registration / Heartbeat             │
│ Durable Local Inbox (SQLite)                │
│ Command Allowlist                           │
│ Profile Registry                            │
│ Connector Runner                            │
│ Evidence Upload Queue                       │
└──────────────────────┬──────────────────────┘
                       │ http://127.0.0.1:9527
                       │ Bearer Token
┌──────────────────────▼──────────────────────┐
│          Existing Browser Service           │
│ Persistent Chromium / Patchright            │
│ Page / DOM / Fetch / Upload / Download       │
│ Screenshot / Network / Storage              │
└──────────────────────┬──────────────────────┘
                       │
             云仓 / 抖音 / 蝉妈妈 / 其他后台
```

## 4.1 云端部署

### 开发环境

Docker Compose：

- `yc-api`
- `yc-worker`
- `postgres`
- `minio`（可选，用于开发证据文件）
- `frontend`（第二周开始）

### 试点环境

阿里云：

- 1台ECS运行API、Worker和反向代理；
- RDS PostgreSQL；
- OSS；
- HTTPS；
- 基础日志和备份。

## 4.2 Edge部署

### 试点

- 企业内部一台稳定Windows电脑；
- Windows任务计划程序自动启动；
- 禁止自动休眠；
- 专用Windows账号；
- Browser Profile使用专用目录。

### 正式运行

- 专用迷你主机或企业服务器；
- Windows Service；
- 托盘诊断工具；
- 自动更新、版本回滚和多Profile后续增加。

## 4.3 Browser Service生产约束

- 只监听`127.0.0.1`；
- Bearer Token强制开启；
- 每个企业账号使用独立`profile_id/user_data_dir`；
- 不直接暴露到企业局域网或公网；
- Browser Service保持单一职责，不加入云端调度、业务状态、本体或Wiki逻辑。

---

# 5. 多租户与权限设计

## 5.1 账号与租户模型

```text
User
  └─ TenantMembership
       ├─ MembershipRole（Tenant或BusinessUnit作用域）
       └─ Tenant
            └─ BusinessUnit（品牌/店铺/部门，可选）
```

### 全局对象

- `users`
- `tenants`
- `business_units`（P0可仅建表和可选外键）
- `roles`
- `permissions`

### 关系对象

- `tenant_memberships`
- `membership_roles`
- `role_permissions`

## 5.2 MVP角色

| 角色 | 用途 |
|---|---|
| `TENANT_OWNER` | 企业所有者，查看全部业务与配置 |
| `TENANT_ADMIN` | 企业管理员、成员和Edge管理 |
| `KOC_MANAGER` | KOC负责人，审批、看板、规则确认 |
| `KOC_OPERATOR` | 媒介专员，维护案例和处理异常 |
| `FINANCE_VIEWER` | 查看成本、报销和核算结果 |
| `AUDITOR` | 只读运行证据和审计 |
| `YC_OPERATOR` | YC实施支持，必须记录支持操作 |
| `EDGE_DEVICE` | 设备服务身份，只能领取本租户命令 |

## 5.3 MVP权限

```text
tenant.member.read
tenant.member.manage
koc.case.read
koc.case.write
koc.case.assign
koc.cost.read
koc.cost.write
cell.deployment.read
cell.deployment.manage
cell.run.read
cell.run.start
cell.run.approve
cell.run.retry
exception.read
exception.resolve
evidence.read
edge.read
edge.manage
config.read
config.manage
audit.read
```

## 5.4 Tenant Context规则

每个业务请求必须经过：

1. JWT解析用户；
2. 获取`X-Tenant-ID`或令牌中的当前租户；
3. 校验有效Membership；
4. 加载角色与权限；
5. 构造`TenantContext`；
6. Repository所有查询强制使用`tenant_id`。

禁止在Router中直接使用裸Session查询租户业务表。

## 5.5 BusinessUnit扩展口

`Tenant`是合同、数据和设备隔离边界；`BusinessUnit`用于同一企业内的品牌、店铺或部门。P0允许角色主要作用于Tenant，但业务表预留可选`business_unit_id`，避免第二家大客户出现品牌隔离时大规模迁移。

## 5.6 隔离验收

必须有自动化测试证明：

- 租户A不能读取租户B的KOC案例；
- 租户A不能领取租户B的Edge命令；
- 同一外部记录ID可在不同租户重复存在；
- 租户级唯一约束不会跨企业冲突；
- `YC_OPERATOR`访问客户数据会写入审计日志。

---

# 6. 代码仓库结构

建议采用Monorepo，但现有Browser Service保持独立服务，不在明天重构。

```text
yc/
├─ apps/
│  ├─ cloud_api/
│  │  └─ app/
│  │     ├─ main.py
│  │     ├─ api/v1/
│  │     ├─ identity/
│  │     ├─ tenancy/
│  │     ├─ koc_hub/
│  │     ├─ cell_runtime/
│  │     ├─ integrations/
│  │     ├─ edge_control/
│  │     ├─ approvals/
│  │     ├─ exceptions/
│  │     ├─ evidence/
│  │     ├─ audit/
│  │     ├─ db/
│  │     └─ core/
│  ├─ cloud_worker/
│  │  └─ worker/
│  │     ├─ main.py
│  │     ├─ job_claim.py
│  │     ├─ orchestrator.py
│  │     └─ handlers/
│  └─ edge_agent/
│     └─ edge/
│        ├─ main.py
│        ├─ device.py
│        ├─ command_inbox.py
│        ├─ policy.py
│        ├─ browser_client.py
│        ├─ profile_registry.py
│        ├─ connectors/
│        ├─ evidence.py
│        └─ local_db.py
├─ packages/
│  ├─ contracts/
│  │  └─ yc_contracts/
│  └─ domain_common/
├─ frontend/
│  └─ web/                 # 第二周：Vue 3 + TypeScript + Element Plus
├─ infra/
│  ├─ docker-compose.yml
│  ├─ migrations/
│  └─ scripts/
├─ tests/
│  ├─ unit/
│  ├─ integration/
│  └─ e2e/
└─ docs/
   ├─ architecture/
   ├─ decisions/
   └─ product/
```

## 6.1 推荐技术栈

### Cloud

- Python 3.11+
- FastAPI
- Pydantic v2
- SQLAlchemy 2.0（同步模式，减少复杂度）
- psycopg 3
- Alembic
- PostgreSQL 16
- httpx
- structlog
- PyJWT
- Argon2密码哈希
- pytest

### Edge

- Python 3.11+
- FastAPI或轻量后台进程
- httpx
- SQLite
- tenacity
- Windows Task Scheduler（试点）
- 现有Browser Service HTTP Client

### Frontend

- Vue 3
- TypeScript
- Vite
- Element Plus

前端不是明日Day 1的主任务；先保证API、状态和契约正确。

---

# 7. KOC业务领域模型（本体v0）

## 7.1 核心对象

| 对象 | 说明 |
|---|---|
| `Tenant` | 一家企业 |
| `BusinessUnit` | 品牌、部门或店铺，第一版可选 |
| `User` | 登录YC的用户 |
| `Person` | 员工、联系人、负责人 |
| `Creator` | KOC达人 |
| `Product` | 产品 |
| `SKU` | 具体SKU |
| `CooperationCase` | 一次KOC合作根对象 |
| `CaseTrack` | 合作案例的一条业务轨道 |
| `BusinessTask` | 人工或系统下一动作 |
| `CommercialTerm` | 固定费用、佣金、内容要求等 |
| `SampleFulfillment` | 寄样履约对象 |
| `WarehouseShipment` | 云仓发货单 |
| `SelfPurchaseReimbursement` | 达人自行购买与报销 |
| `DouyinCooperationOrder` | 抖音合作单 |
| `ContentItem` | 视频、脚本、审核稿 |
| `PerformanceSnapshot` | 播放、销售、ROI等数据快照 |
| `CostItem` | 拍摄费、样品、物流、佣金、投流等 |
| `Evidence` | 截图、文件、外部结果 |
| `ExceptionCase` | 业务或系统异常 |

## 7.2 关键关系

```text
Person 负责 CooperationCase
Creator 参与 CooperationCase
CooperationCase 关联 Product/SKU
CooperationCase 拥有 CaseTrack
CooperationCase 产生 SampleFulfillment
SampleFulfillment 产生 WarehouseShipment
CooperationCase 产生 DouyinCooperationOrder
CooperationCase 产生 ContentItem
CooperationCase 产生 PerformanceSnapshot
CooperationCase 归集 CostItem
CellRun 作用于 CooperationCase
Action 产生 Evidence 与 Outcome
```

## 7.3 多轨道状态

`case_tracks`采用扩展表，而不是在`cooperation_cases`中堆七个固定列：

```text
case_id
track_key
status
owner_person_id
next_action
due_at
last_changed_at
version
```

唯一约束：

```text
unique(tenant_id, case_id, track_key)
```

---

# 8. 配置与连接器模型

## 8.1 ConnectorInstance

表示一个租户已经配置的外部系统连接。

字段：

```text
id
tenant_id
connector_type
display_name
execution_location      # CLOUD / EDGE / HUMAN
status                  # ACTIVE / PAUSED / ERROR
secret_ref
config_json
version
created_at
updated_at
```

首批`connector_type`：

- `LARK_BITABLE`
- `CLOUD_WAREHOUSE`
- `DOUYIN_ECOMMERCE`
- `BROWSER_PROFILE`

## 8.2 MappingProfile

解决飞书表结构变化和第二家企业适配：

```text
id
tenant_id
connector_instance_id
name
source_resource_id
source_schema_fingerprint
field_mapping_json
transform_rules_json
version
status
created_at
```

YC内部标准字段与企业列名解耦。

## 8.3 Schema Drift

每次同步计算表结构指纹。如果发生：

- 必填字段删除；
- 字段类型变化；
- 映射失效；
- 表被替换；

处理：

```text
暂停受影响的CellDeployment
→ 创建SCHEMA_DRIFT异常
→ 生成候选映射
→ 人工确认
→ 使用历史样本Dry Run
→ 发布MappingProfile新版本
→ 新Run使用新版本
```

不允许AI未经确认自动修改高风险字段映射。

## 8.4 CellDefinition与CellDeployment

### CellDefinition

平台级、不可变版本：

```text
cell_key
version
input_schema
output_schema
step_definition
command_types
risk_level
```

### CellDeployment

租户实际部署实例：

```text
tenant_id
business_unit_id
cell_definition_id
name
connector_bindings
mapping_profile_ids
rule_set_id
execution_policy
status
```

## 8.5 字段所有权与同步方向

每个映射字段必须指定：

| 类型 | 说明 |
|---|---|
| `EMPLOYEE_OWNED` | 员工/飞书是事实来源，YC只读 |
| `YC_OWNED` | YC计算或外部回写，员工不应覆盖 |
| `BIDIRECTIONAL` | 双向可改，必须有版本和冲突检测 |
| `DERIVED` | YC派生指标，不作为业务输入 |

同步时保存：

```text
source_record_id
source_revision/source_updated_at
last_synced_at
source_snapshot_hash
mapping_profile_version
```

若源记录在Run启动后发生关键字段变化，写入前必须重新校验；高风险动作不得使用过期快照。

## 8.6 同步触发

优先级：

1. 官方Webhook；
2. 官方API增量轮询；
3. 定时浏览器/接口查询；
4. 人工触发。

任何轮询均应带租户限流、游标和最后成功同步时间。不得每次全表扫描。

## 8.7 同步冲突

冲突处理：

```text
检测到关键字段版本变化
→ 暂停写回或外部执行
→ 创建SYNC_CONFLICT异常
→ 展示源值、YC值和差异
→ 负责人选择保留/合并
→ 生成新快照后继续
```

---

# 9. Cell Runtime

## 9.1 核心实体

| 实体 | 用途 |
|---|---|
| `CellRun` | 一次Cell业务运行 |
| `StepRun` | 运行中的一个确定步骤 |
| `Command` | 已持久化、等待Edge或云Worker执行的命令 |
| `Attempt` | 一次物理执行尝试 |
| `Approval` | 人工决策 |
| `ExceptionCase` | 可处理异常 |
| `Evidence` | 运行证据 |
| `ActivityEvent` | 只追加事实流 |
| `Job` | Cloud Worker待处理任务 |
| `OutboxEvent` | 事务后通知、同步和命令投递 |

## 9.2 运行原则

- 先写数据库，再执行外部副作用；
- Worker自称成功不等于业务成功；
- 外部结果必须二次验证；
- 每个副作用必须有幂等键；
- 重试不能盲目重复创建订单；
- 旧Attempt或过期租约回报必须被拒绝；
- 人工修复后只重跑必要下游；
- 每个Run固定使用Cell、映射、规则和Connector版本。

## 9.3 通用Run状态

```text
CREATED
→ VALIDATING
→ WAITING_APPROVAL
→ READY
→ DISPATCHED
→ EXECUTING
→ VERIFYING
→ WRITING_BACK
→ COMPLETED
```

异常状态：

```text
VALIDATION_FAILED
PAUSED
RESULT_UNKNOWN
WRITEBACK_FAILED
MANUAL_REQUIRED
FAILED
CANCELLED
```

## 9.4 Step状态

```text
PENDING
READY
RUNNING
SUCCEEDED
FAILED_RETRYABLE
FAILED_TERMINAL
WAITING_EXTERNAL
WAITING_HUMAN
SKIPPED
CANCELLED
```

## 9.5 Job领取

Cloud Worker循环：

```sql
select *
from jobs
where status = 'PENDING'
  and available_at <= now()
order by priority desc, created_at
for update skip locked
limit 1;
```

领取后写入租约：

```text
status = RUNNING
lease_owner
lease_expires_at
attempt_count += 1
```

租约过期可重新领取。

---

# 10. Edge Agent与Browser Service集成

## 10.1 Edge职责

- 设备注册与心跳；
- 领取本租户业务命令；
- 本地SQLite保存已领取命令和上传队列；
- 校验命令类型、版本、租户和风险策略；
- 调用本机Browser Service；
- 执行后验证外部结果；
- 上传证据；
- 断网恢复；
- 登录态失效告警；
- 一键暂停。

## 10.2 Edge命令格式

```json
{
  "command_id": "cmd_01",
  "tenant_id": "tenant_01",
  "edge_node_id": "edge_01",
  "cell_run_id": "run_01",
  "cell_key": "warehouse_sample_fulfillment",
  "cell_version": "1.0.0",
  "command_type": "WAREHOUSE_CREATE_SHIPMENT",
  "generation": 1,
  "attempt": 1,
  "idempotency_key": "tenant_01:warehouse:...",
  "profile_id": "warehouse_profile_01",
  "risk_level": "L2",
  "deadline": "2026-08-07T18:00:00+08:00",
  "payload": {},
  "evidence_policy": {
    "screenshot_before": false,
    "screenshot_after": true,
    "save_response_summary": true
  }
}
```

## 10.3 Edge回报

```json
{
  "command_id": "cmd_01",
  "generation": 1,
  "attempt": 1,
  "status": "SUCCEEDED",
  "external_ref": {
    "type": "WAREHOUSE_SHIPMENT",
    "id": "shipment_123"
  },
  "result": {},
  "evidence_manifest": [],
  "completed_at": "2026-08-07T10:20:00+08:00"
}
```

## 10.4 Browser Service调用边界

Edge通过本机Client调用：

```text
health
page/profile preparation
registered connector workflow
screenshot/evidence
```

业务Connector内部可以调用Browser Service现有页面、DOM、Fetch、上传和下载能力，但这些低级能力不暴露给YC Cloud。

## 10.5 Profile管理

```text
profile_id
tenant_id
platform
account_label
user_data_dir
status
last_health_at
last_login_verified_at
browser_version
```

一个账号一个Profile目录；同一Profile同一时间只允许一个写入型命令持有锁。

## 10.6 Edge Connector SDK

平台业务逻辑必须封装在Edge Connector，而不是散落在Cell编排或Browser Service中：

```python
class EdgeConnector(Protocol):
    connector_key: str
    version: str
    supported_commands: set[str]

    def preflight(self, ctx, payload): ...
    def execute(self, ctx, command): ...
    def verify(self, ctx, execution_result): ...
    def reconcile(self, ctx, external_hint): ...
    def health_check(self, ctx): ...
```

Connector内部可以调用Browser Service的页面、DOM、Fetch、上传和截图，但Cloud只看业务命令和结构化结果。

## 10.7 稳定错误分类

| 错误码 | 含义 | 默认处理 |
|---|---|---|
| `BUSINESS_VALIDATION_FAILED` | 缺字段、业务规则不满足 | 人工修复 |
| `AUTH_REQUIRED` | 登录失效、验证码 | 暂停并通知 |
| `PERMISSION_DENIED` | 平台账号无权限 | 人工处理 |
| `SCHEMA_DRIFT` | 飞书/来源结构变化 | 暂停映射 |
| `CONNECTOR_CHANGED` | 页面/API变化 | 暂停Connector |
| `RATE_LIMITED` | 平台限流 | 延迟重试 |
| `NETWORK_ERROR` | 网络和超时 | 指数退避 |
| `BUSINESS_REJECTED` | 外部平台业务拒绝 | 人工处理 |
| `RESULT_UNKNOWN` | 可能已成功但未确认 | 必须对账 |
| `WRITEBACK_FAILED` | 外部成功、飞书写回失败 | 只重试写回 |
| `INCOMPATIBLE_RUNTIME` | Edge/Connector版本不兼容 | 升级或回滚 |

## 10.8 结果未知与对账

所有写入型Connector必须实现`reconcile`：

```text
执行超时/崩溃
→ RESULT_UNKNOWN
→ 使用幂等字段、外部ID、手机号/商品/时间窗口查询
→ 找到唯一外部记录：接受成功
→ 明确不存在：允许安全重试
→ 多条或仍不确定：MANUAL_REQUIRED
```

---

# 11. 深Cell A：云仓寄样履约

## 11.1 输入契约

```json
{
  "cooperation_case_id": "case_01",
  "source_record_id": "lark_record_01",
  "creator_id": "creator_01",
  "recipient": {
    "name": "张三",
    "phone": "13800000000",
    "address": "浙江省杭州市..."
  },
  "items": [
    {
      "product_id": "product_01",
      "sku_id": "sku_01",
      "quantity": 1
    }
  ],
  "owner_person_id": "person_01",
  "remark": "KOC寄样"
}
```

## 11.2 步骤

```text
read_source_snapshot
normalize_input
validate_required_fields
resolve_sku_mapping
check_recent_sample_history
evaluate_risk
wait_approval_if_needed
create_edge_command
execute_warehouse_create
verify_external_shipment
persist_external_ref
writeback_lark_shipment
schedule_shipment_poll
poll_shipment_status
writeback_tracking_number
complete_track
```

## 11.3 幂等键

```text
hash(
 tenant_id
 + cooperation_case_id
 + recipient_phone
 + normalized_address
 + sorted(sku_id, quantity)
 + sample_round
)
```

## 11.4 风险

自动阻断：

- 缺少手机号或地址；
- SKU映射失败；
- 同一合作案例已有有效发货单；
- 最近周期疑似重复寄样；
- 数量超过硬限制。

需要审批：

- 高价值样品；
- 多件寄样；
- 特殊地址；
- 同一达人近期已寄样但业务人员要求再次寄样。

---

# 12. 深Cell B：抖音合作单创建与确认

## 12.1 输入契约

```json
{
  "cooperation_case_id": "case_01",
  "creator_id": "creator_01",
  "douyin_creator_id": "dy_creator_01",
  "cooperation_code": "code_xxx",
  "product_ids": ["product_01"],
  "commercial_terms": {
    "fixed_fee": 500,
    "commission_rate": 20,
    "content_due_at": "2026-08-20"
  },
  "owner_person_id": "person_01"
}
```

## 12.2 步骤

```text
read_source_snapshot
normalize_input
validate_creator_and_code
check_existing_cooperation_order
evaluate_risk
wait_approval_if_needed
create_edge_command
execute_douyin_create
verify_external_order
persist_external_ref
writeback_lark
schedule_confirmation_poll
poll_creator_confirmation
notify_timeout_or_rejection
complete_track
```

## 12.3 幂等键

```text
hash(
 tenant_id
 + cooperation_case_id
 + douyin_creator_id
 + cooperation_code
 + sorted(product_ids)
)
```

## 12.4 第一版明确不做

- 自动修改佣金；
- 自动提交大额固定费用；
- 自动追加投流；
- 自动处理达人个人微信沟通；
- 未经审批自动覆盖已有合作单。

---

# 13. 飞书员工前台与YC管理前台

## 13.1 飞书保留的字段

建议员工日常看到：

- 达人；
- 产品；
- 合作负责人；
- 沟通状态；
- 商务条件；
- 寄样方式；
- 寄样状态；
- 云仓发货单号；
- 快递单号；
- 抖音合作单号；
- 达人确认状态；
- 内容状态；
- 视频链接；
- 下一步动作；
- 截止时间；
- 异常原因；
- 最后同步时间。

## 13.2 YC P0页面（30天）

1. **KOC合作总览**：漏斗、待办、异常和关键指标；
2. **合作案例详情**：七轨道、责任人、下一步和时间线；
3. **审批与异常中心**：同一入口处理需要人的事项；
4. **Cell运行详情与证据**：Run、Step、Command、外部ID和截图；
5. **Edge与账号健康**：节点、Profile、登录态和Connector；
6. **配置与字段映射**：管理员使用。

租户/成员管理可以先采用基础管理页或后台；独立经营看板在总览中完成。P1再拆分高级报表和财务视图。

## 13.3 看板指标

- 合作案例总数及阶段漏斗；
- 寄样待处理、已发货、已签收；
- 合作单待创建、待确认、已确认；
- 超时和异常；
- 自动完成率；
- 人工介入率；
- 每日/每周节省工时；
- 数据最后更新时间；
- 基础KOC成本完整度。

---

# 14. 变更管理

## 14.1 变更申请

先用飞书表单：

- 变更内容；
- 原因；
- 生效时间；
- 修改前样例；
- 修改后样例；
- 是否影响金额、订单、投流等高风险动作。

## 14.2 变更分类

| 类型 | 示例 | 技术动作 |
|---|---|---|
| 配置变更 | 字段改名、商品映射新增 | 发布MappingProfile/Config新版本 |
| 规则变更 | 重复寄样周期30天改45天 | 发布RuleSet新版本 |
| 流程变更 | 增加财务审批 | 发布CellDefinition新版本 |
| Connector变更 | 平台API或页面变化 | 发布Connector版本 |

## 14.3 版本规则

- 旧Run继续使用启动时固定的旧版本；
- 新Run使用已激活的新版本；
- 任何配置发布前必须Dry Run；
- 高风险映射必须人工确认；
- 可回滚到上一有效版本。

---

# 15. 安全与审计

## 15.1 必须满足

- 所有云端通信使用HTTPS；
- Edge使用设备身份和可轮换Token；
- Browser Service仅本机可访问；
- 浏览器Profile不上传；
- 日志对Cookie、Token、手机号、地址进行脱敏；
- 高风险操作支持审批与暂停；
- 任何副作用保留证据；
- 所有支持人员访问写入审计事件；
- 删除、支付、价格和投流预算不进入首版自动化。

## 15.2 数据分类与保留

| 类别 | 示例 | 默认策略 |
|---|---|---|
| C0公开/非敏感 | 产品公开信息 | 可正常保存 |
| C1内部业务 | 合作状态、任务 | 租户隔离 |
| C2个人信息 | 手机、地址、微信号 | 加密、脱敏、最小化 |
| C3凭证 | Token、Cookie、Profile | Edge本地，不上传 |
| C4高风险经营 | 预算、支付、价格 | 首版不自动执行 |

必须明确：

- PII保存期限；
- Evidence保存期限；
- 客户终止后的导出和删除；
- OSS前缀按租户隔离；
- 备份中数据删除的现实延迟；
- 任何调试包默认脱敏。

## 15.3 Secret和设备身份

- Cloud Secret只保存引用，不把明文写入CellDefinition或日志；
- Edge注册使用一次性注册码，换取可轮换设备凭证；
- 设备凭证绑定tenant、edge_node和允许能力；
- Token泄露时可以吊销；
- Browser Service本地Token由Edge安全生成和保存。

## 15.4 审计事件

至少记录：

```text
USER_LOGIN
TENANT_SWITCH
MEMBER_ROLE_CHANGED
CELL_DEPLOYMENT_CHANGED
RUN_STARTED
APPROVAL_DECIDED
EDGE_COMMAND_CLAIMED
EXTERNAL_ACTION_EXECUTED
EXCEPTION_RESOLVED
MAPPING_PROFILE_PUBLISHED
SUPPORT_ACCESS
```

---

# 16. 数据库表清单

## 16.1 身份与多租户

- `users`
- `tenants`
- `business_units`
- `tenant_memberships`
- `roles`
- `permissions`
- `membership_roles`
- `role_permissions`

## 16.2 KOC业务壳

- `people`
- `creators`
- `products`
- `skus`
- `cooperation_cases`
- `cooperation_case_products`
- `case_tracks`
- `business_tasks`
- `commercial_terms`
- `sample_fulfillments`
- `warehouse_shipments`
- `self_purchase_reimbursements`
- `douyin_cooperation_orders`
- `content_items`
- `performance_snapshots`
- `cost_items`
- `external_references`

## 16.3 Cell Runtime

- `cell_definitions`
- `cell_deployments`
- `cell_runs`
- `step_runs`
- `jobs`
- `commands`
- `attempts`
- `approvals`
- `exceptions`
- `evidences`
- `activity_events`
- `outbox_events`

## 16.4 连接与设备

- `connector_instances`
- `mapping_profiles`
- `sync_states`
- `sync_conflicts`
- `rule_sets`
- `edge_nodes`
- `edge_profiles`
- `device_tokens`
- `connector_health_snapshots`

## 16.5 约束原则

- 租户业务表均包含`tenant_id NOT NULL`；
- 外键尽量同时校验租户作用域；
- 所有业务唯一键以`tenant_id`开头；
- `cell_definitions`是平台级不可变定义；
- `cell_deployments`是租户级配置；
- 证据对象保留摘要、哈希、URI和来源；
- 重要状态更新使用乐观锁`version`。

---

# 17. API边界

## 17.1 Auth / Tenant

```text
POST /api/v1/auth/login
POST /api/v1/auth/refresh
GET  /api/v1/me
GET  /api/v1/tenants
POST /api/v1/tenants
GET  /api/v1/tenants/{tenant_id}/members
POST /api/v1/tenants/{tenant_id}/members
PUT  /api/v1/tenants/{tenant_id}/members/{membership_id}/roles
```

## 17.2 KOC Hub

```text
GET  /api/v1/koc/cases
POST /api/v1/koc/cases
GET  /api/v1/koc/cases/{case_id}
PATCH /api/v1/koc/cases/{case_id}
GET  /api/v1/koc/cases/{case_id}/timeline
POST /api/v1/koc/cases/{case_id}/tasks
PATCH /api/v1/koc/tracks/{track_id}
```

## 17.3 Cell Runtime

```text
GET  /api/v1/cells/definitions
POST /api/v1/cells/deployments
GET  /api/v1/cells/deployments
POST /api/v1/cells/deployments/{id}/runs
GET  /api/v1/cell-runs
GET  /api/v1/cell-runs/{run_id}
POST /api/v1/cell-runs/{run_id}/retry
POST /api/v1/cell-runs/{run_id}/cancel
```

## 17.4 Approval / Exception / Evidence

```text
GET  /api/v1/approvals
POST /api/v1/approvals/{id}/approve
POST /api/v1/approvals/{id}/reject
GET  /api/v1/exceptions
POST /api/v1/exceptions/{id}/resolve
GET  /api/v1/evidences/{id}
```

## 17.5 Edge

```text
POST /api/v1/edge/register
POST /api/v1/edge/heartbeat
POST /api/v1/edge/commands/claim
POST /api/v1/edge/commands/{id}/ack
POST /api/v1/edge/commands/{id}/report
POST /api/v1/edge/evidence/upload
```

---

# 18. 运维、发布与可观测性

## 18.1 环境

- `local`：开发与Fake Connector；
- `staging`：脱敏数据、真实Edge或沙箱账号；
- `production`：真实业务；
- `demo`：确定性销售演示。

各环境数据库、OSS、Token、域名和Browser Profile必须隔离。

## 18.2 发布

- Cloud release不可变；
- 数据库迁移有向前兼容窗口和回滚方案；
- CellDefinition不可变；
- Connector先在staging和单节点灰度；
- 旧Run固定旧版本并允许完成；
- Edge心跳上报版本兼容；
- 不兼容时拒绝执行而非勉强运行。

## 18.3 备份与恢复（试点最低标准）

- PostgreSQL每日备份；
- OSS版本/生命周期；
- 每月恢复演练；
- 目标RPO 24小时、RTO 8小时（试点），后续按客户SLA调整；
- Edge本地Inbox和Profile分别处理：Inbox可恢复，Profile由客户重新登录而不是上传云端备份。

## 18.4 关键运行指标

- Run/Step成功率和P95耗时；
- Job等待时间；
- Edge在线率；
- 登录态失效；
- Connector错误分类；
- `RESULT_UNKNOWN`数量；
- 自动重试与人工恢复率；
- 飞书同步延迟；
- Evidence上传失败；
- 每个租户处理量与人工介入率。

## 18.5 告警与严重度

- P0：可能重复发货、越租户、凭证泄露，立即停止相关Cell；
- P1：生产Connector大面积失败、结果未知积压；
- P2：单租户登录失效、写回延迟；
- P3：报表和非关键展示问题。

# 19. 测试策略

## 19.1 测试层次

1. Domain unit tests：状态机、权限、幂等键、风险规则；
2. Repository isolation tests：跨租户不可访问；
3. Runtime integration tests：Job、Command、Attempt、Outbox和恢复；
4. Connector contract tests：统一输入、结果、错误与对账；
5. Recorded fixture tests：使用脱敏响应和页面快照；
6. Staging E2E：真实测试账号；
7. Demo E2E：每次发布前一键演示。

## 19.2 必测故障

- Cloud Worker在外部执行前后崩溃；
- Edge离线和重连；
- 外部成功但回写失败；
- 命令重复投递；
- 迟到结果；
- 登录失效；
- 飞书字段变更；
- Connector版本不兼容；
- Evidence上传失败；
- 多租户越权尝试。

---

# 20. Demo Mode

第一交付物必须内建Demo Mode，不能依赖真实平台随时可用。

## 20.1 Demo租户

- 独立`demo_tenant`；
- 脱敏KOC、商品和飞书记录；
- Fake Lark Connector；
- Fake Warehouse Connector；
- Fake Douyin Connector；
- 成功、缺字段、重复订单、登录失效、回写失败等固定场景。

## 20.2 Demo能力

- 一键重置；
- 一键触发成功流程；
- 一键触发异常流程；
- 运行时间线回放；
- 展示审批；
- 展示证据；
- 展示看板变化。

Demo与生产代码共用同一Cell Runtime和领域模型，只替换Connector。

---

# 21. 30天实施路径

## Week 1：纵向骨架、Fake流程和飞书映射

- Day 1按独立任务书完成多租户、KOC案例、Fake Cell和Fake Edge；
- 补齐Attempt、Exception、Outbox、结果未知和恢复；
- 建立MappingProfile、字段所有权和Schema Fingerprint；
- 创建正式Edge骨架并连接Browser Service健康接口；
- 完成确定性Demo Mode和最小页面。

Week 1退出条件：无需真实平台即可完整演示成功与异常流程；飞书脱敏样本可以映射成KOC案例。

## Week 2：优先接入一个真实深Cell

优先选择更稳定、可对账、能快速创造价值的一条（建议云仓）：

- 完成Connector preflight/execute/verify/reconcile；
- 幂等和外部唯一ID；
- 飞书写回；
- 真实内部小批量试运行；
- 登录失效和平台变化处理；
- 基础证据和指标。

Week 2不同时要求两个真实Connector全部上线。

## Week 3：第二个深Cell + KOC薄业务壳

- 接入抖音合作单；
- 完善七轨道和责任人；
- 审批/异常中心；
- 定时查询达人确认；
- 配置发布和回滚；
- 内部真实业务并行运行。

## Week 4：产品化、连续运行与采购演示

- KOC总览、案例详情、运行时间线、Edge健康和配置；
- 工时、成功率、人工介入和异常指标；
- 5分钟Demo、90秒视频、一页纸和试点方案；
- 至少完成一次故障演练；
- 内部连续运行并向3位品牌老板演示。

## 30天范围纪律

如果时间不足，优先级如下：

1. 不重复发货/建单；
2. 外部成功可对账和恢复；
3. 员工飞书可用；
4. Demo稳定；
5. 管理看板；
6. Wiki候选和高级成本。

# 22. 验收标准

## 22.1 架构

- 多租户隔离测试通过；
- 角色权限可配置；
- Cloud API、Worker、Edge和Browser Service职责清晰；
- Browser Service不暴露公网；
- 真实Connector可以替换Fake Connector而不改运行时。

## 22.2 可靠性

- 同一命令重复投递不会重复创建外部订单；
- 外部创建成功、飞书写回失败可以恢复；
- Worker或Edge重启后状态不丢；
- 过期Attempt结果被拒绝；
- 登录态失效后暂停并可恢复；
- 每个外部副作用都有证据。

## 22.3 产品

- 一条合作案例可展示多轨道状态；
- 员工可继续从飞书完成日常工作；
- 管理者在YC看到全局、异常和结果；
- 成功和异常流程均能一键Demo；
- 采购者五分钟能准确复述产品价值。

## 22.4 商业

- 内部真实运行至少四周；
- 节省工时可量化；
- 你的每周维护时间逐步下降；
- 至少向3位品牌老板演示；
- 至少1位进入付费试点讨论。

---

# 23. Codex实现纪律

1. **先保证租户隔离，再增加功能。**
2. **先建立业务对象和运行事实，再接真实平台。**
3. **先做Fake Connector端到端，再替换真实Connector。**
4. **幂等、对账、证据优先于自动执行速度。**
5. **不写通用DAG编辑器。**
6. **不把低级浏览器能力暴露给云端。**
7. **不硬编码飞书列名、企业ID和平台账号。**
8. **所有高风险动作必须有明确风险等级。**
9. **每次提交必须包含测试或可重复验证步骤。**
10. **发现业务语义不清时停止猜测，创建`OPEN_QUESTION`文档。**
11. **不要重构现有Browser Service；通过HTTP Client适配。**
12. **只抽象已被两个Cell共同需要的能力。**

---

# 24. Day 1实施入口

明日实现以独立文档为准：

> `YC_Codex_明日启动任务书_Day1_v2.0.md`

Day 1只要求完成：

```text
两个租户隔离
→ 一个KOC合作案例及七轨道
→ 一个代码定义的Fake Cell
→ 一个持久化Command
→ Fake Edge领取并回报
→ Evidence/Activity时间线
→ Run完成
```

Day 1不接真实飞书、云仓和抖音，不建立全部业务表，不做正式前端。

# 25. 开放问题

实现中不得猜测的问题维护于：

> `YC_关键开放问题与决策门_v1.0.md`

尤其包括：飞书数据源类型、CooperationCase唯一身份、字段所有权、云仓/抖音真实接口、PII存储边界、审批规则、成本口径和实践企业知识产权。
