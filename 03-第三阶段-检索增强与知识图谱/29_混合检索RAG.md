# 29. 混合检索 RAG：多路召回 + 重排模型

> 阶段：第三阶段 | 解决问题：单一检索方式的局限性

## 为什么需要混合检索？

| 检索方式 | 优点 | 缺点 |
|----------|------|------|
| 向量检索 | 语义匹配好，能理解同义词 | 精确关键词差，可能漏关键信息 |
| 全文检索（ES/BM25） | 关键词精确，速度快 | 语义理解差，同义词匹配不到 |
| 混合检索 | 兼顾语义和精确 | 实现复杂，需要重排 |

**结论**：企业级 RAG 必须用混合检索，召回率和准确率都更高。

## 混合检索架构

```
用户提问
    ├── 向量检索（语义匹配）──┐
    ├── 全文检索（关键词匹配）──┤── 合并去重 ── 重排模型(Reranker) ── Top-K 结果
    └── 知识图谱检索（关系）───┘
```

## 完整实现

```python
# hybrid_rag.py
from typing import List, Dict
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from elasticsearch import Elasticsearch
from sentence_transformers import CrossEncoder
import numpy as np

# ============ 初始化 ============
llm = ChatOpenAI(model="gpt-4o", temperature=0)
embeddings = OpenAIEmbeddings()
es = Elasticsearch("http://localhost:9200")
reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')

# ============ 1. 准备数据 ============
loader = TextLoader("knowledge_base.txt", encoding='utf-8')
docs = loader.load()
splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=50)
chunks = splitter.split_documents(docs)

# 向量库
vectorstore = FAISS.from_documents(chunks, embeddings)

# ES 索引（假设已创建）
# es.indices.create(index="kb", body={"mappings": {"properties": {"text": {"type": "text", "analyzer": "ik_max_word"}}}})

# ============ 2. 多路召回 ============
def vector_search(query: str, top_k: int = 10) -> List[Dict]:
    """向量检索"""
    results = vectorstore.similarity_search_with_score(query, k=top_k)
    return [{"text": doc.page_content, "score": float(score), "source": "vector"} for doc, score in results]

def keyword_search(query: str, top_k: int = 10) -> List[Dict]:
    """全文检索（BM25）"""
    result = es.search(index="kb", body={
        "query": {"match": {"text": query}},
        "size": top_k
    })
    return [{"text": hit["_source"]["text"], "score": hit["_score"], "source": "bm25"} for hit in result["hits"]["hits"]]

def hybrid_recall(query: str, top_k: int = 20) -> List[Dict]:
    """混合召回：向量 + 关键词，合并去重"""
    vec_results = vector_search(query, top_k=10)
    kw_results = keyword_search(query, top_k=10)

    # 合并去重（按文本内容去重）
    seen = set()
    merged = []
    for item in vec_results + kw_results:
        key = item["text"][:50]  # 用前50字作为去重键
        if key not in seen:
            seen.add(key)
            merged.append(item)
    return merged

# ============ 3. 重排（Reranker）============
def rerank(query: str, candidates: List[Dict], top_k: int = 5) -> List[Dict]:
    """用 CrossEncoder 重排"""
    if not candidates:
        return []

    # 构造 (query, text) 对
    pairs = [[query, item["text"]] for item in candidates]
    scores = reranker.predict(pairs)

    # 按重排分数排序
    for item, score in zip(candidates, scores):
        item["rerank_score"] = float(score)

    candidates.sort(key=lambda x: x["rerank_score"], reverse=True)
    return candidates[:top_k]

# ============ 4. RAG 问答 ============
def rag_qa(query: str) -> str:
    # 1. 混合召回
    candidates = hybrid_recall(query, top_k=20)
    # 2. 重排
    top_chunks = rerank(query, candidates, top_k=5)
    # 3. 拼接上下文
    context = "\n\n".join([f"[{i+1}] {c['text']}" for i, c in enumerate(top_chunks)])
    # 4. 大模型生成
    prompt = f"""根据以下资料回答问题，引用资料编号。
资料：
{context}
问题：{query}"""
    return llm.invoke(prompt).content

# 使用
print(rag_qa("什么是 RAG？"))
```

## 重排模型推荐

| 模型 | 说明 | 速度 | 效果 |
|------|------|------|------|
| cross-encoder/ms-marco-MiniLM-L-6-v2 | 轻量，英文好 | 快 | 中 |
| BAAI/bge-reranker-base | 中文效果好 | 中 | 好 |
| BAAI/bge-reranker-large | 中文最佳 | 慢 | 最好 |
| cohere rerank API | 商用 API | 快 | 好 |

中文场景推荐用 `BAAI/bge-reranker-base`。

## 学习要点

1. 混合检索 = 向量检索 + 全文检索，兼顾语义和精确
2. 多路召回后要去重，避免重复片段
3. 重排模型（Reranker）大幅提升准确率，是企业级 RAG 标配
4. 召回阶段多召一些（20-50），重排后取 Top-K（3-5）

---
**上一篇**：[ElasticSearch 全文检索](../02-第二阶段-企业级后端与进阶框架/28_ElasticSearch全文检索.md) | **下一篇**：[Neo4j 知识图谱](./30_Neo4j知识图谱和Graph-RAG.md)
