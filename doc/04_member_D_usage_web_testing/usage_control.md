# 用量控制设计

用量控制是 IE-Agent 的重要加分点。它说明系统不是简单调用大模型，而是一个可控、可管理、可审计的 Agent。用量控制由成员 D 负责，但成员 A、B、C 都必须在调用模型、RAG 和工具前后接入它。

## 1. 控制目标

系统需要控制以下资源：

- 远程模型调用次数。
- 远程模型 token 数。
- 本地模型调用次数，选做。
- RAG 检索次数。
- 工具调用次数。
- MCP 调用次数。
- Skill 调用次数。
- 单会话轮次。
- 估算费用。

## 2. 预算配置

建议提供三个 profile。

### 2.1 saving

省钱模式，适合平时开发。

```json
{
  "max_remote_calls_per_session": 3,
  "max_model_tokens_per_session": 8000,
  "max_tool_calls_per_turn": 3,
  "max_rag_calls_per_turn": 1,
  "max_mcp_calls_per_session": 2,
  "max_skill_calls_per_session": 5,
  "max_turns_per_session": 30
}
```

### 2.2 normal

默认模式，适合普通演示。

```json
{
  "max_remote_calls_per_session": 10,
  "max_model_tokens_per_session": 20000,
  "max_tool_calls_per_turn": 5,
  "max_rag_calls_per_turn": 3,
  "max_mcp_calls_per_session": 5,
  "max_skill_calls_per_session": 10,
  "max_turns_per_session": 50
}
```

### 2.3 performance

展示模式，适合课堂演示前准备。

```json
{
  "max_remote_calls_per_session": 30,
  "max_model_tokens_per_session": 80000,
  "max_tool_calls_per_turn": 8,
  "max_rag_calls_per_turn": 5,
  "max_mcp_calls_per_session": 10,
  "max_skill_calls_per_session": 20,
  "max_turns_per_session": 100
}
```

## 3. UsageAction

所有可能消耗资源的动作，在执行前都要构造 `UsageAction`。

```python
class UsageAction(BaseModel):
    session_id: str
    action_type: Literal["remote_model", "local_model", "rag", "tool", "mcp", "skill"]
    estimated_tokens: int = 0
    estimated_calls: int = 1
    metadata: dict = {}
```

示例：

```python
action = UsageAction(
    session_id="demo",
    action_type="remote_model",
    estimated_tokens=1200,
)
status = usage_manager.check(action)
```

## 4. BudgetStatus

```python
class BudgetStatus(BaseModel):
    allowed: bool
    reason: str = ""
    current: dict = {}
    limits: dict = {}
```

允许调用：

```json
{
  "allowed": true,
  "reason": "within budget"
}
```

拦截调用：

```json
{
  "allowed": false,
  "reason": "remote_calls reached limit 10",
  "current": {"remote_calls": 10},
  "limits": {"remote_calls": 10}
}
```

## 5. UsageRecord

调用完成后记录真实用量。

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
    metadata: dict = {}
```

示例：

```python
usage_manager.record(
    UsageRecord(
        session_id="demo",
        model_tokens=830,
        remote_calls=1,
        estimated_cost_cny=0.02,
    )
)
```

## 6. UsageManager

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

    def reset(self, session_id: str | None = None) -> None:
        ...
```

## 7. 拦截点

### 7.1 远程模型前

成员 A 在调用 API 前：

```python
status = usage_manager.check(
    UsageAction(
        session_id=session_id,
        action_type="remote_model",
        estimated_tokens=estimated_tokens,
    )
)
if not status.allowed:
    return fallback_answer(status.reason)
```

### 7.2 RAG 前

成员 A 或 B 在检索前：

```python
status = usage_manager.check(
    UsageAction(session_id=session_id, action_type="rag")
)
```

### 7.3 工具前

成员 A 或 C 在工具调用前：

```python
status = usage_manager.check(
    UsageAction(session_id=session_id, action_type="tool")
)
```

### 7.4 MCP/Skill 前

成员 C 在扩展调用前：

```python
status = usage_manager.check(
    UsageAction(session_id=session_id, action_type="mcp")
)
```

## 8. 超预算处理

不能只报错退出。应该给用户选择：

```text
budget blocked: remote_calls reached limit 10

可选操作：
1. /model local 使用本地或规则模式
2. /budget set remote_calls 20 提高上限
3. /rag off 暂时关闭知识库检索
```

对于工具超限：

```text
budget blocked: tool calls reached per-turn limit 5
请拆分问题，或者使用 /budget set max_tool_calls_per_turn 8
```

## 9. CLI 展示

`/budget show` 输出：

```text
Budget Profile: normal

remote_calls: 2 / 10
model_tokens: 1600 / 20000
tool_calls_this_turn: 1 / 5
rag_calls_this_turn: 1 / 3
mcp_calls: 0 / 5
skill_calls: 0 / 10
turns: 4 / 50
estimated_cost_cny: 0.04
```

`/budget set`：

```text
/budget set remote_calls 20
```

`/budget reset`：

```text
/budget reset
```

## 10. Web 展示

Usage 页显示：

- 当前 profile。
- 每项当前用量。
- 每项上限。
- 估算费用。
- 最近 10 次调用记录。

不需要复杂图表，表格即可。

## 11. 费用估算

由于不同 API 价格会变化，代码中不要写死不可修改的价格。建议用配置：

```json
{
  "remote_input_per_1k_cny": 0.001,
  "remote_output_per_1k_cny": 0.002
}
```

费用估算：

```python
cost = input_tokens / 1000 * input_price + output_tokens / 1000 * output_price
```

报告中可以说明这是估算费用，用于展示用量控制思路。

## 12. 测试用例

### 12.1 正常记录

输入：

```python
record(remote_calls=1, model_tokens=500)
```

期望：

```text
summary.remote_calls == 1
summary.model_tokens == 500
```

### 12.2 超出远程调用预算

设置：

```text
max_remote_calls_per_session = 1
```

连续调用两次 remote model。

期望：

```text
第二次 check 返回 allowed=false
```

### 12.3 工具单轮超限

设置：

```text
max_tool_calls_per_turn = 2
```

一轮中调用 3 次工具。

期望：

```text
第 3 次被拦截
```

## 13. 成员 D 交付清单

- `BudgetConfig` 可用。
- `UsageManager.check()` 可用。
- `UsageManager.record()` 可用。
- `UsageManager.summary()` 可用。
- `/budget show` 可显示。
- 超预算能拦截。
- Web Usage 页能展示。
- 至少 5 个用量控制测试。

