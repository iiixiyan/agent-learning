# 28. ElasticSearch 全文检索：倒排索引 + IK 分词器 + BM25 算法

> 阶段：第二阶段 | 适用场景：关键词检索、日志分析、全文搜索

## 为什么需要 ElasticSearch？

向量检索（RAG）擅长语义匹配，但不擅长精确关键词搜索。ElasticSearch 是全文检索引擎，擅长：
- 关键词精确匹配
- 中文分词搜索
- 复杂条件过滤
- 高亮显示匹配词

**混合检索**（向量 + ES）是企业级 RAG 的标配，第 29 篇会详细讲。

## 核心概念

| 概念 | 说明 |
|------|------|
| 倒排索引 | 词 → 文档列表的映射，类似书的索引页 |
| IK 分词器 | 中文分词插件，把"我爱北京天安门"分成"我/爱/北京/天安门" |
| BM25 算法 | 相关性评分算法，考虑词频、文档长度、逆文档频率 |
| Index（索引） | 类似数据库的表 |
| Document（文档） | 类似数据库的行 |

## Docker 启动 ES + IK

```bash
# docker-compose.yml
version: '3'
services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data
  # IK 分词器需要安装到 ES 插件目录
  # 或使用带 IK 的镜像：elasticsearch-ik:8.11.0
volumes:
  es_data:
```

```bash
docker-compose up -d
# 安装 IK 分词器（进入容器执行）
docker exec -it elasticsearch bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v8.11.0/elasticsearch-analysis-ik-8.11.0.zip
docker restart elasticsearch
```

## Python 操作 ES

```python
# es_demo.py
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk

# 连接
es = Elasticsearch("http://localhost:9200")

# 1. 创建索引（指定 IK 分词器）
index_body = {
    "mappings": {
        "properties": {
            "title": {"type": "text", "analyzer": "ik_max_word", "search_analyzer": "ik_smart"},
            "content": {"type": "text", "analyzer": "ik_max_word", "search_analyzer": "ik_smart"},
            "author": {"type": "keyword"},
            "date": {"type": "date"}
        }
    }
}
es.indices.create(index="articles", body=index_body, ignore=400)

# 2. 批量插入文档
docs = [
    {"title": "Python入门教程", "content": "Python是一种简单易学的编程语言", "author": "张三", "date": "2024-01-01"},
    {"title": "LangChain实战", "content": "LangChain是大模型应用开发框架", "author": "李四", "date": "2024-02-01"},
    {"title": "RAG检索增强", "content": "RAG结合检索和生成，提升问答准确率", "author": "王五", "date": "2024-03-01"},
]
actions = [{"_index": "articles", "_source": doc} for doc in docs]
bulk(es, actions)

# 3. 全文检索
result = es.search(
    index="articles",
    body={
        "query": {
            "multi_match": {
                "query": "大模型编程",
                "fields": ["title^2", "content"]  # title 权重2倍
            }
        },
        "highlight": {
            "fields": {"title": {}, "content": {}}
        },
        "size": 5
    }
)

for hit in result["hits"]["hits"]:
    print(f"得分: {hit['_score']:.2f}")
    print(f"标题: {hit['_source']['title']}")
    if "highlight" in hit:
        print(f"高亮: {hit['highlight']}")
    print()
```

## BM25 算法简介

BM25 是 ES 默认的相关性评分算法，公式：

```
BM25 = Σ IDF(qi) * (f(qi,D) * (k1+1)) / (f(qi,D) + k1 * (1 - b + b * |D|/avgdl))
```

关键参数：
- `k1`：词频饱和度（默认 1.2），控制词频增加时分数增长速度
- `b`：长度归一化（默认 0.75），文档越长分数越低
- `IDF`：逆文档频率，词在越少文档中出现越重要

## 学习要点

1. ES 用倒排索引实现快速全文检索
2. 中文必须用 IK 分词器，ik_max_word 索引时用，ik_smart 搜索时用
3. BM25 是默认评分算法，考虑词频、文档长度、IDF
4. ES 和向量检索结合（混合检索）效果最好

---
**上一篇**：[Docker Compose 部署](./27_基于Docker-Compose的部署.md) | **下一篇**：[混合检索 RAG](../03-第三阶段-检索增强与知识图谱/29_混合检索RAG.md)
