# 成员 B 任务领取卡：RAG Service

你负责知识库检索模块。你的主要工作是写代码完成资料导入、分块、检索和引用返回，不是花大量时间整理资料。资料只需要准备少量示例，重点是接口完整、可运行、可对接。

## 你要阅读的文档

1. `../00_shared/interfaces.md`
2. `rag_design.md`
3. `../00_shared/programming_manual.md`
4. `../00_shared/team_work_split.md`

## 你负责的代码目录

```text
ie_agent/
  rag/
    ingest.py
    chunker.py
    embeddings.py
    vector_store.py
    retriever.py
data/
  raw/
  processed/
  vector_store/
```

## 最小交付

第一版必须完成：

1. `RagService.ingest(path)` 能导入 `.md` 和 `.txt`。
2. PDF 导入可作为第二优先级，但建议实现。
3. 能把文本切成 chunk。
4. 每个 chunk 有 `chunk_id`、`text`、`citation`、`course_tags`。
5. `RagService.search(query)` 能返回 `RagResult`。
6. embedding 模型不可用时，有关键词 fallback。
7. 至少准备 4 个示例资料文件。

## 核心接口

你必须实现：

```python
class RagService:
    def ingest(self, path: str, course_tags: list[str] | None = None) -> IngestResult:
        ...

    def search(self, query: RagQuery) -> RagResult:
        ...
```

必须返回：

```python
RagResult(
    chunks=[RagChunk(...)],
    citations=[Citation(...)],
    confidence=0.0,
)
```

## 课程标签

导入资料时使用：

- `circuits`
- `signals`
- `digital_logic`
- `information_theory`
- `ckm`
- `general_ai`

示例：

```python
rag_service.ingest(
    "data/raw/signals/convolution_note.md",
    course_tags=["signals"],
)
```

## 示例资料要求

建议先放这些小文件：

```text
data/raw/
  circuits/ohm_law_note.md
  signals/convolution_note.md
  digital_logic/truth_table_note.md
  information_theory/entropy_note.md
```

每个文件 300 到 800 字即可。内容可以是小组自己写的课程笔记，不需要大量清洗。

## Chunk 规则

默认：

```text
chunk_size = 500 到 800 中文字符
chunk_overlap = 50 到 100 中文字符
```

如果是 Markdown，尽量保留标题：

```text
section = 最近的 Markdown 标题
```

如果是 PDF，记录页码：

```text
page = 当前页码
```

## Embedding 和 fallback

推荐优先用 Chroma 或 FAISS，但必须有 fallback。

fallback 方案：

1. 对 query 分词或按字词切分。
2. 计算 query 和 chunk 的关键词重叠数量。
3. 按分数排序返回 top_k。

这样即使 embedding 依赖装不上，系统也能展示。

## 与成员 A 对接

成员 A 会调用：

```python
rag_result = rag_service.search(
    RagQuery(
        query="互信息 定义",
        course_tags=["information_theory"],
        top_k=5,
    )
)
```

你返回的 `RagResult.citations` 会直接显示在 CLI 里。

## 与成员 D 对接

成员 D 的测试会检查：

- `chunks` 是否非空。
- `citation` 是否存在。
- `course_tags` 是否匹配。
- `confidence` 是否有合理数值。

所以不要只返回字符串，必须返回结构化对象。

## 单元测试

至少写 5 个：

1. Markdown 导入成功。
2. TXT 导入成功。
3. 分块后 chunk 数量大于 0。
4. `course_tags=["signals"]` 能检索到信号资料。
5. embedding 不可用时 fallback 仍能返回结果。

## 验收标准

- 至少 4 个示例资料可导入。
- `RagService.search()` 能被成员 A 直接调用。
- 回答需要引用时能返回 `Citation`。
- 向量库文件可保存到 `data/vector_store/`。
- 没有向量库时 fallback 不崩。

