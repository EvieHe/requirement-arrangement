# 真实 XDPai API mapping（已查证）

> 本文固化「页面运行时直连」用到的真实接口契约。查证方式：
> 1. xdpai-apps 组件市场契约（`xdp-app get-component`）—— task-detail / workflow-variables / sop-progress
> 2. 星斗研发平台 proto（`xd get-proto`）—— `xdpai.gateway.taskcgi`、`xdpai.gateway.workflowcgi`、`xdpai.workbench.task`、`xdpai.workbench.sop`
>
> 浏览器侧鉴权约定：`credentials: "include"`（内网域名自动带登录态），请求头只带 `Content-Type`（网关跨域白名单只放行 Content-Type / Accept / Tip-Channel）。

## 1. 读任务详情（= 需求）

| 项 | 值 |
|---|---|
| 网关 | `https://xdclaw.woa.com/xdpai.gateway.taskcgi.taskcgi` |
| 方法 | `GetTask`（POST） |
| 入参 | `{"task_id":"..."}` |
| 返回 | `task`（TaskInfo） |
| 鉴权 | credentials: include，仅 Content-Type |

`TaskInfo` 关键字段：`task_id / title / description / status / priority / creator_name / created_at / updated_at / module_id / project_id / assignees[] / attachments[]`。

> **需求与任务是同一实体**：xdpai 的「任务」即「需求」，`task_id` 就是需求 ID，没有独立的 requirement 实体。`description` 为需求正文（Markdown）。

## 2. 读流程实例（workflow vars + SOP 节点）

| 项 | 值 |
|---|---|
| 网关 | `https://xdclaw.woa.com/xdpai.gateway.workflowcgi.workflowcgi` |
| 方法 | `GetInstance`（POST） |
| 入参 | `{"task_id":"..."}`（或 `instance_id`） |
| 返回 | `instance`（InstanceInfo） |
| 鉴权 | credentials: include，仅 Content-Type |

`InstanceInfo` 关键字段：`instance_id / status / variables_json / node_states[] / nodes[] / edges[] / current_node_id / active_node_ids[] / history_node_ids[]`。

- `variables_json` 是 **JSON 对象字符串**：key = 变量名，value = 变量值（JSON 字符串或纯文本）。
- 本需求读取的 5 个变量：

```text
requirement_arrangement.init
requirement_arrangement.understanding
requirement_arrangement.breakdown
requirement_arrangement.estimation
requirement_arrangement.scheduling
```

## 3. 写 workflow var —— ⚠️ 浏览器侧无开放接口（核心限制）

| 层 | 接口 | 入参 | 浏览器可直连 |
|---|---|---|---|
| sop 后端 | `xdpai.workbench.sop` 的 `SetWorkflowVar` | `{instance_id, name, value}` | ❌（内部服务） |
| workflowcgi 网关 | 未透传 SetWorkflowVar | — | ❌ |
| agent 侧 mcp | `xdpai.aihelper.mcp.mcp` 的 `SetWorkflowVar` | `{task_id, name, value}` | ❌（Bearer token，浏览器拿不到） |

**结论**：读链路（GetTask / GetInstance）浏览器可完整直连；**写链路（写回 workflow var）当前平台网关未向浏览器开放**。页面据此诚实处理「保存写回」，不造假按钮。

## 4. 重跑 AI 节点

workflowcgi 有 `AdvanceWorkflow` / `RejectNode`（流程控制，需 instance_id + 权限），但没有「按阶段重跑」的语义；SOP 后端有 `RollbackWorkflow`。当前均未在浏览器侧开放为「重跑某阶段」的入口。页面明确标记限制，不做假按钮。

## 5. 测试数据

- 测试 task_id：`8b4dd104-eec2-40b4-9160-85ae0a7d788b`（与仓库 5 份 JSON 的 `requirement_id` 一致；xdpai 中 task 即 requirement）。
- 5 份 JSON 仅作为开发 / Mock Mode 数据，非生产事实源。
