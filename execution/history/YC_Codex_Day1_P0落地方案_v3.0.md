---
document_id: yc-codex-day1-p0-plan
title: YC Codex Day 1 P0落地方案
version: 3.0
status: implemented
updated_at: 2026-08-06
owner: YC Founder
---

# YC Codex Day 1 P0落地方案 v3.0

## 1. 今日目标

在 `codex-yunchen/yc` 建立可运行、可迁移、可测试的FastAPI项目，证明：

```text
种子租户A
→ KOC案例及7条轨道
→ Fake Sample Cell Run
→ Job
→ Command
→ CommandAttempt
→ Fake Edge
→ 后续Job
→ Evidence
→ SAMPLE_FULFILLMENT轨道完成
```

强制验收包括两个租户隔离、重复报告幂等、冲突报告拒绝、Attempt generation租约恢复、迟到结果隔离，以及Edge报告不在HTTP请求内推进完整业务流程。

## 2. 已锁定决策

1. 使用Python 3.11、FastAPI、SQLAlchemy 2、Alembic和PostgreSQL 16；测试不使用MySQL替代。
2. 使用两个种子租户，不开放租户创建、更新或删除API。
3. User只实现租户成员校验；完整RBAC为P1。
4. Edge使用独立机器身份和令牌，不属于User RBAC。
5. Job负责云端内部工作，Command负责外部动作，两者严格分离。
6. CommandAttempt独立保存generation、Edge、租约、租约令牌哈希与执行结果。
7. Run必须绑定CellDeployment，并冻结Definition、Deployment和Subject版本快照。
8. 数据库负责约束与乐观锁；领域服务负责合法状态转换；不使用复杂Trigger。
9. JSONB只用于定义、配置、输入输出、主体和扩展快照，不替代核心领域列和关系表。
10. Evidence、ActivityEvent、AuditEvent分别承担业务证据、用户时间线和安全审计语义。
11. Browser Service状态统一标记为“来源不一致待核验”；Day 1不接入、不重构。
12. Docker排查最多30分钟，失败即切换本地PostgreSQL 16，不阻塞P0。

## 3. P0领域范围

身份与租户：`users`、`tenants`、`tenant_memberships`、`edge_nodes`。

KOC薄业务壳：`creators`、`cooperation_cases`、`case_tracks`。案例创建时在同一事务生成以下7条唯一轨道：

- `DISCOVERY_COMMUNICATION`
- `COMMERCIAL_TERMS`
- `SAMPLE_FULFILLMENT`
- `DOUYIN_COOP_ORDER`
- `CONTENT_PUBLICATION`
- `DATA_MEDIA_BUYING`
- `FINANCE_COST`

Cell Runtime：`cell_definitions`、`cell_deployments`、`cell_runs`、`step_runs`、`jobs`、`commands`、`command_attempts`。

证据与事件：`evidences`、`activity_events`、`audit_events`。

## 4. 运行语义

Run API原子创建Run、StepRun和首个 `START_FAKE_SAMPLE` Job。Worker使用PostgreSQL `FOR UPDATE SKIP LOCKED` 和独立Job租约领取工作，生成Fake Sample Command后结束本次Job。

Edge Claim锁定Command并创建新CommandAttempt。每次Claim递增generation，并只返回一次Attempt租约令牌明文；数据库只保存哈希。过期Attempt先标记为 `EXPIRED`，再创建下一generation。

Edge Report只校验并保存Attempt结果、规范化结果哈希，同时创建唯一的 `APPLY_COMMAND_RESULT` Job。报告响应时Run和寄样轨道仍未完成。后续Worker处理该Job时才创建Evidence，并通过领域服务完成Command、StepRun、CellRun和寄样轨道。

结果处理顺序：

1. 已报告且结果哈希相同：幂等返回，不重复创建Job或Evidence。
2. 已报告但结果不同：返回 `409 REPORT_CONFLICT`。
3. generation非当前值、Attempt过期或已被替代：返回 `409 STALE_ATTEMPT`。
4. Edge不匹配或租约令牌错误：拒绝访问。

## 5. P0 API

用户接口：

- `POST /api/v1/auth/login`
- `GET /api/v1/me`
- `GET /api/v1/tenants`
- `POST /api/v1/koc/cases`
- `GET /api/v1/koc/cases/{case_id}`
- `GET /api/v1/koc/cases/{case_id}/timeline`
- `POST /api/v1/cell-runs`
- `GET /api/v1/cell-runs/{run_id}`

Edge接口：

- `POST /api/v1/edge/commands/claim`
- `POST /api/v1/edge/attempts/{attempt_id}/report`

明确不存在 `POST /api/v1/tenants` 及角色权限管理接口。

## 6. P0验收

- Alembic可从空PostgreSQL数据库升级。
- Tenant A与Tenant B的数据和身份相互隔离。
- 案例自动且仅生成7条轨道。
- Run绑定正确Deployment并保存三个版本快照。
- 同幂等键同请求返回已有Run；不同请求返回409。
- Job过期租约可由另一Worker恢复。
- Attempt过期后产生下一generation，旧generation迟到报告被拒绝。
- Edge首次报告只生成一个后续Job；重复相同报告幂等；冲突报告拒绝。
- 后续Worker处理完成后才创建Evidence，且只有 `SAMPLE_FULFILLMENT` 轨道完成。

## 7. P1后置范围

完整Role/Permission、Transactional Outbox、自动Reconcile、真实Browser/抖音/云仓Connector、Browser Service重构与版本定论、真实进程故障注入、前端页面和阿里云部署均不进入Day 1。

## 8. 实施记录

- YC独立PostgreSQL 16容器已启动并通过健康检查，主机端口为55432。
- 开发库 `yc_dev` 与测试库 `yc_test` 隔离。
- Alembic空库迁移、幂等种子初始化和P0集成测试已通过。
- 项目运行说明见 `yc/README.md`。

