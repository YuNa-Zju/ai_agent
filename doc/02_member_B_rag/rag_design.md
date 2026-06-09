# RAG 模块设计

RAG 是 IE-Agent 的核心能力之一。它让系统回答时能够引用课程资料，而不是只依赖模型记忆。对于本课程项目，RAG 的重点是工程链路完整、接口清楚、可展示，不要求整理非常大的知识库。

## 1. RAG 目标

RAG 模块需要支持：

1. 导入 PDF、Markdown、TXT。
2. 按课程标签组织资料。
3. 自动分块。
4. 生成 embedding。
5. 存入向量库。
6. 根据问题检索相关 chunk。
7. 返回引用来源。
8. 提供 fallback 检索，防止模型或向量库不可用时无法展示。

## 2. 数据来源

推荐资料来源：

- 小组自己的课程笔记。
- 老师课件中允许用于课程项目的内容。
- 实验讲义。
- 自己总结的公式表。
- 公开资料补充，例如 MIT OCW、OpenStax。

不建议一开始就导入太多资料。第一版每门课准备 1 到 2 个小文件即可：

```text
data/raw/
  circuits/
    ohm_law_note.md
    rc_circuit_note.md
  signals/
    convolution_note.md
    fourier_note.md
  digital_logic/
    truth_table_note.md
    sequential_logic_note.md
  information_theory/
    entropy_note.md
    mutual_information_note.md
  ckm/
    ckm_intro_note.md
```

## 3. 课程标签

RAG 检索支持以下标签：

| 标签 | 课程方向 |
| --- | --- |
| `circuits` | 电子电路基础 |
| `signals` | 信号与系统 |
| `digital_logic` | 数字电路设计 |
| `information_theory` | 信息理论 |
| `ckm` | CKM 拓展专题 |
| `general_ai` | Agent、RAG、模型相关资料 |

每个资料导入时都要保存 `course_tags`。

## 4. 文件导入

### 4.1 支持格式

| 格式 | 处理方式 |
| --- | --- |
| `.txt` | 直接读取 |
| `.md` | 保留标题层级，去除过多空行 |
| `.pdf` | 用 `pypdf` 提取文本，记录页码 |

### 4.2 IngestResult

```python
class IngestResult(BaseModel):
    path: str
    chunks_added: int
    status: Literal["success", "error"]
    error: str | None = None
```

### 4.3 导入流程

```text
文件路径
  -> 判断格式
  -> 提取文本
  -> 清理文本
  -> 分块
  -> 生成 metadata
  -> embedding
  -> 写入向量库
```

## 5. 文本清理

清理只做必要工作，不做大量人工整理。

建议规则：

- 去掉连续空行。
- 去掉明显页眉页脚。
- 保留公式附近文字。
- 保留 Markdown 标题。
- PDF 提取失败的页面跳过，并记录 warning。

不建议做：

- 大量手工改写。
- 为每个公式单独整理解释。
- 长时间清洗 OCR 错误。

## 6. Chunk 设计

默认参数：

```text
chunk_size = 500 到 800 中文字符
chunk_overlap = 50 到 100 中文字符
```

每个 chunk 保存：

```python
{
  "chunk_id": "signals-convolution-0001",
  "text": "...",
  "source_id": "signals-convolution",
  "title": "离散卷积笔记",
  "path": "data/raw/signals/convolution_note.md",
  "page": null,
  "section": "卷积定义",
  "course_tags": ["signals"]
}
```

## 7. Embedding 方案

优先级：

1. 本地 embedding 模型，例如中文 sentence-transformers。
2. API embedding，如果已有可用 API。
3. fallback：关键词匹配或 TF-IDF。

为了确保展示不失败，必须实现 fallback：

```text
如果 embedding 模型加载失败：
  使用关键词重叠分数检索
```

## 8. 向量库

推荐 FAISS 或 Chroma 二选一。

### 8.1 FAISS

优点：

- 快。
- 本地运行。
- 依赖相对简单。

缺点：

- metadata 管理需要自己维护。

### 8.2 Chroma

优点：

- metadata 管理方便。
- API 更直观。

缺点：

- 依赖稍多。

建议选择：Chroma 更适合课程项目，因为 metadata 和过滤更方便。

## 9. 检索接口

```python
class RagQuery(BaseModel):
    query: str
    course_tags: list[str] = []
    top_k: int = 5
    min_score: float = 0.2
```

检索步骤：

1. 如果 `course_tags` 不为空，先按课程标签过滤。
2. 对 query 生成 embedding。
3. 检索 top_k。
4. 过滤低分结果。
5. 返回 `RagResult`。

## 10. 引用格式

每个检索结果必须带引用。推荐显示：

```text
[1] 信号与系统卷积笔记, section=卷积定义
[2] 信息理论笔记, page=3
```

内部结构：

```python
class Citation(BaseModel):
    source_id: str
    title: str
    path: str
    page: int | None = None
    section: str | None = None
    chunk_id: str
```

## 11. 给 Agent 的上下文格式

RAG 返回给模型的 prompt 片段建议如下：

```text
以下是课程资料检索结果。回答时优先依据资料内容，并在相关句子后标注引用编号。

[1] 来源：信号与系统卷积笔记 section=卷积定义
内容：卷积描述了输入信号与系统冲激响应之间的叠加关系...

[2] 来源：信号与系统讲义 p.12
内容：离散时间卷积可写作 y[n] = sum_k x[k]h[n-k]...
```

如果没有检索结果：

```text
没有找到可靠课程资料。请不要编造引用，可以基于通用知识回答，并明确说明未找到课程资料来源。
```

## 12. RAG 评估

成员 D 的评估脚本需要检查：

- 是否返回 chunk。
- chunk 是否包含正确课程标签。
- 是否返回 citation。
- citation 是否显示在最终回答里。
- 检索耗时。

测试样例：

```yaml
- id: rag_info_entropy_001
  query: "根据课程资料解释信息熵的定义"
  expected_course_tags: ["information_theory"]
  expect_citation: true
```

## 13. 成员 B 交付清单

- `RagService.ingest()` 可用。
- `RagService.search()` 可用。
- 至少 4 个示例资料文件。
- 能保存和加载向量库。
- embedding 不可用时有 fallback。
- 至少 5 个 RAG 单元测试。

