# IE-Agent 信息工程学习科研助手

IE-Agent 是一个面向信息工程专业课程学习、实验辅助和资料问答的 Agent 项目。系统主入口设计为 CLI，交互方式类似 Codex 或 Claude Code；网页端只做轻量展示。项目重点是做出一个接口清楚、分工明确、能运行、能评估、能展示的完整 Agent 系统。

## 项目目标

本项目对应 `requirements.md` 中的大模型与智能体课程要求，核心能力包括：

- LLM + Agent：用大模型生成自然语言回答，用 Agent 完成任务拆解、路由和工具调用。
- RAG：检索课程资料、讲义、笔记，并在回答中展示引用来源。
- 信息工程专业工具：支持电路、信号系统、数字逻辑、信息理论相关计算。
- MCP 与 Skill：支持扩展外部工具和安装专业能力包。
- 用量控制：统计并限制远程模型、RAG、工具、MCP、Skill 的调用。
- CLI 与 Web：CLI 是主展示入口，Web 端用于简单聊天、资料上传和用量展示。

## 文档入口

所有详细开发资料放在 `doc/` 文件夹中。

```text
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
```

建议阅读顺序：

1. 先读 `doc/README.md`。
2. 所有人共同阅读 `doc/00_shared/interfaces.md`。
3. 按个人分工进入对应成员文件夹。
4. 开发时如果接口有变化，同步修改 `doc/00_shared/interfaces.md`。

## 四人分工

| 成员 | 分工 | 文档位置 |
| --- | --- | --- |
| 成员 A | CLI + Agent Core | `doc/01_member_A_cli_agent/` |
| 成员 B | RAG Service | `doc/02_member_B_rag/` |
| 成员 C | Tools + MCP/Skill | `doc/03_member_C_tools_extensions/` |
| 成员 D | Usage + Web + Evaluation | `doc/04_member_D_usage_web_testing/` |

每个成员都需要完成编程任务，不把主要工作放在资料整理和清洗上。每个人先做 mock 和最小可运行版本，再做完整功能。

## 推荐开发流程

### 1. 不直接在 `master` 开发

`master` 只放稳定版本和最终提交。平时不要直接在 `master` 上写代码、改接口或提交实验性功能。

### 2. 每个人新建自己的开发分支

分支命名统一使用：

```text
dev/名字
```

示例：

```text
dev/yuna
dev/member-a
dev/zhangsan
```

创建分支：

```bash
git checkout master
git pull --ff-only origin master
git checkout -b dev/your-name
git push -u origin dev/your-name
```

之后个人日常开发都在自己的 `dev/名字` 分支完成。

### 3. 连调时使用集成分支

需要模块连调时，不要直接把所有人的分支合到 `master`。建议先建一个临时集成分支：

```text
integration/dev
```

负责人可以这样操作：

```bash
git checkout master
git pull --ff-only origin master
git checkout -b integration/dev
git merge dev/member-a
git merge dev/member-b
git merge dev/member-c
git merge dev/member-d
```

如果 `integration/dev` 已经存在：

```bash
git checkout integration/dev
git pull origin integration/dev
git merge dev/member-a
```

连调成功后，再把 `integration/dev` 合并回 `master`。

### 4. 合并回 `master`

合并回 `master` 前至少满足：

- 自己模块的单元测试能跑。
- 与其他模块对接的接口没有私自改名。
- 如果公共接口变化，已经更新 `doc/00_shared/interfaces.md`。
- CLI 主流程能启动，或者不影响当前稳定功能。
- 没有提交 API key、缓存、大模型权重、临时数据。

推荐流程：

```bash
git checkout master
git pull --ff-only origin master
git merge integration/dev
git push origin master
```

### 5. 冲突处理规则

出现冲突时按下面顺序处理：

1. 公共接口以 `doc/00_shared/interfaces.md` 为准。
2. Agent 主流程以成员 A 的 `AgentOrchestrator` 对接方式为准。
3. RAG、工具、用量模块不能绕过统一接口直接互相调用。
4. 先解决代码冲突，再跑对应模块测试。
5. 冲突解决后在提交信息里写清楚改了哪些接口。

### 6. 强制推送规则

- 不要强推别人的 `dev/名字` 分支。
- 个人分支如果需要整理提交，可以强推自己的分支，但要先确认没有其他人在这个分支工作。
- `master` 只有负责人可以 force push。
- 远端已有别人新提交时，不要用普通 `--force`，优先用：

```bash
git push --force-with-lease
```

## 推荐提交信息

提交信息尽量写清楚模块和动作：

```text
Add CLI chat loop
Implement RAG fallback search
Add information theory tools
Add budget usage tracker
Update shared interfaces
```

不要使用太模糊的提交信息，例如：

```text
update
fix
test
```

## 最小可交付版本

如果时间紧，优先保证下面功能：

1. `ie-agent chat` 可以进入连续对话。
2. RAG 能导入少量资料并返回引用。
3. Agent 能调用至少 3 类专业工具。
4. `/budget show` 能展示本轮会话用量。
5. `/mcp list` 和 `/skill list` 能展示扩展管理能力。
6. Web demo 能打开聊天页和用量统计页。
7. 30 条测试问题能跑出评估结果。

## 不要提交的内容

以下内容不要提交到仓库：

- `.env`
- API key
- 大模型权重
- `data/vector_store/` 中的大型索引文件
- Python 缓存
- 临时日志
- 个人无关文件

建议后续补充 `.gitignore`，排除缓存、密钥和大文件。

