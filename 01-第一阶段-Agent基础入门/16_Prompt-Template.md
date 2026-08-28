# 16. Prompt Template：组件化管理 prompt

> 阶段：第一阶段 | 为什么需要模板：prompt 写死在代码里难以维护和复用

## 为什么需要 Prompt Template？

新手常见的写法：
```python
prompt = f"你是一个翻译官，把以下文本翻译成{language}：\n{text}"
```

问题：
- prompt 散落在代码各处，难以统一管理
- 变量多了字符串拼接容易出错
- 无法复用和测试

LangChain 的 PromptTemplate 解决了这些问题。

## 基础用法

```python
from langchain_core.prompts import PromptTemplate

# 方式一：from_template
prompt = PromptTemplate.from_template("你是一个{role}，请回答：{question}")
print(prompt.format(role="Python专家", question="什么是装饰器？"))

# 方式二：直接构造
prompt2 = PromptTemplate(
    input_variables=["language", "text"],
    template="把以下文本翻译成{language}：\n{text}"
)
```

## 常用模板类型

### 1. ChatPromptTemplate（对话模板）
```python
from langchain_core.prompts import ChatPromptTemplate

chat_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}，用中文回答，简洁专业。"),
    ("human", "{question}"),
])

messages = chat_prompt.format_messages(role="Python讲师", question="解释GIL")
```

### 2. FewShotPromptTemplate（少样本提示）
```python
from langchain_core.prompts import FewShotPromptTemplate

examples = [
    {"input": "开心", "output": "😊"},
    {"input": "难过", "output": "😢"},
    {"input": "生气", "output": "😠"},
]

example_prompt = PromptTemplate.from_template("输入：{input}\n输出：{output}")

few_shot = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    suffix="输入：{input}\n输出：",
    input_variables=["input"]
)

print(few_shot.format(input="惊讶"))
```

### 3. PipelinePromptTemplate（管道组合）
```python
from langchain_core.prompts import PipelinePromptTemplate

# 把大 prompt 拆成多个小组件
introduction = PromptTemplate.from_template("你是{persona}。")
task = PromptTemplate.from_template("你的任务是{task}。")
example = PromptTemplate.from_template("示例：{example}")

full_prompt = PromptTemplate.from_template("{introduction}\n{task}\n{example}\n{input}")

pipeline = PipelinePromptTemplate(
    final_prompt=full_prompt,
    pipeline_prompts=[
        ("introduction", introduction),
        ("task", task),
        ("example", example),
    ]
)

print(pipeline.format(
    persona="Python专家",
    task="解释代码",
    example="代码: print(1) → 输出1",
    input="代码: [x*2 for x in range(5)]"
))
```

## 实战：RAG 问答模板

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

# RAG 标准模板
rag_template = ChatPromptTemplate.from_messages([
    ("system", """你是一个专业的问答助手。
根据以下参考资料回答用户问题。
要求：
1. 只根据参考资料回答，不要编造
2. 如果资料中没有，回答"参考资料中未找到相关信息"
3. 回答要简洁准确

参考资料：
{context}"""),
    ("human", "{question}"),
])

llm = ChatOpenAI(model="gpt-4o", temperature=0)
chain = rag_template | llm

# 使用
response = chain.invoke({
    "context": "Python是一种解释型、面向对象的编程语言，由Guido van Rossum于1991年发布。",
    "question": "Python是谁发明的？"
})
print(response.content)
```

## 最佳实践

1. **变量命名清晰**：用 `context`、`question` 而不是 `a`、`b`
2. **System prompt 放角色和规则**：Human prompt 放具体问题
3. **FewShot 用 2-5 个示例**：太多会增加 token，太少效果不好
4. **模板单独存放**：复杂项目可以把 prompt 放到单独的 .py 或 .yaml 文件
5. **用 partial_variables 预填常量**：如 format_instructions

## 学习要点

1. PromptTemplate 让 prompt 可复用、可维护
2. ChatPromptTemplate 支持多角色消息（system/human/ai）
3. FewShotPromptTemplate 用示例引导大模型输出格式
4. PipelinePromptTemplate 把大 prompt 拆成小组件组合
5. RAG 场景一定要在 system prompt 里强调"只根据资料回答"

---
**上一篇**：[Output Parser 实战](./15_Output-Parser实战.md) | **下一篇**：[Runnable](./17_Runnable-把写逻辑变成组装chain.md)
