# RAG：把文档向量化，基于向量实现真正的语义搜索

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2025年12月29日 23:55

大模型所知道的知识，取决于在训练的时候给它的数据集。

如果你问它最近发生的事情，或者你企业内部私有文档的一些事情，它是不知道的。

但它很可能不会说自己不知道，而是会胡乱回答，也就是所谓的**幻觉**（以为自己知道）。

如何解决大模型的幻觉呢？

其实也很容易想到：

用户要查询的内容，我们先去内部知识库里查一下，把它放到 prompt 里再给大模型。

这样大模型通过这些文档知道了背景知识，就可以回答响应的问题了。

这就是 RAG：

Retrieval 检索 - Augmented 增强 - Generation 生成

去知识库里**检索**用户问的知识的相关文档片段，作为背景知识加到 prompt 里**增强**它，让大模型根据这些来**生成**回答。

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/0_公众号_Yi昭.png)

这个是很容易想到的思路，也是很贴切的名字。

但有个问题：

用户问了一个问题，你怎么把相关的文档片段查出来呢？

比如用户查水果的信息，你要把苹果、香蕉、草莓的相关文档查出来。

想想怎么做？

关键词搜索可以么？

很明显不行。

这种语义搜索就需要向量（Vector）了。

比如如果按照两个维度存储信息，分为可食用性、硬度：

- 维度 1： 食用性（0 = 无，1 = 高）
- 维度 2： 硬度（0 = 软/液体，1 = 硬）

那这几个概念大概是这样的向量：

- 水果：[0.9, 0.3] 极高食用性，中低硬度
- 苹果：[0.9, 0.5] 高食用性，硬度适中
- 香蕉：[0.9, 0.1] 高食用性，非常软
- 石头：[0.1, 0.9] 几乎不可食用，非常硬

可视化一下是这样：

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/1_公众号_Yi昭.png)

明显可以看出来，苹果、水果、香蕉，这三个概念相关性很大，而水果和石头相关性就不大。

计算的话，可以通过夹角判断相似度，夹角越小相似度越高：

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/2_公众号_Yi昭.png)

也就是**余弦相似度**（两个向量夹角的余弦值）。

当然，具体的向量数据肯定不会只有二维，可能会是几百维。

虽然高纬度没法可视化，但是原理是一样的。

我们都是通过两个概念对应的向量的余弦相似度来判断相关性。

也就是说**通过向量计算实现语义检索！**

是不是很巧妙！

这就是为啥 RAG 一般都结合向量化来做，虽然基于关键词来做也是 RAG，但是那种没法语义搜索，意义不大。

有的同学可能会问，那给你一个概念，怎么计算它的向量值呢？

这个需要用到专门的模型，叫**嵌入模型（Embedding Model）**。

它和大语言模型（LLM）是不一样的，它的功能就只有把知识转成向量。

这个知识可以是文本、图片、语音等，向量化之后，就都可以实现语义搜索了！

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/3_公众号_Yi昭.png)

我们写代码会用专门的嵌入模型，收费比大模型便宜很多很多。

那加上向量化之后的 RAG 流程是什么样的呢？

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/4_公众号_Yi昭.png)

用户的 prompt 会通过嵌入模型转成向量，然后 retriever 基于这个向量去向量数据库中检索，找到相似的向量，把对应的文档块返回，加到 prompt 里作为背景知识，给大模型。

存的不是向量么？怎么记录向量关联的文档？

文档在向量化的时候，会在向量的元信息里记录来源文档。

综上，我们可以**在原始 prompt 给到大模型之前，查询下知识库，把相关的文档作为背景知识加入到 Prompt 里，再让大模型回答，这就是 RAG。**

RAG 要实现语义查询，需要基于向量来做，把文档向量化存储到向量数据库，查询的时候也把 Prompt 向量化，去数据库中做相似度检索，这样就可以找到语义相近的文档块。

知道了什么是 RAG，我们来写代码试一下：

    mkdir rag-testcd&nbsp;rag-testnpm init -y

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/5_公众号_Yi昭.png)

进入项目，安装下依赖：

    ppip install langchain_core langchain_openai dotenv

创建 src/hello-rag.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ ChatOpenAI, OpenAIEmbeddings }&nbsp;from"langchain_openai";import&nbsp;{ Document }&nbsp;from"langchain_core/documents";import&nbsp;{ MemoryVectorStore }&nbsp;from"langchain_classic/vectorstores/memory";const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({temperature:&nbsp;0,model: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; },});const&nbsp;embeddings =&nbsp;new&nbsp;OpenAIEmbeddings({apiKey: process.env.OPENAI_API_KEY,model: process.env.EMBEDDINGS_MODEL_NAME,configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL&nbsp; },});const&nbsp;documents = [new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`光光是一个活泼开朗的小男孩，他有一双明亮的大眼睛，总是带着灿烂的笑容。光光最喜欢的事情就是和朋友们一起玩耍，他特别擅长踢足球，每次在球场上奔跑时，就像一道阳光一样充满活力。`,&nbsp; &nbsp;&nbsp;metadata: {&nbsp;&nbsp; &nbsp; &nbsp;&nbsp;chapter:&nbsp;1,&nbsp;&nbsp; &nbsp; &nbsp;&nbsp;character:&nbsp;"光光",&nbsp;&nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;"角色介绍",&nbsp;&nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;"活泼"&nbsp; &nbsp; },&nbsp; }),new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`东东是光光最好的朋友，他是一个安静而聪明的男孩。东东喜欢读书和画画，他的画总是充满了想象力。虽然性格不同，但东东和光光从幼儿园就认识了，他们一起度过了无数个快乐的时光。`,&nbsp; &nbsp;&nbsp;metadata: {&nbsp;&nbsp; &nbsp; &nbsp;&nbsp;chapter:&nbsp;2,&nbsp;&nbsp; &nbsp; &nbsp;&nbsp;character:&nbsp;"东东",&nbsp;&nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;"角色介绍",&nbsp;&nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;"温馨"&nbsp; &nbsp; },&nbsp; }),new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`有一天，学校要举办一场足球比赛，光光非常兴奋，他邀请东东一起参加。但是东东从来没有踢过足球，他担心自己会拖累光光。光光看出了东东的担忧，他拍着东东的肩膀说："没关系，我们一起练习，我相信你一定能行的！"`,&nbsp; &nbsp;&nbsp;metadata: {&nbsp; &nbsp; &nbsp;&nbsp;chapter:&nbsp;3,&nbsp; &nbsp; &nbsp;&nbsp;character:&nbsp;"光光和东东",&nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;"友情情节",&nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;"鼓励",&nbsp; &nbsp; },&nbsp; }),new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`接下来的日子里，光光每天放学后都会教东东踢足球。光光耐心地教东东如何控球、传球和射门，而东东虽然一开始总是踢不好，但他从不放弃。东东也用自己的方式回报光光，他画了一幅画送给光光，画上是两个小男孩在球场上一起踢球的场景。`,&nbsp; &nbsp;&nbsp;metadata: {&nbsp; &nbsp; &nbsp;&nbsp;chapter:&nbsp;4,&nbsp; &nbsp; &nbsp;&nbsp;character:&nbsp;"光光和东东",&nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;"友情情节",&nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;"互助",&nbsp; &nbsp; },&nbsp; }),new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`比赛那天终于到了，光光和东东一起站在球场上。虽然东东的技术还不够熟练，但他非常努力，而且他用自己的观察力帮助光光找到了对手的弱点。在关键时刻，东东传出了一个漂亮的球，光光接球后射门得分！他们赢得了比赛，更重要的是，他们的友谊变得更加深厚了。`,&nbsp; &nbsp;&nbsp;metadata: {&nbsp; &nbsp; &nbsp;&nbsp;chapter:&nbsp;5,&nbsp; &nbsp; &nbsp;&nbsp;character:&nbsp;"光光和东东",&nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;"高潮转折",&nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;"激动",&nbsp; &nbsp; },&nbsp; }),new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`从那以后，光光和东东成为了学校里最要好的朋友。光光教东东运动，东东教光光画画，他们互相学习，共同成长。每当有人问起他们的友谊，他们总是笑着说："真正的朋友就是互相帮助，一起变得更好的人！"`,&nbsp; &nbsp;&nbsp;metadata: {&nbsp; &nbsp; &nbsp;&nbsp;chapter:&nbsp;6,&nbsp; &nbsp; &nbsp;&nbsp;character:&nbsp;"光光和东东",&nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;"结局",&nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;"欢乐",&nbsp; &nbsp; },&nbsp; }),new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`多年后，光光成为了一名职业足球运动员，而东东成为了一名优秀的插画师。虽然他们走上了不同的道路，但他们的友谊从未改变。东东为光光设计了球衣上的图案，光光在每场比赛后都会给东东打电话分享喜悦。他们证明了，真正的友情可以跨越时间和距离，永远闪闪发光。`,&nbsp; &nbsp;&nbsp;metadata: {&nbsp; &nbsp; &nbsp;&nbsp;chapter:&nbsp;7,&nbsp; &nbsp; &nbsp;&nbsp;character:&nbsp;"光光和东东",&nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;"尾声",&nbsp; &nbsp; &nbsp;&nbsp;mood:&nbsp;"温馨",&nbsp; &nbsp; },&nbsp; }),];const&nbsp;vectorStore =&nbsp;await&nbsp;MemoryVectorStore.fromDocuments(&nbsp; documents,&nbsp; embeddings,);const&nbsp;retriever = vectorStore.asRetriever({&nbsp;k:&nbsp;3&nbsp;});const&nbsp;questions = ["东东和光光是怎么成为朋友的？"];for&nbsp;(const&nbsp;question&nbsp;of&nbsp;questions) {console.log("=".repeat(80));console.log(`问题:&nbsp;${question}`);console.log("=".repeat(80));// 使用 retriever 获取文档const&nbsp;retrievedDocs =&nbsp;await&nbsp;retriever.invoke(question);// 使用 similaritySearchWithScore 获取相似度评分const&nbsp;scoredResults =&nbsp;await&nbsp;vectorStore.similaritySearchWithScore(question,&nbsp;3);// 打印用到的文档和相似度评分console.log("\n【检索到的文档及相似度评分】");&nbsp; retrievedDocs.forEach((doc, i) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;// 找到对应的评分&nbsp; &nbsp;&nbsp;const&nbsp;scoredResult = scoredResults.find(([scoredDoc]) =&gt;&nbsp; &nbsp; &nbsp; scoredDoc.pageContent === doc.pageContent&nbsp; &nbsp; );&nbsp; &nbsp;&nbsp;const&nbsp;score = scoredResult ? scoredResult[1] :&nbsp;null;&nbsp; &nbsp;&nbsp;const&nbsp;similarity = score !==&nbsp;null&nbsp;? (1&nbsp;- score).toFixed(4) :&nbsp;"N/A";&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;console.log(`\n[文档&nbsp;${i +&nbsp;1}] 相似度:&nbsp;${similarity}`);&nbsp; &nbsp;&nbsp;console.log(`内容:&nbsp;${doc.pageContent}`);&nbsp; &nbsp;&nbsp;console.log(`元数据: 章节=${doc.metadata.chapter}, 角色=${doc.metadata.character}, 类型=${doc.metadata.type}, 心情=${doc.metadata.mood}`);&nbsp; });// 构建 promptconst&nbsp;context = retrievedDocs&nbsp; &nbsp; .map((doc, i) =&gt;`[片段${i +&nbsp;1}]\n${doc.pageContent}`)&nbsp; &nbsp; .join("\n\n━━━━━\n\n");const&nbsp;prompt =&nbsp;`你是一个讲友情故事的老师。基于以下故事片段回答问题，用温暖生动的语言。如果故事中没有提到，就说"这个故事里还没有提到这个细节"。故事片段:${context}问题:&nbsp;${question}老师的回答:`;console.log("\n【AI 回答】");const&nbsp;response =&nbsp;await&nbsp;model.invoke(prompt);console.log(response.content);console.log("\n");}

安装下用到的包：

    ppip install langchain_classic

这里我们用到了大语言模型 LLM，还有嵌入模型 OpenAIEmbeddings

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/6_公众号_Yi昭.png)

具体的 model name 在 .env 里配置下：

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/7_公众号_Yi昭.png)

    # OpenAI API 配置OPENAI_API_KEY=sk-xxxOPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1MODEL_NAME=qwen-plusEMBEDDINGS_MODEL_NAME=text-embedding-v3

这几个 Document 比较容易理解。这个故事直接问大模型，显然它是不知道的：

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/8_公众号_Yi昭.png)

知识库里存的就是这些文档，可以加一些元数据。

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/9_公众号_Yi昭.png)

用嵌入模型把这些文档向量化之后存入向量数据库。

并且返回一个 retriever，k 是 3 就是返回余弦相似度最大的 3 个 Document。

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/10_公众号_Yi昭.png)

用 retriever 把 query 传入，通过向量的余弦相似度，找到语义最相关的 3 个文档片段，传入 prompt：

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/11_公众号_Yi昭.png)

这就是增强后的 Prompt 了，之后问大模型问题的时候，它就有背景知识了。

跑一下：

> 🎬 视频演示（原公众号视频）

可以看到，根据你的问题，查询到了 3 个文档，然后大模型基于这些做了回答。

这样我们就跑通了 RAG 的流程！

回过头来再看下这张图：

![image](../IMG/2025-12-29_RAG：把文档向量化，基于向量实现真正的语义搜索/12_公众号_Yi昭.png)

是不是就很清楚了！

我们对 query 通过嵌入模型向量化，然后查询出了余弦相似度最大的 3 个文档，用它增强 Prompt 后再问大模型，大模型基于这个生成回答。

这就是 RAG。

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

大模型训练完后，知识就不再更新了，它没法知道最新的一些信息，以及一些非互联网上公开的信息。

所以对于它不知道的东西，会胡乱回答，也就是幻觉问题。

解决这个问题的方式就是 RAG。

RAG 是检索、增强、生成，会基于用户的 query 去检索知识库，拿到相关文档后放到 Prompt 里增强它，之后给大大模型来生成回答。

检索肯定是要语义检索，但是关键词检索做不到这点，我们需要用向量来做，通过嵌入模型把知识向量化，这样就可以通过向量的余弦相似度（也就是夹角大小）来计算出两个知识的相关性，从而根据用户的 query 查询出相关的文档。

我们基于 LangChain 写了 RAG 的代码：

- fromDocuments api 基于 embeddings 模型把文档向量化存入数据库。
- asRetriever 指定查询相似度最大的几个文档。
- similaritySearchWithScore 相似度评分

- retriever.invoke 来查询文档。

只要你理解了 RAG 的流程，这些 api 自然也就会用了。

想一下，如果你要做公司内部文档的智能助手，是不是就可以用 RAG 来实现呢？

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光zxg | **发布时间：** 2025-12-29 23:55:04 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247484046&idx=1&sn=56ea6409d3d530e08d58d06e60b5bdde&chksm=f18ffb20121a75c834c50d6f443526d27ff9ba21581ee9d781f2210a66368cd48dc96e82271d&scene=27#wechat_redirect
