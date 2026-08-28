# 24. AGUI 协议：Vercel AI SDK + LangChain 实现流式组件渲染

> 阶段：第二阶段 | 核心概念：AI 生成式 UI

## 什么是 AGUI？

AGUI（AI Generated UI）是一种让大模型直接生成 UI 组件的技术。传统方式是大模型返回文本，前端固定渲染；AGUI 让大模型返回结构化的组件描述，前端动态渲染对应组件。

**应用场景**：
- 智能客服返回卡片、按钮、表单
- 数据分析返回图表组件
- 代码助手返回代码块、运行结果

## 技术方案

| 方案 | 说明 |
|------|------|
| Vercel AI SDK | 前端 SDK，支持流式渲染 UI 组件 |
| LangChain Tool Calling | 后端用工具调用返回组件数据 |
| 自定义协议 | 约定 JSON 格式，前端解析渲染 |

## Python 后端实现（FastAPI + LangChain）

```python
# agui_backend.py
from typing import Literal
from pydantic import BaseModel
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
import json, asyncio

app = FastAPI()
llm = ChatOpenAI(model="gpt-4o", temperature=0, streaming=True)

# 定义 UI 组件工具
@tool
def render_card(title: str, description: str, image_url: str = "") -> dict:
    """渲染一个卡片组件"""
    return {"type": "card", "title": title, "description": description, "image": image_url}

@tool
def render_button(text: str, action: str) -> dict:
    """渲染一个按钮组件"""
    return {"type": "button", "text": text, "action": action}

@tool
def render_chart(chart_type: Literal["bar", "line", "pie"], data: list, title: str) -> dict:
    """渲染一个图表组件"""
    return {"type": "chart", "chart_type": chart_type, "data": data, "title": title}

tools = [render_card, render_button, render_chart]
llm_with_tools = llm.bind_tools(tools)
tool_map = {t.name: t for t in tools}

SYSTEM = """你是一个智能助手，可以渲染 UI 组件。
根据用户需求，选择合适的工具渲染组件。
可以同时渲染多个组件。用中文回答。"""

async def agui_stream(user_input: str):
    messages = [{"role": "system", "content": SYSTEM}, {"role": "user", "content": user_input}]

    # 流式输出文本
    async for chunk in llm_with_tools.astream(messages):
        if chunk.content:
            yield f"data: {json.dumps({'type': 'text', 'content': chunk.content}, ensure_ascii=False)}\n\n"
        if chunk.tool_call_chunks:
            for tc in chunk.tool_call_chunks:
                if tc.get("name"):
                    yield f"data: {json.dumps({'type': 'tool_start', 'name': tc['name']}, ensure_ascii=False)}\n\n"

    # 执行工具并返回组件
    response = llm_with_tools.invoke(messages)
    if response.tool_calls:
        for tc in response.tool_calls:
            tool = tool_map[tc["name"]]
            result = tool.invoke(tc["args"])
            yield f"data: {json.dumps({'type': 'component', 'component': result}, ensure_ascii=False)}\n\n"

    yield "data: [DONE]\n\n"

@app.post("/agui/chat")
async def chat(body: dict):
    return StreamingResponse(
        agui_stream(body["message"]),
        media_type="text/event-stream"
    )
```

## 前端渲染（简单示例）

```javascript
// 前端用 EventSource 接收 SSE，动态渲染组件
const evtSource = new EventSource('/agui/chat?message=展示Python学习路线');
evtSource.onmessage = (e) => {
  if (e.data === '[DONE]') return;
  const data = JSON.parse(e.data);
  if (data.type === 'text') {
    document.getElementById('output').innerHTML += data.content;
  } else if (data.type === 'component') {
    renderComponent(data.component);
  }
};

function renderComponent(comp) {
  const container = document.getElementById('components');
  if (comp.type === 'card') {
    container.innerHTML += `<div class="card"><h3>${comp.title}</h3>${comp.description}</div>`;
  } else if (comp.type === 'button') {
    container.innerHTML += `<button onclick="${comp.action}">${comp.text}</button>`;
  }
}
```

## 学习要点

1. AGUI 的核心是大模型返回结构化组件数据，前端动态渲染
2. 用 Tool Calling 实现组件渲染是最简单的方案
3. SSE 流式传输让文本和组件可以逐步展示
4. 组件类型需要前后端约定好协议

---
**上一篇**：[语音交互](./23_给Agent加上语音交互.md) | **下一篇**：[LangGraph 图编排](./25_图编排引擎-LangGraph和多Agent架构.md)
