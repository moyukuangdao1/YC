---
document_id: yc-codex-day2-1-semantic-cleanup
title: YC Codex Day 2.1语义清理与Release冻结执行方案
version: 1.0
status: EXECUTION_BASELINE
updated_at: 2026-08-06
depends_on:
  - day1-p0
target_release: day2-reliability
---

# YC Codex Day 2.1语义清理与Release冻结执行方案 v1.0

## 0. 唯一目标

本轮只完成Day 2.1，不进入Day 3。

拆分：

```text
CommandAttempt.result_status
- SUCCEEDED
- FAILED
- RESULT_UNKNOWN

CommandAttempt.reconcile_outcome
- EXISTS
- NOT_FOUND
- UNKNOWN
- NULL
```

随后完成迁移、测试、Demo、Release、Commit和`day2-reliability` Tag。

---

## 1. 语义规则

### 普通写Command

允许：

```text
SUCCEEDED + NULL
FAILED + NULL
RESULT_UNKNOWN + NULL
```

禁止携带`reconcile_outcome`。

### Reconcile查询Command

允许：

```text
SUCCEEDED + EXISTS
SUCCEEDED + NOT_FOUND
SUCCEEDED + UNKNOWN
FAILED + NULL
RESULT_UNKNOWN + NULL
```

其中：

- 查询成功必须携带Outcome；
- 查询执行失败或查询本身结果未知时Outcome为空；
- 原写动作继续保持未知并转人工。

---

## 2. 迁移0003

新增：

```text
0003_day2_1_semantic_cleanup
```

顺序：

1. 新增nullable `reconcile_outcome`；
2. 将0002遗留的`EXISTS / NOT_FOUND / UNKNOWN`迁移到新字段；
3. 将这些Attempt的`result_status`改为`SUCCEEDED`；
4. 如仍存在非法旧状态，迁移失败；
5. 替换旧约束；
6. 添加新值域和基础组合约束。

必须测试：

- 空库连续升级；
- 0002带旧Attempt数据升级到0003；
- `alembic check`无漂移。

---

## 3. API错误语义

### 422

- 未知枚举；
- 字段类型错误；
- 请求结构错误。

### 409 `RESULT_SEMANTICS_INVALID`

- 普通写Command携带Outcome；
- Reconcile成功缺Outcome；
- Reconcile失败携带Outcome；
- 其他合法字段的非法组合。

报告幂等哈希包含：

```text
status
reconcile_outcome
result
evidences
```

---

## 4. 状态变化

### 写Command成功

```text
Write Attempt = SUCCEEDED
Write Command = SUCCEEDED
Step = SUCCEEDED
Run = SUCCEEDED
Track = COMPLETED
```

### 写Command明确失败

```text
Write Attempt = FAILED
Write Command = FAILED
Step = FAILED
Run = FAILED
Track = BLOCKED
```

### 写Command结果未知

```text
Write Attempt = RESULT_UNKNOWN
Write Command = RESULT_UNKNOWN
Step = RECONCILING
Run = RECONCILING
创建Reconcile Command
```

### Reconcile `EXISTS`

```text
Reconcile Attempt = SUCCEEDED + EXISTS
Reconcile Command = SUCCEEDED
原Write Attempt仍为RESULT_UNKNOWN
原Write Command = SUCCEEDED
Step/Run/Track完成
```

### Reconcile `NOT_FOUND`

```text
Reconcile Attempt = SUCCEEDED + NOT_FOUND
Reconcile Command = SUCCEEDED
原Write Command = PENDING
Run返回RUNNING
创建更高Generation的新Attempt
旧Attempt永久保留
```

### Reconcile `UNKNOWN`

```text
Reconcile Attempt = SUCCEEDED + UNKNOWN
原Write Command = RESULT_UNKNOWN
Run/Step = WAITING_HUMAN
创建或复用MANUAL_REQUIRED Exception
```

### Reconcile查询失败或结果未知

```text
原Write Command保持RESULT_UNKNOWN
Run/Step = WAITING_HUMAN
创建或复用MANUAL_REQUIRED Exception
```

---

## 5. Reconcile关系验证

必须校验：

- 同一Tenant；
- 同一Run；
- 非自引用；
- 原Command不是另一个Reconcile Command；
- Reconcile命令类型在白名单；
- 当前阶段只允许一层Reconcile。

---

## 6. `NOT_FOUND`重试要求

原Command恢复`PENDING`后：

- 下一次领取创建`generation + 1`；
- 新Attempt ID不同；
- 旧Attempt不删除、不覆盖；
- 旧Attempt再次报告被拒绝；
- 新Attempt成功后Run完成。

必须检查当前`claim_command()`是否会因旧Attempt为`REPORTED/APPLIED`而永久跳过；如会，修正为仅在Command为`DISPATCHED`时跳过终态当前Attempt，Command明确恢复`PENDING`时允许新Generation。

---

## 7. 测试

必须通过：

1. Day 1回归；
2. Day 2五条路径；
3. 0002旧数据迁移；
4. 语义组合参数化矩阵；
5. 普通Command携带Outcome拒绝；
6. Reconcile成功缺Outcome拒绝；
7. 旧Status枚举422；
8. 报告幂等和冲突；
9. Job generation/token；
10. 跨租户Exception；
11. JSON大小限制；
12. NOT_FOUND新Generation；
13. EXISTS不篡改原Attempt历史；
14. Reconcile关系约束。

---

## 8. Demo

更新`demo_day2.ps1`，明确输出：

```text
执行状态
业务事实状态
原写Attempt状态
原写Command状态
Reconcile状态
Run/Step/Track状态
Exception
Generation
```

仍展示五条路径。

---

## 9. Release

补齐：

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

已知缺口：

- 单次Reconcile；
- 无退避；
- 无Outbox；
- 无Job心跳；
- 无正式HumanTask；
- 无真实Connector。

完成后：

```text
commit: feat: complete YC Day 2 reliability loop
tag: day2-reliability
git status clean
```

然后停止，不创建Day 3模型、API或迁移。
