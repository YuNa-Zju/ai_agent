# IE-Agent 编程手册

## 1. 项目定位

IE-Agent 是一个面向信息工程专业学生的学习科研 Agent。它不是普通聊天机器人，而是一个可以在终端中连续工作的专业助手。用户可以像使用 Codex 或 Claude Code 一样在 CLI 中提问，系统会自动判断问题类型，然后选择合适能力：

- 课程资料检索：电子电路基础、信号与系统、数字电路设计、信息理论、CKM 等资料。
- 专业工具计算：电路参数、信号卷积、傅里叶变换辅助、真值表、信息熵、互信息、信道容量。
- 大模型解释：对复杂概念、推导过程、实验问题进行自然语言解释。
- MCP 扩展：连接外部工具服务。
- Skill 安装：增加新的专业能力模板。
- 用量控制：限制远程模型、工具、RAG 和会话资源。

网页端只作为展示，不作为主产品。课堂展示时优先使用 CLI，网页端用于展示聊天、上传资料和用量统计。

## 2. 设计目标

### 2.1 功能目标

1. 支持连续对话，并保留会话上下文。
2. 支持自动路由：简单问题本地处理，复杂问题调用远程大模型。
3. 支持课程知识库 RAG，回答时给出引用来源。
4. 支持信息工程专业工具调用，工具结果进入最终回答。
5. 支持 MCP 和 Skill 的安装、启用、禁用、删除。
6. 支持预算管理，防止 API 用量和工具调用失控。
7. 支持测试集评估，能够生成展示用结果表。

### 2.2 工程目标

1. 四个人可以并行开发。
2. 每个人都有明确编程任务。
3. 每个模块有统一接口，方便 mock 和集成。
4. 文档、测试、演示脚本完整。
5. 不依赖四卡 4090，普通电脑也能跑最小系统。

### 2.3 课程目标

项目需要体现以下课程能力：

- 理论：LLM、Agent、RAG、工具调用、MCP、Skill、预算控制。
- 实验：完整系统设计和实现。
- 优化：路由策略、知识库检索、工具增强、用量管理。
- 评估：正确性、引用质量、工具调用准确率、用户满意度。
- 展示：CLI 演示、Web demo、测试报告。

## 3. 系统总架构

```text
用户
  |
  v
CLI / Web Demo
  |
  v
AgentOrchestrator
  |
  +-- UsageManager.check()
  |
  +-- Router
  |     +-- 规则路由
  |     +-- 本地小模型路由，选做
  |
  +-- RAG Service
  |     +-- 资料导入
  |     +-- 向量检索
  |     +-- 引用返回
  |
  +-- ToolRegistry
  |     +-- 专业工具
  |     +-- MCP 工具
  |     +-- Skill 工具
  |
  +-- ModelManager
        +-- 远程大模型 API
        +-- 本地小模型，选做
```

主流程：

1. 用户在 CLI 输入问题。
2. CLI 把输入封装为 `AgentRequest`。
3. `UsageManager` 检查是否超出预算。
4. `Router` 判断问题类型、课程标签、复杂度和是否需要 RAG/工具。
5. 如果需要 RAG，调用 `RagService.search()`。
6. 如果需要工具，调用 `ToolRegistry.run()`。
7. `ModelManager` 调用本地模型或远程模型生成最终回答。
8. `AgentResponse` 返回答案、引用、工具结果、路由轨迹和用量。
9. CLI 以可读格式显示结果。

## 4. 推荐目录结构

```text
ie_agent/
  cli/
    main.py              # CLI 入口
    commands.py          # slash commands
    renderer.py          # Rich 输出
    session_store.py     # 会话保存和加载
  agent/
    orchestrator.py      # Agent 主流程
    router.py            # 意图识别和模型分流
    prompt_builder.py    # prompt 构造
    response_builder.py  # 答案整合
  models/
    base.py              # 模型接口
    local_model.py       # 本地模型，选做
    remote_model.py      # 远程 API
    model_manager.py     # 模型选择
  rag/
    ingest.py            # 资料导入
    chunker.py           # 分块
    embeddings.py        # embedding
    vector_store.py      # 向量库
    retriever.py         # 检索
  tools/
    registry.py          # 工具注册
    circuits.py          # 电路工具
    signals.py           # 信号系统工具
    digital_logic.py     # 数字逻辑工具
    information_theory.py# 信息论工具
  extensions/
    mcp_manager.py       # MCP 管理
    skill_manager.py     # Skill 管理
    permission.py        # 权限确认
  usage/
    budget.py            # 预算配置
    tracker.py           # 用量记录
    pricing.py           # 费用估算
  web/
    app.py               # Streamlit 或 Gradio
  evaluation/
    evaluator.py         # 测试集运行
    testset.yaml         # 30 条测试问题
```

## 5. 运行方式

### 5.1 安装依赖

推荐使用 `requirements.txt` 或 `pyproject.toml` 管理依赖。最小依赖：

```text
typer
rich
pydantic
requests
openai
faiss-cpu 或 chromadb
sentence-transformers
pypdf
python-dotenv
pytest
streamlit 或 gradio
```

### 5.2 环境变量

```text
IE_AGENT_MODEL_PROVIDER=deepseek
DEEPSEEK_API_KEY=your_key
QWEN_API_KEY=your_key
IE_AGENT_VECTOR_STORE=data/vector_store
IE_AGENT_BUDGET_PROFILE=normal
```

不要把 API key 写入代码或报告截图。展示时只展示配置项名称，不展示真实密钥。

### 5.3 CLI 启动

```bash
python -m ie_agent.cli.main chat
```

或安装命令后：

```bash
ie-agent chat
```

示例：

```text
> 请解释信号与系统中的卷积，并帮我算 x[n]=[1,2,1], h[n]=[1,1] 的结果

[route] intent=calculation, course_tags=[signals], need_tool=true, need_rag=true
[tool] discrete_convolution success
[source] 信号与系统讲义 第2章

卷积表示一个系统对输入信号的加权叠加响应...
计算结果为 y[n]=[1,3,3,1]。
```

## 6. Agent 核心逻辑

### 6.1 意图类型

`intent` 取值：

- `concept`：概念解释，例如“什么是互信息”。
- `calculation`：计算题，例如“求这个信道容量”。
- `rag_qa`：需要资料引用的问题，例如“老师课件里怎么定义傅里叶变换”。
- `experiment_debug`：实验故障排查，例如“Verilog 仿真波形不对怎么办”。
- `report`：报告写作辅助，例如“帮我总结本实验设计思路”。
- `extension`：MCP 或 Skill 管理。
- `unknown`：无法判断。

### 6.2 课程标签

`course_tags` 取值：

- `circuits`
- `signals`
- `digital_logic`
- `information_theory`
- `ckm`
- `general_ai`

可以多标签。例如“用信息论解释通信系统中的噪声影响”可以同时有 `information_theory` 和 `signals`。

### 6.3 模型分流策略

默认不依赖本地模型。建议先实现规则路由：

- 短问题、明确公式、命令类问题：本地规则处理。
- 需要专业解释、长文本总结、复杂推导：远程大模型。
- 需要资料出处：先 RAG，再远程大模型整合。
- 需要数值计算：先工具，再远程大模型解释。

本地小模型是选做项，可用于：

- 改写检索 query。
- 判断 intent。
- 判断是否需要工具。
- 简单概念问答。

如果本地小模型不可用，系统必须自动回退到规则路由。

## 7. RAG 设计

RAG 不需要依赖大量人工清洗。只要准备少量高质量示例资料即可：

- 每门课程 2 到 5 页讲义或笔记。
- 每个资料文件有明确标题和课程标签。
- 检索结果返回 chunk 文本、来源文件、页码或段落号。

示例引用格式：

```text
[1] 信号与系统讲义, 第2章, p.12
[2] 信息理论笔记, 互信息.md, section 3
```

Agent 最终回答必须把引用和回答分开显示，避免用户误以为模型自己编造出处。

## 8. 专业工具设计

第一版至少实现以下工具：

| 工具 | 输入 | 输出 |
| --- | --- | --- |
| `ohms_law` | `voltage/current/resistance` 三选二 | 缺失量和计算过程 |
| `voltage_divider` | `vin, r1, r2` | `vout` |
| `discrete_convolution` | `x, h` | 卷积序列 |
| `truth_table` | 逻辑表达式 | 真值表 |
| `entropy` | 概率分布 | 信息熵 |
| `mutual_information` | 联合概率矩阵 | 互信息 |
| `channel_capacity_bsc` | 交叉概率 `p` | BSC 信道容量 |

工具输出不要只返回数字，要返回公式和单位，方便最终回答解释。

## 9. MCP 和 Skill 扩展

项目允许完全开放安装，但必须加控制：

- MCP 可以添加任意 stdio 或 HTTP/SSE server。
- Skill 可以从本地目录、Git 仓库或 ZIP 安装。
- 新扩展默认禁用，需要用户显式启用。
- 第一次调用外部工具前显示权限确认。
- 所有扩展调用都记录到 usage log。

展示时至少准备一个简单 MCP 或 Skill 示例，例如“公式格式化 Skill”或“课程资料摘要 Skill”。

## 10. 用量控制

用量控制是本项目的一个核心亮点。系统要统计：

- 远程模型请求次数。
- 输入 token、输出 token、估算费用。
- RAG 检索次数。
- 工具调用次数。
- MCP/Skill 调用次数。
- 当前会话轮次。

当预算超出时，系统不能直接失败，要给出替代方案：

```text
当前 remote_calls 已达到预算上限。
可以选择：
1. 使用本地/规则模式给出简短回答。
2. 运行 /budget set remote_calls 20 提高上限。
3. 关闭 RAG 或减少 top_k。
```

## 11. Web Demo

Web 端保持简单，只实现三页：

1. Chat：输入问题，显示回答、引用、工具结果。
2. Knowledge：上传资料，显示导入状态。
3. Usage：显示本次运行的用量统计。

Web 后端直接调用 `AgentOrchestrator.run()`，不要另写一套逻辑。

## 12. 开发顺序

第一阶段：接口和 mock

1. 所有人一起确认 `interfaces.md`。
2. 成员 A 写 CLI 和 Agent mock。
3. 成员 B 写 RAG mock。
4. 成员 C 写工具 mock。
5. 成员 D 写 Usage mock 和测试框架。

第二阶段：真实模块

1. 成员 A 接入远程模型。
2. 成员 B 接入 embedding 和向量库。
3. 成员 C 实现专业工具、MCP、Skill。
4. 成员 D 实现 Web 和评估脚本。

第三阶段：集成和展示

1. 跑 30 条测试集。
2. 修复接口不一致。
3. 准备 3 个 CLI 演示场景。
4. 准备报告截图。

## 13. 验收标准

项目完成时必须能做到：

- CLI 能连续对话。
- RAG 能返回引用。
- 工具能被 Agent 调用。
- MCP/Skill 有安装和管理命令。
- 用量控制能显示并拦截超预算调用。
- Web demo 能简单展示。
- 测试集能运行并输出结果。

## 14. 参考资料

- DeepSeek API Docs: https://api-docs.deepseek.com/
- Qwen Model Studio Docs: https://www.alibabacloud.com/help/en/model-studio/
- LangChain Python Docs: https://python.langchain.com/docs/
- MIT OCW 6.002 Circuits and Electronics: https://ocw.mit.edu/courses/6-002-circuits-and-electronics-spring-2007/
- MIT OCW 6.003 Signals and Systems: https://ocw.mit.edu/courses/6-003-signals-and-systems-fall-2011/
- MIT OCW 6.004 Computation Structures: https://ocw.mit.edu/courses/6-004-computation-structures-spring-2017/
- MIT OCW 6.050J Information and Entropy: https://ocw.mit.edu/courses/6-050j-information-and-entropy-spring-2008/
- OpenStax University Physics Volume 2: https://openstax.org/details/books/university-physics-volume-2

