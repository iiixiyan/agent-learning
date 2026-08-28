# 51. 企业级知识库项目：基于消息队列的异步 RAG 流水线

> 阶段：第五阶段 | 前置知识：RabbitMQ、RAG 基础、文档解析

## 为什么需要异步流水线？

用户上传一个 100 页的 PDF，同步处理需要几分钟，用户会等得不耐烦甚至超时。

**异步方案**：上传后立即返回"处理中"，后台用消息队列异步处理，处理完通知用户。

## 流水线架构

```
用户上传文档
    ↓
保存文件到 OSS/MinIO
    ↓
数据库状态: pending → parsing
    ↓
发送消息到 RabbitMQ (doc_parse_queue)
    ↓立即返回 doc_id
    ↓
消费者按阶段处理：
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  文档解析     │ → │  文本切分     │ → │  向量化       │ → │  建立索引     │
│  (parse)     │    │  (split)     │    │  (embed)     │    │  (index)     │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
    ↓                  ↓                  ↓                  ↓
状态: parsing      状态: splitting    状态: embedding    状态: indexed(完成)
```

## 完整实现

### 1. FastAPI 上传接口

```python
# main.py
from fastapi import FastAPI, UploadFile, File
from pydantic import BaseModel
import uuid, pika, json
from minio import Minio

app = FastAPI()

# MinIO 客户端
minio_client = Minio("localhost:9000", access_key="admin", secret_key="password", secure=False)

# RabbitMQ 连接
def get_rabbit_channel():
    conn = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    return conn.channel()

class UploadResponse(BaseModel):
    doc_id: str
    status: str

@app.post("/upload", response_model=UploadResponse)
async def upload_document(file: UploadFile = File(...)):
    doc_id = str(uuid.uuid4())

    # 1. 保存文件到 MinIO
    file_path = f"documents/{doc_id}/{file.filename}"
    minio_client.put_object("kb-bucket", file_path, file.file, -1, part_size=10*1024*1024)

    # 2. 数据库插入记录（status=pending）
    # db.execute("INSERT INTO documents ...")

    # 3. 发送消息到解析队列
    channel = get_rabbit_channel()
    channel.queue_declare(queue='doc_parse_queue', durable=True)
    channel.basic_publish(
        exchange='',
        routing_key='doc_parse_queue',
        body=json.dumps({'doc_id': doc_id, 'file_path': file_path, 'stage': 'parse'}),
        properties=pika.BasicProperties(delivery_mode=2)
    )
    channel.close()

    return {"doc_id": doc_id, "status": "processing"}

@app.get("/documents/{doc_id}/status")
async def get_status(doc_id: str):
    # 从数据库查询状态
    # doc = db.query(Document).filter_by(id=doc_id).first()
    return {"doc_id": doc_id, "status": "parsing", "progress": 50}
```

### 2. 消费者：文档解析阶段

```python
# worker_parse.py
import pika, json
from unstructured.partition.auto import partition
from minio import Minio

minio_client = Minio("localhost:9000", access_key="admin", secret_key="password", secure=False)

def parse_document(ch, method, properties, body):
    task = json.loads(body)
    doc_id = task['doc_id']
    file_path = task['file_path']

    try:
        print(f"解析文档: {doc_id}")

        # 1. 从 MinIO 下载文件
        response = minio_client.get_object("kb-bucket", file_path)
        file_data = response.read()

        # 2. 用 unstructured 解析
        elements = partition(file=file_data, file_filename=file_path)
        text_content = "\n".join([str(e) for e in elements])

        # 3. 保存解析结果到 MongoDB/数据库
        # mongo.document_contents.insert_one({...})

        # 4. 更新状态，发送到下一阶段队列
        ch.queue_declare(queue='doc_split_queue', durable=True)
        ch.basic_publish(
            exchange='', routing_key='doc_split_queue',
            body=json.dumps({'doc_id': doc_id, 'stage': 'split'}),
            properties=pika.BasicProperties(delivery_mode=2)
        )

        ch.basic_ack(delivery_tag=method.delivery_tag)
        print(f"解析完成: {doc_id}")

    except Exception as e:
        print(f"解析失败: {doc_id}, {e}")
        # 更新状态为 failed
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

# 启动消费者
conn = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = conn.channel()
channel.queue_declare(queue='doc_parse_queue', durable=True)
channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='doc_parse_queue', on_message_callback=parse_document)
print("等待解析任务...")
channel.start_consuming()
```

### 3. 消费者：切分 + 向量化 + 索引（合并阶段）

```python
# worker_embed.py
import pika, json
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import PGVector

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

def process_document(ch, method, properties, body):
    task = json.loads(body)
    doc_id = task['doc_id']

    try:
        print(f"处理文档: {doc_id}")

        # 1. 从 MongoDB 读取解析后的文本
        # doc = mongo.document_contents.find_one({"document_id": doc_id})
        text = "..."  # doc['raw_text']

        # 2. 切分
        splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
        chunks = splitter.split_text(text)
        print(f"切分为 {len(chunks)} 块")

        # 3. 向量化并存入 PGVector
        CONNECTION_STRING = "postgresql+psycopg2://user:password@localhost:5432/kb"
        vectorstore = PGVector.from_texts(
            texts=chunks,
            embedding=embeddings,
            collection_name=doc_id,
            connection_string=CONNECTION_STRING
        )

        # 4. 更新状态为完成
        # db.update status='ready', chunk_count=len(chunks)

        ch.basic_ack(delivery_tag=method.delivery_tag)
        print(f"处理完成: {doc_id}")

    except Exception as e:
        print(f"处理失败: {doc_id}, {e}")
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

conn = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = conn.channel()
channel.queue_declare(queue='doc_split_queue', durable=True)
channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='doc_split_queue', on_message_callback=process_document)
print("等待切分任务...")
channel.start_consuming()
```

## 关键设计点

### 1. 状态机
```
pending → parsing → splitting → embedding → indexing → ready
                ↓           ↓           ↓           ↓
              failed      failed      failed      failed
```
每个阶段失败都要记录错误信息，支持重试。

### 2. 死信队列
处理失败的消息进入死信队列，方便排查和手动重试。

```python
# 声明死信队列
channel.queue_declare(queue='doc_parse_dlq', durable=True)
# 失败时发送到死信队列
channel.basic_publish(exchange='', routing_key='doc_parse_dlq', body=body)
```

### 3. 进度通知
处理完成后可以通过 WebSocket / SSE 通知前端，或者前端轮询状态接口。

### 4. 并发控制
用 `basic_qos(prefetch_count=1)` 让每个消费者一次只处理一个任务，避免被压垮。可以启动多个消费者实例提升吞吐量。

## 学习要点

1. 异步流水线解决大文档处理超时问题，上传后立即返回
2. 用 RabbitMQ 解耦各阶段，每个阶段独立扩展
3. 状态机设计是核心，每个阶段都要有成功/失败状态
4. 死信队列处理失败消息，支持重试
5. 切分和向量化可以合并为一个阶段，减少队列数量
6. 生产环境用 Celery + Redis/RabbitMQ 更成熟，这里用原生 pika 演示原理

---
**上一篇**：[文件解析为 md](./50_企业级知识库项目-文件解析为md.md) | **下一篇**：[全文检索链路](./52_企业级知识库项目-全文检索链路.md)
