---
document_id: yc-codex-day1-brief
version: 2.0
date: 2026-08-07
status: ready
parent_document: YC_KOC合作经营中枢_产品技术方案_v2.1.md
supersedes: YC_Codex_明日启动任务书_Day1_v1.0.md
---

# YC Codex明日启动任务书｜Day 1 v2.0

> **Day 1唯一目标**：完成一个最小但正确的纵向切片——两个租户隔离、一个KOC案例、一个代码定义的Fake Cell、一个持久化命令、一个Fake Edge结果，最终形成Evidence和Activity时间线。

> **纠偏**：Day 1不要求一次建立全部KOC表、完整RBAC、审批、异常、Outbox和真实Connector。先证明骨架能够从请求跑到完成，并且不会跨租户。

## 1. 开工前30分钟

Codex先：

1. 阅读本任务书、产品技术方案、Browser Service README；
2. 检查现有仓库、Python版本、包管理和数据库；
3. 输出`docs/OPEN_QUESTIONS.md`；
4. 输出`docs/architecture/DAY1_PLAN.md`，列出准备修改的文件；
5. 不得自行重构Browser Service。

## 2. Day 1硬性决策

- FastAPI模块化单体；
- PostgreSQL 16；
- SQLAlchemy 2同步模式；
- Alembic；
- 独立Cloud Worker；
- PostgreSQL Job/Command持久化；
- 多租户从第一张业务表开始；
- Day 1使用Fake Edge和Fake Connector；
- 不做正式前端；
- 不引入Celery、Temporal、DBOS、Kafka、Redis或通用DAG；
- 不接真实飞书、云仓和抖音。

## 3. Day 1最小代码结构

```text
yc/
├─ apps/
│  ├─ cloud_api/app/
│  └─ cloud_worker/worker/
├─ packages/contracts/yc_contracts/
├─ infra/docker-compose.yml
├─ tests/
├─ scripts/demo_day1.py
└─ docs/
```

`edge_agent`目录可以创建README和协议占位，但Day 1不实现正式Edge进程。

## 4. Day 1表：只实现P0

### 4.1 身份和租户

- `users`
- `tenants`
- `tenant_memberships`
- `roles`
- `permissions`
- `membership_roles`
- `role_permissions`

### 4.2 最小KOC业务壳

- `creators`
- `cooperation_cases`
- `case_tracks`

`CooperationCase`创建时自动生成七条Track。

### 4.3 最小Cell Runtime

- `cell_definitions`
- `cell_deployments`
- `cell_runs`
- `step_runs`
- `jobs`
- `commands`
- `evidences`
- `activity_events`
- `audit_events`
- `edge_nodes`

Day 2再增加：`attempts`、`approvals`、`exceptions`、`outbox_events`、完整业务对象。

## 5. 租户上下文

请求：

```text
Authorization: Bearer <token>
X-Tenant-ID: <tenant_id>
```

实现：

```python
get_current_user()
get_tenant_context()
require_permission(key)
```

规则：

- User必须有当前Tenant的有效Membership；
- Repository必须显式带`tenant_id`；
- 业务表均包含`tenant_id NOT NULL`；
- 用ID读取时仍需同时过滤`tenant_id`；
- Edge只能领取自己租户且绑定本设备的Command。

## 6. 种子数据

- Tenant A：Demo Brand A；
- Tenant B：Demo Brand B；
- User A仅属于Tenant A；
- 预置基础角色与权限；
- Tenant A中创建1个Creator、1个CooperationCase和7个Track；
- 创建1个Fake CellDefinition和Deployment；
- 注册1个Fake Edge Node。

## 7. Fake Cell

步骤固定在代码中：

```text
validate_input
→ create_edge_command
→ wait_edge_result
→ verify_result
→ save_evidence
→ complete
```

Run状态：

```text
CREATED → RUNNING → WAITING_EXTERNAL → VERIFYING → COMPLETED
```

失败只需支持：`FAILED`。

Day 1不做动态StepDefinition解析，不做人工审批。

## 8. Worker

独立进程：

```bash
python -m apps.cloud_worker.worker.main
```

实现：

- 从`jobs`使用`FOR UPDATE SKIP LOCKED`领取；
- `available_at`；
- `lease_owner`；
- `lease_expires_at`；
- 处理成功/失败；
- 结构化日志；
- 进程重启后可重新领取过期Job。

## 9. Fake Edge协议

最低API：

```text
POST /api/v1/edge/register
POST /api/v1/edge/commands/claim
POST /api/v1/edge/commands/{command_id}/report
```

Fake Edge回报：

```json
{
  "status": "SUCCEEDED",
  "generation": 1,
  "attempt": 1,
  "external_ref": {
    "type": "FAKE_ORDER",
    "id": "fake_001"
  },
  "result": {
    "message": "demo success"
  }
}
```

Cloud收到后：

- 校验Tenant、Edge、Command、generation；
- 重复回报保持幂等；
- Run进入`VERIFYING`；
- 创建Evidence与ActivityEvent；
- Run进入`COMPLETED`。

## 10. Day 1 API

```text
POST /api/v1/auth/login
GET  /api/v1/me
GET  /api/v1/tenants

POST /api/v1/koc/cases
GET  /api/v1/koc/cases
GET  /api/v1/koc/cases/{case_id}

POST /api/v1/cell-deployments/{deployment_id}/runs
GET  /api/v1/cell-runs/{run_id}
GET  /api/v1/cell-runs/{run_id}/timeline

POST /api/v1/edge/register
POST /api/v1/edge/commands/claim
POST /api/v1/edge/commands/{command_id}/report

GET /health
GET /ready
```

## 11. 必须测试

### 租户

- User A能访问Tenant A；
- User A访问Tenant B返回403；
- Tenant A不能用ID读取Tenant B案例；
- Tenant A Edge不能领取Tenant B Command。

### Runtime

- Fake Run完成；
- 重复Edge回报不会重复Evidence；
- 同一`idempotency_key`不会创建两个Run；
- Worker租约过期可重领；
- 错误generation回报被拒绝。

### 迁移

- 空数据库可完整`alembic upgrade head`；
- 种子脚本可重复执行而不创建重复记录。

## 12. 8小时建议顺序

### 09:00—10:00

仓库检查、目录、Docker Compose、PostgreSQL、Alembic。

### 10:00—12:00

Tenant/User/Membership/RBAC最小模型、JWT、Tenant Context、隔离测试。

### 13:00—14:30

CooperationCase、CaseTrack、种子数据和API。

### 14:30—16:30

CellRun、Job、Command、Worker和Fake Cell。

### 16:30—17:30

Fake Edge领取与回报、Evidence和Timeline。

### 17:30—18:00

Demo脚本、测试、README和Codex汇报。

若进度不足，优先保证：多租户隔离 + Fake Run闭环。不得通过删除测试来赶进度。

## 13. Definition of Done

- `docker compose up`启动PostgreSQL、API、Worker；
- Alembic空库迁移成功；
- 登录和Tenant Context成功；
- 跨租户访问被拒绝；
- 创建KOC案例自动生成7条轨道；
- 启动Fake Run；
- Worker生成Command；
- Fake Edge领取并回报；
- Run完成；
- Timeline能看到Run、Command、Evidence和Activity；
- 测试通过；
- `scripts/demo_day1.py`一键演示；
- 没有接真实平台；
- 没有修改Browser Service核心代码。

## 14. Stretch Goals（仅在DoD完成后）

- 加入`attempts`；
- 加入`outbox_events`；
- 增加简单异常对象；
- 生成OpenAPI Client；
- 添加极简调试HTML页。

## 15. Codex最终汇报

1. 修改文件清单；
2. 架构和数据库说明；
3. 运行命令；
4. 测试结果；
5. Demo输出；
6. 未完成项；
7. 新发现的开放问题；
8. Day 2建议，但不得自行开始。
