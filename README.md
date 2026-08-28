# Agent 全栈工程师转型之路 — Python 版

> 原课程基于 Node.js(Nest.js) + LangChain JS 技术栈
> 本项目转换为 Python(FastAPI) + LangChain Python 技术栈

## 项目简介

本项目是「Agent 全栈工程师转型之路：企业级知识库项目」系列文章的 Python 版本。通过 LangChain、LangGraph 等 Agent 框架学习 Agent 开发基础，并且学习 Milvus、ElasticSearch、PostgreSQL 等后端基础，然后实现企业级知识库等 Agent 项目，全面掌握 AI Agent 全栈开发技能。

- **原公众号**：神光的幸福生活
- **文章总数**：56 篇
- **已转换（有参考资料）**：32 篇
- **待补充（无参考资料）**：24 篇
- **图片资源**：435 张（33 个文章图片目录）

## 技术栈映射

| 原技术栈（Node.js） | Python 版对应 |
|---------------------|---------------|
| Nest.js (TypeScript) | FastAPI (Python) |
| LangChain JS | LangChain Python |
| LangGraph JS | LangGraph Python |
| npm / yarn / pnpm | pip / poetry / uv |
| `package.json` | `pyproject.toml` / `requirements.txt` |
| `@nestjs/schedule` | APScheduler / Celery Beat |
| SSE (`@sse`) | FastAPI `StreamingResponse` |
| Prisma / TypeORM | SQLAlchemy / Tortoise-ORM |

## 文章目录

### 第一阶段：Agent 基础入门（第 1-19 篇）

| 序号 | 标题 | 状态 |
|------|------|------|
| 1 | 学员真实转型 Agent 成功经验，给大家一些信心 | ✅ 已转换 |
| 2 | 前言：Agent 课程脉络梳理，有方向感的学 | ⏳ 待补充 |
| 3 | AI Agent 开发要学什么？ | ✅ 已转换 |
| 4 | 从 Tool 开始：让大模型自动调工具读文件 | ✅ 已转换 |
| 5 | 实现 mini cursor：大模型自动调用 tool 执行命令 | ⏳ 待补充 |
| 6 | MCP：可跨进程调用的 Tool | ✅ 已转换 |
| 7 | 高德 MCP + 浏览器 MCP：LangChain 复用别人的 MCP Server 有多爽！ | ✅ 已转换 |
| 8 | RAG：把文档向量化，基于向量实现真正的语义搜索 | ✅ 已转换 |
| 9 | 知识库的 loader 和 splitter：从各种来源加载文档并分割成小块 | ✅ 已转换 |
| 10 | LangChain 全部 Splitter，其实只需要其中的一个 | ✅ 已转换 |
| 11 | 向量数据库 Milvus：做 AI Agent 开发必备技术 | ✅ 已转换 |
| 12 | Milvus + RAG 实战：电子书语义检索助手 | ⏳ 待补充 |
| 13 | Memory 管理的三大策略：截断、总结、检索 | ⏳ 待补充 |
| 14 | 结构化大模型输出：output parser 还是 tool？ | ✅ 已转换 |
| 15 | Output Parser 实战：智能录入 + 流式版 mini cursor | ⏳ 待补充 |
| 16 | Prompt Template：组件化管理 prompt | ⏳ 待补充 |
| 17 | Runnable：把写逻辑变成组装 chain | ⏳ 待补充 |
| 18 | 实战练习 LCEL 组装 chain | ✅ 已转换 |
| 19 | LangChain 整体总结：AI Agent 第一阶段学习完成 | ⏳ 待补充 |

### 第二阶段：企业级后端与进阶框架（第 20-28 篇）

| 序号 | 标题 | 状态 |
|------|------|------|
| 20 | Nest + LangChain 实现基于 SSE 的流式 ai 接口 | ✅ 已转换 |
| 21 | Nest + tool 实现 OpenClaw 同款定时任务功能（上） | ✅ 已转换 |
| 22 | Nest + tool 实现 OpenClaw 同款定时任务功能（下） | ⏳ 待补充 |
| 23 | 给 Agent 加上语音交互：ASR + 流式 TTS | ✅ 已转换 |
| 24 | AGUI 协议：Vercel AI SDK + LangChain 实现流式组件渲染 | ⏳ 待补充 |
| 25 | 图编排引擎：LangGraph 和多 Agent 架构 | ✅ 已转换 |
| 26 | Agentic RAG：基于 LangGraph 实现大模型自主决策的 RAG 闭环系统 | ✅ 已转换 |
| 27 | 基于 Docker Compose 的本地开发提效和生产环境部署 | ✅ 已转换 |
| 28 | ElasticSearch 全文检索：倒排索引表 + IK 分词器 + BM25 算法 | ⏳ 待补充 |

### 第三阶段：检索增强与知识图谱（第 29-36 篇）

| 序号 | 标题 | 状态 |
|------|------|------|
| 29 | 混合检索 RAG：多路召回 + 重排模型 | ⏳ 待补充 |
| 30 | Neo4j 知识图谱和 Graph RAG | ✅ 已转换 |
| 31 | LangSmith 全链路观测：从 Agent 调试到 RAG 量化评估 | ✅ 已转换 |
| 32 | DeepAgents：开箱即用的 skill、上下文压缩等 middleware | ✅ 已转换 |
| 33 | DeepAgents 实战：多 Agent 架构的深度调研助手 | ⏳ 待补充 |
| 34 | PostgreSQL：AI 时代最适合的数据库 | ⏳ 待补充 |
| 35 | Redis：实现 Agent 短期记忆存储的最佳方案 | ✅ 已转换 |
| 36 | Mem0：分层记忆 + 三路召回的长期记忆方案 | ✅ 已转换 |

### 第四阶段：存储、消息与监控（第 37-43 篇）

| 序号 | 标题 | 状态 |
|------|------|------|
| 37 | Nest 进阶：企业级 Node.js 后端最主流框架 | ✅ 已转换 |
| 38 | Agent 的对象存储方案：MinIO、RustFS、阿里云 OSS | ✅ 已转换 |
| 39 | 多模态与 OSS 前端直传实战：AI 画板 | ✅ 已转换 |
| 40 | RabbitMQ：Agent 中异步处理的标配方案 | ⏳ 待补充 |
| 41 | LangFuse：开源可内网部署的 Agent 全链路监测方案 | ⏳ 待补充 |
| 42 | 图解 Transformer 架构：大模型底层原理 | ⏳ 待补充 |
| 43 | 大模型训练、推理全流程详细图解 | ⏳ 待补充 |

### 第五阶段：简历、面试与实战（第 44-56 篇）

| 序号 | 标题 | 状态 |
|------|------|------|
| 44 | 真实 Agent 简历参考库 | ✅ 已转换 |
| 45 | 真实 Agent 简历参考库（二） | ✅ 已转换 |
| 46 | Agent 面试题押题精讲：RAG 篇 | ✅ 已转换 |
| 47 | 真实Agent 简历参考库(知识库 RAG专场) | ⏳ 待补充 |
| 48 | 企业级知识库项目：项目介绍、多模态 RAG 流程梳理 | ✅ 已转换 |
| 49 | 企业级知识库项目：PostgreSQL+MongoDB 的文档模块数据库设计 | ⏳ 待补充 |
| 50 | 企业级知识库项目：PDF/XLSX/DOCX/PPTX 文件解析为 md 文档 | ✅ 已转换 |
| 51 | 企业级知识库项目：基于消息队列的异步 RAG 流水线 | ⏳ 待补充 |
| 52 | 企业级知识库项目：全文检索链路 | ⏳ 待补充 |
| 53 | 企业级知识库项目：文档抽取 Neo4j知识图谱的实体 | ✅ 已转换 |
| 54 | 企业级知识库项目：文档审核机制、四种状态流转 | ✅ 已转换 |
| 55 | 企业级知识库项目：用户鉴权流程 | ⏳ 待补充 |
| 56 | 企业级知识库项目：用户模块功能完善 | ⏳ 待补充 |

## 目录结构

```
agent-learning-python/
├── README.md                          ← 本文件
├── 文章目录.md                         ← 完整文章目录
├── IMG/                               ← 图片资源（435张，33个目录）
├── 01-第一阶段-Agent基础入门/          ← 19篇
├── 02-第二阶段-企业级后端与进阶框架/    ← 9篇
├── 03-第三阶段-检索增强与知识图谱/      ← 8篇
├── 04-第四阶段-存储消息与监控/          ← 7篇
└── 05-第五阶段-简历面试与实战/          ← 13篇
```

## 转换说明

1. **图片路径**：原文章使用本地绝对路径 `../IMG/`，已修正为相对路径 `../IMG/`
2. **微信残留**：已清理 iframe 视频标签、付费标记、公众号专用标签等
3. **技术术语**：B 类技术文章已将 Nest.js/Node.js 相关术语替换为 FastAPI/Python 对应
4. **代码截图**：原文章代码多以图片截图形式存在，保留原图（无法自动转换代码截图）
5. **空模板**：无参考资料的文章输出标准空模板，标注"待补充"

## 学习路线

| 阶段 | 主题 | 核心技术 |
|------|------|----------|
| 一 | Agent 基础入门 | Tool、MCP、RAG、Milvus、Memory、LangChain LCEL |
| 二 | 企业级后端与进阶框架 | FastAPI、SSE、语音交互、AGUI、LangGraph、Docker |
| 三 | 检索增强与知识图谱 | 混合检索、Graph RAG、LangSmith、DeepAgents、PostgreSQL、Redis、Mem0 |
| 四 | 存储、消息与监控 | 对象存储、OSS 直传、RabbitMQ、LangFuse、Transformer 原理 |
| 五 | 简历、面试与企业级项目实战 | 简历参考、面试题、企业级知识库全流程开发 |

---

*本文档由自动化转换脚本生成*
*转换时间：2026-08-28*
