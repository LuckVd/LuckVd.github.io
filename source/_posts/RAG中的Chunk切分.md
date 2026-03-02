---
title: RAG中的Chunk切分
date: 2026-03-02 23:01:10
tags:
---
# RAG 中的 Chunk 切分策略

在构建 RAG（Retrieval-Augmented Generation）系统时，Chunk 切分是最容易被忽视但影响最关键的步骤之一。

很多人把 RAG 的失败归咎于：
- LLM 模型不够强
- 检索算法不先进
- 向量数据库选型错误

但真正的常见原因是：
> Chunk 切分策略选择不当

本文将系统讲清楚：

1. 为什么 RAG 必须进行 chunk 切分
    
2. 5 种主流 chunking 方法及适用场景
    
3. chunk_size 和 chunk_overlap 如何配置
    
4. 不同文档类型应该用什么策略
    
5. 如何评估和优化 chunk 效果
    
6. 生产环境的工程化实践

---

# 一、为什么 RAG 必须进行 Chunk 切分？

## 1.1 Embedding 模型的上下文窗口限制

所有 embedding 模型都有最大输入长度：

- OpenAI `text-embedding-3-small`: 8191 tokens
- OpenAI `text-embedding-3-large`: 8191 tokens
- Cohere `embed-english-v3.0`: 512 tokens
- Llama 2 embeddings: 2048 tokens

如果输入超过这个限制：
- 超出的 tokens 会被截断
- 重要上下文信息可能丢失
- 检索质量严重下降

## 1.2 检索精度 vs 上下文完整性的权衡

假设有一个 50 页的技术文档：

**如果整个文档作为一个 chunk：**
- ✅ 上下文完整
- ❌ 向量表示过于"平均化"
- ❌ 检索时可能召回不相关的内容

**如果切分成 500 字符的小块：**
- ✅ 检索精准
- ❌ 单个 chunk 可能缺少必要上下文
- ❌ LLM 生成答案时信息不完整

这就是：
> RAG 的核心矛盾 - 召回粒度 vs 信息完整性

## 1.3 长上下文 LLM 也不能解决全部问题

即使使用 Claude 4 Sonnet（200k 上下文）：
- **Lost in the Middle 问题**：埋藏在长文档中间的相关信息被忽略
- **延迟增加**：处理更长输入，推理变慢
- **成本上升**：更多 tokens = 更高的 API 费用

因此，chunk 切分仍然是必要的。

---

# 二、Chunk 切分的 5 种主流方法

## 2.1 Fixed-size Chunking（固定大小切分）

### 原理
设定一个固定的 token 数量或字符数，简单粗暴地切分。

### 优点
- ✅ 实现简单，一行代码搞定
- ✅ 性能可预测
- ✅ 适合大多数通用场景

### 缺点
- ❌ 可能切分在句子中间
- ❌ 不考虑语义边界
- ❌ 同一话题可能被分散到多个 chunk

### 代码示例（LangChain）

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,      # 每个 chunk 1000 字符
    chunk_overlap=200,     # 重叠 200 字符
    length_function=len,     # 按字符计算
)

chunks = text_splitter.split_text(your_text)
```

### 适用场景
- 快速原型开发
- 无明显结构的文档
- 大规模批量处理

---

## 2.2 Content-aware Chunking（内容感知切分）

### 原理
利用文档的自然结构（段落、章节、标题）来切分。

### 子方法

#### a. 句子级切分

```python
import nltk
from langchain_text_splitters import NLTKTextSplitter

text_splitter = NLTKTextSplitter(chunk_size=1000, chunk_overlap=200)
chunks = text_splitter.split_text(your_text)
```

- **优点**：保持语义完整性
- **缺点**：句子长短不一，chunk 大小不固定

#### b. 段落级切分

```python
text_splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n", " ", ""],  # 按顺序尝试分隔符
    chunk_size=1000,
    chunk_overlap=200,
)
```

- **优先级**：双换行（段落）→ 单换行（行）→ 空格 → 字符
- **优点**：保留文档结构
- **缺点**：某些段落可能很长

### 适用场景
- 有清晰结构的文章
- Markdown/HTML 文档
- 需要语义连贯性的场景

---

## 2.3 Recursive Character Chunking（递归字符切分）

### 原理
按分隔符优先级递归切分，直到达到目标大小。

### 工作流程

```
原文
  ↓ (尝试 \n\n 切分)
大段落列表
  ↓ (如果段落太长，尝试 \n 切分)
行列表
  ↓ (如果行太长，尝试 空格 切分)
单词列表
  ↓ (如果单词太多，按字符切分)
最终 chunks
```

### LangChain 实现

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", " ", ""],  # 递归分隔符
    add_start_index=True,  # 追踪原始位置
)

chunks = text_splitter.split_documents(documents)
```

### 为什么是"推荐选择"？

- **平衡性**：既考虑了结构，又控制了大小
- **通用性**：适用于大多数文本类型
- **LangChain 默认**：有良好的生态支持

---

## 2.4 Semantic Chunking（语义切分）

### 原理
使用 embedding 计算语义相似度，在主题转变处切分。

### 实现步骤

```
1. 将文档切分成句子
2. 计算每句的 embedding
3. 滑动窗口组合句子
4. 计算相邻组合的相似度
5. 在相似度突降处切分（主题转换）
```

### 代码示意

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer('all-MiniLM-L6-v2')

# 1. 分句
sentences = [s.strip() for s in text.split('.') if s.strip()]

# 2. 生成 embeddings
embeddings = model.encode(sentences)

# 3. 计算相邻相似度
similarities = np.dot(embeddings[:-1], embeddings[1:].T)

# 4. 找切分点（相似度低的点）
threshold = 0.3
chunks = []
current_chunk = [sentences[0]]

for i in range(1, len(sentences)):
    if similarities[i-1] < threshold:
        chunks.append('. '.join(current_chunk))
        current_chunk = [sentences[i]]
    else:
        current_chunk.append(sentences[i])

chunks.append('. '.join(current_chunk))
```

### 优点
- ✅ 语义完整性最好
- ✅ 每个 chunk 都是自包含的主题
- ✅ 召回质量高

### 缺点
- ❌ 计算成本高（需要 embedding）
- ❌ 处理速度慢
- ❌ chunk 大小不均匀

### 适用场景
- **高质量要求**的 RAG 系统
- 知识库问答
- 学术文档检索

---

## 2.5 Document Structure Chunking（文档结构切分）

### 原理
解析文档的原始格式，按结构元素切分。

### 各类文档的处理

#### PDF 文档
- 识别标题层级（H1, H2, H3）
- 按章节切分
- 保留表格、图片位置

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import MarkdownHeaderTextSplitter

loader = PyPDFLoader("document.pdf")
docs = loader.load()

# 如果提取了 Markdown，可以按标题切分
markdown_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "Header 1"),
        ("##", "Header 2"),
        ("###", "Header 3"),
    ]
)
splits = markdown_splitter.split_text(markdown_text)
```

#### HTML 文档
- 解析 `<h1>`, `<h2>`, `<h3>` 标签
- 识别 `<p>`, `<ul>`, `<table>` 等块级元素
- 保留链接和元数据

```python
from langchain_text_splitters import HTMLHeaderTextSplitter

html_splitter = HTMLHeaderTextSplitter(
    headers_to_split_on=[
        ("h1", "Header 1"),
        ("h2", "Header 2"),
        ("h3", "Header 3"),
    ]
)

splits = html_splitter.split_text(html_content)
```

#### Markdown 文档
- 识别 `#`, `##`, `###` 标题
- 按章节切分
- 保留代码块

#### 代码文件
- 按函数、类切分
- 保留注释和文档字符串

### 优点
- ✅ 保留原始结构
- ✅ 上下文最完整
- ✅ 适合专业文档

### 缺点
- ❌ 需要特定解析器
- ❌ 实现复杂度高
- ❌ 不同格式需要不同策略

### 适用场景
- 技术文档
- API 文档
- 学术论文
- 代码库

---

## 2.6 Contextual Chunking（上下文切分）

### 原理
为每个 chunk 生成上下文摘要，保持全局信息。

### Anthropic 的 Contextual Retrieval

```
1. 读取完整文档
2. 切分成基础 chunks
3. 用 LLM 为每个 chunk 生成简短上下文描述
4. 将描述添加到 chunk 内容中
5. 对增强后的 chunk 进行 embedding
```

### 示例

**原始 chunk：**
```
向量数据库使用索引加速搜索。FAISS 是一个高效的近似最近邻搜索库。
```

**添加上下文后：**
```
上下文：本文介绍向量数据库和检索技术，重点是性能优化
内容：向量数据库使用索引加速搜索。FAISS 是一个高效的近似最近邻搜索库。
```

### 优点
- ✅ 每个 chunk 都有全局上下文
- ✅ 召回质量最高
- ✅ 减少检索噪声

### 缺点
- ❌ 成本最高（需要 LLM 生成上下文）
- ❌ 处理速度最慢
- ❌ 文档更新后需要重新生成

### 适用场景
- 复杂文档检索
- 多主题混合文档
- 企业级知识库

---

# 三、关键参数配置

## 3.1 chunk_size（块大小）

### 常见选择

| chunk_size | 适用场景 | 优点 | 缺点 |
|-----------|---------|------|------|
| **128-256 tokens** | 高精度检索、细粒度问题 | 召回精准 | 上下文不足 |
| **512 tokens** | 平衡选择 | 通用性好 | 需要仔细调优 |
| **1000-1500 tokens** | 问答系统、长文档 | 上下文丰富 | 召回范围大 |
| **2000+ tokens** | 长文档总结 | 信息完整 | 精度下降 |

### 选择原则

```
1. 看用户查询类型
   - 短具体问题 → 较小 chunk（256-512）
   - 开放性问题 → 较大 chunk（1000+）

2. 看文档类型
   - 技术文档 → 较小 chunk（精确）
   - 叙事文章 → 较大 chunk（连贯）

3. 看 LLM 上下文窗口
   - 确保总召回 tokens < LLM 上下文 * 50%
   - 给 Prompt 预留空间
```

### 经验公式

```
chunk_size ≈ (目标召回数量 × 平均答案长度) / 2

示例：
- 目标召回 3 个 chunk
- 平均答案 300 tokens
- chunk_size ≈ (3 × 300) / 2 = 450 tokens
```

---

## 3.2 chunk_overlap（块重叠）

### 为什么需要 overlap？

假设有两个句子：
```
Chunk A: "用户可以使用 API 密钥访问服务..."
Chunk B: "API 密钥应该定期更..."
```

用户问："如何管理 API 密钥？"

如果没有 overlap：
- 可能只召回 Chunk B
- 但 A 包含了"访问服务"的信息
- 答案不完整

有了 overlap：
- 两个 chunk 都会被召回
- 信息更完整

### 常见 overlap 设置

| overlap 比例 | 适用场景 |
|-------------|---------|
| **10%** | 文档连贯性强、句子长 |
| **15-20%** | **通用推荐** |
| **25%+** | 文档碎片化、需要更多冗余 |

### 经验公式

```
chunk_overlap = chunk_size × 0.15 ~ 0.20

示例：
- chunk_size = 1000
- overlap = 150 ~ 200
```

### 重要提示

**overlap 不是越大越好：**

- 过小：信息丢失
- **适中：** 最佳平衡点（15-20%）
- 过大：冗余信息多，检索噪声增加

---

# 四、不同文档类型的策略选择

## 4.1 文档类型决策树

```
开始
  ↓
文档是代码吗？
  ├─ 是 → 按函数/类切分
  └─ 否 ↓
文档有清晰结构吗（标题、章节）？
  ├─ 是 → 文档结构切分
  └─ 否 ↓
是长文档吗（>50 页）？
  ├─ 是 → Semantic + Fixed-size 混合
  └─ 否 ↓
是通用文本吗？
  ├─ 是 → Recursive Character（推荐）
  └─ 否 → 根据特点定制
```

---

## 4.2 具体策略

### 技术文档
- **方法**：文档结构切分 + 适度 overlap
- **chunk_size**：500-800 tokens
- **overlap**：15%
- **理由**：保持上下文，又保证精度

### 学术论文
- **方法**：章节切分（Abstract, Introduction, Conclusion 分开）
- **chunk_size**：1000-1500 tokens
- **overlap**：20%
- **理由**：每个章节是独立主题

### 新闻文章
- **方法**：递归字符切分
- **chunk_size**：256-512 tokens
- **overlap**：10%
- **理由**：新闻主题集中，小块即可

### 对话记录
- **方法**：按对话轮次切分
- **chunk_size**：3-5 轮对话
- **overlap**：1 轮
- **理由**：对话有自然边界

### 代码库
- **方法**：按文件 + 函数切分
- **chunk_size**：1-3 个函数
- **overlap**：0
- **理由**：代码有清晰结构，不需要 overlap

---

# 五、如何评估 Chunk 效果

## 5.1 评估指标

### 召回质量

**手工评估：**
- 准备 50-100 个测试问题
- 检查召回的 chunks 是否包含答案
- 计算召回率（Recall@k）

**自动化评估：**
```python
# 假设有标注的答案位置
from sklearn.metrics import recall_score

def evaluate_recall(retrieved_chunks, gold_chunks, k=5):
    """计算 Top-k 召回率"""
    hits = sum(1 for gold in gold_chunks if gold in retrieved_chunks[:k])
    return hits / len(gold_chunks)

recall_3 = evaluate_recall(retrieved, gold_standard, k=3)
recall_5 = evaluate_recall(retrieved, gold_standard, k=5)
```

### 检索效率

- **平均响应时间**：应该 < 500ms
- **数据库大小**：chunk 数量可控
- **索引构建时间**：可接受范围内

---

## 5.2 A/B 测试框架

```python
# 伪代码：测试不同 chunk 策略

strategies = {
    "fixed_500": RecursiveCharacterTextSplitter(chunk_size=500, overlap=100),
    "fixed_1000": RecursiveCharacterTextSplitter(chunk_size=1000, overlap=200),
    "semantic": SemanticChunker(threshold=0.3),
}

for strategy_name, splitter in strategies.items():
    # 1. 切分文档
    chunks = splitter.split_documents(docs)

    # 2. 建立索引
    vector_store.index_documents(chunks)

    # 3. 运行测试集
    results = run_test_set(test_questions, vector_store)

    # 4. 记录指标
    print(f"{strategy_name}: Recall={results['recall']:.3f}")
```

---

## 5.3 优化流程

```
1. 建立测试集（50-100 个真实问题）
2. 实现多种 chunk 策略
3. 并行测试，记录指标
4. 选择表现最好的策略
5. 在生产环境小规模测试
6. 收集用户反馈
7. 迭代优化
```

---

# 六、生产环境工程化实践

## 6.1 配置管理

### 配置示例（YAML）

```yaml
chunking:
  strategy: recursive  # fixed, recursive, semantic, structural
  chunk_size: 1000
  chunk_overlap: 200
  separators:
    - "\n\n"
    - "\n"
    - " "
    - ""

  # 文档类型覆盖
  document_types:
    markdown:
      strategy: structural
      chunk_size: 800

    pdf:
      strategy: structural
      chunk_size: 1000

    code:
      strategy: file_based
      chunk_size: 500
```

---

## 6.2 Chunk 扩展（Chunk Expansion）

### 原理
检索到 chunks 后，自动扩展其上下文。

### 实现方式

```python
def expand_chunks(retrieved_chunks, all_chunks, window=1):
    """扩展召回的 chunks，包含前后邻居"""
    expanded = []

    for chunk in retrieved_chunks:
        # 获取原始文档中的位置
        idx = all_chunks.index(chunk)

        # 获取前后 neighbors
        start = max(0, idx - window)
        end = min(len(all_chunks), idx + window + 1)

        # 合并
        expanded_chunk = " ".join(
            c.page_content for c in all_chunks[start:end]
        )

        expanded.append(expanded_chunk)

    return expanded
```

### 优点
- ✅ 低延迟检索（不增加索引）
- ✅ 高质量上下文
- ✅ 灵活性强

### 缺点
- ❌ 增加 LLM 输入 tokens
- ❌ 可能引入无关信息

---

## 6.3 缓存策略

### Embedding 缓存

```python
from functools import lru_cache

@lru_cache(maxsize=10000)
def get_embedding(text: str):
    """缓存 embedding 结果"""
    return embedding_model.embed(text)
```

### Chunk 缓存

```python
import hashlib

def chunk_cache_key(text: str, size: int, overlap: int):
    """生成 chunk 唯一标识"""
    data = f"{text}:{size}:{overlap}".encode()
    return hashlib.md5(data).hexdigest()

# 检查缓存
cache_key = chunk_cache_key(original_text, 1000, 200)
if cached := redis.get(cache_key):
    chunks = json.loads(cached)
else:
    chunks = splitter.split_text(original_text)
    redis.set(cache_key, json.dumps(chunks), ex=3600)
```

---

# 七、常见问题与解决方案

## Q1: 如何处理多语言文档？

**问题**：中英文混合文档，切分可能出问题。

**解决方案**：
```python
# 使用支持多语言的分句器
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", "。", "！", "？", " ", ""],
)
```

---

## Q2: 如何处理表格和图片？

**问题**：表格被切分后信息不完整。

**解决方案**：
```python
# 1. 检测表格
def is_table(chunk: str) -> bool:
    return "|" in chunk and "\n|"

# 2. 特殊处理
if is_table(chunk):
    # 将整个表格作为一个 chunk
    chunks.append(extract_full_table(document))
```

---

## Q3: 如何动态调整 chunk 大小？

**问题**：固定大小不能适应所有文档。

**解决方案**：
```python
def adaptive_chunking(document: str):
    """根据文档内容动态调整"""

    # 检测文档类型
    if is_technical_doc(document):
        size = 500
        overlap = 100
    elif is_news_article(document):
        size = 300
        overlap = 50
    else:
        size = 1000
        overlap = 200

    splitter = RecursiveCharacterTextSplitter(
        chunk_size=size,
        chunk_overlap=overlap,
    )

    return splitter.split_text(document)
```

---

## Q4: 如何处理实时更新的文档？

**问题**：文档更新后，chunks 需要重新生成。

**解决方案**：
```python
# 使用文档哈希作为版本标识
def document_version(doc: str) -> str:
    return hashlib.sha256(doc.encode()).hexdigest()

# 对比版本
if current_version != stored_version:
    # 重新切分和索引
    chunks = splitter.split_text(doc)
    vector_store.update(document_id, chunks)
    redis.set(f"doc:{doc_id}:version", current_version)
```

---

# 八、最终总结

## 核心原则

1. **没有银弹**：不同场景需要不同策略
2. **从简单开始**：先 Fixed-size，再迭代优化
3. **数据驱动**：用测试集评估，不要凭感觉
4. **工程化优先**：配置化、缓存、监控

## 推荐流程

```
第一阶段（MVP）：
  → 使用 RecursiveCharacterTextSplitter
  → chunk_size=1000, overlap=200
  → 快速验证 RAG 流程

第二阶段（优化）：
  → 根据文档类型定制策略
  → 调整参数
  → 建立 A/B 测试

第三阶段（生产）：
  → 配置化策略
  → 添加缓存
  → 监控指标
  → 持续优化
```

## 一句话总结

> Chunk 切分不是技术问题，而是产品设计问题

关键不是"怎么切"，而是：
> "为你的用户和文档类型，选择什么策略"

---

**参考资源：**
- [Pinecone - Chunking Strategies](https://www.pinecone.io/learn/chunking-strategies/)
- [LangChain - RAG Tutorial](https://python.langchain.com/docs/tutorials/rag/)
- [Greg Kamradt - 5 Levels of Text Splitting](https://github.com/FullStackRetrieval-com/RetrievalTutorials)
