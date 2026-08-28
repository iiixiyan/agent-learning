# 19. LangChain 整体总结：AI Agent 第一阶段学习完成

> 阶段：第一阶段总结 | 恭喜你完成了 Agent 基础入门！

## 第一阶段知识地图

```
Agent 基础入门
├── 核心概念
│   ├── LLM 大模型调用
│   ├── Prompt 提示词工程
│   └── Output Parser 结构化输出
├── Tool 工具调用
│   ├── @tool 装饰器定义工具
│   ├── Function Calling 原理
│   └── ReAct 循环（思考+行动）
├── MCP 协议
│   ├── Model Context Protocol
│   ├── MCP Server/Client
│   └── 跨进程跨语言复用工具
├── RAG 检索增强
│   ├── Document Loader 文档加载
│   ├── Text Splitter 文本切分
│   ├── Embedding 向量化
│   ├── Vector Store 向量数据库（Milvus/FAISS）
│   └── Retriever 检索器
├── Memory 记忆管理
│   ├── 截断（WindowMemory）
│   ├── 总结（SummaryMemory）
│   └── 检索（VectorStoreMemory）
└── LCEL 链式调用
    ├── Runnable 接口
    ├── | 管道符组装
    ├── RunnableParallel 并行
    └── RunnablePassthrough 透传
```

## 核心能力清单

学完第一阶段，你应该能：

| 能力 | 对应技术 |
|------|----------|
| 调用大模型对话 | ChatOpenAI / invoke/stream |
| 让大模型调用工具 | @tool / bind_tools / ReAct |
| 做一个知识库问答 | RAG（Loader→Splitter→Embedding→VectorStore→Retriever） |
| 让 Agent 记住对话 | Memory（截断/总结/检索） |
| 结构化输出数据 | PydanticOutputParser / Function Calling |
| 组装复杂工作流 | LCEL（prompt\|llm\|parser） |
| 跨进程复用工具 | MCP 协议 |

## 最小可用 Agent 代码

```python
# 一个完整的 Agent 示例（第一阶段知识整合）
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import SystemMessage, HumanMessage, ToolMessage
from langchain.memory import ConversationSummaryBufferMemory

@tool
def search_web(query: str) -> str:
    """搜索网络信息"""
    return f"关于'{query}'的搜索结果：..."

@tool
def calculate(expression: str) -> str:
    """数学计算"""
    return str(eval(expression))

llm = ChatOpenAI(model="gpt-4o", temperature=0)
tools = [search_web, calculate]
llm_with_tools = llm.bind_tools(tools)
memory = ConversationSummaryBufferMemory(llm=llm, max_token_limit=500, return_messages=True)

def agent_chat(user_input: str):
    messages = [SystemMessage(content="你是一个智能助手，可搜索网络和计算。")]
    messages += memory.load_memory_variables({})["history"]
    messages.append(HumanMessage(content=user_input))

    for _ in range(5):  # 最多5轮工具调用
        resp = llm_with_tools.invoke(messages)
        messages.append(resp)
        if not resp.tool_calls:
            break
        for tc in resp.tool_calls:
            tool = {"search_web": search_web, "calculate": calculate}[tc["name"]]
            result = tool.invoke(tc["args"])
            messages.append(ToolMessage(content=result, tool_call_id=tc["id"]))

    memory.save_context({"input": user_input}, {"output": resp.content})
    return resp.content

# 使用
print(agent_chat("你好，我叫小明"))
print(agent_chat("我叫什么名字？"))  # 能记住
print(agent_chat("123*456等于多少？"))  # 能调用工具计算
```

## 第二阶段预告

第二阶段我们将学习企业级后端与进阶框架：
- **FastAPI + SSE**：把 Agent 封装成流式 API 接口
- **定时任务**：让 Agent 定时自动执行
- **语音交互**：ASR 语音识别 + TTS 语音合成
- **LangGraph**：图编排引擎，多 Agent 协作
- **Agentic RAG**：自主决策的高级 RAG
- **Docker Compose**：一键部署开发环境

## 学习建议

1. **多动手**：每个知识点都写代码跑一遍，不要只看
2. **做项目**：用第一阶段知识做一个完整的 Agent 应用（如智能客服、知识库问答）
3. **查文档**：LangChain 官方文档很详细，遇到问题先查文档
4. **关注更新**：LangChain 更新很快，关注官方博客和 GitHub Release

## 恭喜！

你已经完成了 AI Agent 开发的第一阶段，掌握了核心基础知识。接下来的第二阶段会更有挑战性，也更接近真实企业项目。

记住：**Agent 开发的核心不是框架，而是解决问题的思维方式**。框架只是工具，理解原理才能灵活运用。

---
**上一篇**：[实战练习 LCEL](./18_实战练习LCEL组装chain.md) | **下一篇**：[FastAPI + LangChain SSE 流式接口](../02-第二阶段-企业级后端与进阶框架/20_FastAPI+LangChain实现SSE流式接口.md)
