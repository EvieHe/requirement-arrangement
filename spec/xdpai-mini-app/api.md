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

> 已参考 xdpai skill 的 `workflow_set_var.sh` 逻辑并实测验证（2026-11-05 用 ego-browser 在应用页内 fetch 实测）：

| 层 | 接口 | 入参 | 浏览器可直连 |
|---|---|---|---|
| sop 后端 | `xdpai.workbench.sop` 的 `SetWorkflowVar` | `{instance_id, name, value}` | ❌（内部 trpc 服务，无对外网关） |
| workflowcgi 网关 | **未透传 SetWorkflowVar**（proto v0.0.9 无此 rpc） | — | ❌ |
| agent 侧 mcp | `xdpai.aihelper.mcp.mcp` 的 `SetWorkflowVar` | `{task_id, name, value}` | ❌（① apipai.woa.com 跨域 CORS 拦截 → `Failed to fetch`；② Bearer `XDPAI_MCP_TOKEN` 为平台注入 agent 凭证，浏览器无） |

**实测证据**（应用页 `requirement-arrangement.app.xdpai.woa.com` 内 fetch）：

- `xdclaw.woa.com/xdpai.gateway.workflowcgi.workflowcgi/GetInstance` → 200（读链路 ✅，返回 `instance_id`）
- `xdclaw.woa.com/xdpai.aihelper.mcp.mcp/GetWorkflowVar` → `Failed to fetch`（xdclaw 域无 mcp 入口）
- `apipai.woa.com/xdpai.aihelper.mcp.mcp/GetWorkflowVar` → `Failed to fetch`（CORS 白名单不含 app.xdpai.woa.com）

**根因**：浏览器可达的唯一 workflow 网关 `workflowcgi`（xdclaw.woa.com，cookie + CORS 白名单）**没有透传 sop 后端的 `SetWorkflowVar`**；而带 `SetWorkflowVar` 的 mcp 网关（apipai.woa.com）是 agent 侧（Bearer token + 无 CORS），浏览器不可达。

**修复建议（平台侧，一行 proto）**：在 `xdpai.gateway.workflowcgi` 照 `GetInstance` 的透传模式新增

```proto
rpc SetWorkflowVar(xdpai.workbench.sop.SetWorkflowVarReq) returns (xdpai.workbench.sop.SetWorkflowVarRsp);
```

并转发到 sop 后端。页面已能 `GetInstance` 拿到 `instance_id`，透传后写回链路即可打通（`GetInstance → SetWorkflowVar(instance_id, name, value)`）。更友好可改为收 `task_id`（由网关转 instance_id，与 mcp 服务一致）。

**当前结论**：读链路完整直连；写链路受平台网关缺口阻塞。

**已采用的折中实现（先跑通，后提需）**：页面「保存写回」不再造假按钮，改为**生成可执行的写回命令**（`curl` 调 `apipai.woa.com/xdpai.aihelper.mcp.mcp/SetWorkflowVar`，token 用 `$XDPAI_MCP_TOKEN` 环境变量占位、不硬编码），复制后由用户在具备 `XDPAI_MCP_TOKEN` 的环境（本地虾 / 沙箱）执行即可真正写回。同时保留「导出本 Tab JSON」兜底。

> 曾评估「页面让用户填个人 token 后浏览器直连」，实测**不可行**：即使带 token，浏览器跨域到 `apipai.woa.com` 仍被 CORS 拦截（`Failed to fetch`）。CORS 是浏览器强制机制，JS 无法绕过，与 token 无关。

**后续待办（平台侧提需）**：在 `workflowcgi` 透传 `SetWorkflowVar` 后，页面把「生成命令」改为浏览器直连写回（`GetInstance → SetWorkflowVar`），去掉对 `XDPAI_MCP_TOKEN` 的依赖。

## 4. 重跑 AI 节点

workflowcgi 有 `AdvanceWorkflow` / `RejectNode`（流程控制，需 instance_id + 权限），但没有「按阶段重跑」的语义；SOP 后端有 `RollbackWorkflow`。当前均未在浏览器侧开放为「重跑某阶段」的入口。页面明确标记限制，不做假按钮。

## 5. 测试数据

- 测试 task_id：`8b4dd104-eec2-40b4-9160-85ae0a7d788b`（与仓库 5 份 JSON 的 `requirement_id` 一致；xdpai 中 task 即 requirement）。
- 5 份 JSON 仅作为开发 / Mock Mode 数据，非生产事实源。
