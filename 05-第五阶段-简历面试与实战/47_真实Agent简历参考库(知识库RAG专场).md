# 47. 真实 Agent 简历参考库（知识库 RAG 专场）

> 阶段：第五阶段 | 适用人群：有 RAG/知识库项目经验，想求职 Agent 岗位

## 写在前面

这篇专门针对**知识库 RAG 方向**的简历优化。RAG 是目前 Agent 岗位最常见的项目方向，也是面试重点。

## 简历核心结构

```
个人信息
├── 求职意向：AI Agent 开发工程师 / RAG 工程师
├── 技术栈：Python、LangChain、LangGraph、向量数据库、大模型
└── 联系方式

专业技能
├── 大模型应用：LangChain、LangGraph、LlamaIndex、Prompt Engineering
├── 检索增强：RAG、混合检索、Reranker、向量数据库(Milvus/Pinecone/PGVector)
├── 后端开发：FastAPI、Python、PostgreSQL、Redis、RabbitMQ
└── 工程化：Docker、Git、CI/CD、LangSmith/LangFuse 观测

项目经验（重点）
├── 企业级知识库系统（核心项目，详细写）
├── 智能客服 RAG 系统
└── 其他相关项目

工作/实习经历
教育背景
```

## 核心项目写法示例

### 企业级知识库系统（重点项目）

**项目描述**：
> 基于 RAG 技术构建企业级知识库问答系统，支持 PDF/Word/Excel 等多格式文档解析，实现混合检索（向量+全文）+ Reranker 重排，问答准确率提升 40%。

**技术栈**：Python、FastAPI、LangChain、LangGraph、Milvus、ElasticSearch、PostgreSQL、Redis、BGE-Reranker、Docker

**个人职责与成果**（用 STAR 法则，量化成果）：
- **文档解析模块**：基于 unstructured + pypdf 实现 10+ 格式文档解析，支持表格、图片提取，解析准确率 95%+
- **检索优化**：设计混合检索架构（向量检索 + BM25 全文检索 + BGE-Reranker 重排），Top-5 召回率从 68% 提升至 92%
- **异步流水线**：基于 RabbitMQ 实现文档处理异步流水线，支持多阶段处理（解析→切分→向量化→索引），吞吐量提升 3 倍
- **系统架构**：FastAPI 后端 + PostgreSQL 存储 + Redis 缓存 + Milvus 向量库，Docker Compose 一键部署，支持 1000+ 并发
- **效果评估**：构建 200+ 条测试集，用 LangSmith 做 RAG 评估，答案准确率从 65% 提升至 88%

**项目亮点**（面试加分项）：
- 实现了 Agentic RAG，大模型自主决定检索策略（直接回答 / 检索 / 多轮检索）
- 支持多轮对话记忆，基于 Redis 实现短期记忆 + Mem0 长期记忆
- 全链路可观测，LangFuse 追踪每次调用的 token、耗时、成本

## 简历常见问题

### ❌ 错误写法
```
做了一个 RAG 项目，用了 LangChain 和向量数据库，能回答问题。
```
问题：太笼统，没有量化成果，没有技术细节。

### ✅ 正确写法
```
企业级知识库 RAG 系统
- 基于 LangChain + Milvus 构建 RAG 问答系统，支持 PDF/Word 多格式文档
- 实现混合检索（向量+BM25）+ BGE-Reranker 重排，Top-5 召回率 92%
- 用 FastAPI 封装 RESTful API，Redis 缓存热点问题，P99 响应 < 2s
- 构建 200+ 条评估集，答案准确率 88%，支持 1000+ 并发
```

## 面试高频问题（RAG 方向）

1. **RAG 的完整流程是什么？**
   答：文档加载→切分→向量化→存入向量库→用户问题向量化→检索相关片段→拼接 Prompt→大模型生成回答

2. **如何提升 RAG 准确率？**
   答：优化切分策略（按语义/章节切分）、混合检索（向量+全文）、加 Reranker 重排、优化 Prompt、多轮检索、Query 改写

3. **向量数据库怎么选？**
   答：小规模用 FAISS/Chroma，中规模用 PGVector（PostgreSQL 插件），大规模用 Milvus/Pinecone/Qdrant

4. **什么是 Agentic RAG？**
   答：传统 RAG 是固定流程（检索→生成），Agentic RAG 让大模型自主决定是否检索、检索什么、是否多轮检索，用 LangGraph 实现决策循环

5. **如何评估 RAG 效果？**
   答：构建测试集（问题+标准答案+相关文档），评估召回率（Recall@K）、准确率、答案相关性，用 LangSmith/Ragas 工具

## 学习要点

1. RAG 项目简历要量化成果（准确率、召回率、并发数、响应时间）
2. 技术栈要写具体版本和用途，不要只列名词
3. 项目描述用 STAR 法则：情境(S)→任务(T)→行动(A)→结果(R)
4. 重点突出你做了什么优化，带来了什么提升
5. 准备好面试高频问题，理解原理比会用框架更重要

---
**上一篇**：[真实 Agent 简历参考库（二）](./45_真实Agent简历参考库(二).md) | **下一篇**：[企业级知识库项目介绍](./48_企业级知识库项目-项目介绍.md)
