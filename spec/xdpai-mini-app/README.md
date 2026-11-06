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

展示：

- goal
- scope
- out_of_scope
- frontend modules
- backend modules
- assumptions / confirmed rules（如果后续 schema 拆分，则按最新 Skill）
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

> 曾评估「页面让用户填个人 token 后浏览器直连」，实测不可行：带 token 跨域到 `apipai.woa.com` 仍被 CORS 拦截。CORS 与 token 无关，JS 无法绕过。

**后续待办**：平台在 `workflowcgi` 透传 `SetWorkflowVar` 后，页面改为浏览器直连写回（`GetInstance → SetWorkflowVar`），不依赖 AI 助手、可实时校验。当前「发消息 + 回退」依赖会话中 AI 助手在线且理解指令，作为过渡方案。

### 防重复提交

「提交变更」成功后**刷新原始快照**（`saveOriginal`），再次点击时 diff 为空，提示「未检测到改动」——不会把同一批 diff 重复发给 agent。

### 对话浮层（与 AI 对话，复用 tipai/web 组件 + imcgi 消息接口）

页面右下角提供「与 AI 对话」悬浮按钮，点开浮层即可在页面内直接和任务会话里的 AI 对话、看到它的回复（含节点流转）。复用 `git.woa.com/tipai/web` 的聊天组件设计 + `imcgi` 消息接口契约，未从零自研。

- **入口**：右下角悬浮按钮，仅在获取到会话 ID（`conv_id`）后显示
- **消息列表**：三类气泡
  - 自己发的（`sender_id` == 当前 uid）靠右紫色气泡，纯文本
  - AI（`sender_type` = silicon/agent，或 `type` = agent_response）带「AI」标签，正文 **Markdown 渲染**（标题 / 列表 / 代码块 / 表格 / 引用 / 加粗 / 行内代码）
  - 系统（`type` = system，或 `sender_type` = system）居中灰色卡片，承载**节点流转 / 状态变更**展示（如「🎉 SOP 流程已全部完成」「何艺萍 变更状态：审核中 → 已完成」）
- **拉取**：`GetMessages` 拉会话历史（返回倒序 → 恢复正序）
- **发送**：`SendMessage` 发到任务会话，Enter 发送 / Shift+Enter 换行，发送后自动刷新
- **刷新**：手动刷新按钮拉最新回复
- **布局（可拖动 + 调整大小，设计待实现）**：浮层当前 `position:fixed` 固定右下角（`right:24px; bottom:150px; width:380px; height:520px`），不可拖动、不可调大小。规划支持：
  - 拖动：标题栏作为 drag handle，可拖到页面任意位置
  - 调整大小：右下角 resize handle，可拖拽改宽高，设最小尺寸（如 320×400）
  - 位置 / 尺寸记忆到 `localStorage`，下次打开保持

### 流式（折中：轮询，真流式在小应用域不可行）

tipai/web 的真流式走 WebSocket 通道 `wss://xdclaw.woa.com/xdpai.gateway.ws.ws/?token=<t_uid>`，订阅 `agent:stream`（text/thinking chunk 增量）+ `agent:stream_end`。但该通道在小应用域**不可行**：

1. `t_uid` 是 `xdclaw.woa.com` 域的登录 cookie，`app.xdpai.woa.com` 跨域读不到（实测 cookie 列表无 `t_uid`）；
2. xdpai-apps `check` 硬约束禁止代码读取登录 cookie。

折中：浮层打开时**每 3 秒轮询 `GetMessages`**，检测到 `status=sending` 的 AI 消息时显示「正在回复」+ 光标闪烁，回复落库后自动刷新出完整消息（带 Markdown 渲染）。关闭浮层即停轮询。

> 真流式待办：平台提供免 token 的流式通道（如 imcgi 加 SSE 拉流，或 WS 网关加 CORS + 跨域 token 透传）后，可替换轮询为逐字流式。

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
| 发消息 | `xdclaw.woa.com/xdpai.gateway.imcgi.imcgi/SendMessage` | `{conversation_id, sender_type, type, content:{text}}` | ✅ |
| 读消息 | `xdclaw.woa.com/xdpai.gateway.imcgi.imcgi/GetMessages` | `{conversation_id, limit}` | ✅ |

关键结论：**读链路完整直连，写链路经「发消息 + AI 助手」间接实现**（详见 §13 与 `api.md §3`）。

### 已实现

- Task ID 入口（输入框 + `?task_id=` URL 参数 + 自动加载）
- `GetTask` → 「需求」Tab（标题 / ID / 优先级 / 状态 / 正文 Markdown 渲染）
- `GetInstance` → 解析 `variables_json` → 4 个 AI Tab（需求理解 / 任务拆解 / 工作量评估 / 排期生成，含甘特图、关键路径）
- 编辑（内存态）+ 上游修改 → 下游 stale 联动
- 错误状态：Task 不存在 / SOP 未运行 / 变量缺失 / JSON 非法 / API 失败，均不白屏
- **提交变更**：diff（只发人改的字段）+ 回退下游节点重新评估（见 §13，实测打通）+ 防重复提交
- **对话浮层**：右下角悬浮按钮，imcgi 消息接口，气泡式列表（自己 / AI / 系统三类），AI 消息 Markdown 渲染，系统消息节点流转展示，输入发送 + 刷新（见 §13「对话浮层」）
- **流式折中**：浮层打开每 3 秒轮询 `GetMessages`，检测 `status=sending` 显示「正在回复」，回复落库自动刷新（真流式在小应用域不可行，见 §13「流式」）

### 尚无法实现

- **浏览器直连写 workflow var**：平台网关未向浏览器开放 `SetWorkflowVar`（原因见 `api.md §3`），当前经「发消息 + AI 助手」间接实现
- **真流式**：tipai/web 的 WS 通道需 `t_uid` cookie（跨域读不到）+ 触发 check 硬约束，折中为轮询；待平台提供免 token 流式通道

### 待验证

- 真实 task_id 端到端：`8b4dd104-eec2-40b4-9160-85ae0a7d788b`（已实测读链路 + 发消息链路 + 对话浮层收发消息）

---

## 23. 动效（设计，待实现）

现状痛点：进入页面「闪现」、切 Tab「闪切」，无任何过渡，显得生硬。

### 进入页面（加载）

- 加载中：hero 卡片 / 阶段进度条 / Tab 内容区显示**骨架屏**（灰底 shimmer 微光扫过），替代空白等待
- 数据就绪后：整卡 **fade + 轻微上移**（`opacity 0→1` + `translateY 8px→0`，约 300ms）
- **错峰**：hero → 进度条 → 主内容，各延迟 60~80ms 依次浮现，有节奏不突兀

### Tab 切换

- 内容区**交叉淡入淡出**：旧内容快速淡出（约 100ms）→ 新内容淡入 + 上移（约 200ms）
- Tab 高亮下划线 / 背景色用 `transition` 平滑滑动，不跳变
- 表格 / 甘特图进入时轻微渐入，避免整块「啪」地出现

### 细节动效（克制的点缀）

- 甘特图任务条：进入时从左向右「生长」（`scaleX` 或宽度过渡）
- 阶段状态标签（completed / stale）：背景色平滑过渡
- 进度条填充：宽度过渡
- 按钮 hover / active：轻微按压反馈

### 技术约束

- 纯 CSS `transition` + `@keyframes`，不引第三方动画库（保持自包含单文件）
- 尊重 `prefers-reduced-motion: reduce`：系统开启「减弱动态效果」时自动关闭动画
- 时长克制（100~300ms），不做弹跳 / 过冲类花哨动效

