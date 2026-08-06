---
document_id: yc-codex-day3a-smoke-acceptance
title: YC Codex Day 3A HumanTask核心｜人工Smoke验收记录
version: 1.0
status: ACCEPTED_PENDING_CANCEL_INVARIANT
updated_at: 2026-08-06
release:
  commit: e55c999
  tag: day3a-human-task-core
  alembic_revision: 0004_day3a_human_task_core
  full_test_result: 27 passed
  demo_result: 3 passed
---

# YC Codex Day 3A HumanTask核心｜人工Smoke验收记录 v1.0

## 1. 已完成的人工验收

创始人在本地实际运行并确认：

```text
git工作区：clean
Tag Commit：e55c999
Alembic：0004_day3a_human_task_core (head)
alembic check：无模型漂移
全量测试：27 passed
Day 3A Demo：3 passed
```

唯一警告为FastAPI/Starlette TestClient上游弃用提示，不阻塞本Release。

---

## 2. 三条核心路径已验证

### A. 人工确认外部动作已完成

实际结果：

```text
HumanTask = RESOLVED
Submission = APPLIED
Exception = RESOLVED
ResumeSignal = CONSUMED
Run / Step / Command = SUCCEEDED
原Write Attempt = RESULT_UNKNOWN
```

确认：

- 人工结论没有篡改原机器Attempt历史；
- 人工结论通过Worker应用；
- 业务运行按成功路径恢复并完成。

### B. 人工确认外部动作未执行

实际结果：

```text
HumanTask = RESOLVED
Run = RUNNING
Step = WAITING_EXTERNAL
Command = DISPATCHED
旧Generation = 1
新Generation = 2
原Write Attempt = RESULT_UNKNOWN
```

确认：

- 系统可以安全重试；
- 新Attempt使用更高Generation；
- 旧未知Attempt历史保留。

### C. KEEP_PAUSED

实际结果：

```text
HumanTask = WAITING_USER
Submissions = [RECORDED, RECORDED]
Run / Step = WAITING_HUMAN
ResumeSignal数量 = 0
duplicate = true
```

确认：

- 同一HumanTask可以保留多次人工Submission；
- KEEP_PAUSED不制造后继任务；
- 不创建ResumeSignal；
- 不创建新Generation；
- 业务继续等待人。

---

## 3. 当前唯一待确认不变量

若系统保留HumanTask取消API，需要确认：

> Run/Step处于WAITING_HUMAN且只有一个有效HumanTask时，取消该任务不能留下“仍等待人但无人可处理”的死状态。

接受的实现：

```text
A. 返回409 ACTIVE_HUMAN_TASK_REQUIRED
```

或：

```text
B. 取消与创建替代HumanTask在同一事务完成
```

禁止：

```text
Run/Step = WAITING_HUMAN
Exception = OPEN
有效HumanTask = 0
```

如果Day 3A最终没有开放取消API，则该检查标记为`NOT_APPLICABLE`，Day 3A可直接最终验收。

---

## 4. 最终状态转换

取消不变量确认后，将本文状态更新为：

```yaml
status: ACCEPTED
```

并正式授权进入Day 3B。

如需代码修正：

- 不移动或重写现有`day3a-human-task-core` Tag；
- 新增最小Patch Commit；
- 建议新Tag：`day3a-human-task-core.1`；
- 增加取消不变量测试；
- 更新Release KNOWN_GAPS与验收记录。
