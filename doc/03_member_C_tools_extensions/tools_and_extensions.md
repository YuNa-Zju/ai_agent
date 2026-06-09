# 专业工具、MCP 与 Skill 设计

本文件定义 IE-Agent 的工具系统。工具系统分为三类：

1. 内置信息工程专业工具。
2. MCP 外部工具。
3. Skill 扩展能力。

所有工具都必须通过 `ToolRegistry` 统一注册和调用。

## 1. ToolRegistry 总接口

```python
class ToolRegistry:
    def list(self) -> list[ToolSpec]:
        ...

    def get(self, tool_name: str) -> ToolSpec | None:
        ...

    def run(self, tool_name: str, arguments: dict) -> ToolCall:
        ...
```

Agent 不直接调用 `entropy()`、`ohms_law()` 这类函数，而是调用：

```python
tool_registry.run("entropy", {"probabilities": [0.5, 0.5]})
```

这样 MCP 和 Skill 工具也可以用同样方式接入。

## 2. ToolSpec

```python
class ToolSpec(BaseModel):
    name: str
    description: str
    input_schema: dict
    output_schema: dict
    permission_level: Literal["safe", "confirm", "dangerous"] = "safe"
    provider: Literal["builtin", "mcp", "skill"] = "builtin"
```

示例：

```json
{
  "name": "entropy",
  "description": "计算离散随机变量的信息熵",
  "input_schema": {
    "probabilities": "list[float]"
  },
  "output_schema": {
    "entropy": "float",
    "unit": "bit",
    "formula": "str",
    "steps": "list[str]"
  },
  "permission_level": "safe",
  "provider": "builtin"
}
```

## 3. ToolCall

```python
class ToolCall(BaseModel):
    tool_name: str
    provider: Literal["builtin", "mcp", "skill"]
    arguments: dict
    result: dict
    status: Literal["success", "error", "blocked"]
    error: str | None = None
    cost: dict = {}
```

工具失败时不要抛出未处理异常，返回：

```json
{
  "tool_name": "entropy",
  "provider": "builtin",
  "arguments": {"probabilities": [0.2, 0.2]},
  "result": {},
  "status": "error",
  "error": "probabilities must sum to 1"
}
```

## 4. 内置专业工具

### 4.1 电路工具

#### `ohms_law`

功能：已知电压、电流、电阻中的两个，求第三个。

输入：

```json
{
  "voltage": 5.0,
  "current": null,
  "resistance": 1000.0
}
```

输出：

```json
{
  "missing": "current",
  "value": 0.005,
  "unit": "A",
  "formula": "I = U / R",
  "steps": ["I = 5.0 / 1000.0 = 0.005 A"]
}
```

#### `voltage_divider`

功能：计算分压电路输出。

输入：

```json
{
  "vin": 5.0,
  "r1": 1000.0,
  "r2": 1000.0
}
```

输出：

```json
{
  "vout": 2.5,
  "unit": "V",
  "formula": "Vout = Vin * R2 / (R1 + R2)"
}
```

#### `rc_time_constant`

功能：计算 RC 时间常数。

输入：

```json
{
  "resistance": 10000.0,
  "capacitance": 0.000001
}
```

输出：

```json
{
  "tau": 0.01,
  "unit": "s",
  "formula": "tau = R * C"
}
```

### 4.2 信号系统工具

#### `discrete_convolution`

功能：计算两个离散序列的线性卷积。

输入：

```json
{
  "x": [1, 2, 1],
  "h": [1, 1]
}
```

输出：

```json
{
  "y": [1, 3, 3, 1],
  "formula": "y[n] = sum_k x[k] h[n-k]",
  "steps": [
    "y[0] = 1*1 = 1",
    "y[1] = 1*1 + 2*1 = 3"
  ]
}
```

#### `linear_convolution_length`

输入：

```json
{
  "len_x": 3,
  "len_h": 4
}
```

输出：

```json
{
  "length": 6,
  "formula": "N = len_x + len_h - 1"
}
```

#### `dft_basic`

功能：计算短序列 DFT，适合展示，不用于大规模数值计算。

输入：

```json
{
  "x": [1, 0, 0, 0]
}
```

输出：

```json
{
  "X": [[1.0, 0.0], [1.0, 0.0], [1.0, 0.0], [1.0, 0.0]],
  "note": "complex number represented as [real, imag]"
}
```

### 4.3 数字逻辑工具

#### `truth_table`

输入：

```json
{
  "expression": "A and (not B)"
}
```

输出：

```json
{
  "variables": ["A", "B"],
  "rows": [
    {"A": 0, "B": 0, "Y": 0},
    {"A": 0, "B": 1, "Y": 0},
    {"A": 1, "B": 0, "Y": 1},
    {"A": 1, "B": 1, "Y": 0}
  ]
}
```

#### `binary_convert`

输入：

```json
{
  "value": "1010",
  "from_base": 2,
  "to_base": 10
}
```

输出：

```json
{
  "result": "10"
}
```

### 4.4 信息论工具

#### `entropy`

输入：

```json
{
  "probabilities": [0.5, 0.5]
}
```

输出：

```json
{
  "entropy": 1.0,
  "unit": "bit",
  "formula": "H(X) = - sum p(x) log2 p(x)"
}
```

#### `mutual_information`

输入：

```json
{
  "joint": [[0.25, 0.25], [0.25, 0.25]]
}
```

输出：

```json
{
  "mutual_information": 0.0,
  "unit": "bit",
  "formula": "I(X;Y)=sum p(x,y) log2(p(x,y)/(p(x)p(y)))"
}
```

#### `channel_capacity_bsc`

输入：

```json
{
  "p": 0.1
}
```

输出：

```json
{
  "capacity": 0.531,
  "unit": "bit/use",
  "formula": "C = 1 - H(p)"
}
```

## 5. MCP 管理

MCP 允许 Agent 接入外部工具。项目支持完全开放安装，但默认受权限和预算控制。

### 5.1 MCP 配置

```python
class McpServerConfig(BaseModel):
    name: str
    transport: Literal["stdio", "http", "sse"]
    command: str | None = None
    args: list[str] = []
    url: str | None = None
    env: dict[str, str] = {}
    enabled: bool = False
    permission_level: Literal["confirm", "dangerous"] = "confirm"
```

### 5.2 MCP 命令

```text
/mcp add demo --transport stdio --command python --args server.py
/mcp list
/mcp enable demo
/mcp disable demo
/mcp remove demo
```

### 5.3 调用控制

第一次调用 MCP 工具前显示：

```text
MCP tool request
server: demo
tool: search_course_file
arguments: {"query": "entropy"}
permission: confirm
allow? [y/N]
```

用户拒绝时返回：

```json
{
  "status": "blocked",
  "error": "user denied MCP tool call"
}
```

## 6. Skill 管理

Skill 是一种可安装能力包，可以包含 prompt 模板、工具声明、执行脚本或说明文档。

### 6.1 Skill Manifest

```json
{
  "name": "formula-explainer",
  "description": "解释信息工程公式并生成推导步骤",
  "entrypoint": "SKILL.md",
  "allowed_tools": ["entropy", "discrete_convolution"],
  "enabled": false
}
```

### 6.2 Skill 命令

```text
/skill install ./skills/formula-explainer
/skill install https://github.com/example/skill.git
/skill install ./skill.zip
/skill list
/skill enable formula-explainer
/skill disable formula-explainer
/skill remove formula-explainer
```

### 6.3 第一版实现范围

必做：

- 本地目录安装。
- list/enable/disable/remove。
- 读取 manifest。
- 注册 Skill 中声明的 prompt 模板或工具。

选做：

- Git 仓库安装。
- ZIP 安装。
- 自动安装依赖。
- Skill 内脚本执行。

## 7. Agent 如何选择工具

Router 可以用关键词选择工具：

| 关键词 | 工具 |
| --- | --- |
| 电压、电流、电阻、欧姆 | `ohms_law` |
| 分压 | `voltage_divider` |
| 卷积 | `discrete_convolution` |
| 真值表 | `truth_table` |
| 信息熵、熵 | `entropy` |
| 互信息 | `mutual_information` |
| 信道容量、BSC | `channel_capacity_bsc` |

如果多个工具都可能需要，先调用确定性工具，再交给模型解释。

## 8. 成员 C 交付清单

- `ToolRegistry` 可用。
- 至少 8 个内置工具可运行。
- 每个工具有输入校验。
- 每个工具有公式和计算过程输出。
- MCP 管理命令可用。
- Skill 管理命令可用。
- 至少 8 个工具单元测试。

