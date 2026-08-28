# 向量数据库 Milvus：做 AI Agent 开发必备技术

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年1月13日 14:11

前面我们实现了 RAG：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/0_公众号_Yi昭.png)

文档向量化放到向量数据库，每次查询根据向量化的 query 去数据库做相似度匹配，查出相关文档放到 prompt 里给大模型，大模型来生成回答。

但之前向量数据库是放在内存里的：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/1_公众号_Yi昭.png)

而实际上 AI Agent 产品都会用 Milvus 这种向量数据库。

就像 web 应用会把数据存在 mysql 里，基于对数据的增删改查实现各种业务功能。

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/2_公众号_Yi昭.png)

根据 id 或者关键词去关联查询一系列表的数据。

而 AI Agent 应用会把知识、记忆放在 Milvus 数据库中，基于对知识的检索、增删改实现各种功能。

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/3_公众号_Yi昭.png)

不同的是这里涉及到向量化，就需要嵌入模型，比如检索、新增、修改。

但是删除直接根据 id，不需要嵌入模型。

有同学可能会问，把数据存在 MySQL 里，和现在存在 Milvus 里有什么不同么？

你在 MySQL 里查询数据，只能用 id、关键词匹配。

而在 Milvus 里查询知识，是根据语义匹配的，你可以用自然语言来检索。

这两种功能一般都需要。

比如你做了一个 AI 日记本：

- 查询日记列表可以从 MySQL 来查，不走 AI
- 查询“我哪几天的日记心情比较好”，就要去 Milvus 做向量相似度检索，然后交给 AI 生成回答

所以一般会做 mysql 和 milvus 的双写，也就是同时对两个数据库做增删改，保持数据同步。

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/4_公众号_Yi昭.png)

这节我们先学下 Milvus，做下增删改查，跑通基于 Mivlus 的 RAG 流程。

本地跑 Milvus 需要安装 docker：

https://www.docker.com/

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/5_公众号_Yi昭.png)

下载后安装，会有桌面端和命令行工具：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/6_公众号_Yi昭.png)

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/7_公众号_Yi昭.png)

如果 docker 命令可用了，就代表装好了。

打开桌面端：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/8_公众号_Yi昭.png)

images 是下载的镜像列表。

containers 是镜像跑起来的容器列表。

这里对 docker 不熟也没关系，下节会讲，这节重点是 mivlus。

创建一个目录用来放 milvus 的 docker 配置文件和数据：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/9_公众号_Yi昭.png)

从这里下载 milvus 的 docker compose 配置文件：

https://github.com/milvus-io/milvus/releases

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/10_公众号_Yi昭.png)

把配置文件拿到刚才这个目录，跑一下 docker compose

    docker compose -f ./milvus-standalone-docker-compose.yml up -d

用到的镜像根据配置文件自动下载：

> 🎬 视频演示（原公众号视频）

跑起来之后，在 docker 桌面端这里也可以看到：

下载的镜像：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/11_公众号_Yi昭.png)

跑起来的容器：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/12_公众号_Yi昭.png)

milvus 数据库是跑在 19530 这个端口。

访问这个 url 可以做健康度检查：

http://localhost:9091/healthz

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/13_公众号_Yi昭.png)

然后我们用 python 来连接 milvus 服务做增删改查。

创建项目：

    mkdir milvus-testcd&nbsp;milvus-testnpm init -y

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/14_公众号_Yi昭.png)

安装 milvus 的 python sdk：

    ppip install @zilliz/milvus2-sdk-node

还有 langchain：

    ppip install langchain_openai dotenv

这里的配置文件 .env 大家自己创建下：

    # OpenAI API 配置OPENAI_API_KEY=sk-xxxOPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1MODEL_NAME=qwen-plusEMBEDDINGS_MODEL_NAME=text-embedding-v3

写下插入数据的代码：

创建 src/insert.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ MilvusClient, DataType, MetricType, IndexType }&nbsp;from&nbsp;'@zilliz/milvus2-sdk-node';import&nbsp;{ OpenAIEmbeddings }&nbsp;from&nbsp;"langchain_openai";const&nbsp;COLLECTION_NAME =&nbsp;'ai_diary';const&nbsp;VECTOR_DIM =&nbsp;1024;const&nbsp;embeddings =&nbsp;new&nbsp;OpenAIEmbeddings({&nbsp;&nbsp;apiKey: process.env.OPENAI_API_KEY,&nbsp;&nbsp;model: process.env.EMBEDDINGS_MODEL_NAME,&nbsp;&nbsp;configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL&nbsp; },&nbsp;&nbsp;dimensions: VECTOR_DIM});const&nbsp;client =&nbsp;new&nbsp;MilvusClient({&nbsp;&nbsp;address:&nbsp;'localhost:19530'});async&nbsp;function&nbsp;getEmbedding(text)&nbsp;{&nbsp;&nbsp;const&nbsp;result =&nbsp;await&nbsp;embeddings.embedQuery(text);&nbsp;&nbsp;return&nbsp;result;}async&nbsp;function&nbsp;main()&nbsp;{&nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp;&nbsp;console.log('Connecting to Milvus...');&nbsp; &nbsp;&nbsp;await&nbsp;client.connectPromise;&nbsp; &nbsp;&nbsp;console.log('✓ Connected\n');&nbsp; &nbsp;&nbsp;// 创建集合&nbsp; &nbsp;&nbsp;console.log('Creating collection...');&nbsp; &nbsp;&nbsp;await&nbsp;client.createCollection({&nbsp; &nbsp; &nbsp;&nbsp;collection_name: COLLECTION_NAME,&nbsp; &nbsp; &nbsp;&nbsp;fields: [&nbsp; &nbsp; &nbsp; &nbsp; {&nbsp;name:&nbsp;'id',&nbsp;data_type: DataType.VarChar,&nbsp;max_length:&nbsp;50,&nbsp;is_primary_key:&nbsp;true&nbsp;},&nbsp; &nbsp; &nbsp; &nbsp; {&nbsp;name:&nbsp;'vector',&nbsp;data_type: DataType.FloatVector,&nbsp;dim: VECTOR_DIM },&nbsp; &nbsp; &nbsp; &nbsp; {&nbsp;name:&nbsp;'content',&nbsp;data_type: DataType.VarChar,&nbsp;max_length:&nbsp;5000&nbsp;},&nbsp; &nbsp; &nbsp; &nbsp; {&nbsp;name:&nbsp;'date',&nbsp;data_type: DataType.VarChar,&nbsp;max_length:&nbsp;50&nbsp;},&nbsp; &nbsp; &nbsp; &nbsp; {&nbsp;name:&nbsp;'mood',&nbsp;data_type: DataType.VarChar,&nbsp;max_length:&nbsp;50&nbsp;},&nbsp; &nbsp; &nbsp; &nbsp; {&nbsp;name:&nbsp;'tags',&nbsp;data_type: DataType.Array,&nbsp;element_type: DataType.VarChar,&nbsp;max_capacity:&nbsp;10,&nbsp;max_length:&nbsp;50&nbsp;}&nbsp; &nbsp; &nbsp; ]&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;console.log('Collection created');&nbsp; &nbsp;&nbsp;// 创建索引&nbsp; &nbsp;&nbsp;console.log('\nCreating index...');&nbsp; &nbsp;&nbsp;await&nbsp;client.createIndex({&nbsp; &nbsp; &nbsp;&nbsp;collection_name: COLLECTION_NAME,&nbsp; &nbsp; &nbsp;&nbsp;field_name:&nbsp;'vector',&nbsp; &nbsp; &nbsp;&nbsp;index_type: IndexType.IVF_FLAT,&nbsp; &nbsp; &nbsp;&nbsp;metric_type: MetricType.COSINE,&nbsp; &nbsp; &nbsp;&nbsp;params: {&nbsp;nlist:&nbsp;1024&nbsp;}&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;console.log('Index created');&nbsp; &nbsp;&nbsp;// 加载集合&nbsp; &nbsp;&nbsp;console.log('\nLoading collection...');&nbsp; &nbsp;&nbsp;await&nbsp;client.loadCollection({&nbsp;collection_name: COLLECTION_NAME });&nbsp; &nbsp;&nbsp;console.log('Collection loaded');&nbsp; &nbsp;&nbsp;// 插入日记数据&nbsp; &nbsp;&nbsp;console.log('\nInserting diary entries...');&nbsp; &nbsp;&nbsp;const&nbsp;diaryContents = [&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;id:&nbsp;'diary_001',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content:&nbsp;'今天天气很好，去公园散步了，心情愉快。看到了很多花开了，春天真美好。',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;date:&nbsp;'2026-01-10',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;'happy',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;tags: ['生活',&nbsp;'散步']&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;id:&nbsp;'diary_002',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content:&nbsp;'今天工作很忙，完成了一个重要的项目里程碑。团队合作很愉快，感觉很有成就感。',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;date:&nbsp;'2026-01-11',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;'excited',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;tags: ['工作',&nbsp;'成就']&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;id:&nbsp;'diary_003',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content:&nbsp;'周末和朋友去爬山，天气很好，心情也很放松。享受大自然的感觉真好。',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;date:&nbsp;'2026-01-12',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;'relaxed',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;tags: ['户外',&nbsp;'朋友']&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;id:&nbsp;'diary_004',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content:&nbsp;'今天学习了 Milvus 向量数据库，感觉很有意思。向量搜索技术真的很强大。',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;date:&nbsp;'2026-01-12',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;'curious',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;tags: ['学习',&nbsp;'技术']&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;id:&nbsp;'diary_005',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content:&nbsp;'晚上做了一顿丰盛的晚餐，尝试了新菜谱。家人都说很好吃，很有成就感。',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;date:&nbsp;'2026-01-13',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;'proud',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;tags: ['美食',&nbsp;'家庭']&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; ];&nbsp; &nbsp;&nbsp;console.log('Generating embeddings...');&nbsp; &nbsp;&nbsp;const&nbsp;diaryData =&nbsp;await&nbsp;Promise.all(&nbsp; &nbsp; &nbsp; diaryContents.map(async&nbsp;(diary) =&gt; ({&nbsp; &nbsp; &nbsp; &nbsp; ...diary,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;vector:&nbsp;await&nbsp;getEmbedding(diary.content)&nbsp; &nbsp; &nbsp; }))&nbsp; &nbsp; );&nbsp; &nbsp;&nbsp;const&nbsp;insertResult =&nbsp;await&nbsp;client.insert({&nbsp; &nbsp; &nbsp;&nbsp;collection_name: COLLECTION_NAME,&nbsp; &nbsp; &nbsp;&nbsp;data: diaryData&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;console.log(`✓ Inserted&nbsp;${insertResult.insert_cnt}&nbsp;records\n`);&nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp;&nbsp;console.error('Error:', error.message);&nbsp; }}main();

在 milvus 里是这样存储数据的：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/15_公众号_Yi昭.png)

可以分为多个 database，每个 database 下有多个 collection

每个 collection 下是符合 schema 的 entity，也就是数据。

所以我们插入数据，就定义一个 schema，然后插入 entity 就好了。

同时要建立一个向量字段的索引，用来快速查询。

也就是这样：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/16_公众号_Yi昭.png)

这就是 schema，创建 collection 集合的时候需要指定。

具体字段包含 id、vector、content、date、mode、tags

其实和 mysql 的表差不多，唯一的区别是 vector 这个字段，我们设置了 FloatVector 类型，也就是向量，指定维度是 1024 维。

这样我们后面插入数据，也要把嵌入模型指定为 1024 的维度。

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/17_公众号_Yi昭.png)

这个集合名是 ai\_diary，用来放日记数据的。

向量字段需要建立索引：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/18_公众号_Yi昭.png)

metric\_type 指定用余弦相似度作为距离度量

余弦相似度的原理前面讲过：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/19_公众号_Yi昭.png)

之后就可以插入数据了：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/20_公众号_Yi昭.png)

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/21_公众号_Yi昭.png)

插入数据比较简单，就是调用 insert 方法，指定 collection name 和 data

只不过这里的 vector 字段需要用嵌入模型来向量化一下。

跑一下：

> 🎬 视频演示（原公众号视频）

接下来做一下查询。

先不着急用代码写，我们可以安装一个 GUI 工具：

https://github.com/zilliztech/attu?tab=readme-ov-file#quick-start

Attu 是 Milvus 生态最好的 GUI 工具。

https://github.com/zilliztech/attu/releases

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/22_公众号_Yi昭.png)

下载后安装下：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/23_公众号_Yi昭.png)

用默认配置连接就行：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/24_公众号_Yi昭.png)

和 node.js 那边一样。

> 🎬 视频演示（原公众号视频）

可以看到所有的集合，集合下所有的 Entity

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/25_公众号_Yi昭.png)

可以看到我们刚创建的 ai\_diary 的 collection，以及下面的 5 条数据

vector 是向量，用来做语义检索的。

其他字段是元信息，会一并查出来返回。

我们写下查询：

创建 src/query.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ MilvusClient, MetricType }&nbsp;from&nbsp;'@zilliz/milvus2-sdk-node';import&nbsp;{ OpenAIEmbeddings }&nbsp;from&nbsp;"langchain_openai";const&nbsp;COLLECTION_NAME =&nbsp;'ai_diary';const&nbsp;VECTOR_DIM =&nbsp;1024;const&nbsp;embeddings =&nbsp;new&nbsp;OpenAIEmbeddings({&nbsp;&nbsp;apiKey: process.env.OPENAI_API_KEY,&nbsp;&nbsp;model: process.env.EMBEDDINGS_MODEL_NAME,&nbsp;&nbsp;configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL&nbsp; },&nbsp;&nbsp;dimensions: VECTOR_DIM});const&nbsp;client =&nbsp;new&nbsp;MilvusClient({&nbsp;&nbsp;address:&nbsp;'localhost:19530'});async&nbsp;function&nbsp;getEmbedding(text)&nbsp;{&nbsp;&nbsp;const&nbsp;result =&nbsp;await&nbsp;embeddings.embedQuery(text);&nbsp;&nbsp;return&nbsp;result;}async&nbsp;function&nbsp;main()&nbsp;{&nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp;&nbsp;console.log('Connecting to Milvus...');&nbsp; &nbsp;&nbsp;await&nbsp;client.connectPromise;&nbsp; &nbsp;&nbsp;console.log('✓ Connected\n');&nbsp; &nbsp;&nbsp;// 向量搜索&nbsp; &nbsp;&nbsp;console.log('Searching for similar diary entries...');&nbsp; &nbsp;&nbsp;const&nbsp;query =&nbsp;'我想看看关于户外活动的日记';&nbsp; &nbsp;&nbsp;console.log(`Query: "${query}"\n`);&nbsp; &nbsp;&nbsp;const&nbsp;queryVector =&nbsp;await&nbsp;getEmbedding(query);&nbsp; &nbsp;&nbsp;const&nbsp;searchResult =&nbsp;await&nbsp;client.search({&nbsp; &nbsp; &nbsp;&nbsp;collection_name: COLLECTION_NAME,&nbsp; &nbsp; &nbsp;&nbsp;vector: queryVector,&nbsp; &nbsp; &nbsp;&nbsp;limit:&nbsp;2,&nbsp; &nbsp; &nbsp;&nbsp;metric_type: MetricType.COSINE,&nbsp; &nbsp; &nbsp;&nbsp;output_fields: ['id',&nbsp;'content',&nbsp;'date',&nbsp;'mood',&nbsp;'tags']&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;console.log(`Found&nbsp;${searchResult.results.length}&nbsp;results:\n`);&nbsp; &nbsp; searchResult.results.forEach((item, index) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp;&nbsp;console.log(`${index +&nbsp;1}. [Score:&nbsp;${item.score.toFixed(4)}]`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(` &nbsp; ID:&nbsp;${item.id}`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(` &nbsp; Date:&nbsp;${item.date}`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(` &nbsp; Mood:&nbsp;${item.mood}`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(` &nbsp; Tags:&nbsp;${item.tags?.join(', ')}`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(` &nbsp; Content:&nbsp;${item.content}\n`);&nbsp; &nbsp; });&nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp;&nbsp;console.error('Error:', error.message);&nbsp; }}main();

是把 query 向量化，做余弦相似度的检索：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/26_公众号_Yi昭.png)

跑一下：

> 🎬 视频演示（原公众号视频）

可以看到，检索出了两条户外活动的日记。

改一下 query，查询做饭、学习的日记：

再试一下：

> 🎬 视频演示（原公众号视频）

这次查了做饭和学习的日记，也搜出来了。

你用 MySQL 做关键词搜索可以做到么？

很明显不能，这就是为啥用向量数据库！

然后我们把它和 RAG 流程结合来跑一下完整流程：

创建 src/rag.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ MilvusClient, MetricType }&nbsp;from&nbsp;'@zilliz/milvus2-sdk-node';import&nbsp;{ ChatOpenAI, OpenAIEmbeddings }&nbsp;from&nbsp;"langchain_openai";const&nbsp;COLLECTION_NAME =&nbsp;'ai_diary';const&nbsp;VECTOR_DIM =&nbsp;1024;// 初始化 OpenAI Chat 模型const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({&nbsp;&nbsp;temperature:&nbsp;0.7,&nbsp;&nbsp;model: process.env.MODEL_NAME,&nbsp;&nbsp;apiKey: process.env.OPENAI_API_KEY,&nbsp;&nbsp;configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; },});// 初始化 Embeddings 模型const&nbsp;embeddings =&nbsp;new&nbsp;OpenAIEmbeddings({&nbsp;&nbsp;apiKey: process.env.OPENAI_API_KEY,&nbsp;&nbsp;model: process.env.EMBEDDINGS_MODEL_NAME,&nbsp;&nbsp;configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL&nbsp; },&nbsp;&nbsp;dimensions: VECTOR_DIM});// 初始化 Milvus 客户端const&nbsp;client =&nbsp;new&nbsp;MilvusClient({&nbsp;&nbsp;address:&nbsp;'localhost:19530'});/**&nbsp;* 获取文本的向量嵌入&nbsp;*/async&nbsp;function&nbsp;getEmbedding(text)&nbsp;{&nbsp;&nbsp;const&nbsp;result =&nbsp;await&nbsp;embeddings.embedQuery(text);&nbsp;&nbsp;return&nbsp;result;}/**&nbsp;* 从 Milvus 中检索相关的日记条目&nbsp;*/async&nbsp;function&nbsp;retrieveRelevantDiaries(question, k =&nbsp;2)&nbsp;{&nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp;&nbsp;// 生成问题的向量&nbsp; &nbsp;&nbsp;const&nbsp;queryVector =&nbsp;await&nbsp;getEmbedding(question);&nbsp; &nbsp;&nbsp;// 在 Milvus 中搜索相似的日记&nbsp; &nbsp;&nbsp;const&nbsp;searchResult =&nbsp;await&nbsp;client.search({&nbsp; &nbsp; &nbsp;&nbsp;collection_name: COLLECTION_NAME,&nbsp; &nbsp; &nbsp;&nbsp;vector: queryVector,&nbsp; &nbsp; &nbsp;&nbsp;limit: k,&nbsp; &nbsp; &nbsp;&nbsp;metric_type: MetricType.COSINE,&nbsp; &nbsp; &nbsp;&nbsp;output_fields: ['id',&nbsp;'content',&nbsp;'date',&nbsp;'mood',&nbsp;'tags']&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;return&nbsp;searchResult.results;&nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp;&nbsp;console.error('检索日记时出错:', error.message);&nbsp; &nbsp;&nbsp;return&nbsp;[];&nbsp; }}/**&nbsp;* 使用 RAG 回答关于日记的问题&nbsp;*/async&nbsp;function&nbsp;answerDiaryQuestion(question, k =&nbsp;2)&nbsp;{&nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp;&nbsp;console.log('='.repeat(80));&nbsp; &nbsp;&nbsp;console.log(`问题:&nbsp;${question}`);&nbsp; &nbsp;&nbsp;console.log('='.repeat(80));&nbsp; &nbsp;&nbsp;// 1. 检索相关日记&nbsp; &nbsp;&nbsp;console.log('\n【检索相关日记】');&nbsp; &nbsp;&nbsp;const&nbsp;retrievedDiaries =&nbsp;await&nbsp;retrieveRelevantDiaries(question, k);&nbsp; &nbsp;&nbsp;if&nbsp;(retrievedDiaries.length ===&nbsp;0) {&nbsp; &nbsp; &nbsp;&nbsp;console.log('未找到相关日记');&nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;'抱歉，我没有找到相关的日记内容。';&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;// 2. 打印检索到的日记及相似度&nbsp; &nbsp; retrievedDiaries.forEach((diary, i) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp;&nbsp;console.log(`\n[日记&nbsp;${i +&nbsp;1}] 相似度:&nbsp;${diary.score.toFixed(4)}`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(`日期:&nbsp;${diary.date}`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(`心情:&nbsp;${diary.mood}`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(`标签:&nbsp;${diary.tags?.join(', ')}`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(`内容:&nbsp;${diary.content}`);&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;// 3. 构建上下文&nbsp; &nbsp;&nbsp;const&nbsp;context = retrievedDiaries&nbsp; &nbsp; &nbsp; .map((diary, i) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;`[日记&nbsp;${i +&nbsp;1}]日期:&nbsp;${diary.date}心情:&nbsp;${diary.mood}标签:&nbsp;${diary.tags?.join(', ')}内容:&nbsp;${diary.content}`;&nbsp; &nbsp; &nbsp; })&nbsp; &nbsp; &nbsp; .join('\n\n━━━━━\n\n');&nbsp; &nbsp;&nbsp;// 4. 构建 prompt&nbsp; &nbsp;&nbsp;const&nbsp;prompt =&nbsp;`你是一个温暖贴心的 AI 日记助手。基于用户的日记内容回答问题，用亲切自然的语言。请根据以下日记内容回答问题：${context}用户问题:&nbsp;${question}回答要求：1. 如果日记中有相关信息，请结合日记内容给出详细、温暖的回答2. 可以总结多篇日记的内容，找出共同点或趋势3. 如果日记中没有相关信息，请温和地告知用户4. 用第一人称"你"来称呼日记的作者5. 回答要有同理心，让用户感到被理解和关心AI 助手的回答:`;&nbsp; &nbsp;&nbsp;// 5. 调用 LLM 生成回答&nbsp; &nbsp;&nbsp;console.log('\n【AI 回答】');&nbsp; &nbsp;&nbsp;const&nbsp;response =&nbsp;await&nbsp;model.invoke(prompt);&nbsp; &nbsp;&nbsp;console.log(response.content);&nbsp; &nbsp;&nbsp;console.log('\n');&nbsp; &nbsp;&nbsp;return&nbsp;response.content;&nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp;&nbsp;console.error('回答问题时出错:', error.message);&nbsp; &nbsp;&nbsp;return&nbsp;'抱歉，处理您的问题时出现了错误。';&nbsp; }}async&nbsp;function&nbsp;main()&nbsp;{&nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp;&nbsp;console.log('连接到 Milvus...');&nbsp; &nbsp;&nbsp;await&nbsp;client.connectPromise;&nbsp; &nbsp;&nbsp;console.log('✓ 已连接\n');&nbsp; &nbsp;&nbsp;await&nbsp;answerDiaryQuestion("我最近做了什么让我感到快乐的事情？",&nbsp;2);&nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp;&nbsp;console.error('错误:', error.message);&nbsp; }}main();

这次把温度调高点，让 AI 可以发挥创造性回答：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/27_公众号_Yi昭.png)

我们先把 query 向量化，去 Milvus 里查出相关数据：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/28_公众号_Yi昭.png)

然后把这些加到 prompt 里让大模型回答：

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/29_公众号_Yi昭.png)

跑一下：

> 🎬 视频演示（原公众号视频）

可以看到，大模型基于我们的问题，查询了相关的日记，然后做了回答。

完全是根据语义检索的！

实际的 AI Agent 里就是这样来做 RAG 的。

最后，我们做了 query、insert，自然要把 update 和 delete 也测一下：

创建 src/update.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ MilvusClient }&nbsp;from&nbsp;'@zilliz/milvus2-sdk-node';import&nbsp;{ OpenAIEmbeddings }&nbsp;from&nbsp;"langchain_openai";const&nbsp;COLLECTION_NAME =&nbsp;'ai_diary';const&nbsp;VECTOR_DIM =&nbsp;1024;const&nbsp;embeddings =&nbsp;new&nbsp;OpenAIEmbeddings({&nbsp;&nbsp;apiKey: process.env.OPENAI_API_KEY,&nbsp;&nbsp;model: process.env.EMBEDDINGS_MODEL_NAME,&nbsp;&nbsp;configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL&nbsp; },&nbsp;&nbsp;dimensions: VECTOR_DIM});const&nbsp;client =&nbsp;new&nbsp;MilvusClient({&nbsp;&nbsp;address:&nbsp;'localhost:19530'});async&nbsp;function&nbsp;getEmbedding(text)&nbsp;{&nbsp;&nbsp;const&nbsp;result =&nbsp;await&nbsp;embeddings.embedQuery(text);&nbsp;&nbsp;return&nbsp;result;}async&nbsp;function&nbsp;main()&nbsp;{&nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp;&nbsp;console.log('Connecting to Milvus...');&nbsp; &nbsp;&nbsp;await&nbsp;client.connectPromise;&nbsp; &nbsp;&nbsp;console.log('✓ Connected\n');&nbsp; &nbsp;&nbsp;// 更新数据（Milvus 通过 upsert 实现更新）&nbsp; &nbsp;&nbsp;console.log('Updating diary entry...');&nbsp; &nbsp;&nbsp;const&nbsp;updateId =&nbsp;'diary_001';&nbsp; &nbsp;&nbsp;const&nbsp;updatedContent = {&nbsp; &nbsp; &nbsp;&nbsp;id: updateId,&nbsp; &nbsp; &nbsp;&nbsp;content:&nbsp;'今天下了一整天的雨，心情很糟糕。工作上遇到了很多困难，感觉压力很大。一个人在家，感觉特别孤独。',&nbsp; &nbsp; &nbsp;&nbsp;date:&nbsp;'2026-01-10',&nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;'sad',&nbsp; &nbsp; &nbsp;&nbsp;tags: ['生活',&nbsp;'散步',&nbsp;'朋友']&nbsp; &nbsp; };&nbsp; &nbsp;&nbsp;console.log('Generating new embedding...');&nbsp; &nbsp;&nbsp;const&nbsp;vector =&nbsp;await&nbsp;getEmbedding(updatedContent.content);&nbsp; &nbsp;&nbsp;const&nbsp;updateData = { ...updatedContent, vector };&nbsp; &nbsp;&nbsp;const&nbsp;result =&nbsp;await&nbsp;client.upsert({&nbsp; &nbsp; &nbsp;&nbsp;collection_name: COLLECTION_NAME,&nbsp; &nbsp; &nbsp;&nbsp;data: [updateData]&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;console.log(`✓ Updated diary entry:&nbsp;${updateId}`);&nbsp; &nbsp;&nbsp;console.log(` &nbsp;New content:&nbsp;${updatedContent.content}`);&nbsp; &nbsp;&nbsp;console.log(` &nbsp;New mood:&nbsp;${updatedContent.mood}`);&nbsp; &nbsp;&nbsp;console.log(` &nbsp;New tags:&nbsp;${updatedContent.tags.join(', ')}\n`);&nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp;&nbsp;console.error('Error:', error.message);&nbsp; }}main();

因为要向量化，所以也要嵌入模型。

![image](../IMG/2026-01-13_向量数据库Milvus：做AIAgent开发必备技术/30_公众号_Yi昭.png)

调用 upsert 方法，数据里带上 id 即可。

> 🎬 视频演示（原公众号视频）

这样，更新就完成了。

最后测一下删除：

创建 src/delete.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ MilvusClient }&nbsp;from&nbsp;'@zilliz/milvus2-sdk-node';const&nbsp;COLLECTION_NAME =&nbsp;'ai_diary';const&nbsp;client =&nbsp;new&nbsp;MilvusClient({&nbsp;&nbsp;address:&nbsp;'localhost:19530'});async&nbsp;function&nbsp;main()&nbsp;{&nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp;&nbsp;console.log('Connecting to Milvus...');&nbsp; &nbsp;&nbsp;await&nbsp;client.connectPromise;&nbsp; &nbsp;&nbsp;console.log('✓ Connected\n');&nbsp; &nbsp;&nbsp;// 删除单条数据&nbsp; &nbsp;&nbsp;console.log('Deleting diary entry...');&nbsp; &nbsp;&nbsp;const&nbsp;deleteId =&nbsp;'diary_005';&nbsp; &nbsp;&nbsp;const&nbsp;result =&nbsp;await&nbsp;client.delete({&nbsp; &nbsp; &nbsp;&nbsp;collection_name: COLLECTION_NAME,&nbsp; &nbsp; &nbsp;&nbsp;filter:&nbsp;`id == "${deleteId}"`&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;console.log(`✓ Deleted&nbsp;${result.delete_cnt}&nbsp;record(s)`);&nbsp; &nbsp;&nbsp;console.log(` &nbsp;ID:&nbsp;${deleteId}\n`);&nbsp; &nbsp;&nbsp;// 批量删除&nbsp; &nbsp;&nbsp;console.log('Batch deleting diary entries...');&nbsp; &nbsp;&nbsp;const&nbsp;deleteIds = ['diary_002',&nbsp;'diary_003'];&nbsp; &nbsp;&nbsp;const&nbsp;idsStr = deleteIds.map(id&nbsp;=&gt;&nbsp;`"${id}"`).join(', ');&nbsp; &nbsp;&nbsp;const&nbsp;batchResult =&nbsp;await&nbsp;client.delete({&nbsp; &nbsp; &nbsp;&nbsp;collection_name: COLLECTION_NAME,&nbsp; &nbsp; &nbsp;&nbsp;filter:&nbsp;`id in [${idsStr}]`&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;console.log(`✓ Batch deleted&nbsp;${batchResult.delete_cnt}&nbsp;record(s)`);&nbsp; &nbsp;&nbsp;console.log(` &nbsp;IDs:&nbsp;${deleteIds.join(', ')}\n`);&nbsp; &nbsp;&nbsp;// 条件删除&nbsp; &nbsp;&nbsp;console.log('Deleting by condition...');&nbsp; &nbsp;&nbsp;const&nbsp;conditionResult =&nbsp;await&nbsp;client.delete({&nbsp; &nbsp; &nbsp;&nbsp;collection_name: COLLECTION_NAME,&nbsp; &nbsp; &nbsp;&nbsp;filter:&nbsp;`mood == "sad"`&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;console.log(`✓ Deleted&nbsp;${conditionResult.delete_cnt}&nbsp;record(s) with mood="sad"\n`);&nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp;&nbsp;console.error('Error:', error.message);&nbsp; }}main();

这个不用向量化数据，也就不用嵌入模型。

这里用了 filter

根据条件来删除，或者 id in [1,2,3] 这样来批量删除。

我们这里删了一个 mood 为 sad 的，一个 id 为 2、3 的，一个 id 为 5 的

跑一下：

> 🎬 视频演示（原公众号视频）

可以看到，数据都被正确删除了。

这样我们就完成了对 Milvus 数据的增删改查。

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

这节我们学了 Milvus 向量数据库。

MySQL 数据库只能根据 id、关键词去检索，涉及到语义检索的，我们都会存到 Milvus 里。

我们用 docker compose 跑了 Milvus 数据库，然后在 attu （GUI 工具） 和 python 代码里连上，并做了增删改查。

Milvus 分为 database、collection、entity 这三级，collection 要指定数据结构也就是 schema。

vector 向量字段需要做索引，用来快速检索。

我们把 Milvus 接入了 RAG 流程，实现了 AI 日记本的功能。可以根据自然语言去做语义检索，查出最相关的日记。

MySQL 和 Milvus 分别用于不同的场景，一个是做精确查询，可以关联查出很多表的数据，一个是做语义检索，可以用自然语言来查询。

实际上一般会做双写，同时对两者做增删改查。

后面项目里我们也会同时用 MySQL 和 Milvus。

做 AI Agent 项目，Milvus 向量数据库是是必备技术，可以写到简历上，围绕这个聊很多功能的实现，比如知识、记忆等，需要重点掌握。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-01-13 14:11:43 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247484247&idx=1&sn=dedd4d7ee1cb9c36c267382c11236fba&chksm=f1d54531ec987ac348bb26d782c319ff4b92eeb003f6e7365d9d74e910fd0f22ee84f2f5d779&scene=27#wechat_redirect
