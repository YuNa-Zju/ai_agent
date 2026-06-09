# 成员 D 任务领取卡：Usage + Web + Evaluation

你负责用量控制、Web demo 和测试评估。这个岗位不是只整理报告，而是写项目中很重要的控制和评估代码。用量控制会让项目看起来更像真实 Agent 系统，而不是简单 API 调用。

## 你要阅读的文档

1. `../00_shared/interfaces.md`
2. `usage_control.md`
3. `testing_and_demo.md`
4. `../00_shared/programming_manual.md`
5. `../00_shared/team_work_split.md`

## 你负责的代码目录

```text
ie_agent/
  usage/
    budget.py
    tracker.py
    pricing.py
  web/
    app.py
  evaluation/
    evaluator.py
    testset.yaml
```

## 最小交付

第一版必须完成：

1. `UsageManager.check()` 能判断是否超预算。
2. `UsageManager.record()` 能累计用量。
3. `UsageManager.summary()` 能返回当前 session 用量。
4. `/budget show` 需要的数据可返回给成员 A。
5. Web demo 有 Chat、Knowledge、Usage 三页。
6. `Evaluator.run()` 能读取 30 条测试集并输出结果。
7. 超预算时能返回清楚的拦截原因。

## 核心接口

你必须实现：

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

评估接口：

```python
class Evaluator:
    def run(self, testset_path: str) -> dict:
        ...
```

## 预算 Profile

实现三个模式：

- `saving`
- `normal`
- `performance`

默认使用 `normal`。

至少限制：

- `max_remote_calls_per_session`
- `max_model_tokens_per_session`
- `max_tool_calls_per_turn`
- `max_rag_calls_per_turn`
- `max_mcp_calls_per_session`
- `max_skill_calls_per_session`
- `max_turns_per_session`

## 与成员 A 对接

成员 A 会调用：

```python
status = usage_manager.check(
    UsageAction(
        session_id=session_id,
        action_type="remote_model",
        estimated_tokens=1200,
    )
)
```

你要返回：

```python
BudgetStatus(allowed=True)
```

或者：

```python
BudgetStatus(
    allowed=False,
    reason="remote_calls reached limit 10",
    current={"remote_calls": 10},
    limits={"remote_calls": 10},
)
```

## 与成员 B/C 对接

RAG 调用要记录：

```python
UsageRecord(session_id=session_id, rag_calls=1)
```

工具调用要记录：

```python
UsageRecord(session_id=session_id, tool_calls=1)
```

MCP 和 Skill 调用也要分别统计。

## Web Demo

Web 端简单即可，用 Streamlit 或 Gradio。

### Chat 页

功能：

- 输入问题。
- 调用 `AgentOrchestrator.run()`。
- 显示回答。
- 显示引用。
- 显示工具结果。
- 显示用量。

### Knowledge 页

功能：

- 上传 `.md`、`.txt` 或 `.pdf`。
- 调用 `RagService.ingest()`。
- 显示 chunk 数和状态。

### Usage 页

功能：

- 显示当前预算 profile。
- 显示用量表格。
- 显示最近调用记录。

## 测试集

`testset.yaml` 至少 30 条：

- 电路 6 条。
- 信号系统 6 条。
- 数字逻辑 6 条。
- 信息理论 6 条。
- RAG 资料问答 4 条。
- MCP/Skill/预算 2 条。

不要做消融实验，把时间用在真实测试和演示稳定性上。

## 评估输出

`Evaluator.run()` 返回：

```json
{
  "total": 30,
  "answer_non_empty": 30,
  "route_correct": 25,
  "tool_called_correctly": 8,
  "citation_returned": 7,
  "budget_respected": 30,
  "average_latency_seconds": 3.2
}
```

这个结果可以直接写入报告。

## 单元测试

至少写 5 个：

1. `record()` 能累计 token 和 calls。
2. 超出 remote calls 能拦截。
3. 超出 tool calls 能拦截。
4. `summary()` 返回正确 session。
5. `Evaluator.run()` 能读取测试集。

## 验收标准

- `/budget show` 能显示数据。
- 超预算能拦截远程模型或工具。
- Web demo 能启动。
- Chat 页能调用 Agent。
- Knowledge 页能上传资料。
- Usage 页能显示统计。
- 30 条测试集能跑出结果表。

