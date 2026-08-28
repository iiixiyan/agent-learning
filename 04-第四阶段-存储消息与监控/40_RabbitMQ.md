# 40. RabbitMQ：Agent 中异步处理的标配方案

> 阶段：第四阶段 | 适用场景：异步任务、削峰填谷、解耦系统

## 为什么需要消息队列？

Agent 系统中很多操作耗时很长（文档解析、向量化、大模型调用），如果同步等待会导致：
- 用户体验差，请求超时
- 系统脆弱，一个慢任务拖垮整个服务
- 无法削峰，突发流量直接打垮后端

RabbitMQ 消息队列解决这些问题：
- **异步**：耗时任务丢到队列，立即返回，后台慢慢处理
- **解耦**：生产者和消费者互不依赖
- **削峰**：队列缓冲突发流量，消费者按自己的速度处理

## 核心概念

| 概念 | 说明 |
|------|------|
| Producer（生产者） | 发送消息的一方 |
| Consumer（消费者） | 接收并处理消息的一方 |
| Queue（队列） | 存储消息的缓冲区 |
| Exchange（交换机） | 接收生产者消息，路由到队列 |
| Binding（绑定） | 交换机和队列的绑定关系 |
| Routing Key（路由键） | 消息的路由标识 |

## Docker 启动

```bash
docker run -d \
  --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=password \
  rabbitmq:3-management
# 管理界面: http://localhost:15672
```

## Python 实现

```python
# pip install pika
import pika
import json

# ============ 生产者 ============
def publish_task(task_type: str, data: dict):
    conn = pika.BlockingConnection(pika.ConnectionParameters('localhost', 5672, credentials=pika.PlainCredentials('admin', 'password')))
    ch = conn.channel()
    ch.queue_declare(queue='agent_tasks', durable=True)
    ch.basic_publish(
        exchange='',
        routing_key='agent_tasks',
        body=json.dumps({'type': task_type, 'data': data}, ensure_ascii=False),
        properties=pika.BasicProperties(delivery_mode=2)  # 持久化
    )
    conn.close()

# 发送文档解析任务
publish_task('parse_document', {'doc_id': '123', 'file_path': '/docs/report.pdf'})

# ============ 消费者 ============
def callback(ch, method, properties, body):
    task = json.loads(body)
    print(f"处理任务: {task['type']}")
    try:
        if task['type'] == 'parse_document':
            # 执行文档解析（耗时操作）
            print(f"解析文档: {task['data']['file_path']}")
            # ... 实际处理逻辑
        ch.basic_ack(delivery_tag=method.delivery_tag)  # 确认处理成功
    except Exception as e:
        print(f"处理失败: {e}")
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)  # 失败重新入队

conn = pika.BlockingConnection(pika.ConnectionParameters('localhost', credentials=pika.PlainCredentials('admin', 'password')))
ch = conn.channel()
ch.queue_declare(queue='agent_tasks', durable=True)
ch.basic_qos(prefetch_count=1)  # 一次只处理一个任务
ch.basic_consume(queue='agent_tasks', on_message_callback=callback)
print('等待任务...')
ch.start_consuming()
```

## Agent 系统中的典型应用

```python
# 场景：用户上传文档 → 异步解析 → 向量化 → 存入知识库
# API 层（FastAPI）
@app.post("/upload")
async def upload(file: UploadFile):
    doc_id = str(uuid.uuid4())
    # 保存文件...
    # 发送异步任务，立即返回
    publish_task('rag_pipeline', {'doc_id': doc_id, 'stage': 'parse'})
    return {"doc_id": doc_id, "status": "processing"}

# 消费者按阶段处理
def process_rag_pipeline(task):
    stage = task['data']['stage']
    if stage == 'parse':
        # 1. 解析文档
        parse_document(task['data']['doc_id'])
        # 2. 发送下一阶段任务
        publish_task('rag_pipeline', {'doc_id': task['data']['doc_id'], 'stage': 'embed'})
    elif stage == 'embed':
        # 3. 向量化
        embed_document(task['data']['doc_id'])
        # 4. 发送下一阶段
        publish_task('rag_pipeline', {'doc_id': task['data']['doc_id'], 'stage': 'index'})
    elif stage == 'index':
        # 5. 存入索引
        index_document(task['data']['doc_id'])
        # 6. 更新状态为完成
        update_doc_status(task['data']['doc_id'], 'ready')
```

## 学习要点

1. RabbitMQ 用 AMQP 协议，支持持久化、确认机制、死信队列
2. `basic_qos(prefetch_count=1)` 实现公平分发，避免一个消费者被压垮
3. 处理成功要 `basic_ack`，失败要 `basic_nack` 并决定是否重新入队
4. 多阶段流水线可以通过不同队列或消息中的 stage 字段实现
5. 生产环境要配置死信队列处理失败消息，避免无限重试

---
**上一篇**：[多模态与 OSS 直传](./39_多模态与OSS前端直传实战.md) | **下一篇**：[LangFuse](./41_LangFuse.md)
