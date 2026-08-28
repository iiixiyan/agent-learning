# 13. Memory 管理的三大策略：截断、总结、检索

> 阶段：第一阶段 | 解决问题：大模型上下文窗口有限，如何管理对话历史

## 为什么需要 Memory 管理？

大模型的上下文窗口有限（如 GPT-4o 是 128K token），对话越来越长时会遇到：
- **超出窗口**：历史消息被截断，模型丢失上下文
- **成本增加**：每次请求都带全部历史，token 消耗大
- **响应变慢**：输入越长，生成越慢
- **注意力分散**：太多无关历史干扰模型判断

**三大策略**：截断（Window）、总结（Summary）、检索（Retrieval），实际项目中通常组合使用。

## 策略一：截断（Window Buffer Memory）

### 原理
只保留最近 N 轮对话，更早的直接丢弃。最简单粗暴。

```
完整对话: [轮1] [轮2] [轮3] [轮4] [轮5] [轮6]
保留最近3轮:          [轮4] [轮5] [轮6]
```

### 实现

```python
from langchain.memory import ConversationBufferWindowMemory
from langchain_openai import ChatOpenAI
from langchain.chains import ConversationChain

# 只保留最近 5 轮
memory = ConversationBufferWindowMemory(k=5)

llm = ChatOpenAI(model="gpt-4o", temperature=0)
conversation = ConversationChain(llm=llm, memory=memory, verbose=False)

# 对话
conversation.predict(input="我叫小明")
conversation.predict(input="我喜欢Python")
conversation.predict(input="我在学Agent开发")
conversation.predict(input="我前面说了什么？")
# 模型能记住最近5轮的内容

# 查看内存中的历史
print(memory.load_memory_variables({}))
```

### 优缺点
| 优点 | 缺点 |
|------|------|
| 实现简单，零额外成本 | 早期信息完全丢失 |
| 响应快，token 可控 | 不适合需要长期记忆的场景 |
| 适合短对话、闲聊 | 重要信息可能被挤掉 |

## 策略二：总结（Summary Memory）

### 原理
用大模型把历史对话总结成一段摘要，新对话继续追加，超出阈值时重新总结。

```
对话历史 → 大模型总结 → "用户叫小明，喜欢Python，在学Agent开发..."
新对话 → 追加到摘要 → 定期重新总结
```

### 实现

```python
from langchain.memory import ConversationSummaryMemory
from langchain_openai import ChatOpenAI
from langchain.chains import ConversationChain

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 总结型记忆，每次对话后自动更新摘要
memory = ConversationSummaryMemory(llm=llm)

conversation = ConversationChain(llm=llm, memory=memory)

# 多轮对话
conversation.predict(input="我叫小明，是一名后端开发")
conversation.predict(input="我想转型做Agent开发")
conversation.predict(input="你有什么学习路线建议吗？")

# 查看总结内容
print(memory.buffer)
# 输出类似: "用户介绍自己叫小明，是一名后端开发，想转型做Agent开发，询问学习路线建议..."
```

### 进阶：总结 + 缓冲（SummaryBufferMemory）

```python
from langchain.memory import ConversationSummaryBufferMemory

# 最近 N 轮保留原文，更早的总结
memory = ConversationSummaryBufferMemory(
    llm=llm,
    max_token_limit=500  # 超过 500 token 就开始总结
)

conversation = ConversationChain(llm=llm, memory=memory)
```

### 优缺点
| 优点 | 缺点 |
|------|------|
| 保留长期信息的概要 | 细节会丢失，总结可能有偏差 |
| token 消耗相对稳定 | 每次总结需要额外调用大模型（有成本） |
| 适合中长对话 | 总结质量依赖大模型能力 |

## 策略三：检索（Retrieval Memory）

### 原理
把所有历史消息向量化存入向量库，需要时检索与当前问题最相关的历史片段。类似 RAG，但检索的是对话历史。

```
所有对话历史 → 向量化 → 存入向量库
当前问题 → 向量化 → 检索相关历史 → 拼入 Prompt
```

### 实现

```python
from langchain.memory import VectorStoreRetrieverMemory
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain.chains import ConversationChain

# 1. 创建向量库
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vector_store = FAISS.from_texts([""], embeddings)

# 2. 创建检索型记忆
retriever = vector_store.as_retriever(search_kwargs={"k": 3})
memory = VectorStoreRetrieverMemory(retriever=retriever)

# 3. 保存一些历史信息
memory.save_context(
    {"input": "我叫小明，在字节跳动工作"},
    {"output": "你好小明，很高兴认识你！"}
)
memory.save_context(
    {"input": "我主要用Python做后端开发"},
    {"output": "Python是很好的语言，适合Agent开发"}
)
memory.save_context(
    {"input": "我最近在学LangChain和LangGraph"},
    {"output": "这两个是Agent开发的主流框架"}
)

# 4. 提问时自动检索相关历史
llm = ChatOpenAI(model="gpt-4o", temperature=0)
conversation = ConversationChain(llm=llm, memory=memory)

# 问一个需要历史信息的问题
response = conversation.predict(input="我是做什么工作的？用什么语言？")
print(response)
# 模型会检索到相关历史："在字节跳动工作"、"用Python做后端开发"
```

### 优缺点
| 优点 | 缺点 |
|------|------|
| 可以保留无限长的历史 | 需要额外维护向量库 |
| 只检索相关信息，不浪费 token | 检索可能漏掉重要但不相关的信息 |
| 适合需要长期记忆的场景 | 实现复杂度较高 |

## 三大策略对比

| 维度 | 截断 | 总结 | 检索 |
|------|------|------|------|
| 实现难度 | ⭐ 简单 | ⭐⭐ 中等 | ⭐⭐⭐ 复杂 |
| 额外成本 | 无 | 每次总结调用LLM | 向量化+向量库 |
| 信息保留 | 只保留最近N轮 | 保留概要，丢细节 | 保留全部，按需检索 |
| 适用场景 | 短对话、闲聊 | 中长对话、客服 | 长期记忆、个人助手 |
| Token 控制 | 最好 | 中等 | 取决于检索数量 |

## 最佳实践：组合使用

生产环境通常组合多种策略：

```python
from langchain.memory import ConversationSummaryBufferMemory
from langchain_openai import ChatOpenAI

# 推荐方案：总结+缓冲
# - 最近 1000 token 保留原文（精确）
# - 更早的对话总结成摘要（保留长期信息）
# - 重要信息可以额外存入向量库做检索
memory = ConversationSummaryBufferMemory(
    llm=ChatOpenAI(model="gpt-4o"),
    max_token_limit=1000,
    return_messages=True
)
```

### 企业级 Agent 的 Memory 架构

```
┌─────────────────────────────────────────────┐
│              当前对话上下文                    │
│  ┌───────────────────────────────────────┐  │
│  │  系统 Prompt + 最近 5 轮原文（Window） │  │
│  └───────────────────────────────────────┘  │
│  ┌───────────────────────────────────────┐  │
│  │  历史对话摘要（Summary）                 │  │
│  └───────────────────────────────────────┘  │
│  ┌───────────────────────────────────────┐  │
│  │  检索到的相关历史/知识库（Retrieval）    │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

## 学习要点

1. 三大策略：截断（简单但丢信息）、总结（保概要但丢细节）、检索（保全部但复杂）
2. 实际项目中组合使用：最近对话保留原文 + 早期对话总结 + 重要信息检索
3. `ConversationSummaryBufferMemory` 是 LangChain 中最实用的记忆类型
4. 检索型记忆类似 RAG，把对话历史当知识库检索
5. 记忆管理的核心是在"信息完整性"和"token 成本"之间找平衡
6. 生产环境还要考虑：多用户隔离、记忆持久化、重要信息手动标注

---
**上一篇**：[Milvus + RAG 实战](./12_Milvus+RAG实战.md) | **下一篇**：[结构化大模型输出](./14_结构化大模型输出.md)
