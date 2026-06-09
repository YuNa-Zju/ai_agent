# 成员 C 任务领取卡：Tools + MCP + Skill

你负责专业工具库和扩展系统。这个模块是 Agent 区别于普通问答机器人的关键：Agent 不仅会说，还会调用确定性工具算出结果，并且可以安装 MCP 和 Skill 扩展能力。

## 你要阅读的文档

1. `../00_shared/interfaces.md`
2. `tools_and_extensions.md`
3. `../00_shared/programming_manual.md`
4. `../00_shared/team_work_split.md`

## 你负责的代码目录

```text
ie_agent/
  tools/
    registry.py
    circuits.py
    signals.py
    digital_logic.py
    information_theory.py
  extensions/
    mcp_manager.py
    skill_manager.py
    permission.py
```

## 最小交付

第一版必须完成：

1. `ToolRegistry.list()` 能列出内置工具。
2. `ToolRegistry.run()` 能调用工具。
3. 至少 8 个专业工具可运行。
4. MCP 支持 add/list/enable/disable/remove 配置管理。
5. Skill 支持 install/list/enable/disable/remove 配置管理。
6. 外部扩展默认 disabled。
7. `permission_level` 为 `confirm` 或 `dangerous` 的工具不能静默执行。

## 核心接口

你必须实现：

```python
class ToolRegistry:
    def list(self) -> list[ToolSpec]:
        ...

    def get(self, tool_name: str) -> ToolSpec | None:
        ...

    def run(self, tool_name: str, arguments: dict) -> ToolCall:
        ...
```

成员 A 只会通过这个接口调用工具。

## 内置工具清单

### 电路

- `ohms_law`
- `voltage_divider`
- `rc_time_constant`

### 信号系统

- `discrete_convolution`
- `linear_convolution_length`
- `dft_basic`

### 数字逻辑

- `truth_table`
- `binary_convert`

### 信息理论

- `entropy`
- `mutual_information`
- `channel_capacity_bsc`

如果时间紧，至少完成前 8 个；如果时间够，完成全部。

## 工具输出要求

每个工具不能只输出数字。必须包含：

- `formula`
- `steps`
- `unit`，如果有单位
- `result`
- 输入参数回显

示例：

```json
{
  "result": 1.0,
  "unit": "bit",
  "formula": "H(X) = - sum p(x) log2 p(x)",
  "steps": ["H = -0.5log2(0.5)-0.5log2(0.5)=1"]
}
```

## 输入校验

必须校验：

- 概率不能为负。
- 概率和应接近 1。
- 电阻不能为 0 或负数。
- 分压电阻不能为 0。
- 序列输入必须是数字列表。
- 真值表表达式不能执行任意 Python 代码。

工具失败时返回 `ToolCall(status="error")`，不要让程序崩溃。

## MCP 任务

先做配置管理，不要求第一版真实连通所有 MCP server。

配置字段：

```json
{
  "name": "demo",
  "transport": "stdio",
  "command": "python",
  "args": ["server.py"],
  "url": null,
  "env": {},
  "enabled": false,
  "permission_level": "confirm"
}
```

命令：

```text
/mcp add demo --transport stdio --command python --args server.py
/mcp list
/mcp enable demo
/mcp disable demo
/mcp remove demo
```

## Skill 任务

Skill 安装后默认 disabled。

Manifest 示例：

```json
{
  "name": "formula-explainer",
  "description": "解释公式并生成推导步骤",
  "entrypoint": "SKILL.md",
  "allowed_tools": ["entropy", "discrete_convolution"],
  "enabled": false
}
```

命令：

```text
/skill install ./skills/formula-explainer
/skill list
/skill enable formula-explainer
/skill disable formula-explainer
/skill remove formula-explainer
```

第一版可以只支持本地目录安装，Git 和 ZIP 作为加分项。

## 与成员 A 对接

成员 A 会调用：

```python
tool_registry.list()
tool_registry.run("channel_capacity_bsc", {"p": 0.1})
```

返回：

```python
ToolCall(
    tool_name="channel_capacity_bsc",
    provider="builtin",
    arguments={"p": 0.1},
    result={...},
    status="success",
)
```

## 与成员 D 对接

成员 D 需要统计工具、MCP、Skill 调用次数。你需要在 `ToolCall.cost` 或 metadata 里保留：

```json
{
  "calls": 1,
  "provider": "builtin"
}
```

外部工具调用前要给 UsageManager 检查的机会。

## 单元测试

至少写 8 个：

1. 熵 `[0.5,0.5]` 返回 1 bit。
2. BSC `p=0.1` 返回约 0.531 bit/use。
3. 卷积 `[1,2,1]` 和 `[1,1]` 返回 `[1,3,3,1]`。
4. 分压 `5V,1k,1k` 返回 `2.5V`。
5. 真值表返回 4 行。
6. 错误概率输入返回 error。
7. MCP add/list/enable/disable 可用。
8. Skill install/list/enable/disable 可用。

## 验收标准

- `/tools list` 能看到内置工具。
- Agent 能调用至少 3 个工具。
- MCP 管理命令可用。
- Skill 管理命令可用。
- 外部扩展默认不自动执行。
- 工具错误不会导致 CLI 崩溃。

