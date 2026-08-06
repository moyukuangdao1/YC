---
document_id: yc-day2-reliability-acceptance
title: YC Codex Day 2可靠性闭环｜验收意见
version: 1.0
status: ACCEPTED_WITH_GATES
updated_at: 2026-08-06
baseline:
  - day1-p0
next_stage:
  - day2.1-semantic-cleanup
  - day3-human-task
---

# YC Codex Day 2可靠性闭环｜验收意见 v1.0

## 1. 验收结论

根据Codex提交的完成报告，Day 2的目标已经基本达成：

- Job租约增加Generation与Token校验；
- Command支持`RESULT_UNKNOWN`；
- Run/Step支持`RECONCILING`与`WAITING_HUMAN`；
- 建立Reconcile命令与原写命令关联；
- 支持`EXISTS / NOT_FOUND / UNKNOWN`三条对账路径；
- `UNKNOWN`进入`MANUAL_REQUIRED Exception`；
- Edge报告仍只持久化结果并创建后续Job；
- Day 1回归测试继续通过；
- Day 2新增测试通过；
- 多租户Exception隔离、结果幂等、冲突报告和JSON大小限制已有测试覆盖。

本阶段可判定为：

> **工程目标达成，允许进入Release冻结和Day 3准备；但在正式开始HumanTask前，必须完成一个小型Day 2.1语义清理。**

说明：本验收基于Codex报告、测试摘要和既有Day 1基线；尚未对Day 2全部修改文件进行逐行代码审查。

---

## 2. 本阶段真正完成的能力

系统已经从：

```text
成功 / 失败
```

升级为：

```text
成功
明确失败
结果未知
  → 查询外部事实
     → 已存在
     → 不存在，可安全重试
     → 仍未知，交给人
```

这是YC接入真实写入型Connector前必须拥有的最低可靠性。

它解决的核心风险是：

> 浏览器超时、网络断开或回写失败时，不能因为“没有收到成功响应”就再次创建一张真实订单。

---

## 3. Release冻结前的强制门

### 3.1 状态语义拆分

Codex报告中提到：

```text
Attempt结果：
RESULT_UNKNOWN / EXISTS / NOT_FOUND / UNKNOWN
```

需要核对当前实现是否把这五个值都放在`CommandAttempt.result_status`中。

如果是，必须在Day 2.1修正。

建议模型：

```text
CommandAttempt.result_status
- SUCCEEDED
- FAILED
- RESULT_UNKNOWN
```

Reconcile查询的业务结果放在：

```text
result_snapshot.reconcile_outcome
- EXISTS
- NOT_FOUND
- UNKNOWN
```

或增加独立的`reconcile_outcome`字段。

原因：

- `result_status`描述“本次命令执行是否成功”；
- `reconcile_outcome`描述“查询后发现原外部动作处于什么事实状态”。

例如：

```text
查询命令执行成功
result_status = SUCCEEDED
reconcile_outcome = NOT_FOUND
```

不能把`NOT_FOUND`当成命令执行失败。

如果不拆分，未来每个Connector都会把自己的业务结果塞进通用Attempt状态，运行时会逐渐失去通用性。

### 3.2 固定Release

完成Day 2.1后：

```text
git status干净
commit: feat: complete YC Day 2 reliability loop
tag: day2-reliability
```

Release目录：

```text
docs/releases/day2-reliability/
├─ README.md
├─ openapi.json
├─ pytest-output.txt
├─ alembic-check.txt
├─ demo-transcript.txt
├─ migration-summary.md
└─ KNOWN_GAPS.md
```

### 3.3 人工体验

创始人亲自运行：

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\demo_day2.ps1
```

至少确认五条路径在时间线和数据库状态上可理解：

1. 正常成功；
2. 明确失败；
3. 未知后确认已存在；
4. 未知后确认不存在并重试；
5. 仍未知进入人工。

---

## 4. 当前风险判断

### 可以接受并后移

- Reconcile只有单次查询；
- 没有退避和定时轮询；
- 没有Outbox；
- Job没有心跳续租；
- FastAPI TestClient上游弃用警告。

原因：

- 当前Cloud Job为短任务；
- Edge采用拉取式Command；
- 通知系统尚未接入；
- 暂无长程Agent Worker。

### 不能带入真实Connector

- Attempt状态与Reconcile业务结果混用；
- `WAITING_HUMAN`没有正式HumanTask和解决接口；
- 对账结果仍不确定时没有可审计的人工结论；
- 真实写入命令没有最大重试、退避和Connector级对账契约。

---

## 5. 当前里程碑状态

| 能力 | 状态 |
|---|---|
| Day 1 Fake纵向闭环 | VALIDATED |
| 多租户基础 | VALIDATED_FOR_MVP |
| Job/Command/Attempt分离 | VALIDATED |
| RESULT_UNKNOWN与Fake Reconcile | VALIDATED_FOR_FAKE |
| Exception最小模型 | IMPLEMENTED |
| HumanTask | NOT_IMPLEMENTED |
| 真实Connector | NOT_IMPLEMENTED |
| Agent Runtime | NOT_IMPLEMENTED |
| 企业产品界面 | NOT_IMPLEMENTED |
| 真实业务价值 | NOT_VALIDATED |

---

## 6. 下一步唯一主任务

先完成不超过半天的：

> **Day 2.1：Attempt执行状态与Reconcile业务结果语义拆分，并冻结Day 2 Release。**

随后进入：

> **Day 3：HumanTask、Notification、SecureEphemeralArtifact与ResumeSignal。**

禁止跳过HumanTask直接接入抖音Browser写入动作。
