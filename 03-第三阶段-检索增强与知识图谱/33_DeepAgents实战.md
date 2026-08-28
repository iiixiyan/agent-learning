# 33. DeepAgents 实战：多 Agent 架构的深度调研助手

> 阶段：第三阶段 | 前置知识：DeepAgents 基础、LangGraph

## 项目目标

做一个深度调研助手：用户输入一个主题，Agent 自动规划研究方向、搜索资料、整理报告。

涉及多 Agent 协作：
- **Planner（规划者）**：拆解研究任务，制定计划
- **Researcher（研究员）**：搜索资料，提取信息
- **Writer（写作者）**：整理资料，生成报告
- **Reviewer（审核者）**：审核报告质量，提出修改意见

## 技术选型

- **LangGraph**：图编排，管理多 Agent 状态流转
- **DeepAgents**：提供 skill、上下文压缩等 middleware
- **Tavily Search**：搜索工具（也可以用 SerpAPI、Bing API）

## 核心实现

```python
# research_agent.py
from typing import TypedDict, List, Annotated
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage
import operator

# ============ 状态定义 ============
class ResearchState(TypedDict):
    topic: str
    research_plan: List[str]
    findings: List[str]
    report: str
    review_feedback: str
    iteration: int

# ============ 工具定义 ============
@tool
def web_search(query: str) -> str:
    """搜索网络信息"""
    # 实际项目用 Tavily/SerpAPI
    return f"关于'{query}'的搜索结果：...（模拟）"

@tool
def read_article(url: str) -> str:
    """读取网页内容"""
    return f"网页 {url} 的内容：...（模拟）"

tools = [web_search, read_article]
tool_node = ToolNode(tools)
llm = ChatOpenAI(model="gpt-4o", temperature=0)
llm_with_tools = llm.bind_tools(tools)

# ============ Agent 节点 ============
def planner_node(state: ResearchState):
    """规划者：拆解研究任务"""
    prompt = f"""你是研究规划专家。请为以下主题制定研究计划，列出3-5个研究方向。
主题：{state['topic']}
用JSON数组返回，每个元素是一个研究方向字符串。"""
    response = llm.invoke(prompt)
    import json
    try:
        plan = json.loads(response.content)
    except:
        plan = [response.content]
    return {"research_plan": plan, "findings": []}

def researcher_node(state: ResearchState):
    """研究员：搜索资料"""
    findings = []
    for direction in state["research_plan"]:
        query = f"{state['topic']} {direction}"
        result = web_search.invoke({"query": query})
        findings.append(f"【{direction}】\n{result}")
    return {"findings": findings}

def writer_node(state: ResearchState):
    """写作者：生成报告"""
    context = "\n\n".join(state["findings"])
    prompt = f"""根据以下研究资料，写一份关于'{state['topic']}'的深度调研报告。
要求：结构清晰，包含摘要、正文、结论，字数1000字左右。
研究资料：
{context}"""
    response = llm.invoke(prompt)
    return {"report": response.content}

def reviewer_node(state: ResearchState):
    """审核者：审核报告质量"""
    if state["iteration"] >= 2:  # 最多修改2次
        return {}
    prompt = f"""请审核以下调研报告，指出不足和改进建议。如果质量合格，回复"通过"。
报告：
{state['report']}"""
    response = llm.invoke(prompt)
    if "通过" in response.content:
        return {}
    return {"review_feedback": response.content, "iteration": state["iteration"] + 1}

# ============ 构建图 ============
workflow = StateGraph(ResearchState)
workflow.add_node("planner", planner_node)
workflow.add_node("researcher", researcher_node)
workflow.add_node("writer", writer_node)
workflow.add_node("reviewer", reviewer_node)

workflow.set_entry_point("planner")
workflow.add_edge("planner", "researcher")
workflow.add_edge("researcher", "writer")
workflow.add_edge("writer", "reviewer")

# 条件边：审核通过则结束，否则重新写
def should_rewrite(state):
    return "writer" if state.get("review_feedback") and state["iteration"] < 2 else END

workflow.add_conditional_edges("reviewer", should_rewrite)

app = workflow.compile()

# ============ 运行 ============
result = app.invoke({
    "topic": "2024年AI Agent行业发展趋势",
    "iteration": 0
})
print(result["report"])
```

## DeepAgents Middleware 集成

```python
# DeepAgents 提供的高级功能
from deepagents import DeepAgent
from deepagents.middleware import SkillMiddleware, ContextCompressionMiddleware

# Skill：预定义的能力包
skills = [
    SkillMiddleware(skill_name="web_research"),
    SkillMiddleware(skill_name="data_analysis"),
]

# 上下文压缩：自动压缩历史消息，节省 token
compression = ContextCompressionMiddleware(max_tokens=2000)

agent = DeepAgent(
    llm=llm,
    tools=tools,
    middleware=[compression] + skills
)
```

## 学习要点

1. 多 Agent 架构适合复杂任务，每个 Agent 职责单一
2. LangGraph 用图结构管理状态流转，比链式调用更灵活
3. 审核-修改循环提升输出质量，但要限制迭代次数防止死循环
4. DeepAgents 的 skill 和上下文压缩是生产级必备功能

---
**上一篇**：[DeepAgents 基础](./32_DeepAgents.md) | **下一篇**：[PostgreSQL](./34_PostgreSQL.md)
