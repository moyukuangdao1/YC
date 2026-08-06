---
document_id: yc-codex-week1-plan
version: 1.0
status: ready
week: 2026-08-07_to_2026-08-13
---

# YC Codex第1周实施计划 v1.0

## 周目标

完成一个能够演示的Fake产品骨架，并让真实飞书数据进入KOC薄业务壳；Edge能够连接现有Browser Service并完成健康检查。第1周不要求真实创建云仓或抖音订单。

## Day 1：纵向骨架

执行`YC_Codex_明日启动任务书_Day1_v2.0.md`。

输出：多租户、KOC案例、Fake Cell、Fake Edge、Evidence、Timeline。

## Day 2：运行可靠性

增加：

- `attempts`；
- `exceptions`；
- `outbox_events`；
- Command ACK/lease/deadline；
- `RESULT_UNKNOWN`状态；
- 重复回报、迟到结果和取消；
- Worker/Job恢复测试；
- 稳定错误码。

验收：模拟“外部成功、写回失败”，只重试写回；模拟进程崩溃后恢复。

## Day 3：Mapping和飞书Fake/真实只读

增加：

- `connector_instances`；
- `mapping_profiles`；
- Schema Fingerprint；
- 字段所有权；
- Fake Lark Connector；
- 真实飞书只读Spike（不写回生产表）；
- CooperationCase去重策略。

验收：使用脱敏样本从不同列名映射到统一KOC字段。

## Day 4：Edge骨架与Browser Service

增加：

- 正式Edge Agent最小进程；
- 设备注册、心跳和命令领取；
- SQLite本地Inbox；
- Browser Service Client；
- `127.0.0.1`与Bearer Token；
- Profile注册和健康状态；
- Fake Connector通过Edge调用Browser Service健康接口。

验收：云端看到Edge在线、Browser Service版本和Profile健康；断网后恢复。

## Day 5：Demo Mode和基础页面

增加：

- Demo Tenant一键重置；
- 成功、缺字段、重复寄样、登录失效和写回失败场景；
- 极简Web页面：总览、案例详情、运行时间线、异常；
- 管理指标Fake数据；
- 5分钟Demo第一次排练。

验收：无真实平台也能稳定完成完整演示。

## 周末复盘

回答：

1. 架构是否支持第二个租户而无需复制代码？
2. 哪些表和抽象过度设计？
3. 飞书真实字段与本体有何冲突？
4. Browser Service需要改代码还是只需Client？
5. Week 2先接云仓还是抖音？
6. Demo中采购者最难理解什么？

输出：

- `WEEK1_REVIEW.md`；
- 修订后的开放问题；
- Week 2任务书；
- 不得同时接两个真实Connector，优先选择稳定、可对账的一条。
