# 数据契约（Data Contract）

> 本文件归档自 GitHub 提交 `0c336d2`（docs: refine XDPai mini app product spec and fixed data contract，2026-11-05 08:59）。
> 该提交是产品 spec 的早期 refine 版本，其中「章节 6-11」为字段级数据契约（schema 4.0），是本地 spec README 未完整文档化的部分。
> 字段最终以 requirement-arrangement-planner Skill 的 schema 4.0 为准，本文件与其保持一致。

---

# XDPai 小应用：需求智能评估与排期

## 1. 背景与目标

当前 `requirement-arrangement` 已有可运行的需求智能评估页面，并已经确定了需求理解、任务拆解、工作量评估、排期生成四个 AI 阶段的数据模型。

本需求要将当前页面建设成一个真正可在 XDPai 中使用的小应用。

用户以一个真实 XDPai Task 为入口，应用从 XDPai 动态读取：

- Task / Requirement 信息；
- 需求智能排期相关 Workflow Vars；

然后展示、校对和编辑 AI 已生成的规划结果，并把修改后的结果写回该 Task 对应的流程变量。

核心链路：

```text
XDPai Task
  ↓
真实 Requirement
  ↓
需求理解
  ↓
任务拆解
  ↓
工作量评估
  ↓
排期生成
  ↓
人工查看 / 修改
  ↓
写回 XDPai Workflow Vars
```

最终交付物应该是一个真实可用的小应用，而不是读取本地 JSON 的静态 Demo。

---

# 2. 本 Spec 中哪些内容已经确定

以下内容是当前产品和数据协议已经达成一致的部分，实现时应保持兼容。

如果实现过程中发现真实 XDPai 能力与这些约定存在冲突，可以提出调整方案，但不要在没有说明的情况下自行换成另一套模型。

已经确定的内容包括：

1. 页面有「需求 + 四个 AI 阶段」五个 Tab；
2. 需求是事实输入，不属于 AI 阶段；
3. 四个 AI 阶段分别维护独立 JSON；
4. 使用 1 个 init Workflow Var + 4 个阶段 Workflow Var；
5. 各 JSON 的字段格式和 ID 关联方式；
6. 上游数据修改后的 stale 传播规则；
7. 页面必须基于真实 Task / Requirement / Workflow Var，而不是生产环境读取 Mock；
8. 真实任务必须用于联调与验收。

以下内容由实现者根据工程判断自行决定：

- 技术栈；
- 前后端具体架构；
- 代码目录组织；
- XDPai API 如何封装；
- 是否需要中间服务；
- 状态管理方案；
- UI 组件拆分；
- Markdown 渲染方案；
- 缓存、并发和性能优化方式；
- 在不改变核心产品语义前提下的页面体验优化。

---

# 3. 真实验收任务

开发和测试不能只使用 Mock 数据。

必须至少使用以下真实 XDPai Task 完成端到端验证：

**Task ID**

```text
8b4dd104-eec2-40b4-9160-85ae0a7d788b
```

**XDPai Task**

```text
https://xdclaw.woa.com/tipai.web.xdpai/#/workbench/a4b2e5de-8287-449c-b771-68d221ea9f5a/1872817c-f9a6-4c45-9bd1-98b20c3797bb/task/8b4dd104-eec2-40b4-9160-85ae0a7d788b
```

该任务已经实际跑过「需求智能排期」SOP，并产生了真实的需求内容和四阶段流程变量。

实现者应通过 XDPai 的真实能力读取该 Task 的数据进行开发和验收，而不是把该任务当前的数据复制进代码中作为静态测试数据。

---

# 4. 数据来源

## 4.1 Task ID 是数据入口

小应用需要能够定位一个 XDPai Task。

至少支持用户输入 Task ID 加载。

如果 XDPai 小应用运行环境存在更自然的上下文传递方式，例如 URL 参数、当前 Task 上下文或页面跳转参数，也可以同时支持。

具体交互方式可以由实现者优化，但不能写死某个 Task ID。

## 4.2 原始需求

「需求」Tab 的事实数据来自当前 Task 实际关联的 Requirement / 需求详情。

需要尽可能保留真实需求中的信息结构，例如：

- 标题；
- 正文；
- Markdown 标题；
- 列表；
- 表格；
- 引用；
- 验收规则；
- 优先级；
- XDPai 可以读取到的其他有价值信息。

页面展示以真实 XDPai Requirement 为事实来源。

## 4.3 AI 评估结果

需求智能排期由以下 5 个 Workflow Vars 表达：

```text
requirement_arrangement.init
requirement_arrangement.understanding
requirement_arrangement.breakdown
requirement_arrangement.estimation
requirement_arrangement.scheduling
```

其中：

| Workflow Var | 含义 |
|---|---|
| requirement_arrangement.init | 公共需求元信息 + 阶段状态 |
| requirement_arrangement.understanding | 需求理解 |
| requirement_arrangement.breakdown | 任务拆解 |
| requirement_arrangement.estimation | 工作量评估 |
| requirement_arrangement.scheduling | 排期生成 |

四个 AI 阶段分别写自己的 Workflow Var，不使用一个不断扩大的单体 Planning JSON。

---

# 5. 页面信息架构

页面包含 5 个 Tab：

```text
需求 | 需求理解 | 任务拆解 | 工作量评估 | 排期生成
```

## 5.1 需求

表示原始事实输入。

它不是 AI 生成阶段，不计入 AI 阶段完成度。

## 5.2 需求理解

回答：

> AI 如何理解这个需求？

## 5.3 任务拆解

回答：

> 为了完成这个需求，需要做哪些研发工作？

## 5.4 工作量评估

回答：

> 每项工作需要多少研发投入，估算可信度如何？

## 5.5 排期生成

回答：

> 这些任务如何安排资源和时间，预计什么时候交付？

五个 Tab 应形成连续的规划链路，而不是五个独立的 JSON 查看器。

---

# 6. 已确定的数据契约

当前数据契约版本：

```text
schema_version = 4.0
```

详细规则同时以 `requirement-arrangement-planner` Skill 为准。

## 6.1 init

Workflow Var：

```text
requirement_arrangement.init
```

格式：

```json
{
  "schema_version": "4.0",
  "requirement": {
    "id": "<requirement-id>",
    "title": "<requirement-title>",
    "description": "<requirement-description>",
    "priority": "medium"
  },
  "stage_status": {
    "understanding": "completed",
    "breakdown": "completed",
    "estimation": "completed",
    "scheduling": "completed"
  },
  "revision": {
    "version": 1,
    "last_modified_by": "ai",
    "changes": []
  }
}
```

阶段状态枚举：

```text
pending | ready | completed | stale
```

---

## 6.2 understanding

Workflow Var：

```text
requirement_arrangement.understanding
```

格式：

```json
{
  "schema_version": "4.0",
  "requirement_id": "<requirement-id>",
  "goal": "<goal>",
  "scope": [
    "<scope item>"
  ],
  "out_of_scope": [],
  "modules": [
    {
      "id": "M1",
      "side": "frontend",
      "name": "<module-name>",
      "repository": "<repository-or-empty>",
      "change_summary": "<one-sentence-change>"
    }
  ],
  "assumptions": [],
  "review_items": [],
  "revision": {
    "version": 1,
    "last_modified_by": "ai",
    "changes": []
  }
}
```

### Module

字段：

- `id`
- `side`
- `name`
- `repository`
- `change_summary`

`side`：

```text
frontend | backend
```

repository 无法从真实上下文确定时允许为空，不应伪造。

### Review Item

支持两种类型：

```text
choice | text
```

Choice 示例：

```json
{
  "id": "Q1",
  "type": "choice",
  "question": "有效对局局数的统计实现落在哪一层？",
  "required": true,
  "ai_assumption": "在任务引擎侧统计",
  "value": null,
  "custom_value": "",
  "options": [
    "任务引擎",
    "数据中心",
    "对局服务"
  ],
  "impact": "影响模块划分、接口契约和工作量"
}
```

「自定义」由 UI 提供，不写入 `options`。

Text 示例：

```json
{
  "id": "Q2",
  "type": "text",
  "question": "TGS 管理端前端仓库与负责人是谁？",
  "required": true,
  "ai_assumption": "沿用现有运营后台",
  "value": "",
  "impact": "影响任务拆解和排期"
}
```

---

# 7. breakdown 数据契约

Workflow Var：

```text
requirement_arrangement.breakdown
```

格式：

```json
{
  "schema_version": "4.0",
  "requirement_id": "<requirement-id>",
  "tasks": [
    {
      "id": "BE-01",
      "module_id": "M3",
      "title": "<task-title>",
      "category": "backend",
      "description": "<task-description>",
      "dependencies": []
    }
  ],
  "revision": {
    "version": 1,
    "last_modified_by": "ai",
    "changes": []
  }
}
```

Task 字段：

- `id`
- `module_id`
- `title`
- `category`
- `description`
- `dependencies`

category：

```text
frontend | backend | other
```

关联规则：

```text
breakdown.tasks[].module_id
        ↓
understanding.modules[].id
```

跨模块联调 / QA 等任务允许：

```json
"module_id": null
```

dependencies 必须引用现有 task ID，并保持无环。

breakdown 中不保存人日、负责人和日期。

---

# 8. estimation 数据契约

Workflow Var：

```text
requirement_arrangement.estimation
```

格式：

```json
{
  "schema_version": "4.0",
  "requirement_id": "<requirement-id>",
  "summary": {
    "total_effort_days": 8,
    "confidence_counts": {
      "high": 3,
      "medium": 4,
      "low": 1
    }
  },
  "estimations": [
    {
      "task_id": "BE-01",
      "effort_days": 2,
      "confidence": "medium",
      "estimate_basis": [
        {
          "item": "领域逻辑",
          "effort_days": 1
        },
        {
          "item": "异常与自测",
          "effort_days": 1
        }
      ],
      "uncertainties": []
    }
  ],
  "revision": {
    "version": 1,
    "last_modified_by": "ai",
    "changes": []
  }
}
```

关联：

```text
estimation.estimations[].task_id
        ↓
breakdown.tasks[].id
```

每个 breakdown task 应恰好对应一条 estimation。

confidence：

```text
high | medium | low
```

必须满足：

```text
sum(estimate_basis.effort_days)
=
estimation.effort_days
```

并且：

```text
sum(all estimation.effort_days)
=
summary.total_effort_days
```

confidence_counts 与实际 estimations 保持一致。

---

# 9. scheduling 数据契约

Workflow Var：

```text
requirement_arrangement.scheduling
```

格式：

```json
{
  "schema_version": "4.0",
  "requirement_id": "<requirement-id>",
  "summary": {
    "start_date": "2026-11-05",
    "delivery_date": "2026-12-10",
    "estimated_working_days": 26,
    "critical_path": [
      "BE-01",
      "BE-07",
      "QA-01"
    ]
  },
  "schedule": [
    {
      "task_id": "BE-01",
      "assignee": "后端待分配-A",
      "start_date": "2026-11-05",
      "end_date": "2026-11-09"
    }
  ],
  "revision": {
    "version": 1,
    "last_modified_by": "ai",
    "changes": []
  }
}
```

关联：

```text
scheduling.schedule[].task_id
        ↓
breakdown.tasks[].id
```

每个 breakdown task 应恰好对应一个 schedule item。

日期：

```text
YYYY-MM-DD
```

依赖关系必须满足排期顺序。

`critical_path` 只能引用真实 task ID。

如果真实人员未知，assignee 可以是角色 / 资源级占位，例如：

```text
后端待分配
前端待分配
测试待分配
```

占位资源不应被解释为真实员工。

---

# 10. 跨 JSON 关联关系

核心关系固定为：

```text
init.requirement.id
  ├── understanding.requirement_id
  ├── breakdown.requirement_id
  ├── estimation.requirement_id
  └── scheduling.requirement_id

understanding.modules[].id
  ↓
breakdown.tasks[].module_id

breakdown.tasks[].id
  ├── breakdown.tasks[].dependencies[]
  ├── estimation.estimations[].task_id
  ├── scheduling.schedule[].task_id
  └── scheduling.summary.critical_path[]
```

页面可以在展示层 join 数据，但不要把完整 task 重复复制到多个 Workflow Var 中。

---

# 11. Agent / SOP 对流程变量的写入职责

当前 SOP 有四个 AI 阶段。

各阶段写入职责已经确定：

## 需求理解 Agent

主要写：

```text
requirement_arrangement.understanding
```

并负责初始化 / 更新必要的：

```text
requirement_arrangement.init
```

## 任务拆解 Agent

主要写：

```text
requirement_arrangement.breakdown
```

并更新 init 中对应阶段状态。

## 工作量评估 Agent

主要写：

```text
requirement_arrangement.estimation
```

并更新 init 中对应阶段状态。

## 排期生成 Agent

主要写：

```text
requirement_arrangement.scheduling
```

并更新 init 中对应阶段状态。

下游 Agent 不应为了方便重新生成并覆盖上游阶段 JSON。

---

# 12. 页面展示和编辑

## 12.1 需求 Tab

展示真实 Requirement。

重点包括：

- 需求标题；
- Task ID；
- Requirement ID；
- 优先级；
- 原始需求正文。

需求正文需要保持良好的结构化阅读体验，包括 Markdown 表格、标题、列表等。

## 12.2 需求理解 Tab

主要展示 / 编辑：

- goal
- scope
- out_of_scope
- modules
- assumptions
- review_items

review item 需要能够完成人工回答。

## 12.3 任务拆解 Tab

主要展示 / 编辑：

- task
- module
- category
- description
- dependencies

支持合理的任务增删改。

## 12.4 工作量评估 Tab

主要展示 / 编辑：

- effort_days
- confidence
- estimate_basis
- uncertainties

页面应能够根据修改实时反映：

- total_effort_days
- confidence_counts

## 12.5 排期生成 Tab

需要提供适合研发审核的排期视图。

至少能够理解：

- task
- assignee / resource
- effort
- start / end
- dependency
- critical path
- delivery date

现有甘特图可以继续演进，但具体交互方式由实现者根据实际体验判断。

---

# 13. 阶段状态与 stale 语义

`requirement_arrangement.init.stage_status` 用于描述四个 AI 阶段的数据是否可用。

状态：

```text
pending
ready
completed
stale
```

基本语义：

- pending：尚未到该阶段；
- ready：前置条件已满足，可以生成；
- completed：当前结果有效；
- stale：结果曾经生成，但上游数据已发生会影响本阶段的变化，需要重新生成。

## 上游修改后的传播

### understanding 发生影响后续规划的修改

```text
understanding = completed
breakdown = stale
estimation = stale
scheduling = stale
```

### breakdown 修改

```text
breakdown = completed
estimation = stale
scheduling = stale
```

### estimation 修改

```text
estimation = completed
scheduling = stale
```

### scheduling 修改

只影响 scheduling 自身。

实际实现可以根据修改内容做更精确的失效判断，但不能让明显已经失效的下游结果继续显示为 completed。

---

# 14. AI 阶段完成度与人工确认是两个概念

页面需要区分：

### AI 阶段生成情况

例如：

```text
阶段生成 4 / 4
```

### 人工仍需确认的问题

例如：

```text
5 项待确认
```

即使四个 AI 阶段全部 completed，只要 required review_items 尚未回答，就不能把产品语义表达成“整个评估已经完全确认”。

---

# 15. 数据保存

页面中的编辑不是纯前端草稿。

用户保存某一个 AI 阶段的数据后，应写回该 Task 对应的 Workflow Var。

基本对应关系：

| Tab | 保存目标 |
|---|---|
| 需求理解 | requirement_arrangement.understanding |
| 任务拆解 | requirement_arrangement.breakdown |
| 工作量评估 | requirement_arrangement.estimation |
| 排期生成 | requirement_arrangement.scheduling |

如果修改会造成下游 stale，需要同时更新：

```text
requirement_arrangement.init
```

刷新 / 重新进入同一个 Task 后，应能够读取到刚才真实保存后的值。

---

# 16. 缺失、异常和部分数据

真实 Task 不一定永远有完整的五份变量。

产品需要合理处理：

- Task 不存在；
- Requirement 获取失败；
- SOP 尚未运行；
- 某个阶段尚未生成；
- 单个 Workflow Var 缺失；
- 单个 Workflow Var JSON 非法；
- 某个阶段 stale；
- XDPai 请求失败；
- 写回失败。

某一个阶段异常不应无条件导致整个应用不可使用。

例如：

- Requirement 已经存在但 SOP 尚未运行时，仍然应该能查看「需求」；
- estimation 异常时，需求理解和任务拆解仍然可以查看。

具体错误 UI 和恢复方式由实现者设计。

---

# 17. 关于重新运行 AI

产品希望用户能够在上游修改后重新生成 stale 的阶段。

具体是否可以直接从小应用触发某个 SOP 节点，需要以 XDPai 当前真实能力为准。

如果真实 API 支持，可以提供合适的重跑能力。

如果当前能力不支持，应如实表达限制，不要求为了满足 UI 而模拟一个不存在的重跑动作。

---

# 18. Skill 与 Schema

需求智能排期数据契约已经沉淀在：

```text
requirement-arrangement-planner
```

当前 schema：

```text
4.0
```

应用应与 Skill 输出保持兼容。

如果开发过程中发现 Skill schema 本身存在明显问题，可以提出并同步修改 Skill 和本 Spec，但不要只修改前端形成新的隐式数据协议。

---

# 19. 真实任务验收要求

必须使用本 Spec 第 3 节给出的真实 Task 做实际联调。

至少验证：

### 读取

- 能定位真实 Task；
- 能读取真实需求详情；
- 能读取当前真实的 init；
- 能读取 understanding；
- 能读取 breakdown；
- 能读取 estimation；
- 能读取 scheduling。

### 展示

使用该真实 Task 时：

- 「需求」展示真实需求；
- 「需求理解」展示真实模块和 review items；
- 「任务拆解」展示真实任务；
- 「工作量评估」展示真实估算；
- 「排期生成」展示真实排期。

不能把目前仓库中的相同数据硬编码到页面后声称已经完成联调。

### 编辑与写回

至少实际完成一次安全的 Workflow Var 修改验证：

```text
读取原值
→ 修改
→ 写回
→ 重新读取
→ 确认新值生效
```

测试时避免无意义破坏真实规划数据。

如果需要做会改变业务结果的测试，应选择可恢复字段，或者测试后恢复原值。

### 一致性

页面读取到的阶段 JSON 应保持前述 ID 关系和 summary 一致性。

---

# 20. 验收结果

完成后，应能证明：

1. 页面已经脱离静态 JSON 作为正式数据源；
2. 输入 / 定位真实 Task 后能够动态加载数据；
3. 五个 Tab 使用真实 Task 数据渲染；
4. Workflow Var 能够真实写回；
5. 刷新后重新读取的数据与保存结果一致；
6. 上游修改后的 stale 语义正确；
7. 部分数据缺失或异常时产品仍有合理行为；
8. 当前真实测试 Task 可以完整使用；
9. 代码实现与本 Spec 和 requirement-arrangement-planner schema 保持一致。

---

# 21. 实现者的设计空间

本 Spec 定义产品目标、数据契约、数据来源和验收语义。

除此之外，实现者拥有充分的工程和产品实现自由度。

可以根据对现有代码、XDPai 平台能力和用户体验的判断：

- 重构现有代码；
- 改变技术实现；
- 优化页面布局；
- 改善交互；
- 增加合理的校验和保护；
- 提出比当前 Demo 更好的实现方案。

如果发现本 Spec 本身存在不合理之处，也应指出问题并提出修改，而不是机械执行。

最终判断标准不是“是否逐字照着文档写代码”，而是：

> 是否在保持已确定的数据契约和核心产品语义的前提下，交付了一个真实、可靠、易用的 XDPai 需求智能评估与排期小应用。
