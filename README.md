# requirement-arrangement

xdpai「需求智能评估与排期」前端 Demo。

## 运行

这是一个零依赖静态页面。

- 直接双击 `index.html`
- 或在目录下运行 `python3 -m http.server 5173`，访问 `http://localhost:5173`

## 当前能力

- 需求概览与核心指标
- 任务拆解 + 甘特图一体化展示
- 按开发类型 / 负责人切换分组
- 关键路径高亮开关
- 点击任务查看详情、依赖、AI 估算依据和风险
- 调整任务人日并联动刷新总人日
- 确认排期 Demo 交互

## 数据协议

`planning-result.json` 是 Skill → 页面之间的 V1 数据协议示例。

当前 `index.html` 内嵌同结构 Mock 数据，以保证双击即可运行。下一步接入真实 Skill 时，只需将这份数据替换为 Skill 输出即可。

> 当前只是前端 Demo，不会真正创建 TAPD / SOP 任务。
