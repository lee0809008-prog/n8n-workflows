<!-- .github/copilot-instructions.md - guidance for AI coding agents -->
# 快速上手说明（给 AI 编码代理）

这是为 AI 编码助手准备的、可直接用于贡献与导航此代码库的精简说明。重点是：架构要点、开发/调试/构建命令、项目约定和关键文件示例。请只基于仓库可观察到的内容给出实作建议。

核心概览
- **用途**: 本仓库是一个面向浏览与搜索大量 n8n 工作流的文档和索引系统（约 2k+ workflows）。主要功能是把 workflows/*.json 索引到 SQLite（FTS5），并通过 FastAPI 提供检索与可视化接口。
- **主要组件**:
  - `workflow_db.py`：索引、FTS5 搜索、文件散列（MD5）检测、格式化文件名等的核心逻辑。
  - `api_server.py`：FastAPI HTTP API，提供检索、下载、Mermaid 图生成和后台重建索引接口。
  - `run.py`：项目启动脚本（封装依赖检查、目录创建、初始化及启动 uvicorn），推荐用于本地快速启动。
  - `workflows/`：包含所有 n8n workflow JSON 文件（递归查找）。
  - `static/`：前端静态文件；若缺失，`/` 路由会返回提示。
  - `context/` 目录：包含分类定义（例如 `def_categories.json`、`search_categories.json`），用于分类/映射。

重要运行与开发命令（在仓库根目录）
```powershell
# 创建虚拟环境（Windows）
python -m venv .venv
.\.venv\Scripts\activate; pip install -r requirements.txt

# 本地启动（推荐）
pip install -r requirements.txt
python run.py            # 默认: http://127.0.0.1:8000

# 开发模式（自动重载 + 强制重建索引）
python run.py --dev --reindex

# 或直接运行 API 服务器（调试）
python api_server.py --reload

# 单独索引（workflow_db CLI）
python workflow_db.py --index --force

# 导入到 n8n（如果需要）
python import_workflows.py
```

关键环境变量与路径
- `WORKFLOW_DB_PATH`: 如果未传入，`WorkflowDatabase` 默认使用 `workflows.db`；`run.py` 在启动时会将其设为 `database/workflows.db`。AI 代理在修改路径或启动脚本时请保持一致。
- `workflows/`：工作流 JSON 的顶级目录；索引逻辑使用 `Path('workflows').rglob('*.json')`（递归匹配），所以支持子目录。

索引与数据模型要点（来自 `workflow_db.py`）
- 索引行为：基于文件 MD5（`get_file_hash`）决定是否重新处理；`index_all_workflows(force_reindex=True)` 可强制重建。
- 数据库存储：主表 `workflows` + FTS5 虚拟表 `workflows_fts`，并通过触发器保持同步。修改 schema 或触发器需同时更新触发器 SQL。
- 名称处理：`format_workflow_name(filename)` 会去掉前缀编号并做特殊词（HTTP/API/Webhook）大小写修正 —— 在生成名称或文案时优先考虑 JSON 内的 `name` 字段（若有且有意义）。

API 与行为約定（来自 `api_server.py`）
- 基本端点：`/api/workflows`（搜索）、`/api/stats`、`/api/categories`、`/api/reindex`、`/api/workflows/{filename}`（详情/原始 JSON）、`/api/workflows/{filename}/diagram`（Mermaid）。
- 文件查找：API 通过在 `workflows/` 下递归查找匹配 `filename` 的文件（严格按文件名匹配）。不要假设数据库中条目仍保证文件存在，代码会在找不到文件时返回 404。
- Mermaid 生成：`generate_mermaid_diagram` 使用 `nodes` 与 `connections` 字段构建图形，代理在更改渲染逻辑时应参考该函数的节点类型和样式决策。

项目特定約定与最佳实践（可直接采用）
- 文件命名模式：`[ID]_[Service1]_[Service2]_[Purpose]_[Trigger].json`。索引器会把 ID 去掉並智能大写常见缩写（`HTTP`, `API` 等）。
- 集成识别：`analyze_nodes` 中维护了丰富的 `service_mappings`（例如 `telegram` → `Telegram`），新增服务时请一并更新该映射以保证分类准确。
- 分类来源：`context/search_categories.json` 與 `context/def_categories.json` 为分类/映射权威源；更改分类脚本请更新 `create_categories.py`。

注意事項与變更风险
- 修改数据库 schema、FTS 列或触发器会破坏索引同步；若必须修改，确保同时更新 `init_database` 内的触发器 SQL 和任何依赖查询。
- 索引性能与 PRAGMA：`init_database` 设置了多个 PRAGMA（WAL、cache_size 等），不建议移除这些以避免性能回退。
- 敏感信息：workflow JSON 可能包含 webhook 或第三方凭证占位；AI 代理在生成补丁或代码片段时不要泄露或默认写入敏感值。

示例任務（agent-friendly）
- “添加新的分类映射”：修改 `workflow_db.py::get_service_categories`，运行 `python workflow_db.py --index --force`，并验证 `GET /api/categories` 返回预期。
- “修复 API 中的 404 场景”：检查 `api_server.py` 的文件查找路径（`workflows_path.rglob('*.json')`），对缺失文件的异常处理应返回清晰日志和 HTTP 404。

反馈请求
如果本文件有遗漏（例如你希望包含其他脚本的执行顺序、CI/CD 细节或更详细的调试步骤），请指出要补充的具体区域，我会基于仓库代码迭代更新。
