# 41. LangFuse：开源可内网部署的 Agent 全链路监测方案

> 阶段：第四阶段 | 解决问题：Agent 调用链追踪、成本统计、效果评估

## 为什么需要 LangFuse？

Agent 系统复杂，调用链很长（Prompt → LLM → Tool → LLM → ...），没有监测工具会导致：
- 不知道每次调用花了多少钱、多少 token
- 出问题无法定位是哪一步出错
- 无法评估 Prompt 改动的效果
- 没有数据支撑优化决策

LangFuse 是开源的 LLM 应用观测平台，支持私有化部署。

## LangSmith vs LangFuse

| 对比 | LangSmith | LangFuse |
|------|-----------|----------|
| 开源 | ❌ 闭源 SaaS | ✅ 开源可自建 |
| 部署 | 只能用官方云 | Docker 一键自建 |
| 数据安全 | 数据在官方服务器 | 数据在自己服务器 |
| 成本 | 按调用量收费 | 免费（自己出服务器费） |
| 功能 | 非常完善 | 核心功能齐全，持续更新 |

**企业内网/数据敏感场景选 LangFuse**。

## Docker 部署

```bash
# docker-compose.yml
version: '3'
services:
  langfuse:
    image: langfuse/langfuse:latest
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/langfuse
      - NEXTAUTH_SECRET=your-secret-key-change-in-production
      - NEXTAUTH_URL=http://localhost:3000
      - TELEMETRY_ENABLED=false
    ports:
      - "3000:3000"
    depends_on:
      - postgres
  postgres:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=langfuse
    volumes:
      - pg_data:/var/lib/postgresql/data
volumes:
  pg_data:
```

```bash
docker-compose up -d
# 访问 http://localhost:3000，注册账号，创建项目，获取 API Key
```

## Python 集成

```python
# pip install langfuse
from langfuse import Langfuse
from langfuse.callback import CallbackHandler
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

# 初始化 LangFuse
langfuse = Langfuse(
    public_key="pk-lf-xxx",
    secret_key="sk-lf-xxx",
    host="http://localhost:3000"
)

# LangChain 回调处理器
langfuse_handler = CallbackHandler(
    public_key="pk-lf-xxx",
    secret_key="sk-lf-xxx",
    host="http://localhost:3000"
)

# ============ 方式一：LangChain 自动追踪 ============
llm = ChatOpenAI(model="gpt-4o", temperature=0)
prompt = ChatPromptTemplate.from_template("用中文解释：{topic}")
chain = prompt | llm

# 调用时传入 handler，自动记录整个链路
result = chain.invoke(
    {"topic": "RAG检索增强"},
    config={"callbacks": [langfuse_handler]}
)
print(result.content)

# ============ 方式二：手动追踪（更灵活）============
def rag_query(question: str):
    # 创建 trace
    trace = langfuse.trace(name="rag_query", metadata={"question": question})

    # 1. 检索阶段
    span_retrieve = trace.span(name="retrieve")
    # ... 执行检索 ...
    retrieved_docs = ["文档1内容", "文档2内容"]
    span_retrieve.end(output={"docs_count": len(retrieved_docs)})

    # 2. LLM 生成阶段
    span_llm = trace.span(name="llm_generate")
    llm_result = llm.invoke(f"根据以下资料回答：{retrieved_docs}\n问题：{question}")
    span_llm.end(
        output={"answer": llm_result.content},
        usage={"input": 100, "output": 50, "total": 150}
    )

    trace.update(output={"answer": llm_result.content})
    return llm_result.content

rag_query("什么是RAG？")
```

## 核心功能

### 1. Trace 追踪
每次请求是一个 Trace，包含多个 Span（阶段），可以看到完整调用链、耗时、token 消耗。

### 2. 成本统计
自动统计每次调用的 token 数和费用，支持按时间、模型、用户维度分析。

### 3. Prompt 管理
在 LangFuse 界面管理 Prompt 版本，支持 A/B 测试，不用改代码就能切换 Prompt。

```python
# 从 LangFuse 获取 Prompt（不用硬编码）
prompt_client = langfuse.get_prompt("rag_prompt", version=1)
prompt = prompt_client.compile(topic="RAG")
```

### 4. 评分与反馈
支持对回答打分（1-5星），收集用户反馈，用于评估和优化。

```python
trace.score(name="relevance", value=4, comment="回答相关但不够详细")
```

### 5. 数据集与评估
上传测试数据集，批量运行评估，对比不同 Prompt/模型的效果。

## 学习要点

1. LangFuse 是开源可自建的 LLM 观测平台，适合企业内网和数据敏感场景
2. LangChain 集成最简单，传一个 CallbackHandler 就自动追踪
3. Trace + Span 模型记录完整调用链，方便定位问题
4. Prompt 管理功能让运营人员不用改代码就能优化 Prompt
5. 生产环境一定要配观测，否则出问题无法排查

---
**上一篇**：[RabbitMQ](./40_RabbitMQ.md) | **下一篇**：[Transformer 架构](./42_图解Transformer架构.md)
