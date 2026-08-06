---
document_id: yc-browser-service-integration
version: 1.0
status: current
updated_at: 2026-08-06
owner: YC Founder
applies_to: Browser Service 2.0.0
---

# YC Browser Service集成规范 v1.0

## 1. 定位

现有Browser Service是YC的 **本地浏览器执行内核**，不是直接面向客户销售的产品，也不是YC云端工作流引擎。

其职责：

- 维护持久化Chromium上下文和平台登录态；
- 页面、DOM、网络、Fetch、上传、下载和截图；
- 为Edge内部的已注册Connector提供低级浏览器能力。

YC Edge职责：

- 接收云端受限业务命令；
- 校验租户、命令、Profile、风险和版本；
- 调用Connector；
- Connector再调用Browser Service；
- 验证外部结果并上传证据。

## 2. 调用链

```text
YC Cloud业务命令
→ YC Edge Command Handler
→ Registered Connector
→ Browser Service HTTP API（127.0.0.1）
→ 外部平台
```

Cloud永远不能直接调用Browser Service。

## 3. 生产启动要求

```powershell
.\scripts\start.ps1 \
  -HostName 127.0.0.1 \
  -Port 9527 \
  -Token "<random-local-token>" \
  -UserDataDir "D:\YC\profiles\<profile_id>"
```

要求：

- 只监听`127.0.0.1`；
- Bearer Token强制开启；
- Browser Service进程由YC Edge启动和监督；
- Windows设备禁止自动休眠；
- 每个平台账号使用独立Profile目录；
- Profile目录不进入代码仓库、云同步盘和普通备份；
- 临时文件有定期清理和容量上限。

## 4. Profile模型

```text
profile_id
edge_node_id
tenant_id
business_unit_id
platform
account_label
user_data_dir
status
lock_owner
lock_expires_at
last_health_at
last_login_verified_at
browser_service_version
chromium_version
```

规则：

- 一个Profile只能属于一个租户；
- 同一Profile同一时间只能有一个写入型命令；
- 只读查询可以按平台特性决定是否并发；
- 登录态失效时Profile进入`AUTH_REQUIRED`，相关Cell暂停；
- 员工重新登录并通过健康检查后恢复。

## 5. Edge Connector接口

每个Connector实现统一接口：

```python
class EdgeConnector(Protocol):
    connector_key: str
    version: str
    supported_commands: set[str]

    def preflight(self, ctx, payload) -> PreflightResult: ...
    def execute(self, ctx, command) -> ExecutionResult: ...
    def verify(self, ctx, execution_result) -> VerificationResult: ...
    def reconcile(self, ctx, external_hint) -> ReconcileResult: ...
    def health_check(self, ctx) -> ConnectorHealth: ...
```

Cloud只识别业务命令，例如：

- `WAREHOUSE_CREATE_SHIPMENT`；
- `WAREHOUSE_QUERY_SHIPMENT`；
- `DOUYIN_CREATE_COOPERATION_ORDER`；
- `DOUYIN_QUERY_COOPERATION_ORDER`。

## 6. 禁止能力

云端命令Payload不得包含：

- 任意JavaScript代码；
- 任意URL；
- 任意CSS/XPath脚本；
- 任意Cookie读取请求；
- 任意Fetch目标；
- 任意本地路径；
- 任意Shell命令。

这些低级能力只能存在于签名、版本化、经过测试的Connector代码中。

## 7. 能力发现与兼容

Edge心跳上报：

```json
{
  "edge_version": "0.1.0",
  "browser_service_version": "2.0.0",
  "chromium_version": "...",
  "connectors": {
    "cloud_warehouse": "1.0.0",
    "douyin_cooperation": "1.0.0"
  },
  "profiles": []
}
```

CellDeployment声明最低兼容版本。版本不兼容时不允许执行，进入`INCOMPATIBLE_RUNTIME`异常。

## 8. 错误映射

Browser Service错误需映射为稳定Connector错误：

| 分类 | 示例 | 是否重试 |
|---|---|---|
| `AUTH_REQUIRED` | 登录失效、验证码 | 人工处理 |
| `ELEMENT_CHANGED` | 页面结构变化 | 暂停Connector |
| `RATE_LIMITED` | 平台限流 | 延迟重试 |
| `NETWORK_ERROR` | 超时、断网 | 自动重试 |
| `BUSINESS_REJECTED` | 平台拒绝业务字段 | 人工处理 |
| `RESULT_UNKNOWN` | 请求可能成功但未确认 | 必须对账 |
| `BROWSER_CRASHED` | Chromium崩溃 | 重启后重试/对账 |
| `PERMISSION_DENIED` | 账号无权限 | 人工处理 |

## 9. 结果验证与对账

外部写入动作不能只依据“点击成功”或HTTP 200判定成功。

必须尽量取得：

- 外部唯一ID；
- 查询接口或页面二次确认；
- 关键字段摘要；
- 完成时间；
- 证据截图或响应摘要。

若执行结果不确定：

```text
Command → RESULT_UNKNOWN
→ Connector.reconcile()
→ 找到外部记录：标记成功并继续写回
→ 确认不存在：允许安全重试
→ 仍不确定：MANUAL_REQUIRED
```

## 10. 证据与隐私

- 截图默认只截关键区域；
- 手机号和地址在日志中掩码；
- 原始页面HTML和网络记录默认不长期保存；
- 证据文件上传前计算SHA-256；
- 证据保留期限按租户策略；
- 支持人工删除或法定要求删除。

## 11. 进程监督

试点阶段：

- Edge使用Windows任务计划程序自启动；
- Edge负责启动Browser Service；
- 每30秒健康检查；
- 连续失败后自动重启，达到阈值则停止并告警；
- 不允许无限重启风暴。

后续：Windows Service、签名更新包、灰度和回滚。

## 12. Browser Service本身暂不修改的内容

- 不加入租户业务模型；
- 不加入工作流和审批；
- 不加入Wiki/本体；
- 不直接连接YC Cloud；
- 不为了某一平台把业务逻辑写入通用页面API。

真实平台逻辑放在Edge Connector中。
