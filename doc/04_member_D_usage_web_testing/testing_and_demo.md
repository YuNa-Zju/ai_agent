# 测试、评估与展示方案

本项目不做消融实验，重点做模块测试、集成测试、专业测试集和课堂展示。测试结果要能写进报告，说明系统确实能工作。

## 1. 测试目标

测试需要证明：

1. CLI 能运行。
2. Agent 能正确路由。
3. RAG 能返回引用。
4. 专业工具能被调用。
5. MCP/Skill 管理功能可用。
6. 用量控制能统计和拦截。
7. Web demo 能展示主要能力。

## 2. 测试类型

| 测试类型 | 负责成员 | 目标 |
| --- | --- | --- |
| 单元测试 | 每个成员 | 验证自己模块 |
| 集成测试 | A/D | 验证 Agent 主流程 |
| RAG 测试 | B/D | 验证检索和引用 |
| 工具测试 | C/D | 验证计算正确 |
| 扩展测试 | C/D | 验证 MCP/Skill 管理 |
| 用量测试 | D | 验证预算控制 |
| 展示测试 | 全员 | 验证课堂演示流程 |

## 3. 单元测试要求

每个成员至少写 5 个测试。

成员 A：

- CLI 普通输入能构造 `AgentRequest`。
- `/trace on` 能改变状态。
- Router 能识别信息论问题。
- Router 能识别卷积计算问题。
- API 失败时能 fallback。

成员 B：

- `.md` 导入成功。
- `.txt` 导入成功。
- chunk 数量大于 0。
- 按 `course_tags` 检索成功。
- 无向量库时 fallback 检索可用。

成员 C：

- `entropy([0.5,0.5]) = 1 bit`。
- `discrete_convolution([1,2,1],[1,1]) = [1,3,3,1]`。
- `voltage_divider(5,1000,1000) = 2.5V`。
- `truth_table("A and not B")` 返回 4 行。
- MCP add/list/enable/disable 状态正确。

成员 D：

- `UsageManager.record()` 能累计。
- 超出 remote call 能拦截。
- 超出 tool call 能拦截。
- `/budget show` 有输出。
- `Evaluator.run()` 能读取测试集。

## 4. 专业测试集

文件：`ie_agent/evaluation/testset.yaml`

至少 30 条。

### 4.1 电路题 6 条

示例：

```yaml
- id: circuits_001
  query: "已知电压 5V，电阻 1k 欧，求电流"
  expected_intent: "calculation"
  expected_course_tags: ["circuits"]
  expected_tool: "ohms_law"
```

```yaml
- id: circuits_002
  query: "Vin=5V，R1=1k，R2=1k，分压输出是多少？"
  expected_tool: "voltage_divider"
```

### 4.2 信号系统题 6 条

```yaml
- id: signals_001
  query: "计算 x[n]=[1,2,1] 和 h[n]=[1,1] 的离散卷积"
  expected_intent: "calculation"
  expected_course_tags: ["signals"]
  expected_tool: "discrete_convolution"
```

```yaml
- id: signals_002
  query: "解释卷积在 LTI 系统中的意义"
  expected_intent: "concept"
  expected_course_tags: ["signals"]
  expect_citation: true
```

### 4.3 数字逻辑题 6 条

```yaml
- id: digital_001
  query: "生成 A and not B 的真值表"
  expected_tool: "truth_table"
```

```yaml
- id: digital_002
  query: "二进制 1010 转十进制是多少？"
  expected_tool: "binary_convert"
```

### 4.4 信息理论题 6 条

```yaml
- id: info_001
  query: "计算概率分布 [0.5, 0.5] 的信息熵"
  expected_tool: "entropy"
```

```yaml
- id: info_002
  query: "二元对称信道 p=0.1 的信道容量是多少？"
  expected_tool: "channel_capacity_bsc"
```

### 4.5 RAG 资料问答 4 条

```yaml
- id: rag_001
  query: "根据课程资料解释互信息的定义"
  expected_course_tags: ["information_theory"]
  expect_citation: true
```

### 4.6 扩展和预算 2 条

```yaml
- id: budget_001
  command: "/budget show"
  expect_output_contains: "remote_calls"
```

```yaml
- id: skill_001
  command: "/skill list"
  expect_output_contains: "Skill"
```

## 5. 评估指标

```python
class EvalResult(BaseModel):
    total: int
    answer_non_empty: int
    route_correct: int
    tool_called_correctly: int
    citation_returned: int
    budget_respected: int
    average_latency_seconds: float
```

指标解释：

- `answer_non_empty`：回答非空。
- `route_correct`：intent 和 course_tags 是否符合预期。
- `tool_called_correctly`：需要工具的问题是否调用了正确工具。
- `citation_returned`：需要 RAG 的问题是否返回引用。
- `budget_respected`：是否遵守预算。
- `average_latency_seconds`：平均耗时。

## 6. 集成测试流程

### 6.1 CLI 到 Agent

输入：

```text
计算 [1,2,1] 和 [1,1] 的卷积
```

期望：

- `intent=calculation`
- `course_tags=[signals]`
- 调用 `discrete_convolution`
- 回答包含 `[1,3,3,1]`

### 6.2 CLI 到 RAG

输入：

```text
根据课程资料解释信息熵
```

期望：

- `need_rag=true`
- 返回 citation。
- 回答中显示来源。

### 6.3 CLI 到预算控制

设置：

```text
/budget set remote_calls 0
```

输入：

```text
请详细解释傅里叶变换
```

期望：

- 远程模型调用被拦截。
- CLI 给出替代方案。

## 7. Web Demo 测试

Web demo 只需要验证三页。

### 7.1 Chat 页

输入：

```text
什么是互信息？
```

期望：

- 显示回答。
- 如果 RAG 开启，显示引用。
- 显示本轮用量。

### 7.2 Knowledge 页

上传：

```text
entropy_note.md
```

期望：

- 显示导入成功。
- 显示 chunk 数。

### 7.3 Usage 页

期望显示：

- profile。
- remote calls。
- token。
- tool calls。
- rag calls。
- cost。

## 8. 课堂展示脚本

### 场景 1：课程资料问答

操作：

```text
/trace on
根据课程资料解释互信息和信息熵的区别
```

展示点：

- Agent 判断为信息理论问题。
- RAG 检索课程资料。
- 回答包含引用。
- 用量统计记录一次 RAG 和远程模型。

### 场景 2：专业工具计算

操作：

```text
计算二元对称信道 p=0.1 的信道容量，并解释这个结果
```

展示点：

- Agent 判断需要工具。
- 调用 `channel_capacity_bsc`。
- 工具给出公式 `C = 1 - H(p)`。
- 大模型解释结果含义。

### 场景 3：扩展和预算控制

操作：

```text
/skill list
/mcp list
/budget show
/budget set remote_calls 0
请详细解释傅里叶变换
```

展示点：

- 系统有 Skill 和 MCP 扩展管理。
- 预算控制可以拦截远程模型。
- Agent 给出替代方案。

## 9. 报告截图清单

建议截图：

1. CLI 启动界面。
2. `/trace on` 后的路由结果。
3. RAG 回答和引用。
4. 工具调用结果。
5. `/tools list`。
6. `/skill list`。
7. `/mcp list`。
8. `/budget show`。
9. 超预算拦截图。
10. Web Chat 页。
11. Web Usage 页。
12. 测试集结果表。

## 10. 最终验收清单

- [ ] CLI 能启动。
- [ ] 普通对话能返回回答。
- [ ] RAG 能导入资料。
- [ ] RAG 能返回引用。
- [ ] 至少 8 个内置工具可运行。
- [ ] Agent 能调用工具。
- [ ] MCP 管理命令可用。
- [ ] Skill 管理命令可用。
- [ ] 用量控制可显示。
- [ ] 超预算能拦截。
- [ ] Web demo 能启动。
- [ ] 30 条测试集能运行。
- [ ] 报告截图准备完成。

