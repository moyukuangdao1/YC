---
document_id: yc-codex-day2-reliability-record
title: YC Codex Day 2可靠性闭环实施记录
version: 1.1
status: awaiting_human_acceptance
updated_at: 2026-08-06
owner: YC Founder
baseline: day1-p0
---

# YC Codex Day 2可靠性闭环实施记录 v1.1

## 1. 边界

本任务只实现 Day 2 与 Day 2.1 可靠性闭环，不接真实平台、不建设 Agent Runtime、不进入 Day 3 HumanTask。

## 2. 已完成

- Job 租约增加 generation 和令牌校验，旧 generation、错误令牌和过期租约均被拒绝。
- Attempt 执行状态固定为 `SUCCEEDED / FAILED / RESULT_UNKNOWN`。
- Reconcile 业务事实独立为 `EXISTS / NOT_FOUND / UNKNOWN / NULL`。
- 新增单层 `FAKE_QUERY_SAMPLE_SHIPMENT` 和最小 `MANUAL_REQUIRED` Exception。
- `EXISTS` 只确认原 Command 成功，原写 Attempt 保持 `APPLIED + RESULT_UNKNOWN`。
- `NOT_FOUND` 恢复原 Command 为 `PENDING`，允许创建更高 generation Attempt，旧 Attempt 不删除、不覆盖、不篡改。
- 查询 `UNKNOWN / FAILED / RESULT_UNKNOWN` 均进入 `WAITING_HUMAN`，Exception 幂等。
- Reconcile 目标必须与查询 Command 同租户、同 Run、同 Step，禁止自引用和嵌套 Reconcile。

## 3. 迁移与验证

- 迁移链：`0001_day1_p0 → 0002_day2_reliability → 0003_day2_1_semantic_cleanup`。
- 0003 会安全迁移 0002 的三类旧状态、转换快照并按运行时算法重算哈希。
- 独立 PostgreSQL 测试覆盖带 0002 历史数据升级到 0003。
- 全量 Day 1、Day 2、Day 2.1 测试通过。
- Alembic 空库升级、开发库升级和模型漂移检查通过。
- 五路径 Fake Demo 通过。

## 4. 待人工验收

Release 产物位于 `yc/docs/releases/day2-reliability/`。本记录不批准进入 Day 3；需人工核对状态、时间线、Demo 和已知风险后再决定下一任务包。
