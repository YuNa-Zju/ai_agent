# IE-Agent 项目资料总入口

本文档文件夹用于指导小组实现“信息工程学习科研 Agent”。系统主形态是 CLI，交互方式类似 Codex 或 Claude Code；网页端只作为简单展示页面。项目重点不是堆模型参数，而是做出一个结构清楚、接口统一、能检索课程资料、能调用专业工具、能安装 MCP/Skill、能控制用量的完整 Agent 系统。

## 阅读顺序

1. `00_shared/programming_manual.md`
   - 先读这一份，了解系统目标、整体架构、目录结构、开发流程。
2. `00_shared/team_work_split.md`
   - 四位成员分别读自己的任务，同时看清楚需要和谁对接。
3. `00_shared/interfaces.md`
   - 所有人必须按这里的字段、函数签名和返回格式开发。
4. 按个人分工进入对应文件夹领取任务：
   - 成员 A：`01_member_A_cli_agent/`
   - 成员 B：`02_member_B_rag/`
   - 成员 C：`03_member_C_tools_extensions/`
   - 成员 D：`04_member_D_usage_web_testing/`

## 文件夹分工

```text
doc/
  README.md
  00_shared/
    programming_manual.md      # 全项目主手册
    interfaces.md              # 所有人必须遵守的接口定义
    team_work_split.md         # 四人分工和对接顺序
  01_member_A_cli_agent/
    README.md                  # 成员 A 任务领取卡
    cli_manual.md              # CLI 和 Agent 主流程手册
  02_member_B_rag/
    README.md                  # 成员 B 任务领取卡
    rag_design.md              # RAG 设计和实现手册
  03_member_C_tools_extensions/
    README.md                  # 成员 C 任务领取卡
    tools_and_extensions.md    # 专业工具、MCP、Skill 手册
  04_member_D_usage_web_testing/
    README.md                  # 成员 D 任务领取卡
    usage_control.md           # 用量控制手册
    testing_and_demo.md        # 测试、Web demo、展示手册
```

## 个人阅读说明

- 成员 A 必读：`00_shared/interfaces.md`、`01_member_A_cli_agent/README.md`、`01_member_A_cli_agent/cli_manual.md`
- 成员 B 必读：`00_shared/interfaces.md`、`02_member_B_rag/README.md`、`02_member_B_rag/rag_design.md`
- 成员 C 必读：`00_shared/interfaces.md`、`03_member_C_tools_extensions/README.md`、`03_member_C_tools_extensions/tools_and_extensions.md`
- 成员 D 必读：`00_shared/interfaces.md`、`04_member_D_usage_web_testing/README.md`、`04_member_D_usage_web_testing/usage_control.md`、`04_member_D_usage_web_testing/testing_and_demo.md`

## 原计划资料对应关系

- `programming_manual.md` 已移入 `00_shared/programming_manual.md`
- `interfaces.md` 已移入 `00_shared/interfaces.md`
- `team_work_split.md` 已移入 `00_shared/team_work_split.md`
- `cli_manual.md` 已移入 `01_member_A_cli_agent/cli_manual.md`
- `rag_design.md` 已移入 `02_member_B_rag/rag_design.md`
- `tools_and_extensions.md` 已移入 `03_member_C_tools_extensions/tools_and_extensions.md`
- `usage_control.md` 已移入 `04_member_D_usage_web_testing/usage_control.md`
- `testing_and_demo.md` 已移入 `04_member_D_usage_web_testing/testing_and_demo.md`

## 各成员第一天要做什么

- 成员 A：建立 CLI 和 Agent mock，保证 `AgentOrchestrator.run()` 能返回假回答。
- 成员 B：建立 RAG mock，保证 `RagService.search()` 能返回假引用。
- 成员 C：建立 ToolRegistry mock，保证 `/tools list` 和一个工具调用能跑。
- 成员 D：建立 UsageManager mock，保证 `/budget show` 和预算检查能跑。

## 领取任务方式

每个成员打开自己的文件夹后，按 `README.md` 的顺序领取任务。建议每个人先完成“最小交付”，再做“加分交付”。这样集成时不会因为某个复杂功能没写完影响全组进度。

## 旧版阅读顺序备忘

如果需要按功能阅读，也可以按下面顺序：

1. `00_shared/programming_manual.md`
2. `00_shared/interfaces.md`
3. `00_shared/team_work_split.md`
4. `01_member_A_cli_agent/cli_manual.md`
5. `02_member_B_rag/rag_design.md`
6. `03_member_C_tools_extensions/tools_and_extensions.md`
7. `04_member_D_usage_web_testing/usage_control.md`
8. `04_member_D_usage_web_testing/testing_and_demo.md`

<!-- The following section keeps the original project explanation. -->

## 项目一句话说明

IE-Agent 是一个面向信息工程专业课程学习和实验辅助的终端智能体。它能够根据用户问题自动判断是否需要课程知识库、专业计算工具、MCP 外部工具或 Skill，然后生成带引用、带计算过程、带用量记录的回答。

## 对应课程要求

| requirements.md 要求 | 本项目对应内容 |
| --- | --- |
| 理论学习和背景研究 | 在报告中介绍 LLM、Agent、RAG、工具调用、MCP、Skill 的理论背景 |
| 实验设计 | 选择信息工程学习科研助手作为应用场景，设计 CLI-first Agent 框架 |
| 系统实现 | 使用 Python、PyTorch/Transformers 可选、本地或 API 大模型、向量库、Web demo 实现 |
| 能力构建和优化 | 设计 RAG、专业工具、Skill、MCP、用量控制和路由策略 |
| 评估和测试 | 设计 30 条专业测试集、模块测试、集成测试、用量测试和展示测试 |
| 报告与展示 | 使用 CLI 演示 3 个完整场景，Web 展示聊天、上传资料和用量统计 |

## 推荐最终仓库结构

```text
project1/
  requirements.md
  doc/
    README.md
    00_shared/
      programming_manual.md
      interfaces.md
      team_work_split.md
    01_member_A_cli_agent/
      README.md
      cli_manual.md
    02_member_B_rag/
      README.md
      rag_design.md
    03_member_C_tools_extensions/
      README.md
      tools_and_extensions.md
    04_member_D_usage_web_testing/
      README.md
      usage_control.md
      testing_and_demo.md
  ie_agent/
    __init__.py
    cli/
      main.py
      commands.py
      renderer.py
      session_store.py
    agent/
      orchestrator.py
      router.py
      prompt_builder.py
      response_builder.py
    models/
      base.py
      local_model.py
      remote_model.py
      model_manager.py
    rag/
      ingest.py
      chunker.py
      embeddings.py
      vector_store.py
      retriever.py
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
    usage/
      budget.py
      tracker.py
      pricing.py
    web/
      app.py
    evaluation/
      evaluator.py
      testset.yaml
  data/
    raw/
    processed/
    vector_store/
  tests/
```

## 四人并行开发原则

- 所有人先实现自己的最小可运行版本，不等待其他人完全完成。
- 所有跨模块调用都走 `00_shared/interfaces.md` 中定义的 Pydantic 类或 JSON 结构。
- 每个成员都要写编程任务，不把主要工作放在资料整理和清洗上。
- 每个模块都要有 mock 版本，方便其他成员提前对接。
- 每天合并一次接口变更，接口字段不要随意改名。

## 推荐技术栈

- 语言：Python 3.10 或以上
- CLI：Typer 或 Click，Rich 用于彩色输出和表格
- Web demo：Streamlit 或 Gradio
- 向量库：FAISS 或 Chroma
- Embedding：`bge-small-zh-v1.5`、`text2vec-base-chinese` 或 API embedding
- 大模型：DeepSeek API、Qwen API 或其他 OpenAI-compatible API
- 本地模型：可选，只做路由和轻问答；不能跑时用规则路由代替
- 数据结构：Pydantic
- 测试：pytest

## 最小可交付版本

如果时间紧，必须保证以下功能可运行：

1. `ie-agent chat` 能进入连续对话。
2. 用户问课程问题时，系统能走 RAG 并返回引用。
3. 用户问计算题时，系统能调用至少 3 类专业工具。
4. `/budget show` 能展示本轮会话用量。
5. `/mcp list`、`/skill list` 能展示扩展管理功能。
6. Web demo 能打开聊天页和用量统计页。
7. 30 条测试问题能跑出评估结果表。

## 加分扩展

- 实现 MCP server 的真实连接和工具调用。
- 实现 Skill 从本地目录或 Git 仓库安装。
- 为每个回答显示 `route_trace`，说明为什么选择 RAG、工具或远程模型。
- 为 RAG 回答展示引用编号，例如 `[电路基础讲义 p12]`。
- 为工具计算输出公式推导过程，而不是只输出数值。
- 为预算控制做 `saving`、`normal`、`performance` 三种策略。
