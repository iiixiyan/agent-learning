# 15. Output Parser 实战：智能录入 + 流式版 mini cursor

> 阶段：第一阶段 | 前置知识：结构化输出、Tool 调用

## 本文目标

上一篇学了 output parser 和 tool 的区别。这篇用两个实战项目巩固：
1. 智能录入：把自然语言转成结构化数据存入数据库
2. 流式版 mini cursor：支持流式输出的命令执行 Agent

## 实战一：智能录入系统

### 场景
用户用自然语言描述一个人，系统自动提取姓名、年龄、职业等字段，存入 JSON/数据库。

### 代码实现

```python
# smart_input.py
from typing import List, Optional
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import PromptTemplate

# 1. 定义数据结构
class Person(BaseModel):
    name: str = Field(description="姓名")
    age: Optional[int] = Field(description="年龄，未知则为null")
    occupation: str = Field(description="职业")
    skills: List[str] = Field(description="技能列表")
    city: Optional[str] = Field(description="所在城市")

# 2. 创建 parser
parser = PydanticOutputParser(pydantic_object=Person)

# 3. 创建 prompt
prompt = PromptTemplate(
    template="根据以下描述提取人物信息。\n{format_instructions}\n描述：{description}",
    input_variables=["description"],
    partial_variables={"format_instructions": parser.get_format_instructions()}
)

# 4. 创建 chain
llm = ChatOpenAI(model="gpt-4o", temperature=0)
chain = prompt | llm | parser

# 5. 运行
result = chain.invoke({"description": "张三，28岁，北京的软件工程师，会Python和Java，喜欢打篮球"})
print(result)
# output: name='张三' age=28 occupation='软件工程师' skills=['Python', 'Java', '打篮球'] city='北京'

# 6. 转字典存储
import json
print(json.dumps(result.dict(), ensure_ascii=False, indent=2))
```

### 批量录入

```python
descriptions = [
    "李四，35岁，上海的数据科学家，精通Python、SQL、机器学习",
    "王五，刚毕业的大学生，学的是计算机专业",
    "赵六在深圳做产品经理，擅长用户研究和需求分析",
]

for desc in descriptions:
    person = chain.invoke({"description": desc})
    print(f"{person.name}: {person.occupation} in {person.city}")
```

## 实战二：流式版 mini cursor

### 为什么需要流式？
普通 Agent 等大模型完整输出后才显示，用户体验差。流式输出可以边生成边显示，类似 ChatGPT 的打字机效果。

### 代码实现

```python
# streaming_agent.py
import subprocess
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import SystemMessage, HumanMessage, ToolMessage

@tool
def run_cmd(command: str) -> str:
    """执行 shell 命令"""
    try:
        r = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=15)
        return (r.stdout + r.stderr)[:3000]
    except Exception as e:
        return f"错误: {e}"

llm = ChatOpenAI(model="gpt-4o", temperature=0, streaming=True)
tools = [run_cmd]
llm_with_tools = llm.bind_tools(tools)

SYSTEM = "你是编程助手，可执行 shell 命令。用中文回答。"

def run_streaming_agent(task: str):
    messages = [SystemMessage(content=SYSTEM), HumanMessage(content=task)]

    while True:
        print("\n🤔 思考中...", end="", flush=True)

        # 流式获取大模型输出
        response = None
        tool_calls = []
        content_chunks = []

        for chunk in llm_with_tools.stream(messages):
            if chunk.content:
                print(chunk.content, end="", flush=True)
                content_chunks.append(chunk.content)
            if chunk.tool_call_chunks:
                for tc in chunk.tool_call_chunks:
                    if tc.get("index") is not None:
                        idx = tc["index"]
                        while len(tool_calls) <= idx:
                            tool_calls.append({"name": "", "args": "", "id": ""})
                        if tc.get("name"):
                            tool_calls[idx]["name"] += tc["name"]
                        if tc.get("args"):
                            tool_calls[idx]["args"] += tc["args"]
                        if tc.get("id"):
                            tool_calls[idx]["id"] += tc["id"]

        # 构建完整 response
        full_content = "".join(content_chunks)
        if tool_calls:
            # 有工具调用，执行工具
            for tc in tool_calls:
                print(f"\n\n🔧 执行命令: {tc['args']}")
                import json
                args = json.loads(tc["args"]) if isinstance(tc["args"], str) else tc["args"]
                result = run_cmd.invoke(args)
                print(f"📋 结果:\n{result}")

                messages.append(ToolMessage(content=result, tool_call_id=tc["id"]))
        else:
            # 没有工具调用，任务完成
            print("\n\n✅ 任务完成")
            break

if __name__ == "__main__":
    run_streaming_agent("查看当前目录有哪些文件，然后创建一个 test.py 写 hello world")
```

## 学习要点

1. PydanticOutputParser 是结构化输出的首选，类型安全
2. 流式输出需要处理 tool_call_chunks 的增量拼接
3. 智能录入的关键是设计好数据结构和 Field 描述
4. 流式 Agent 的体验比普通 Agent 好很多

---
**上一篇**：[结构化大模型输出](./14_结构化大模型输出.md) | **下一篇**：[Prompt Template](./16_Prompt-Template.md)
