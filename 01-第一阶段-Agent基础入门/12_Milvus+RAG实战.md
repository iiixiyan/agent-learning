# 12. Milvus + RAG 实战：电子书语义检索助手

> 阶段：第一阶段 | 前置知识：Milvus 基础、RAG 原理、LangChain

## 项目目标

做一个电子书语义检索助手：上传一本电子书（PDF/TXT），系统自动解析、切分、向量化存入 Milvus，用户可以用自然语言提问，系统检索相关段落并让大模型给出答案。

## 完整流程

```
电子书 → 文档加载 → 文本切分 → 向量化 → 存入 Milvus
                                          ↓
用户提问 → 问题向量化 → Milvus 检索 Top-K → 拼接 Prompt → 大模型生成答案
```

## 环境准备

```bash
# 1. 启动 Milvus（Docker）
wget https://github.com/milvus-io/milvus/releases/download/v2.4.0/milvus-standalone-docker-compose.yml -O docker-compose.yml
docker-compose up -d

# 2. 安装依赖
pip install langchain langchain-openai pymilvus pypdf unstructured python-dotenv
```

## 第一步：文档加载与切分

```python
# document_processor.py
from langchain_community.document_loaders import PyPDFLoader, TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
import os

def load_document(file_path: str):
    """加载文档，支持 PDF 和 TXT"""
    ext = os.path.splitext(file_path)[1].lower()
    if ext == '.pdf':
        loader = PyPDFLoader(file_path)
    elif ext == '.txt':
        loader = TextLoader(file_path, encoding='utf-8')
    else:
        raise ValueError(f"不支持的格式: {ext}")
    return loader.load()

def split_documents(documents, chunk_size=500, chunk_overlap=50):
    """切分文档"""
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
        separators=["\n\n", "\n", "。", "！", "？", ".", " ", ""]
    )
    return splitter.split_documents(documents)

# 使用
docs = load_document("example_book.pdf")
print(f"加载了 {len(docs)} 页")
chunks = split_documents(docs)
print(f"切分为 {len(chunks)} 块")
```

## 第二步：存入 Milvus

```python
# milvus_store.py
from langchain_community.vectorstores import Milvus
from langchain_openai import OpenAIEmbeddings
import os

os.environ["OPENAI_API_KEY"] = "your-api-key"

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

def create_vector_store(chunks, collection_name="ebook_rag"):
    """创建向量库并存入数据"""
    vector_store = Milvus.from_documents(
        documents=chunks,
        embedding=embeddings,
        collection_name=collection_name,
        connection_args={"host": "localhost", "port": "19530"},
        drop_old=True  # 重建集合
    )
    return vector_store

def get_vector_store(collection_name="ebook_rag"):
    """获取已存在的向量库"""
    return Milvus(
        embedding_function=embeddings,
        collection_name=collection_name,
        connection_args={"host": "localhost", "port": "19530"}
    )

# 创建并存入
vector_store = create_vector_store(chunks)
print("向量库创建完成")
```

## 第三步：语义检索

```python
# retriever.py
def search_documents(query: str, vector_store, top_k=5):
    """检索相关文档"""
    # 方式一：相似度搜索
    results = vector_store.similarity_search(query, k=top_k)

    # 方式二：带分数的相似度搜索
    results_with_score = vector_store.similarity_search_with_score(query, k=top_k)

    print(f"查询: {query}")
    print("=" * 50)
    for i, (doc, score) in enumerate(results_with_score, 1):
        print(f"\n[{i}] 相似度: {1-score:.4f}")  # Milvus 距离越小越相似
        print(f"来源: 第{doc.metadata.get('page', '?')}页")
        print(f"内容: {doc.page_content[:200]}...")

    return results

# 测试检索
search_documents("什么是RAG检索增强生成？", vector_store)
```

## 第四步：RAG 问答

```python
# rag_qa.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# RAG Prompt 模板
RAG_PROMPT = ChatPromptTemplate.from_template("""你是一个专业的电子书问答助手。根据以下参考资料回答用户问题。

参考资料：
{context}

用户问题：{question}

回答要求：
1. 只根据参考资料回答，不要编造
2. 引用来源页码，如（第3页）
3. 如果参考资料中没有相关信息，回答"该电子书未涉及此内容"
4. 用中文回答，简洁准确
""")

def format_docs(docs):
    """格式化文档为上下文"""
    return "\n\n".join([
        f"[第{doc.metadata.get('page', '?')}页] {doc.page_content}"
        for doc in docs
    ])

def build_rag_chain(vector_store):
    """构建 RAG Chain"""
    retriever = vector_store.as_retriever(search_kwargs={"k": 5})

    chain = (
        {"context": retriever | format_docs, "question": RunnablePassthrough()}
        | RAG_PROMPT
        | llm
        | StrOutputParser()
    )
    return chain

# 使用
rag_chain = build_rag_chain(vector_store)
answer = rag_chain.invoke("这本书的主要内容是什么？")
print(answer)
```

## 第五步：完整入口

```python
# main.py
from document_processor import load_document, split_documents
from milvus_store import create_vector_store, get_vector_store
from rag_qa import build_rag_chain

class EbookRAG:
    def __init__(self, collection_name="ebook_rag"):
        self.collection_name = collection_name
        self.vector_store = None
        self.chain = None

    def ingest(self, file_path: str):
        """导入电子书"""
        print(f"加载文档: {file_path}")
        docs = load_document(file_path)
        print(f"切分文档...")
        chunks = split_documents(docs)
        print(f"共 {len(chunks)} 块，正在向量化并存入 Milvus...")
        self.vector_store = create_vector_store(chunks, self.collection_name)
        self.chain = build_rag_chain(self.vector_store)
        print("导入完成！")

    def ask(self, question: str) -> str:
        """提问"""
        if self.chain is None:
            self.vector_store = get_vector_store(self.collection_name)
            self.chain = build_rag_chain(self.vector_store)
        return self.chain.invoke(question)

# 使用
if __name__ == "__main__":
    rag = EbookRAG()
    # 首次导入
    rag.ingest("example_book.pdf")
    # 提问
    while True:
        q = input("\n请提问（输入 q 退出）: ")
        if q.lower() == 'q':
            break
        answer = rag.ask(q)
        print(f"\n回答: {answer}")
```

## 学习要点

1. RAG 四步走：加载→切分→向量化→检索生成，每一步都影响最终效果
2. 切分策略很重要：chunk_size 500、overlap 50 是常用配置，按语义分隔符切分效果更好
3. Milvus 用 L2 距离，分数越小越相似，展示时用 1-score 转成相似度
4. RAG Prompt 要明确要求"只根据参考资料回答"，减少幻觉
5. 引用来源页码让答案更可信，也方便用户查证
6. 生产环境要加缓存（Redis）和检索结果重排序（Reranker）提升效果

---
**上一篇**：[向量数据库 Milvus](./11_向量数据库Milvus.md) | **下一篇**：[Memory 管理三大策略](./13_Memory管理的三大策略.md)
