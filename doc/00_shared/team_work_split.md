# 四人分工与模块对接

本项目要求每个人都有明确编程任务。资料整理和清洗只作为辅助工作，不能成为某个成员的主要任务。每个成员都要交付可以运行的代码、接口和测试样例。

## 总体分工表

| 成员 | 模块 | 主要代码 | 必须对接 |
| --- | --- | --- | --- |
| 成员 A | CLI + Agent Core | `ie_agent/cli/`, `ie_agent/agent/` | B 的 RAG，C 的工具，D 的用量 |
| 成员 B | RAG Service | `ie_agent/rag/` | A 的 Agent，D 的评估 |
| 成员 C | Tools + MCP/Skill | `ie_agent/tools/`, `ie_agent/extensions/` | A 的 Agent，D 的用量 |
| 成员 D | Usage + Web + Evaluation | `ie_agent/usage/`, `ie_agent/web/`, `ie_agent/evaluation/` | A/B/C 全部 |

## 成员 A：CLI + Agent Core

### 负责目标

成员 A 负责项目主入口和 Agent 主流程。用户看到的 CLI 基本都由成员 A 实现。成员 A 不需要自己实现 RAG、专业工具、预算细节，但必须把这些模块串起来。

### 需要实现的文件

```text
ie_agent/cli/main.py
ie_agent/cli/commands.py
ie_agent/cli/renderer.py
ie_agent/cli/session_store.py
ie_agent/agent/orchestrator.py
ie_agent/agent/router.py
ie_agent/agent/prompt_builder.py
ie_agent/agent/response_builder.py
ie_agent/models/remote_model.py
ie_agent/models/model_manager.py
```

### 具体编程任务

1. 实现 `ie-agent chat`。
2. 实现连续对话循环：
   - 读取用户输入。
   - 判断是否是 slash command。
   - 普通问题封装为 `AgentRequest`。
   - 调用 `AgentOrchestrator.run()`。
   - 用 Rich 输出答案。
3. 实现 slash commands：
   - `/model`
   - `/rag`
   - `/trace`
   - `/source`
   - `/session`
4. 实现 Agent 路由：
   - 根据关键词判断课程标签。
   - 根据问题类型判断 intent。
   - 判断是否需要 RAG。
   - 判断是否需要工具。
   - 判断是否使用 remote 或 rule。
5. 实现 prompt 构造：
   - 把用户问题、历史对话、RAG chunk、工具结果组织成 prompt。
6. 实现远程模型调用：
   - 支持 OpenAI-compatible API。
   - 支持 DeepSeek 或 Qwen。
   - 支持环境变量读取 API key。
7. 实现异常处理：
   - 远程模型失败时给出错误说明。
   - RAG 无结果时不编造引用。
   - 工具失败时把错误交给模型解释。

### 对外接口

```python
class AgentOrchestrator:
    def run(self, request: AgentRequest) -> AgentResponse:
        ...
```

成员 A 必须保证其他模块只需要调用这个接口就可以得到最终回答。

### 需要调用其他成员接口

成员 B：

```python
rag_result = rag_service.search(RagQuery(...))
```

成员 C：

```python
tool_call = tool_registry.run(tool_name, arguments)
```

成员 D：

```python
status = usage_manager.check(UsageAction(...))
usage_manager.record(UsageRecord(...))
```

### 验收标准

- `ie-agent chat` 能进入对话。
- 输入普通问题能返回回答。
- 输入 `/trace on` 后能显示路由结果。
- 输入需要工具的问题时能调用 mock tool。
- 输入需要 RAG 的问题时能调用 mock RAG。
- 远程 API 不可用时，不会让程序崩溃。

## 成员 B：RAG Service

### 负责目标

成员 B 负责知识库模块。重点不是整理大量资料，而是做出完整可运行的 RAG 工程链路：导入、分块、embedding、存储、检索、引用。

### 需要实现的文件

```text
ie_agent/rag/ingest.py
ie_agent/rag/chunker.py
ie_agent/rag/embeddings.py
ie_agent/rag/vector_store.py
ie_agent/rag/retriever.py
data/raw/
data/processed/
data/vector_store/
```

### 具体编程任务

1. 实现资料导入：
   - 支持 `.txt`
   - 支持 `.md`
   - 支持 `.pdf`
2. 实现文本分块：
   - 默认 chunk size 500 到 800 中文字符。
   - chunk overlap 50 到 100 字符。
   - 每个 chunk 保留来源文件、页码、课程标签。
3. 实现 embedding：
   - 优先使用本地 sentence-transformers。
   - 如果本地模型不可用，可以使用简单 fallback，例如 TF-IDF 或关键词匹配。
4. 实现向量库：
   - FAISS 或 Chroma 二选一。
   - 支持保存到 `data/vector_store/`。
   - 支持重新加载。
5. 实现检索：
   - 输入 `RagQuery`。
   - 按 `course_tags` 过滤。
   - 返回 top_k 个 `RagChunk`。
6. 实现引用格式：
   - 文件名。
   - 页码或段落号。
   - chunk_id。
7. 准备少量示例资料：
   - 每门课程 1 到 2 个小文件即可。
   - 示例资料可以是小组自己整理的短笔记，不需要大规模清洗。

### 对外接口

```python
class RagService:
    def ingest(self, path: str, course_tags: list[str] | None = None) -> IngestResult:
        ...

    def search(self, query: RagQuery) -> RagResult:
        ...
```

### 需要和成员 A 对接

成员 A 会传入：

```python
RagQuery(
    query="离散卷积 定义 计算方法",
    course_tags=["signals"],
    top_k=5,
)
```

成员 B 返回：

```python
RagResult(
    chunks=[...],
    citations=[...],
    confidence=0.73,
)
```

### 需要和成员 D 对接

成员 D 需要统计：

- 检索是否有结果。
- 引用是否来自正确课程。
- top_k chunk 是否相关。

因此成员 B 要保留 `score` 和 `course_tags`。

### 验收标准

- 能导入至少 4 个示例资料文件。
- 能对每个资料生成 chunk。
- 能按课程标签检索。
- 能返回引用。
- RAG 服务即使没有向量库，也要有 fallback 检索，保证展示能跑。

## 成员 C：Tools + MCP/Skill

### 负责目标

成员 C 负责专业工具库和扩展系统。这是项目体现 Agent 能力的重要部分。工具必须能被 Agent 统一调用，不能只是单独脚本。

### 需要实现的文件

```text
ie_agent/tools/registry.py
ie_agent/tools/circuits.py
ie_agent/tools/signals.py
ie_agent/tools/digital_logic.py
ie_agent/tools/information_theory.py
ie_agent/extensions/mcp_manager.py
ie_agent/extensions/skill_manager.py
ie_agent/extensions/permission.py
```

### 专业工具任务

电路工具：

- `ohms_law`
- `voltage_divider`
- `rc_time_constant`

信号系统工具：

- `discrete_convolution`
- `linear_convolution_length`
- `dft_basic`

数字逻辑工具：

- `truth_table`
- `boolean_simplify_basic`
- `binary_convert`

信息论工具：

- `entropy`
- `mutual_information`
- `channel_capacity_bsc`

每个工具都要返回：

- 输入参数。
- 计算公式。
- 计算结果。
- 单位。
- 错误信息，如果输入不合法。

### MCP 任务

实现命令：

```text
/mcp add
/mcp list
/mcp enable
/mcp disable
/mcp remove
```

MCP 配置结构：

```json
{
  "name": "demo-mcp",
  "transport": "stdio",
  "command": "python",
  "args": ["server.py"],
  "url": null,
  "env": {},
  "enabled": false,
  "permission_level": "confirm"
}
```

第一版可以先做到配置管理和 mock 调用。如果时间够，再实现真实 MCP client。

### Skill 任务

实现命令：

```text
/skill install
/skill list
/skill enable
/skill disable
/skill remove
```

Skill 配置结构：

```json
{
  "name": "formula-explainer",
  "description": "解释公式含义并生成推导步骤",
  "path": "skills/formula-explainer",
  "enabled": false,
  "allowed_tools": ["entropy", "discrete_convolution"]
}
```

Skill 安装来源：

- 本地目录。
- Git 仓库。
- ZIP 文件。

如果时间紧，Git 和 ZIP 可以先实现为复制/解压到本地目录，依赖安装作为选做。

### 对外接口

```python
class ToolRegistry:
    def list(self) -> list[ToolSpec]:
        ...

    def get(self, tool_name: str) -> ToolSpec | None:
        ...

    def run(self, tool_name: str, arguments: dict) -> ToolCall:
        ...
```

成员 A 只通过 `ToolRegistry.run()` 调用工具。

### 需要和成员 D 对接

成员 D 需要统计：

- 工具调用次数。
- MCP 调用次数。
- Skill 调用次数。
- 被预算拦截的次数。

所以每次工具调用都要返回 `provider` 和 `cost`。

### 验收标准

- 内置专业工具至少 8 个可运行。
- `/tools list` 能列出工具。
- Agent 能调用至少 3 个专业工具。
- MCP 能 add/list/enable/disable/remove。
- Skill 能 install/list/enable/disable/remove。
- 危险或外部工具默认不自动执行。

## 成员 D：Usage + Web + Evaluation

### 负责目标

成员 D 负责用量控制、Web demo 和测试评估。这个成员不是只写报告，也要写核心代码。用量控制会贯穿整个系统，是项目的加分点。

### 需要实现的文件

```text
ie_agent/usage/budget.py
ie_agent/usage/tracker.py
ie_agent/usage/pricing.py
ie_agent/web/app.py
ie_agent/evaluation/evaluator.py
ie_agent/evaluation/testset.yaml
```

### 用量控制任务

实现以下限制：

- `max_remote_calls_per_session`
- `max_model_tokens_per_session`
- `max_tool_calls_per_turn`
- `max_rag_calls_per_turn`
- `max_mcp_calls_per_session`
- `max_skill_calls_per_session`
- `max_turns_per_session`

实现三个预算 profile：

```text
saving      省钱模式，远程模型调用少
normal      默认模式，效果和成本平衡
performance 展示模式，允许更多调用
```

### Web demo 任务

Web 只做简单展示：

1. Chat 页：
   - 输入问题。
   - 显示回答。
   - 显示引用和工具结果。
2. Knowledge 页：
   - 上传资料。
   - 显示导入结果。
3. Usage 页：
   - 显示当前 session 用量。
   - 显示预算上限。

Web 端必须调用同一个 `AgentOrchestrator.run()`，不能另写回答逻辑。

### 测试评估任务

准备 `testset.yaml`，至少 30 条：

- 电路 6 条。
- 信号系统 6 条。
- 数字逻辑 6 条。
- 信息理论 6 条。
- RAG 资料问答 4 条。
- MCP/Skill/预算控制 2 条。

评估指标：

- `answer_non_empty`：是否返回答案。
- `route_correct`：路由是否符合预期。
- `tool_called_correctly`：是否调用正确工具。
- `citation_returned`：RAG 问题是否返回引用。
- `budget_respected`：是否遵守预算。
- `latency_seconds`：响应耗时。

### 对外接口

```python
class UsageManager:
    def check(self, action: UsageAction) -> BudgetStatus:
        ...

    def record(self, record: UsageRecord) -> None:
        ...

    def summary(self, session_id: str) -> UsageRecord:
        ...

    def set_limit(self, key: str, value: int | float) -> None:
        ...
```

```python
class Evaluator:
    def run(self, testset_path: str) -> dict:
        ...
```

### 验收标准

- `/budget show` 能显示当前用量。
- 超预算时能拦截调用。
- Web demo 能启动并展示三个页面。
- 30 条测试集能跑出结果表。
- 报告里可以引用评估结果。

## 每日对接计划

### 第 1 天

- 所有人读 `interfaces.md`。
- 建好目录结构。
- 每个人写 mock 实现。

### 第 2 到 4 天

- 成员 A 完成 CLI 和 Agent 主流程。
- 成员 B 完成 RAG ingest/search。
- 成员 C 完成专业工具。
- 成员 D 完成 UsageManager。

### 第 5 到 7 天

- 接入远程模型。
- 接入向量库。
- 接入 MCP/Skill 配置管理。
- Web demo 初版。

### 第 8 到 10 天

- 模块集成。
- 修复接口问题。
- 准备测试集。

### 第 11 到 14 天

- 完成展示场景。
- 完成评估结果。
- 写报告和制作答辩材料。

## 模块集成顺序

1. A + D：CLI 调用 UsageManager。
2. A + B：CLI 问题触发 RAG。
3. A + C：CLI 问题触发工具。
4. C + D：工具调用计入预算。
5. B + D：RAG 调用计入预算。
6. A + D：Web 调用 Agent。
7. 全员：跑测试集。

## 接口冲突处理

- 如果字段缺失，优先补可选字段，不删除旧字段。
- 如果返回值不一致，按 `interfaces.md` 为准。
- 如果某模块未完成，用 mock 顶上，不能阻塞其他人。
- 每次集成只改一个模块，方便定位问题。

