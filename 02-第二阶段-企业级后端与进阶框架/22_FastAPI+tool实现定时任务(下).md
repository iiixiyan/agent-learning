# 22. FastAPI + tool 实现 OpenClaw 同款定时任务功能（下）

> 阶段：第二阶段 | 前置知识：定时任务（上）、FastAPI 基础

## 本文目标

上一篇实现了基础的定时任务功能。这篇继续完善：
1. 任务持久化（重启不丢失）
2. 任务管理 API（增删改查）
3. 任务执行日志
4. 并发控制和错误重试

## 完整实现

```python
# scheduler_app.py
import asyncio
import json
import uuid
from datetime import datetime, timedelta
from typing import Optional, List
from pydantic import BaseModel
from fastapi import FastAPI, HTTPException
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

app = FastAPI(title="Agent 定时任务服务")
scheduler = AsyncIOScheduler()
llm = ChatOpenAI(model="gpt-4o", temperature=0)

# ============ 数据模型 ============
class TaskCreate(BaseModel):
    name: str
    cron: str = "0 9 * * *"  # 每天9点
    prompt: str
    tool_name: Optional[str] = None
    tool_args: Optional[dict] = None
    enabled: bool = True

class TaskUpdate(BaseModel):
    name: Optional[str] = None
    cron: Optional[str] = None
    prompt: Optional[str] = None
    enabled: Optional[bool] = None

class TaskInfo(BaseModel):
    id: str
    name: str
    cron: str
    prompt: str
    enabled: bool
    next_run: Optional[str] = None
    last_run: Optional[str] = None
    last_status: Optional[str] = None

# ============ 工具定义 ============
@tool
def send_notification(message: str) -> str:
    """发送通知消息"""
    print(f"[通知] {message}")
    return f"通知已发送: {message}"

@tool
def generate_report(topic: str) -> str:
    """生成报告"""
    result = llm.invoke(f"生成关于{topic}的简短报告")
    return result.content

tools_map = {"send_notification": send_notification, "generate_report": generate_report}

# ============ 任务存储（JSON 文件持久化） ============
TASKS_FILE = "tasks.json"
LOGS_FILE = "task_logs.json"

def load_tasks() -> dict:
    try:
        with open(TASKS_FILE, encoding='utf-8') as f:
            return json.load(f)
    except:
        return {}

def save_tasks(tasks: dict):
    with open(TASKS_FILE, 'w', encoding='utf-8') as f:
        json.dump(tasks, f, ensure_ascii=False, indent=2)

def append_log(task_id: str, status: str, result: str):
    try:
        with open(LOGS_FILE, encoding='utf-8') as f:
            logs = json.load(f)
    except:
        logs = {}
    if task_id not in logs:
        logs[task_id] = []
    logs[task_id].append({
        "time": datetime.now().isoformat(),
        "status": status,
        "result": result[:500]
    })
    logs[task_id] = logs[task_id][-50:]  # 只保留最近50条
    with open(LOGS_FILE, 'w', encoding='utf-8') as f:
        json.dump(logs, f, ensure_ascii=False, indent=2)

# ============ 任务执行函数 ============
async def execute_task(task_id: str):
    tasks = load_tasks()
    if task_id not in tasks:
        return
    task = tasks[task_id]
    print(f"[执行任务] {task['name']}")

    try:
        # 1. 调用大模型生成内容
        result = llm.invoke(task["prompt"])
        content = result.content

        # 2. 如果指定了工具，调用工具
        if task.get("tool_name") and task["tool_name"] in tools_map:
            tool = tools_map[task["tool_name"]]
            tool_args = task.get("tool_args", {})
            tool_result = tool.invoke(tool_args)
            content += f"\n\n工具执行结果: {tool_result}"

        # 3. 更新任务状态
        tasks[task_id]["last_run"] = datetime.now().isoformat()
        tasks[task_id]["last_status"] = "success"
        save_tasks(tasks)
        append_log(task_id, "success", content)
        print(f"[任务完成] {task['name']}")

    except Exception as e:
        tasks[task_id]["last_run"] = datetime.now().isoformat()
        tasks[task_id]["last_status"] = f"error: {str(e)}"
        save_tasks(tasks)
        append_log(task_id, "error", str(e))
        print(f"[任务失败] {task['name']}: {e}")

# ============ API 接口 ============
@app.post("/tasks", response_model=TaskInfo)
def create_task(task: TaskCreate):
    tasks = load_tasks()
    task_id = str(uuid.uuid4())[:8]
    tasks[task_id] = {
        "id": task_id,
        "name": task.name,
        "cron": task.cron,
        "prompt": task.prompt,
        "tool_name": task.tool_name,
        "tool_args": task.tool_args,
        "enabled": task.enabled,
        "last_run": None,
        "last_status": None
    }
    save_tasks(tasks)

    if task.enabled:
        scheduler.add_job(
            execute_task,
            CronTrigger.from_crontab(task.cron),
            args=[task_id],
            id=task_id,
            replace_existing=True
        )

    return tasks[task_id]

@app.get("/tasks", response_model=List[TaskInfo])
def list_tasks():
    tasks = load_tasks()
    result = []
    for tid, task in tasks.items():
        job = scheduler.get_job(tid)
        task["next_run"] = job.next_run_time.isoformat() if job and job.next_run_time else None
        result.append(task)
    return result

@app.put("/tasks/{task_id}")
def update_task(task_id: str, task: TaskUpdate):
    tasks = load_tasks()
    if task_id not in tasks:
        raise HTTPException(404, "任务不存在")
    update_data = task.dict(exclude_unset=True)
    tasks[task_id].update(update_data)
    save_tasks(tasks)

    # 重新调度
    if scheduler.get_job(task_id):
        scheduler.remove_job(task_id)
    if tasks[task_id]["enabled"]:
        scheduler.add_job(
            execute_task,
            CronTrigger.from_crontab(tasks[task_id]["cron"]),
            args=[task_id],
            id=task_id,
            replace_existing=True
        )
    return tasks[task_id]

@app.delete("/tasks/{task_id}")
def delete_task(task_id: str):
    tasks = load_tasks()
    if task_id not in tasks:
        raise HTTPException(404, "任务不存在")
    del tasks[task_id]
    save_tasks(tasks)
    if scheduler.get_job(task_id):
        scheduler.remove_job(task_id)
    return {"message": "删除成功"}

@app.post("/tasks/{task_id}/run")
async def run_task_now(task_id: str):
    """立即执行一次任务"""
    await execute_task(task_id)
    return {"message": "执行完成"}

@app.get("/tasks/{task_id}/logs")
def get_task_logs(task_id: str, limit: int = 20):
    try:
        with open(LOGS_FILE, encoding='utf-8') as f:
            logs = json.load(f)
        return logs.get(task_id, [])[-limit:]
    except:
        return []

# ============ 启动 ============
@app.on_event("startup")
def start_scheduler():
    scheduler.start()
    # 恢复已启用的任务
    tasks = load_tasks()
    for tid, task in tasks.items():
        if task.get("enabled"):
            scheduler.add_job(
                execute_task,
                CronTrigger.from_crontab(task["cron"]),
                args=[tid],
                id=tid,
                replace_existing=True
            )
    print(f"调度器已启动，恢复 {sum(1 for t in tasks.values() if t.get('enabled'))} 个任务")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## 使用示例

```bash
# 1. 启动服务
python scheduler_app.py

# 2. 创建定时任务（每天早上9点生成日报）
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"name":"每日日报","cron":"0 9 * * *","prompt":"生成今日AI行业新闻摘要","tool_name":"send_notification","tool_args":{"message":"日报已生成"}}'

# 3. 查看所有任务
curl http://localhost:8000/tasks

# 4. 立即执行一次
curl -X POST http://localhost:8000/tasks/{task_id}/run

# 5. 查看执行日志
curl http://localhost:8000/tasks/{task_id}/logs
```

## 学习要点

1. APScheduler 的 AsyncIOScheduler 配合 FastAPI 异步运行
2. CronTrigger.from_crontab() 可以直接解析标准 cron 表达式
3. 任务持久化用 JSON 文件，生产环境建议用数据库
4. 任务执行日志方便排查问题
5. 启动时恢复已启用的任务，实现重启不丢失

---
**上一篇**：[定时任务（上）](./21_FastAPI+tool实现定时任务(上).md) | **下一篇**：[语音交互](./23_给Agent加上语音交互.md)
