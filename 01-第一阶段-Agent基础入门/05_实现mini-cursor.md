# 5. 实现 mini cursor：大模型自动调用 tool 执行命令

> 阶段：第一阶段：Agent 基础入门 | 前置知识：Tool 调用基础

## 核心原理

Cursor 这类 AI 编程助手的核心是 **ReAct 循环**：思考 → 行动 → 观察 → 再思考，直到任务完成。

## 环境准备

```bash
pip install langchain langchain-openai python-dotenv
```

## 第一步：定义工具

```python
# tools.py
import subprocess, os
from langchain_core.tools import tool

@tool
def run_shell(command: str) -> str:
    """执行 shell 命令，返回输出（超时30秒）"""
    try:
        r = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=30)
        out = r.stdout + (f"\n[stderr]\n{r.stderr}" if r.stderr else "")
        return out[:5000]
    except Exception as e:
        return f"错误: {e}"

@tool
def read_file(path: str) -> str:
    """读取文件内容"""
    try:
        with open(path, encoding='utf-8') as f:
            return f.read()[:5000]
    except Exception as e:
        return f"读取失败: {e}"

@tool
def write_file(path: str, content: str) -> str:
    """写入文件内容"""
    try:
        os.makedirs(os.path.dirname(path) or '.', exist_ok=True)
        with open(path, 'w', encoding='utf-8') as f:
            f.write(content)
        return f"写入成功: {path}"
    except Exception as e:
        return f"写入失败: {e}"
```

## 第二步：Agent 主循环

```python
# mini_cursor.py
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage, ToolMessage
from tools import run_shell, read_file, write_file

load_dotenv()
llm = ChatOpenAI(model="gpt-4o", temperature=0)
tools = [run_shell, read_file, write_file]
llm_with_tools = llm.bind_tools(tools)
tool_map = {"run_shell": run_shell, "read_file": read_file, "write_file": write_file}

SYSTEM = """你是编程助手，可执行 shell 命令、读写文件。
每次只调用一个工具，根据结果继续操作，任务完成后给出总结。
危险命令（rm -rf /等）禁止执行。"""

def run_agent(task: str, max_steps=10):
    messages = [SystemMessage(content=SYSTEM), HumanMessage(content=task)]
    for step in range(max_steps):
        print(f"\n=== 第 {step+1} 步 ===")
        resp = llm_with_tools.invoke(messages)
        messages.append(resp)
        if not resp.tool_calls:
            print("任务完成:", resp.content)
            return
        for tc in resp.tool_calls:
            print(f"调用 {tc['name']}: {tc['args']}")
            result = tool_map[tc['name']].invoke(tc['args'])
            print(f"结果: {result[:150]}")
            messages.append(ToolMessage(content=result, tool_call_id=tc['id']))

if __name__ == "__main__":
    run_agent("创建 hello.py，写斐波那契数列前10项，然后运行")
```

## 运行

```bash
python mini_cursor.py
```

Agent 会自动：写文件 → 跑命令 → 分析结果 → 给出总结。

## 安全提示

- 加工作目录限制，防止越权访问
- 危险命令加白名单或人工确认
- 设置 max_steps 防止死循环

## 学习要点

1. ReAct 模式 = 思考+行动循环
2. 工具描述要清晰，影响大模型决策
3. 控制输出长度避免 token 溢出
4. 安全第一，shell 执行必须有限制

---
**上一篇**：[从 Tool 开始](./04_从Tool开始-让大模型自动调工具读文件.md) | **下一篇**：[MCP 协议](./06_MCP-可跨进程调用的Tool.md)
