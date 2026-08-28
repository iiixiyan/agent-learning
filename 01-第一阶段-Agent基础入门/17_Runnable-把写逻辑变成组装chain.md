# 17. Runnable：把写逻辑变成组装 chain

> 阶段：第一阶段 | 核心概念：LCEL（LangChain Expression Language）

## 为什么需要 Runnable？

传统写法：一步步调用，代码冗长且难以复用。
```python
# 传统写法
prompt = ChatPromptTemplate.from_template("翻译：{text}")
llm = ChatOpenAI()
messages = prompt.format_messages(text="hello")
result = llm.invoke(messages)
print(result.content)
```

LCEL 写法：用管道符 `|` 组装，简洁优雅。
```python
chain = prompt | llm
print(chain.invoke({"text": "hello"}))
```

## Runnable 核心接口

所有 Runnable 都实现三个方法：

| 方法 | 用途 |
|------|------|
| `invoke()` | 同步调用，输入一个输出一个 |
| `stream()` | 流式输出，逐块返回 |
| `batch()` | 批量调用，输入列表输出列表 |

```python
chain = prompt | llm

# 1. 同步调用
result = chain.invoke({"text": "hello"})

# 2. 流式输出
for chunk in chain.stream({"text": "写一首短诗"}):
    print(chunk.content, end="", flush=True)

# 3. 批量调用
results = chain.batch([{"text": "hello"}, {"text": "world"}])
```

## 常用 Runnable 组件

### 1. PromptTemplate | ChatPromptTemplate
```python
from langchain_core.prompts import ChatPromptTemplate
prompt = ChatPromptTemplate.from_template("用中文解释：{topic}")
```

### 2. ChatModel（大模型）
```python
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o", temperature=0)
```

### 3. OutputParser（输出解析）
```python
from langchain_core.output_parsers import StrOutputParser
parser = StrOutputParser()

# 完整 chain：prompt → llm → parser
chain = prompt | llm | parser
result = chain.invoke({"topic": "递归"})
print(result)  # 直接是字符串，不是 message 对象
```

### 4. RunnableLambda（自定义函数）
```python
from langchain_core.runnables import RunnableLambda

def add_prefix(text):
    return f"【翻译结果】{text}"

chain = prompt | llm | StrOutputParser() | RunnableLambda(add_prefix)
```

### 5. RunnableParallel（并行执行）
```python
from langchain_core.runnables import RunnableParallel

# 同时做翻译和摘要
chain = RunnableParallel({
    "translation": ChatPromptTemplate.from_template("翻译：{text}") | llm | StrOutputParser(),
    "summary": ChatPromptTemplate.from_template("总结：{text}") | llm | StrOutputParser(),
})

result = chain.invoke({"text": "Python is a programming language."})
print(result["translation"])
print(result["summary"])
```

## 实战：RAG Chain

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# 1. 准备向量库
vectorstore = FAISS.from_texts(
    ["Python由Guido发明于1991年", "Python是解释型语言"],
    embedding=OpenAIEmbeddings()
)
retriever = vectorstore.as_retriever()

# 2. RAG Prompt
rag_prompt = ChatPromptTemplate.from_template("""根据以下资料回答问题。
资料：{context}
问题：{question}""")

llm = ChatOpenAI(temperature=0)

# 3. 组装 RAG chain
# retriever 检索 → 拼接 context → prompt → llm → parser
def format_docs(docs):
    return "\n".join([d.page_content for d in docs])

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | rag_prompt
    | llm
    | StrOutputParser()
)

# 4. 使用
print(rag_chain.invoke("Python是谁发明的？"))
```

## 学习要点

1. `|` 管道符是 LCEL 的核心，把各个组件串成 chain
2. 常用组合：`prompt | llm | output_parser`
3. `RunnableParallel` 可以并行执行多个 chain
4. `RunnablePassthrough` 透传输入，常用于 RAG
5. 所有 Runnable 都支持 invoke/stream/batch 三种调用方式

---
**上一篇**：[Prompt Template](./16_Prompt-Template.md) | **下一篇**：[实战练习 LCEL](./18_实战练习LCEL组装chain.md)
