# IE-Agent CLI 使用与实现手册

CLI 是本项目的主产品。课堂展示时应优先展示 CLI，因为它最能体现 Agent 的任务拆解、工具调用、RAG 引用、扩展管理和用量控制。

## 1. CLI 目标

CLI 需要做到：

- 连续对话，而不是单次问答。
- 支持 slash commands。
- 支持流式输出。
- 支持显示路由轨迹。
- 支持显示 RAG 引用。
- 支持显示工具结果。
- 支持会话保存和加载。
- 支持预算查看和设置。

## 2. 启动命令

```bash
ie-agent chat
```

开发阶段可以使用：

```bash
python -m ie_agent.cli.main chat
```

启动后显示：

```text
IE-Agent 信息工程学习科研助手
mode=chat, model=auto, rag=on, budget=normal
输入 /help 查看命令，输入 /exit 退出

ie-agent>
```

## 3. 普通对话流程

用户输入：

```text
ie-agent> 什么是互信息？它和信息熵有什么区别？
```

CLI 内部构造：

```python
request = AgentRequest(
    session_id=current_session_id,
    user_query="什么是互信息？它和信息熵有什么区别？",
    history=session_history,
    mode="study",
    attachments=[],
)
```

输出格式：

```text
[Answer]
互信息衡量两个随机变量之间共享的信息量...

[Sources]
[1] 信息理论笔记, section 互信息

[Usage]
remote_calls=1, rag_calls=1, tool_calls=0, tokens=820
```

如果 `/trace on`，额外输出：

```text
[Route]
intent=concept
course_tags=[information_theory]
need_rag=true
need_tool=false
target_model=remote
reason=用户询问信息论概念，需要结合课程资料解释
```

## 4. Slash Commands 总表

| 命令 | 负责成员 | 功能 |
| --- | --- | --- |
| `/help` | A | 显示帮助 |
| `/exit` | A | 退出 |
| `/model local|remote|auto` | A | 切换模型模式 |
| `/rag on|off` | A/B | 开关 RAG |
| `/tools list` | A/C | 列出工具 |
| `/mcp add|list|enable|disable|remove` | C | 管理 MCP |
| `/skill install|list|enable|disable|remove` | C | 管理 Skill |
| `/budget show|set|reset` | D/A | 查看和设置预算 |
| `/trace on|off` | A | 开关路由轨迹 |
| `/source show` | A/B | 显示上一轮引用 |
| `/session save|load` | A | 保存和加载会话 |
| `/eval run` | D/A | 运行测试集 |

## 5. `/model`

用途：切换模型调用策略。

格式：

```text
/model auto
/model remote
/model local
```

策略：

- `auto`：默认，根据 Router 决定。
- `remote`：强制远程大模型。
- `local`：优先本地模型或规则路由，本地模型不可用时提示回退。

成功输出：

```text
model mode set to auto
```

错误输出：

```text
unknown model mode: fast
available: local, remote, auto
```

## 6. `/rag`

用途：开关知识库检索。

格式：

```text
/rag on
/rag off
```

成功输出：

```text
rag enabled
```

如果 RAG 未初始化：

```text
rag is enabled, but vector store is empty
run /source show or ingest documents from Web Knowledge page
```

## 7. `/tools list`

用途：列出可用工具。

输出示例：

```text
Builtin Tools
- ohms_law: 欧姆定律计算
- voltage_divider: 分压计算
- discrete_convolution: 离散卷积
- truth_table: 逻辑表达式真值表
- entropy: 信息熵计算

MCP Tools
- none

Skill Tools
- formula-explainer disabled
```

## 8. `/mcp`

### `/mcp add`

用途：添加 MCP server。

stdio 示例：

```text
/mcp add demo --transport stdio --command python --args server.py
```

http 示例：

```text
/mcp add search --transport http --url http://localhost:8000/mcp
```

添加后默认禁用：

```text
mcp server added: demo
status: disabled
run /mcp enable demo to enable it
```

### `/mcp list`

输出：

```text
name     transport  enabled  permission
demo     stdio      false    confirm
search   http       true     confirm
```

### `/mcp enable`

```text
/mcp enable demo
```

输出：

```text
mcp server enabled: demo
```

### `/mcp disable`

```text
/mcp disable demo
```

### `/mcp remove`

```text
/mcp remove demo
```

## 9. `/skill`

### `/skill install`

本地目录：

```text
/skill install ./skills/formula-explainer
```

Git 仓库：

```text
/skill install https://github.com/example/ie-agent-skill.git
```

ZIP：

```text
/skill install ./formula-explainer.zip
```

安装后默认禁用：

```text
skill installed: formula-explainer
status: disabled
run /skill enable formula-explainer to enable it
```

### `/skill list`

输出：

```text
name                enabled  allowed_tools
formula-explainer   false    entropy, discrete_convolution
report-helper       true     rag_search
```

## 10. `/budget`

### `/budget show`

输出：

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

### `/budget set`

格式：

```text
/budget set remote_calls 20
/budget set model_tokens 50000
```

成功输出：

```text
budget updated: remote_calls=20
```

### `/budget reset`

```text
/budget reset
```

## 11. `/trace`

用途：显示或隐藏 Agent 决策过程。

```text
/trace on
/trace off
```

开启后每轮显示：

- intent
- course_tags
- need_rag
- need_tool
- target_model
- selected_tools
- reason

## 12. `/source show`

用途：显示上一轮 RAG 引用。

输出：

```text
[1] 信号与系统讲义, p.12
path=data/raw/signals/chapter2.pdf
chunk_id=signals-ch2-0007
score=0.81

[2] MIT OCW 6.003 Signals and Systems
path=data/raw/public/mit_6003_note.md
chunk_id=mit-6003-0012
score=0.65
```

## 13. `/session`

### `/session save`

```text
/session save final-demo
```

保存到：

```text
data/sessions/final-demo.json
```

### `/session load`

```text
/session load final-demo
```

## 14. `/eval run`

用途：运行测试集。

```text
/eval run
/eval run ie_agent/evaluation/testset.yaml
```

输出：

```text
Evaluation Result
total=30
answer_non_empty=30/30
route_correct=25/30
tool_called_correctly=8/10
citation_returned=7/8
budget_respected=30/30
average_latency=3.2s
```

## 15. 错误处理规范

远程 API 失败：

```text
remote model request failed: API key missing
fallback: rule/local mode
```

RAG 无结果：

```text
no relevant course material found
answer will be generated without citations
```

工具输入错误：

```text
tool error: entropy requires probabilities sum to 1
```

预算超限：

```text
budget blocked: remote_calls reached limit 10
try /budget set remote_calls 20 or /model local
```

## 16. CLI 实现建议

推荐使用 Typer 做命令入口，Rich 做显示。

伪代码：

```python
def chat():
    session = SessionStore.create()
    while True:
        user_input = prompt("ie-agent> ")
        if user_input.startswith("/"):
            handle_command(user_input, session)
            continue
        request = AgentRequest(
            session_id=session.id,
            user_query=user_input,
            history=session.history,
            mode=session.mode,
        )
        response = orchestrator.run(request)
        render_response(response, trace=session.trace_enabled)
        session.append(user_input, response)
```

## 17. 展示建议

展示时准备三条问题：

1. RAG 概念题：
   - “根据课程资料解释互信息和熵的区别。”
2. 工具计算题：
   - “计算二元对称信道 p=0.1 的信道容量，并解释意义。”
3. 扩展和预算题：
   - “/budget show”
   - “/skill list”
   - “/mcp list”

