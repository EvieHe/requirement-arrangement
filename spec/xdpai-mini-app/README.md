# XDPai 小应用化：需求智能评估与排期

## 1. 背景

当前 requirement-arrangement 已有一个可运行的前端 Demo，并已使用 5 份 JSON 表达需求及四个 AI 评估阶段：

- `init.json`
- `understanding.json`
- `breakdown.json`
- `estimation.json`
- `scheduling.json`

当前页面仍以仓库静态 JSON 为主要数据源。本需求要将它改造成一个真正可在 XDPai 中使用的小应用。

核心链路：

```text
Task ID
  ↓
XDPai Task / Requirement API
  ↓
需求详情 + Workflow Vars
  ↓
需求智能评估页面
  ↓
用户编辑
  ↓
写回对应 Workflow Var
```

本需求不是重新设计页面，而是把现有页面与 XDPai 的真实任务、需求和流程变量打通。

---

## 2. 目标

用户进入小应用后，可以输入一个 XDPai Task ID，应用自动读取：

1. 当前任务基础信息；
2. 当前任务关联的需求详情；
3. 当前任务对应 SOP 的需求智能排期流程变量。

然后动态渲染：

1. 需求
2. 需求理解
3. 任务拆解
4. 工作量评估
5. 排期生成

用户对四个 AI 阶段的修改能够写回 XDPai workflow vars，刷新后能够再次读取到最新结果。

---

## 3. 非目标

本阶段不做：

- 重新设计 requirement-arrangement 的整体视觉体系；
- 将四个阶段重新合并成一个大 JSON；
- 自建一套与 XDPai 重复的任务/流程系统；
- 编造人员、仓库、需求详情或不存在的 API；
- 用静态 Mock 冒充真实 XDPai 数据；
- 实现与本需求无关的项目管理、资源管理能力。

---

## 4. 用户使用方式

### 4.1 Task ID 入口

页面提供：

- XDPai Task ID 输入框
- 「加载需求」按钮

用户输入 Task ID 后，系统完成：

1. 校验 Task ID；
2. 获取 Task；
3. 获取关联 Requirement；
4. 获取 Workflow Vars；
5. 校验需求智能排期数据；
6. 渲染页面。

### 4.2 URL 参数

支持：

```text
?task_id=<xdpai-task-id>
```

如果 URL 已携带 task_id，应自动加载。

最终应支持 XDPai 内直接跳转：

```text
需求智能排期小应用?task_id={task_id}
```

---

## 5. 数据来源

### 5.1 任务与需求

Task ID 是小应用的数据入口。

应用必须通过 XDPai 当前真实能力读取：

- task_id
- requirement id
- requirement title
- requirement description / detail
- priority
- workflow / SOP 信息
- 页面需要的其他任务上下文

开发前必须加载并阅读 XDPai Skill / SDK / 现有 API，确认：

- 如何根据 task ID 获取任务；
- 如何获取需求详情；
- 如何读取 workflow vars；
- 如何写 workflow vars；
- 是否支持重跑某个 SOP 节点。

禁止自行猜测 API。

### 5.2 Workflow Vars

需求智能排期使用：

```text
requirement_arrangement.init
requirement_arrangement.understanding
requirement_arrangement.breakdown
requirement_arrangement.estimation
requirement_arrangement.scheduling
```

对应：

| 页面 | 数据 |
|---|---|
| 需求 | XDPai Requirement + requirement_arrangement.init |
| 需求理解 | requirement_arrangement.understanding |
| 任务拆解 | requirement_arrangement.breakdown |
| 工作量评估 | requirement_arrangement.estimation |
| 排期生成 | requirement_arrangement.scheduling |

仓库中的 5 份 JSON 只作为开发 / Mock Mode 数据，不是生产事实源。

---

## 6. 页面结构

页面改成 5 个 Tab：

```text
需求 | 需求理解 | 任务拆解 | 工作量评估 | 排期生成
```

「需求」是事实数据展示，不属于 AI 阶段。

AI 评估进度只有：

```text
需求理解 → 任务拆解 → 工作量评估 → 排期生成
```

顶部应分别表达：

- 阶段生成：例如 `4 / 4`
- 待确认项：例如 `5 项待确认`

不能把「4 个 AI 阶段已生成」表达成「整个评估 100% 确定」。

---

## 7. 需求 Tab

数据来源：

- XDPai Requirement API
- `requirement_arrangement.init.requirement`

优先以 XDPai 实际需求详情为事实来源。

展示：

- 需求标题
- Requirement ID
- Task ID
- 优先级
- 原始需求正文

需求正文需要支持 Markdown：

- 标题
- 列表
- 表格
- 引用
- 加粗
- 分节

Hero 不展示完整需求正文，只展示：

- title
- 简短 goal / 摘要
- Task ID / Requirement ID
- 当前阶段状态
- 待确认数量

---

## 8. 需求理解 Tab

数据来源：

`requirement_arrangement.understanding`

遵循 requirement-arrangement-planner schema 4.0。

展示并**支持编辑**：

- goal（textarea 编辑）
- scope（列表，逐项编辑 + 增删）
- out_of_scope（列表，逐项编辑 + 增删）
- assumptions（列表，逐项编辑 + 增删）
- frontend modules / backend modules
- review_items

模块支持编辑：

- name
- repository
- change_summary
- 新增
- 删除

review_items：

### choice

展示 Skill 生成的 options，并由 UI 额外提供「自定义」。

选择自定义后写入 `custom_value`。

### text

直接编辑 `value`。

同时显示：

- ai_assumption
- impact

> 编辑后的写回与其它阶段一致：走「提交变更」（只发 diff + 回退下游节点），详见 §13。

---

## 9. 任务拆解 Tab

数据来源：

`requirement_arrangement.breakdown`

同时读取 understanding，通过：

`module_id -> understanding.modules[].id`

关联模块名称。

展示并允许编辑：

- task id
- title
- module
- category
- description
- dependencies

支持：

- 新增任务
- 删除任务
- 修改 task
- 修改 module
- 修改 category
- 修改 description
- 修改 dependencies

本 Tab 不维护：

- effort
- confidence
- assignee
- start/end date

---

## 10. 工作量评估 Tab

数据来源：

`requirement_arrangement.estimation`

通过：

`task_id -> breakdown.tasks[].id`

关联任务定义。

顶部展示：

- total_effort_days
- high count
- medium count
- low count

任务展示：

- task
- effort_days
- confidence
- estimate_basis
- uncertainties

允许编辑：

- effort_days
- confidence
- estimate_basis

修改后实时重新计算：

- `summary.total_effort_days`
- `summary.confidence_counts`

不得修改 breakdown 中的任务定义。

---

## 11. 排期生成 Tab

数据来源：

`requirement_arrangement.scheduling`

同时读取：

- breakdown
- estimation

通过 task_id join。

主要以甘特图展示：

- task
- assignee
- effort
- start_date
- end_date
- 时间轴

支持：

- 修改负责人 / 资源槽
- 修改开始日期
- 修改结束日期
- 关键路径高亮

展示：

- start_date
- delivery_date
- estimated_working_days
- critical_path

日期修改后重新计算基础 summary。

注意：如果 assignee 当前是 `后端待分配-A` 等占位资源，不得把它解释为真实人员。

---

## 12. Stage Status

读取：

`requirement_arrangement.init.stage_status`

状态枚举：

- pending
- ready
- completed
- stale

页面必须区分显示。

例如：

```json
{
  "understanding": "completed",
  "breakdown": "completed",
  "estimation": "stale",
  "scheduling": "stale"
}
```

stale 必须明确提示：

> 上游数据已修改，该阶段结果需要重新生成。

---

## 13. 编辑写回

当前 Demo 的「保存当前修改」需要改造成真实 workflow var 写回。

### 需求理解

写：

`requirement_arrangement.understanding`

### 任务拆解

写：

`requirement_arrangement.breakdown`

### 工作量评估

写：

`requirement_arrangement.estimation`

### 排期

写：

`requirement_arrangement.scheduling`

不要一次保存覆盖全部变量。

### 写回实现（diff + 回退节点，2026-11-05 实测打通）

浏览器侧**没有直连「写 workflow var」的接口**（详见 `api.md §3`）：`workflowcgi` 未透传 `SetWorkflowVar`，mcp 网关 CORS 拦截。

但**「发消息」的网页 IM 网关 `imcgi`（xdclaw.woa.com）有 CORS 白名单 + cookie 鉴权，浏览器可达**。因此写回走「发消息」：页面「提交变更」把**人改了哪些字段（diff）**发到任务会话，由会话中的 AI 助手（有 `XDPAI_MCP_TOKEN`）更新变量并回退到下游节点重新评估。

**交互**：

```
编辑（加载时存原始快照）
  → 点「提交变更」
  → diff（对比快照 vs 当前数据，只挑出改动的字段）
  → 发一条通知到任务会话：diff 明细 + 「更新变量 + 回退到 xx 节点重新评估」
  → AI 助手更新变量（last_modified_by=human）→ 回退下游节点重跑
  → 刷新看到重新生成的结果
```

**通知语义**（用 SOP 术语，不造新词）：

```
我在工作台审查「工作量评估」时做了以下修正（human edit，权威）：
- BE-01 人日（effort_days）：3 → 4

请先更新 requirement_arrangement.estimation（last_modified_by 置为 human，重算 summary），
然后回退到「排期生成」节点，让它基于更新后的数据重新评估。
```

**阶段 → 回退节点映射**：

| 改的阶段 | 回退到 |
|---|---|
| 需求理解 | 任务拆解 |
| 任务拆解 | 工作量评估 |
| 工作量评估 | 排期生成 |
| 排期生成 | 排期生成 |

**防重复提交**：提交成功后刷新原始快照，再次点击「提交变更」时 diff 为空、不会重复发同一条通知。

> 曾评估「页面让用户填个人 token 后浏览器直连」，实测不可行：带 token 跨域到 `apipai.woa.com` 仍被 CORS 拦截。CORS 与 token 无关，JS 无法绕过。

**后续待办**：平台在 `workflowcgi` 透传 `SetWorkflowVar` 后，页面改为浏览器直连写回（`GetInstance → SetWorkflowVar`），不依赖 AI 助手、可实时校验。当前「发消息 + 回退」依赖会话中 AI 助手在线且理解指令，作为过渡方案。

---

## 14. 上游修改与 stale

### 修改 understanding

保存后：

```text
understanding = completed
breakdown = stale
estimation = stale
scheduling = stale
```

### 修改 breakdown

```text
breakdown = completed
estimation = stale
scheduling = stale
```

### 修改 estimation

```text
estimation = completed
scheduling = stale
```

### 修改 scheduling

只更新 scheduling。

状态变化写回：

`requirement_arrangement.init`

---

## 15. 重新运行 AI

现有「重新评估」不能继续使用页面 reload。

需要调查 XDPai 是否支持：

- rerun node
- retry node
- execute SOP node
- workflow transition

如果支持，根据当前 Tab 重跑对应 AI 节点。

如果暂时不支持，不要做假按钮；页面明确标记限制。

---

## 16. 数据访问层

不要让 UI 到处直接调用 XDPai API。

建议抽象：

```text
services/
  xdpai.ts

planning/
  normalize.ts
  validate.ts
```

建议能力：

```text
getTask(taskId)
getRequirementByTask(taskId)
getWorkflowVars(taskId)
getWorkflowVar(taskId, key)
setWorkflowVar(taskId, key, value)
rerunStage(taskId, stage)
```

planning 层负责：

- JSON parse
- schema validation
- cross-stage validation
- stale calculation
- 数据 join / normalize

---

## 17. 数据校验

必须加载 requirement-arrangement-planner Skill，并遵循 schema 4.0。

如果运行环境允许，复用：

`requirement-arrangement-planner/scripts/validate_stage.py`

至少校验：

- 5 份 requirement_id 一致；
- module_id 存在；
- task_id 存在；
- dependency 存在且无环；
- estimation 覆盖所有 task；
- scheduling 覆盖所有 task；
- effort summary 正确；
- confidence_counts 正确；
- scheduling dependency 顺序正确。

单个阶段数据异常不应导致整个页面白屏。

---

## 18. Loading / Empty / Error State

必须处理：

### Task 不存在

显示：

> 未找到该 XDPai Task。

### SOP 尚未运行

需求 Tab 仍然正常展示。

未生成阶段显示：

> 该阶段尚未生成。

### Workflow Var JSON 非法

例如：

> 工作量评估数据格式异常。

其他 Tab 仍然可用。

### API 请求失败

提供错误说明和 retry。

### stale

明确显示重新生成提示。

禁止 API 失败后静默切到 Mock，让用户误以为是真实结果。

---

## 19. 实现顺序

优先级：

1. 查清真实 XDPai API / Skill；
2. Task ID → Task；
3. Task → Requirement；
4. Task → Workflow Vars；
5. 动态渲染 5 Tab；
6. 编辑 → set workflow var；
7. stale 状态联动；
8. 错误状态；
9. 重跑 AI 节点；
10. 最后再优化 UI。

不要优先重做视觉。

---

## 20. 验收标准

必须实际验证：

1. 输入真实 XDPai Task ID；
2. 成功读取对应 Task；
3. 成功读取 Requirement；
4. 「需求」Tab 显示真实需求正文；
5. 成功读取 5 个 planning workflow vars；
6. 4 个 AI Tab 分别展示对应变量；
7. 页面内容不是静态 Mock；
8. 修改 understanding 并写回；
9. 修改 breakdown 并写回；
10. 修改 estimation，人日汇总实时更新并写回；
11. 修改 scheduling 并写回；
12. 上游修改后下游正确 stale；
13. 刷新后重新读取能看到修改结果；
14. 缺变量 / 非法 JSON / API 失败不会白屏；
15. 浏览器无未处理异常；
16. schema 4.0 不被破坏。

---

## 21. 开发执行要求

开始实现前：

1. 阅读 requirement-arrangement 当前代码；
2. 加载 requirement-arrangement-planner Skill；
3. 加载 xdpai Skill；
4. 查清 Task、Requirement、Workflow Var 的真实读写接口；
5. 先在开发记录中写出真实 API mapping，再编码。

完成后必须汇报：

- 使用的真实 XDPai API / Skill 能力；
- 测试 Task ID；
- 已验证的数据读取链路；
- 已验证的 workflow var 写回链路；
- 已实现能力；
- 尚无法实现的能力及原因。

本需求最核心的验收链路是：

```text
Task ID
→ XDPai Task / Requirement / Workflow Vars
→ 页面
→ 用户修改
→ 写回 XDPai
→ 刷新重新读取成功
```

只完成 UI 不算完成。

---

## 22. 实现结果（2026-11-05）

### 已查证的真实 API（详见 `api.md`）

| 能力 | 网关 / 方法 | 入参 | 浏览器直连 |
|---|---|---|---|
| 读任务（需求） | `xdclaw.woa.com/xdpai.gateway.taskcgi.taskcgi/GetTask` | `{task_id}` | ✅ |
| 读流程实例（vars + 节点） | `xdclaw.woa.com/xdpai.gateway.workflowcgi.workflowcgi/GetInstance` | `{task_id}` | ✅ |
| 写流程变量 | `sop` 后端 `SetWorkflowVar` | `{instance_id,name,value}` | ❌（workflowcgi 未透传） |

关键结论：**读链路完整直连，写链路浏览器无开放接口**（详见 §13 与 `api.md §3`）。

### 已实现

- Task ID 入口（输入框 + `?task_id=` URL 参数 + 自动加载）
- `GetTask` → 「需求」Tab（标题 / ID / 优先级 / 状态 / 正文 Markdown 渲染）
- `GetInstance` → 解析 `variables_json` → 4 个 AI Tab（需求理解 / 任务拆解 / 工作量评估 / 排期生成，含甘特图、关键路径）
- 编辑（内存态）+ 上游修改 → 下游 stale 联动
- 错误状态：Task 不存在 / SOP 未运行 / 变量缺失 / JSON 非法 / API 失败，均不白屏
- **提交变更**：diff（只发人改的字段）+ 回退下游节点重新评估（见 §13，实测打通）
- 防重复提交：提交后刷新快照，重复点击不再发同一条通知

### 尚无法实现

- **浏览器直连写 workflow var**：平台网关未向浏览器开放 `SetWorkflowVar`（原因见 `api.md §3`），当前经「发消息 + AI 助手」间接实现

### 待验证

- 真实 task_id 端到端：`8b4dd104-eec2-40b4-9160-85ae0a7d788b`（已实测读链路 + 发消息链路）

---

## 23. 聊天浮层（与任务会话 AI 对话）

工作台页面提供右下角**聊天浮层**，让人不用跳出页面就能与任务会话里的 AI 对话、查看它的回复——把「人机协作」闭环在页面上。

### 复用与实现

- **复用已有组件契约**：消息历史组件 `message-history`（`cp-67374d13-1832-4abc-abe9-71e96f3199e1`），其契约即 `imcgi` 网关的 `GetMessages`（入参 `conversation_id` / `limit`，返回 `messages`，倒序需 reverse，`text` 取自 `content.text`，`sender_name` 由成员名映射）。
- **发送**：`imcgi` 网关的 `SendMessage`（已在「提交变更」中打通）。
- **UI 风格**：参考 xdpai 前端 `tipai/web` 的聊天样式，不从头造组件。

### 交互

```
右下角悬浮按钮
  → 点击展开浮层（消息列表 + 输入框 + 发送）
  → 消息列表：GetMessages 拉历史（自己靠右、AI 靠左）
  → 输入框发送 → SendMessage → 刷新列表
  → 看到 AI 的回复（发送后刷新 / 手动刷新）
```

### 关键约束

- 浮层只做「读历史 + 发消息」，不做撤回/已读/流式控制（组件契约的边界）
- `conversation_id` 复用「提交变更」的 `conv_id`（来自 `GetInstance` / `GetTask`）
- 消息文本转义展示，不执行消息中的 HTML 或工具输出

