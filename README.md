# Diet-RAG-Agent

> **一个「先规划、再检索、后作答」的中文饮食智能 Agent。** 基于 LangGraph 多节点编排 + RAG，一套框架承载菜谱推荐、营养分析、教程问答与 B 站视频知识入库，每个阶段可观测、可替换、可评测。

![Python](https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white)
![LangGraph](https://img.shields.io/badge/Agent-LangGraph-111827)
![FastAPI](https://img.shields.io/badge/API-FastAPI-009688?logo=fastapi&logoColor=white)
![ChromaDB](https://img.shields.io/badge/Retrieval-ChromaDB-6E56CF)
![PostgreSQL](https://img.shields.io/badge/Storage-PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![Status](https://img.shields.io/badge/Status-Active-2EA043)

---

## 它能做什么

| 能力 | 说明 |
| --- | --- |
| **菜谱推荐** | 基于食材、时长、健康目标、餐次等约束，生成 grounded 推荐 |
| **营养分析** | 面向减脂、控糖、高蛋白等目标，给出保守型营养建议 |
| **食材搭配检查** | 区分「证据结论」与「通用建议」，避免无依据强判断 |
| **教程问答** | 基于 tutorial 知识库回答「怎么做 / 步骤 / 教程」类问题 |
| **B 站视频知识入库** | 把视频 / 教程转成可检索的结构化知识（JSON / PDF / chunk） |
| **多轮承接** | 读回历史消息、反馈信号、推荐锚点与稳定偏好 |

---

## 架构：一次请求怎么走

LangGraph 多节点主链路，不是单轮黑盒调用：

```
Router → Planner → Retriever → Generator → Evaluator
```

- **Router** 识别意图并选择 `active_skill`
- **Planner** 决定澄清、直接承接还是进入检索
- **Retriever** 按 skill 契约执行检索、过滤与精排
- **Generator** 基于证据生成回答，或执行 tutorial / video 工作流
- **Evaluator** 需要时做质量检查与有限重试

每个阶段的输入 / 输出 / 短路条件都可独立观测与调试。

### Skill-aware：一套 graph，多种技能

项目的关键不是「多了几个节点」，而是把节点行为绑定到 `active_skill`：

- **retrieval profile**：菜谱检索、教程检索、视频知识入库使用不同路径
- **hard filter / rerank bias**：不同技能对时长、目标、证据边界的约束不同
- **evidence boundary**：某些回答必须严格基于检索证据，不能越界扩写

---

## 数据分层

不同数据放进不同层，是为了降低耦合、控制成本、让每层只做擅长的事：

| 数据层 | 典型内容 | 作用 |
| --- | --- | --- |
| 向量知识（ChromaDB） | recipes、recipe_tutorials、bilibili_tutorials | 语义召回，不存强结构关系 |
| 结构化持久化（PostgreSQL） | 用户偏好、交互记录、反馈 | 强结构、可追溯、可查询 |
| 运行时记忆（内存） | session history、反馈信号、推荐锚点 | 低延迟承接多轮上下文 |
| 产物层（JSON / PDF / dataset） | tutorial JSON、PDF、离线 benchmark 集 | 可审计、可复现、可离线处理 |

**降级策略**：PostgreSQL 不可用仍可基于向量库 + 内存工作；tutorial / video 导入失败不破坏普通问答链路；`/chat` 与 `/chat/stream` 共用同一 graph 主线。

---

## 快速开始

```powershell
conda create -n myagent python=3.11 -y && conda activate myagent
pip install -r requirements.txt
```

复制 `.env.example` 为 `.env`，至少填 `DASHSCOPE_API_KEY`。

初始化知识库并启动：

```powershell
python scripts/init_database.py          # 完整数据可用 --recipes-file data/recipes_v2.json
uvicorn src.api.main:app --host 0.0.0.0 --port 8000
```

启动后访问：

- `http://127.0.0.1:8000/demo` — 单文件 Web 对话 Demo
- `http://127.0.0.1:8000/docs` — OpenAPI 接口文档
- `POST /chat`（完整响应）/ `POST /chat/stream`（SSE 节点级流式）

---

## API

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| `GET` | `/health` | 健康检查（graph / postgres / chromadb / langfuse） |
| `GET` | `/demo` | 单文件 Web Demo |
| `POST` | `/chat` | 完整回答 + metadata |
| `POST` | `/chat/stream` | SSE 流式：chunk + 节点进度事件 |
| `GET` | `/users/{uid}/sessions` | 会话列表 |
| `GET` | `/users/{uid}/sessions/{sid}/history` | 会话历史恢复 |

---

## 工程亮点

- **多节点显式编排**，而不是单轮黑盒调用——每个阶段可观测、可调试、可替换
- **Skill Contract 驱动的统一框架**——不同任务共享一套 graph，按 `active_skill` 切换策略与约束
- **数据分层清晰**——Chroma 管可检索知识、PostgreSQL 管结构化状态、内存管低延迟承接、JSON/PDF 管可审计产物
- **从外部内容到内部知识的闭环**——tutorial / 视频不仅能被总结，还能转成可再检索的知识资产
- **一条链路覆盖工程面**——API、Graph 编排、RAG、记忆、结构化持久化、可观测性与 benchmark 评测

---

## 仓库结构

```
├── src/
│   ├── api/             # FastAPI、SSE、Schema、Demo
│   ├── graph/           # LangGraph 主图、state、nodes、edges
│   ├── context/         # 上下文拼装、记忆管理
│   ├── database/        # PostgreSQL 客户端
│   ├── evaluation/      # benchmark 与评测逻辑
│   ├── retriever/       # 检索与精排
│   ├── tutorials/       # tutorial / video 处理与导出
│   ├── vectorstore/     # Chroma 接入、检索实现
│   ├── observability/   # 结构化日志 / Langfuse
│   └── utils/
├── diet_agent/          # runtime、skills、integrations、user memory
├── data/                # recipe / tutorial / benchmark 数据
├── scripts/             # 初始化、tutorial/video 构建、PDF 渲染、benchmark
└── web_chat_demo.html   # 单文件 Web Demo
```

## License

[MIT](./LICENSE)
