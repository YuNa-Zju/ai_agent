# IE-Agent 统一接口定义

本文件是四人对接的核心。所有模块必须使用这里定义的数据结构、字段名和函数签名。不要在自己的模块里私自新增同名但不同结构的接口。

建议用 Pydantic 定义这些模型。如果时间紧，也可以先用普通 `dict`，但字段必须保持一致。

## 1. 通用消息结构

```python
from typing import Any, Literal
from pydantic import BaseModel, Field

class Message(BaseModel):
    role: Literal["user", "assistant", "system", "tool"]
    content: str
    timestamp: str | None = None
    metadata: dict[str, Any] = Field(default_factory=dict)
```

说明：

- `role=user`：用户输入。
- `role=assistant`：Agent 回答。
- `role=system`：系统提示词或内部策略。
- `role=tool`：工具返回结果。
- `metadata` 可记录 token、引用、工具名等额外信息。

## 2. AgentRequest

```python
class AgentRequest(BaseModel):
    session_id: str
    user_query: str
    history: list[Message] = Field(default_factory=list)
    mode: Literal["chat", "study", "solve", "report", "debug"] = "chat"
    attachments: list[str] = Field(default_factory=list)
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `session_id` | `str` | 当前会话 ID |
| `user_query` | `str` | 用户本轮输入 |
| `history` | `list[Message]` | 历史对话 |
| `mode` | `str` | 当前工作模式 |
| `attachments` | `list[str]` | 附件路径，例如 PDF、Markdown |

使用场景：

- CLI 每收到一条用户输入，就构造一个 `AgentRequest`。
- Web demo 也复用该结构。

## 3. RouteDecision

```python
class RouteDecision(BaseModel):
    intent: Literal[
        "concept",
        "calculation",
        "rag_qa",
        "experiment_debug",
        "report",
        "extension",
        "unknown",
    ]
    course_tags: list[Literal[
        "circuits",
        "signals",
        "digital_logic",
        "information_theory",
        "ckm",
        "general_ai",
    ]] = Field(default_factory=list)
    complexity: Literal["simple", "medium", "complex"] = "medium"
    need_rag: bool = False
    need_tool: bool = False
    target_model: Literal["local", "remote", "rule"] = "remote"
    query_rewrite: str | None = None
    reason: str = ""
```

字段说明：

- `intent`：问题类型。
- `course_tags`：所属课程方向。
- `complexity`：复杂度。
- `need_rag`：是否需要课程知识库。
- `need_tool`：是否需要计算工具或扩展工具。
- `target_model`：最终回答优先使用哪个模型。
- `query_rewrite`：给 RAG 使用的改写查询。
- `reason`：路由理由，用于 `/trace on` 展示。

路由示例：

```json
{
  "intent": "calculation",
  "course_tags": ["signals"],
  "complexity": "medium",
  "need_rag": true,
  "need_tool": true,
  "target_model": "remote",
  "query_rewrite": "离散卷积 定义 计算 示例",
  "reason": "用户要求解释概念并计算离散卷积"
}
```

## 4. RAG 接口

### 4.1 RagQuery

```python
class RagQuery(BaseModel):
    query: str
    course_tags: list[str] = Field(default_factory=list)
    top_k: int = 5
    min_score: float = 0.2
```

### 4.2 Citation

```python
class Citation(BaseModel):
    source_id: str
    title: str
    path: str
    page: int | None = None
    section: str | None = None
    chunk_id: str
```

### 4.3 RagChunk

```python
class RagChunk(BaseModel):
    chunk_id: str
    text: str
    score: float
    citation: Citation
    course_tags: list[str] = Field(default_factory=list)
```

### 4.4 RagResult

```python
class RagResult(BaseModel):
    chunks: list[RagChunk] = Field(default_factory=list)
    citations: list[Citation] = Field(default_factory=list)
    confidence: float = 0.0
```

### 4.5 RagService 函数签名

```python
class IngestResult(BaseModel):
    path: str
    chunks_added: int
    status: Literal["success", "error"]
    error: str | None = None

class RagService:
    def ingest(self, path: str, course_tags: list[str] | None = None) -> IngestResult:
        ...

    def search(self, query: RagQuery) -> RagResult:
        ...
```

成员 A 只需要调用：

```python
rag_result = rag_service.search(
    RagQuery(query=decision.query_rewrite or request.user_query,
             course_tags=decision.course_tags,
             top_k=5)
)
```

## 5. 工具接口

### 5.1 ToolSpec

```python
class ToolSpec(BaseModel):
    name: str
    description: str
    input_schema: dict[str, Any]
    output_schema: dict[str, Any]
    permission_level: Literal["safe", "confirm", "dangerous"] = "safe"
    provider: Literal["builtin", "mcp", "skill"] = "builtin"
```

权限说明：

- `safe`：纯计算工具，可以直接调用。
- `confirm`：可能访问文件、网络或外部服务，调用前需要确认。
- `dangerous`：可能修改文件或执行命令，默认禁止，展示项目不建议使用。

### 5.2 ToolCall

```python
class ToolCall(BaseModel):
    tool_name: str
    provider: Literal["builtin", "mcp", "skill"] = "builtin"
    arguments: dict[str, Any] = Field(default_factory=dict)
    result: dict[str, Any] = Field(default_factory=dict)
    status: Literal["success", "error", "blocked"] = "success"
    error: str | None = None
    cost: dict[str, Any] = Field(default_factory=dict)
```

### 5.3 ToolRegistry

```python
class ToolRegistry:
    def list(self) -> list[ToolSpec]:
        ...

    def get(self, tool_name: str) -> ToolSpec | None:
        ...

    def run(self, tool_name: str, arguments: dict[str, Any]) -> ToolCall:
        ...
```

成员 C 必须保证每个工具都能通过 `ToolRegistry.run()` 调用，不允许 Agent 直接 import 某个具体工具函数。

## 6. 用量接口

### 6.1 UsageAction

```python
class UsageAction(BaseModel):
    session_id: str
    action_type: Literal["remote_model", "local_model", "rag", "tool", "mcp", "skill"]
    estimated_tokens: int = 0
    estimated_calls: int = 1
    metadata: dict[str, Any] = Field(default_factory=dict)
```

### 6.2 UsageRecord

```python
class UsageRecord(BaseModel):
    session_id: str
    model_tokens: int = 0
    remote_calls: int = 0
    local_calls: int = 0
    tool_calls: int = 0
    rag_calls: int = 0
    mcp_calls: int = 0
    skill_calls: int = 0
    estimated_cost_cny: float = 0.0
    metadata: dict[str, Any] = Field(default_factory=dict)
```

### 6.3 BudgetStatus

```python
class BudgetStatus(BaseModel):
    allowed: bool
    reason: str = ""
    current: dict[str, Any] = Field(default_factory=dict)
    limits: dict[str, Any] = Field(default_factory=dict)
```

### 6.4 UsageManager

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

所有远程模型、RAG、工具、MCP、Skill 调用前都必须先 `check()`，调用后必须 `record()`。

## 7. AgentResponse

```python
class AgentResponse(BaseModel):
    answer: str
    citations: list[Citation] = Field(default_factory=list)
    tool_results: list[ToolCall] = Field(default_factory=list)
    route_trace: RouteDecision | None = None
    usage: UsageRecord | None = None
    confidence: float = 0.0
    followups: list[str] = Field(default_factory=list)
```

CLI 展示顺序：

1. 如果 `/trace on`，先显示 `route_trace`。
2. 显示工具调用结果摘要。
3. 显示正文 `answer`。
4. 显示引用 `citations`。
5. 显示本轮用量 `usage`。
6. 显示建议追问 `followups`。

## 8. AgentOrchestrator

```python
class AgentOrchestrator:
    def __init__(
        self,
        router,
        model_manager,
        rag_service,
        tool_registry,
        usage_manager,
    ):
        ...

    def run(self, request: AgentRequest) -> AgentResponse:
        ...
```

推荐执行逻辑：

```python
def run(self, request: AgentRequest) -> AgentResponse:
    decision = self.router.route(request)

    rag_result = None
    if decision.need_rag:
        self.usage_manager.check(UsageAction(session_id=request.session_id, action_type="rag"))
        rag_result = self.rag_service.search(...)

    tool_calls = []
    if decision.need_tool:
        selected_tools = self.router.select_tools(request, decision)
        for tool_name, args in selected_tools:
            self.usage_manager.check(UsageAction(session_id=request.session_id, action_type="tool"))
            tool_calls.append(self.tool_registry.run(tool_name, args))

    model_input = self.prompt_builder.build(request, decision, rag_result, tool_calls)
    answer = self.model_manager.generate(model_input, target=decision.target_model)

    usage = self.usage_manager.summary(request.session_id)
    return AgentResponse(
        answer=answer,
        citations=rag_result.citations if rag_result else [],
        tool_results=tool_calls,
        route_trace=decision,
        usage=usage,
    )
```

## 9. 模块对接 Mock 约定

每个成员在真实实现前先提供 mock，方便别人联调。

成员 B 的 RAG mock：

```python
class MockRagService:
    def search(self, query: RagQuery) -> RagResult:
        return RagResult(chunks=[], citations=[], confidence=0.0)
```

成员 C 的工具 mock：

```python
class MockToolRegistry:
    def list(self) -> list[ToolSpec]:
        return []

    def run(self, tool_name: str, arguments: dict[str, Any]) -> ToolCall:
        return ToolCall(tool_name=tool_name, arguments=arguments, result={"mock": True})
```

成员 D 的用量 mock：

```python
class MockUsageManager:
    def check(self, action: UsageAction) -> BudgetStatus:
        return BudgetStatus(allowed=True)

    def record(self, record: UsageRecord) -> None:
        pass

    def summary(self, session_id: str) -> UsageRecord:
        return UsageRecord(session_id=session_id)
```

## 10. 接口变更规则

- 字段名不能随意改。
- 如果必须新增字段，只能加可选字段，不能破坏旧调用。
- 每次接口变更要同步修改本文件。
- 集成前先跑所有 mock 测试。

