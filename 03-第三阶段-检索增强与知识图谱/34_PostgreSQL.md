# 34. PostgreSQL：AI 时代最适合的数据库

> 阶段：第三阶段 | 为什么选 PG：一个数据库搞定关系+向量+全文+JSON

## 为什么 PostgreSQL 适合 AI 项目？

| 能力 | PostgreSQL | MySQL | MongoDB | 专用向量库 |
|------|-----------|-------|---------|-----------|
| 关系型数据 | ✅ 强 | ✅ 强 | ❌ | ❌ |
| JSON 存储 | ✅ JSONB | ✅ JSON | ✅ 原生 | ❌ |
| 全文检索 | ✅ tsvector | ✅ 全文索引 | ❌ | ❌ |
| 向量检索 | ✅ pgvector | ❌ | ❌ | ✅ 专业 |
| 事务支持 | ✅ 强 | ✅ 强 | ❌ 弱 | ❌ |

**结论**：PostgreSQL + pgvector 一个数据库搞定 AI 项目的所有存储需求，不用维护多个数据库。

## 环境准备

```bash
# Docker 启动带 pgvector 的 PostgreSQL
docker run -d \
  --name pgvector \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=agent_db \
  -p 5432:5432 \
  pgvector/pgvector:pg16

# Python 依赖
pip install psycopg2-binary sqlalchemy pgvector openai langchain
```

## 核心功能演示

### 1. 向量检索（pgvector）

```python
# pgvector_demo.py
import psycopg2
from openai import OpenAI

client = OpenAI()
conn = psycopg2.connect(host="localhost", dbname="agent_db", user="postgres", password="password")
cur = conn.cursor()

# 启用 pgvector 扩展
cur.execute("CREATE EXTENSION IF NOT EXISTS vector")
cur.execute("""CREATE TABLE IF NOT EXISTS documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)  -- text-embedding-3-small 是 1536 维
)""")

# 插入向量数据
def embed(text):
    return client.embeddings.create(input=text, model="text-embedding-3-small").data[0].embedding

docs = ["Python是编程语言", "LangChain是LLM框架", "PostgreSQL是数据库"]
for doc in docs:
    emb = embed(doc)
    cur.execute("INSERT INTO documents (content, embedding) VALUES (%s, %s)", (doc, str(emb)))
conn.commit()

# 向量检索（余弦相似度）
query_emb = embed("什么是LLM框架？")
cur.execute("""
    SELECT content, 1 - (embedding <=> %s) as similarity
    FROM documents
    ORDER BY embedding <=> %s
    LIMIT 3
""", (str(query_emb), str(query_emb)))
for row in cur.fetchall():
    print(f"相似度: {row[1]:.4f} | {row[0]}")
```

### 2. 全文检索

```sql
-- PostgreSQL 内置全文检索，不需要 ES
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    tsv tsvector GENERATED ALWAYS AS (to_tsvector('simple', coalesce(title,'') || ' ' || coalesce(content,''))) STORED
);
CREATE INDEX idx_articles_tsv ON articles USING GIN(tsv);

-- 搜索
SELECT title, ts_rank(tsv, plainto_tsquery('simple', 'python AI')) as rank
FROM articles
WHERE tsv @@ plainto_tsquery('simple', 'python AI')
ORDER BY rank DESC
LIMIT 10;
```

### 3. JSONB 存储

```python
# JSONB 适合存储灵活的 Agent 配置、对话记录
cur.execute("""CREATE TABLE IF NOT EXISTS conversations (
    id SERIAL PRIMARY KEY,
    user_id INT,
    messages JSONB,
    metadata JSONB DEFAULT '{}'::jsonb
)""")

# 插入 JSON
messages = [{"role": "user", "content": "你好"}, {"role": "assistant", "content": "你好！"}]
cur.execute("INSERT INTO conversations (user_id, messages) VALUES (%s, %s)",
            (1, json.dumps(messages, ensure_ascii=False)))

# 查询 JSON 字段
cur.execute("SELECT messages->0->>'content' as first_msg FROM conversations WHERE user_id = 1")
```

## LangChain 集成

```python
from langchain_community.vectorstores import PGVector
from langchain_openai import OpenAIEmbeddings

CONNECTION_STRING = "postgresql+psycopg2://postgres:password@localhost:5432/agent_db"

vectorstore = PGVector(
    collection_name="my_docs",
    connection_string=CONNECTION_STRING,
    embedding_function=OpenAIEmbeddings(),
)

# 添加文档
vectorstore.add_texts(["Python入门", "LangChain教程"])

# 检索
results = vectorstore.similarity_search("什么是LangChain", k=2)
```

## 学习要点

1. PostgreSQL + pgvector 一个数据库搞定 AI 项目所有存储
2. pgvector 支持余弦相似度、L2距离、内积三种距离计算
3. PG 内置全文检索，小项目不需要额外部署 ES
4. JSONB 适合存储灵活的 Agent 配置和对话记录
5. 生产环境建议用 PGVector 托管服务（如 Supabase、RDS for PG）

---
**上一篇**：[DeepAgents 实战](./33_DeepAgents实战.md) | **下一篇**：[Redis 短期记忆](./35_Redis-实现Agent短期记忆.md)
