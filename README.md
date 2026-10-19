# 智扫通 · 扫地机器人智能客服 Agent

基于 **LangChain ReAct Agent + RAG + 中间件（Middleware）** 的扫地/扫拖机器人领域智能客服系统。配备 Streamlit 聊天前端，支持专业知识问答、环境适配咨询、个性化使用报告生成等能力。

## ✨ 核心特性

- **ReAct 智能体**：遵循「思考 → 行动 → 观察 → 再思考」流程，自主判断何时调用工具、调用哪个工具，最多 5 轮工具调用补全信息
- **RAG 检索增强**：基于 Chroma 向量库 + `text-embedding-v4` 嵌入模型，检索扫地机器人问答、故障排除、维护保养、选购指南等专业知识并总结作答
- **多工具协同**：7 个专用工具覆盖知识检索、天气查询、用户识别、数据获取与报告生成
- **中间件机制**：工具调用监控日志、模型调用前状态日志、报告场景下**动态切换提示词**
- **知识库增量加载**：基于 MD5 去重，相同文件不会重复入库
- **流式输出**：Streamlit 前端逐字符流式展示回答，支持多轮会话记忆

## 🧰 技术栈

| 类别 | 技术 |
| --- | --- |
| 大模型 | 通义千问 `qwen3-max`（ChatTongyi） |
| 向量化 | DashScope `text-embedding-v4` |
| Agent 框架 | LangChain `create_agent` + LangGraph + Middleware |
| 向量数据库 | Chroma（本地持久化） |
| 文本分片 | RecursiveCharacterTextSplitter |
| Web 前端 | Streamlit |
| 环境管理 | uv（虚拟环境） |
| 配置管理 | YAML（`config/`）+ TXT 提示词（`prompts/`） |

## 🏗️ 项目结构

```
.
├── app.py                  # Streamlit 入口：智能客服聊天界面
├── agent/                  # ReAct 智能体
│   ├── react_agent.py      # Agent 组装（模型 + 提示词 + 工具 + 中间件）
│   └── tools/
│       ├── agent_tools.py  # 7 个 Agent 工具定义
│       └── middleware.py   # 工具监控 / 模型前日志 / 动态提示词切换
├── rag/                    # RAG 模块
│   ├── vector_store.py     # 向量库管理、知识库加载（MD5 去重）
│   └── rag_service.py      # 检索 + LLM 总结链路
├── model/factory.py        # 模型工厂（聊天模型 / 嵌入模型）
├── prompts/                # 提示词模板
│   ├── main_prompt.txt     # 客服主系统提示词
│   ├── rag_summarize.txt   # RAG 总结提示词
│   └── report_prompt.txt   # 使用报告生成提示词
├── config/                 # YAML 配置
│   ├── rag.yml             # 模型名称
│   ├── chroma.yml          # 向量库 / 分片 / 知识库路径
│   ├── prompts.yml         # 提示词文件路径
│   └── agent.yml           # 外部数据路径
├── data/                   # 知识库源文件（txt/pdf）与外部使用记录 csv
├── utils/                  # 配置加载、提示词加载、文件/日志/路径工具
└── logs/                   # 运行日志
```

## 🛠️ Agent 工具一览

| 工具 | 入参 | 说明 |
| --- | --- | --- |
| `rag_summarize` | query | 从向量库检索专业知识并总结 |
| `get_weather` | city | 获取城市实时天气、湿度等环境信息 |
| `get_user_location` | 无 | 获取当前用户所在城市 |
| `get_user_id` | 无 | 获取当前用户 ID |
| `get_current_month` | 无 | 获取当前月份（YYYY-MM） |
| `fetch_external_data` | user_id, month | 查询用户指定月份的机器人使用记录 |
| `fill_context_for_report` | 无 | 报告场景标记，触发中间件注入上下文并切换报告提示词 |

**报告生成固定流程**：`get_user_id` → `get_current_month` → `fill_context_for_report` → `fetch_external_data`，由主提示词强约束，非报告场景严禁调用上下文注入工具。

## 🚀 快速开始

### 1. 环境准备

项目使用 [uv](https://docs.astral.sh/uv/) 管理 Python 虚拟环境（Python 3.12+）：

```powershell
# 创建虚拟环境
uv venv

# 安装依赖
uv pip install streamlit langchain langchain-community langchain-chroma chromadb dashscope pypdf pyyaml python-dotenv
```

> 注意：由 uv 管理的 Python 环境不能直接 `pip install`（PEP 668），务必使用 `uv pip install` 安装到虚拟环境。

### 2. 配置 API Key

项目通过 DashScope（阿里云百炼）调用通义千问模型，运行前需设置环境变量：

```powershell
$env:DASHSCOPE_API_KEY = "sk-your-api-key"
```

### 3. 加载知识库

首次运行时将 `data/` 目录下的 txt/pdf 知识文件导入向量库：

```powershell
uv run python rag/vector_store.py
```

已入库文件会依据 MD5 自动跳过，可重复执行。

### 4. 启动应用

```powershell
uv run streamlit run app.py
```

浏览器访问 `http://localhost:8501` 即可与智能客服对话。

## 💬 使用示例

| 用户输入 | Agent 行为 |
| --- | --- |
| 「扫地机器人总是迷路怎么办？」 | 调用 `rag_summarize` 检索故障排除资料并总结回答 |
| 「今天天气适合用扫地机器人吗？」 | `get_user_location` → `get_weather` → 结合湿度/降雨概率给建议 |
| 「生成我的使用报告」 | 按固定流程调用报告工具链，切换报告提示词生成本月度使用分析报告 |

## ⚙️ 关键配置（`config/chroma.yml`）

```yaml
collection_name: agent      # 向量库 collection
persist_directory: chroma_db # 本地持久化目录
k: 3                        # 检索返回条数
chunk_size: 200             # 分片大小
chunk_overlap: 20           # 分片重叠
allow_knowledge_file_type: ["txt", "pdf"]
```

## 📄 License

本项目仅用于学习交流。
