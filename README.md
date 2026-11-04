# requirement-arrangement

xdpai「需求智能评估与排期」前端 Demo。

## 运行

页面现在从 5 份 JSON 读取数据，因此请通过 HTTP Server 运行：

```bash
python3 -m http.server 5173
```

访问 `http://localhost:5173`。

## 数据输入

- `init.json`：需求基础信息 + 四阶段状态
- `understanding.json`：需求理解 Tab
- `breakdown.json`：任务拆解 Tab
- `estimation.json`：工作量评估 Tab
- `scheduling.json`：排期生成 Tab

四个阶段文件互相独立，通过 `requirement_id / module_id / task_id` 关联。页面不再依赖单体 `planning-result.json`。

## 页面映射

- 需求理解 → understanding.json
- 任务拆解 → breakdown.json
- 工作量评估 → estimation.json
- 排期生成 → scheduling.json

当前页面支持对四个阶段数据进行内存态编辑；真实 XDPai 小应用接入时，将对应编辑写回各自流程变量即可。
