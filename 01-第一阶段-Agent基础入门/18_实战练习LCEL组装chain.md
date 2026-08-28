# 实战练习 LCEL 组装 chain

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年2月26日 23:20

我们学了 LangChain 的各种功能：tool、mcp、RAG、memory、prompt template、output parser 等，并且学了 LCEL 的写法，把流程组装成 chain 来调用。

LCEL 就是基于 Runnable 的 api 来声明 chain，然后统一执行。

![image](../IMG/2026-02-26_实战练习LCEL组装chain/0_公众号_Yi昭.png)

声明的 chain 可以用 invoke、batch、stream 等 api 来同步调用、批量调用、流式返回，因为所有 Runnable 都实现了这些方法。

![image](../IMG/2026-02-26_实战练习LCEL组装chain/1_公众号_Yi昭.png)

但是大家可能对用了 Runnable 之后和之前的写法的区别没有具体的认识。

这节我们就把之前做过的一些小实战用 LCEL 的方式再写一遍。

这两个：

- 高德 mcp + Chrome Devtools MCP
- RAG + Milvus 电子书语义助手

功能一样，大家感受下写法上的区别，体会下 LCEL 的好处。

在上节的 runnable-test 项目里继续写：

首先我们分析下之前 tool-test 项目里 mcp 的那个案例用 LCEL 的方式应该怎么写：

> 🎬 视频演示（原公众号视频）

经过分析：

bindTools 之后的 model 是一个 Runnable

Prompt Template 是一个 Runnable

调用大模型返回的结果处理，有个 if else 逻辑，可以封装成 RunnableBranch

然后具体处理 tool call 的逻辑可以封装成 RunnableLamda

把这个 chain 组装好，统一调用就好了。

创建 src/cases/mcp-test.mjs

    import&nbsp;'dotenv/config';import&nbsp;{ MultiServerMCPClient }&nbsp;from'langchain_mcp-adapters';import&nbsp;{ ChatOpenAI }&nbsp;from'langchain_openai';import&nbsp;chalk&nbsp;from'chalk';import&nbsp;{ HumanMessage, ToolMessage }&nbsp;from'langchain_core/messages';import&nbsp;{ ChatPromptTemplate, MessagesPlaceholder }&nbsp;from'langchain_core/prompts';import&nbsp;{ RunnableSequence, RunnableLambda, RunnableBranch, RunnablePassthrough }&nbsp;from'langchain_core/runnables';const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({&nbsp;&nbsp; &nbsp;&nbsp;modelName:&nbsp;"qwen-plus",&nbsp; &nbsp;&nbsp;apiKey: process.env.OPENAI_API_KEY,&nbsp; &nbsp;&nbsp;configuration: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; &nbsp; },});const&nbsp;mcpClient =&nbsp;new&nbsp;MultiServerMCPClient({&nbsp; &nbsp;&nbsp;mcpServers: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"amap-maps-streamableHTTP": {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"url":&nbsp;"https://mcp.amap.com/mcp?key="&nbsp;+ process.env.AMAP_MAPS_API_KEY&nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"chrome-devtools": {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"command":&nbsp;"npx",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"args": [&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"-y",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"chrome-devtools-mcp@latest"&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ]&nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; }});const&nbsp;tools =&nbsp;await&nbsp;mcpClient.getTools();const&nbsp;modelWithTools = model.bindTools(tools);const&nbsp;prompt = ChatPromptTemplate.fromMessages([&nbsp; &nbsp; ["system",&nbsp;"你是一个可以调用 MCP 工具的智能助手。"],&nbsp; &nbsp;&nbsp;new&nbsp;MessagesPlaceholder("messages"),]);const&nbsp;llmChain = prompt.pipe(modelWithTools);// 1. 定义处理工具调用的逻辑 (封装为 Runnable)const&nbsp;toolExecutor =&nbsp;new&nbsp;RunnableLambda({&nbsp; &nbsp;&nbsp;func:&nbsp;async&nbsp;(input) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;{ response, tools } = input;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;toolResults = [];&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;for&nbsp;(const&nbsp;toolCall&nbsp;of&nbsp;response.tool_calls ?? []) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;foundTool = tools.find(t&nbsp;=&gt;&nbsp;t.name === toolCall.name);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!foundTool)&nbsp;continue;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;toolResult =&nbsp;await&nbsp;foundTool.invoke(toolCall.args);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 兼容不同返回格式的字符串化&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;contentStr =&nbsp;typeof&nbsp;toolResult ===&nbsp;'string'&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ? toolResult&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; : (toolResult?.text ||&nbsp;JSON.stringify(toolResult));&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; toolResults.push(new&nbsp;ToolMessage({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content: contentStr,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;tool_call_id: toolCall.id,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }));&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;toolResults;&nbsp; &nbsp; }});// 2. 对结果的处理const&nbsp;agentStepChain = RunnableSequence.from([&nbsp; &nbsp;&nbsp;// step1: 将 LLM 输出挂到 state.response 上&nbsp; &nbsp; RunnablePassthrough.assign({&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;response: llmChain,&nbsp; &nbsp; }),&nbsp; &nbsp;&nbsp;// step2: 使用 RunnableBranch 根据是否有 tool_calls 走不同分支&nbsp; &nbsp; RunnableBranch.from([&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 分支1：没有 tool_calls，认为本轮已经完成&nbsp; &nbsp; &nbsp; &nbsp; [&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;(state) =&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; !state.response?.tool_calls ||&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; state.response.tool_calls.length ===&nbsp;0,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;new&nbsp;RunnableLambda({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;func:&nbsp;async&nbsp;(state) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;{ messages, response } = state;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;newMessages = [...messages, response];&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ...state,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;messages: newMessages,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;done:&nbsp;true,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;final: response.content,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }),&nbsp; &nbsp; &nbsp; &nbsp; ],&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 默认分支：有 tool_calls，调用工具并把 ToolMessage 写回 messages&nbsp; &nbsp; &nbsp; &nbsp; RunnableSequence.from([&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;new&nbsp;RunnableLambda({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;func:&nbsp;async&nbsp;(state) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;{ messages, response } = state;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;newMessages = [...messages, response];&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; chalk.bgBlue(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;`🔍 检测到&nbsp;${response.tool_calls.length}&nbsp;个工具调用`&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; )&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; chalk.bgBlue(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;`🔍 工具调用:&nbsp;${response.tool_calls&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; .map((t) =&gt; t.name)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; .join(', ')}`&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; )&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ...state,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;messages: newMessages,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 调用工具执行器，得到 toolMessages&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; RunnablePassthrough.assign({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;toolMessages: toolExecutor,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;new&nbsp;RunnableLambda({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;func:&nbsp;async&nbsp;(state) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;{ messages, toolMessages } = state;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ...state,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;messages: [...messages, ...(toolMessages ?? [])],&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;done:&nbsp;false,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }),&nbsp; &nbsp; &nbsp; &nbsp; ]),&nbsp; &nbsp; ]),]);asyncfunction&nbsp;runAgentWithTools(query, maxIterations =&nbsp;30)&nbsp;{&nbsp; &nbsp;&nbsp;let&nbsp;state = {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;messages: [new&nbsp;HumanMessage(query)],&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;done:&nbsp;false,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;final:&nbsp;null,&nbsp; &nbsp; &nbsp; &nbsp; tools,&nbsp; &nbsp; };&nbsp; &nbsp;&nbsp;for&nbsp;(let&nbsp;i =&nbsp;0; i &lt; maxIterations; i++) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(chalk.bgGreen(`⏳ 正在等待 AI 思考...`));&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 每一轮都通过一个完整的 Runnable chain（LLM + 工具调用处理）&nbsp; &nbsp; &nbsp; &nbsp; state =&nbsp;await&nbsp;agentStepChain.invoke(state);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(state.done) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(`\n✨ AI 最终回复:\n${state.final}\n`);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;state.final;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;return&nbsp;state.messages[state.messages.length -&nbsp;1].content;}await&nbsp;runAgentWithTools("北京南站附近的酒店，最近的 3 个酒店，拿到酒店图片，打开浏览器，展示每个酒店的图片，每个 tab 一个 url 展示，并且在把那个页面标题改为酒店名");// await mcpClient.close();

我们加了一个 state 在多个 Runnable 之间传递，记录了 messages 数组、是否 done、以及最终的回复 final 以及所有 tools

![image](../IMG/2026-02-26_实战练习LCEL组装chain/2_公众号_Yi昭.png)

然后用 Runnable 的方式写下逻辑：

![image](../IMG/2026-02-26_实战练习LCEL组装chain/3_公众号_Yi昭.png)

首先大模型调用结果用 RunnablePassthrough.assign 加到 state 的 response 属性上。

这里不用手动 invoke，在 chain invoke 的时候，会自动执行所有的 Runnable

然后根据有没有 tool\_calls 来做 if else，也就是 RunnableBranch

if else 分别用 RunnableLambda 来写处理逻辑。

![image](../IMG/2026-02-26_实战练习LCEL组装chain/4_公众号_Yi昭.png)

这里涉及到另一个 chain 的调用，也就是执行工具的 chain，用 RunnablePassthrough.assign 把执行结果加到 toolMessage 属性上

另一个 chain 就是调用 tool，结果封装成 ToolMessage

![image](../IMG/2026-02-26_实战练习LCEL组装chain/5_公众号_Yi昭.png)

这样，整个 chain 就串联好了。

之后统一 invoke 这个组装好的 chain：

![image](../IMG/2026-02-26_实战练习LCEL组装chain/6_公众号_Yi昭.png)

如果返回的 state 是 done 就说明执行完了，没有 tool\_call 了，就返回 final 否则继续循环调用 chain

安装下依赖：

    ppip install langchain_mcp-adapters chalk

加一下配置：

![image](../IMG/2026-02-26_实战练习LCEL组装chain/7_公众号_Yi昭.png)

跑一下：

> 🎬 视频演示（原公众号视频）

逻辑是一样的，只是现在改成了 LCEL 的声明式写法。

然后再改造下之前那个 RAG + Milvus 的电子书语义助手

分析下：

> 🎬 视频演示（原公众号视频）

整个流程比较简单，我们改成 Runnable 版本

安装下依赖：

    ppip install @zilliz/milvus2-sdk-node

创建 src/cases/ebook-reader-rag.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ ChatOpenAI, OpenAIEmbeddings }&nbsp;from"langchain_openai";import&nbsp;{ RunnableSequence, RunnableLambda }&nbsp;from"langchain_core/runnables";import&nbsp;{ MilvusClient, MetricType }&nbsp;from"@zilliz/milvus2-sdk-node";import&nbsp;{ PromptTemplate }&nbsp;from"langchain_core/prompts";import&nbsp;{ StringOutputParser }&nbsp;from"langchain_core/output_parsers";const&nbsp;COLLECTION_NAME =&nbsp;"ebook_collection";const&nbsp;VECTOR_DIM =&nbsp;1024;// 初始化 OpenAI Chat 模型const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({temperature:&nbsp;0.7,modelName: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; },});// 初始化 Embeddings 模型const&nbsp;embeddings =&nbsp;new&nbsp;OpenAIEmbeddings({apiKey: process.env.OPENAI_API_KEY,model: process.env.EMBEDDINGS_MODEL_NAME,configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; },dimensions: VECTOR_DIM,});// 初始化原生 Milvus 客户端const&nbsp;milvusClient =&nbsp;new&nbsp;MilvusClient({address:&nbsp;"localhost:19530",});// 从 Milvus 中检索内容的 Runnableconst&nbsp;milvusSearch =&nbsp;new&nbsp;RunnableLambda({func:&nbsp;async&nbsp;(input) =&gt; {&nbsp; &nbsp;&nbsp;const&nbsp;{ question, k =&nbsp;5&nbsp;} = input;&nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp;&nbsp;// 1. 生成问题向量&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;queryVector =&nbsp;await&nbsp;embeddings.embedQuery(question);&nbsp; &nbsp; &nbsp;&nbsp;// 2. 调用 Milvus 搜索&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;searchResult =&nbsp;await&nbsp;milvusClient.search({&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;collection_name: COLLECTION_NAME,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;vector: queryVector,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;limit: k,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;metric_type: MetricType.COSINE,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;output_fields: ["id",&nbsp;"book_id",&nbsp;"chapter_num",&nbsp;"index",&nbsp;"content"],&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;results = searchResult.results ?? [];&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;retrievedContent = results.map((item, idx) =&gt;&nbsp;({&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;id: item.id,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;book_id: item.book_id,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;chapter_num: item.chapter_num,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;index: item.index ?? idx,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content: item.content,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;score: item.score,&nbsp; &nbsp; &nbsp; }));&nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{ question, retrievedContent };&nbsp; &nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp; &nbsp;&nbsp;console.error("检索内容时出错:", error.message);&nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{ question,&nbsp;retrievedContent: [] };&nbsp; &nbsp; }&nbsp; },});// PromptTemplate：负责把 context / question 拼成最终 promptconst&nbsp;promptTemplate = PromptTemplate.fromTemplate(`你是一个专业的《天龙八部》小说助手。基于小说内容回答问题，用准确、详细的语言。请根据以下《天龙八部》小说片段内容回答问题：{context}用户问题: {question}回答要求：1. 如果片段中有相关信息，请结合小说内容给出详细、准确的回答2. 可以综合多个片段的内容，提供完整的答案3. 如果片段中没有相关信息，请如实告知用户4. 回答要准确，符合小说的情节和人物设定5. 可以引用原文内容来支持你的回答AI 助手的回答:`);// 构建 context + 日志打印的 Runnableconst&nbsp;buildPromptInput =&nbsp;new&nbsp;RunnableLambda({func:&nbsp;async&nbsp;(input) =&gt; {&nbsp; &nbsp;&nbsp;const&nbsp;{ question, retrievedContent } = input;&nbsp; &nbsp;&nbsp;if&nbsp;(!retrievedContent.length) {&nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;hasContext:&nbsp;false,&nbsp; &nbsp; &nbsp; &nbsp; question,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;context:&nbsp;"",&nbsp; &nbsp; &nbsp; &nbsp; retrievedContent,&nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;// 打印检索结果&nbsp; &nbsp;&nbsp;console.log("=".repeat(80));&nbsp; &nbsp;&nbsp;console.log(`问题:&nbsp;${question}`);&nbsp; &nbsp;&nbsp;console.log("=".repeat(80));&nbsp; &nbsp;&nbsp;console.log("\n【检索相关内容】");&nbsp; &nbsp; retrievedContent.forEach((item, i) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp;&nbsp;console.log(`\n[片段&nbsp;${i +&nbsp;1}] 相似度:&nbsp;${item.score ??&nbsp;"N/A"}`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(`书籍:&nbsp;${item.book_id}`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(`章节: 第&nbsp;${item.chapter_num}&nbsp;章`);&nbsp; &nbsp; &nbsp;&nbsp;console.log(`片段索引:&nbsp;${item.index}`);&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;content = item.content ??&nbsp;"";&nbsp; &nbsp; &nbsp;&nbsp;console.log(&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;`内容:&nbsp;${content.substring(0,&nbsp;200)}${&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; content.length &gt;&nbsp;200&nbsp;?&nbsp;"..."&nbsp;:&nbsp;""&nbsp; &nbsp; &nbsp; &nbsp; }`&nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;const&nbsp;context = retrievedContent&nbsp; &nbsp; &nbsp; .map((item, i) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return`[片段&nbsp;${i +&nbsp;1}]章节: 第&nbsp;${item.chapter_num}&nbsp;章内容:&nbsp;${item.content}`;&nbsp; &nbsp; &nbsp; })&nbsp; &nbsp; &nbsp; .join("\n\n━━━━━\n\n");&nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp; &nbsp; &nbsp;&nbsp;hasContext:&nbsp;true,&nbsp; &nbsp; &nbsp; question,&nbsp; &nbsp; &nbsp; context,&nbsp; &nbsp; &nbsp; retrievedContent,&nbsp; &nbsp; };&nbsp; },});// 组合成完整的 RAG Runnable（检索 -&gt; 构建 Prompt 输入 -&gt; PromptTemplate -&gt; LLM -&gt; 文本）const&nbsp;ragChain = RunnableSequence.from([&nbsp; milvusSearch,&nbsp; buildPromptInput,new&nbsp;RunnableLambda({&nbsp; &nbsp;&nbsp;func:&nbsp;async&nbsp;(input) =&gt; {&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;{ hasContext, question, context } = input;&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!hasContext) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;fallback =&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"抱歉，我没有找到相关的《天龙八部》内容。请尝试换一个问题。";&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(fallback);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{ question,&nbsp;context:&nbsp;"",&nbsp;answer: fallback,&nbsp;noContext:&nbsp;true&nbsp;};&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;// PromptTemplate 需要 { question, context }&nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{ question, context,&nbsp;noContext:&nbsp;false&nbsp;};&nbsp; &nbsp; },&nbsp; }),&nbsp; promptTemplate,&nbsp; model,new&nbsp;StringOutputParser(),]);asyncfunction&nbsp;initMilvusCollection()&nbsp;{console.log("连接到 Milvus...");await&nbsp;milvusClient.connectPromise;console.log("✓ 已连接\n");try&nbsp;{&nbsp; &nbsp;&nbsp;await&nbsp;milvusClient.loadCollection({&nbsp;collection_name: COLLECTION_NAME });&nbsp; &nbsp;&nbsp;console.log("✓ 集合已加载\n");&nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp;&nbsp;if&nbsp;(!error.message.includes("already loaded")) {&nbsp; &nbsp; &nbsp;&nbsp;throw&nbsp;error;&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;console.log("✓ 集合已处于加载状态\n");&nbsp; }}asyncfunction&nbsp;main()&nbsp;{try&nbsp;{&nbsp; &nbsp;&nbsp;await&nbsp;initMilvusCollection();&nbsp; &nbsp;&nbsp;const&nbsp;input = {&nbsp; &nbsp; &nbsp;&nbsp;question:&nbsp;"鸠摩智会什么武功？",&nbsp; &nbsp; &nbsp;&nbsp;k:&nbsp;5,&nbsp; &nbsp; };&nbsp; &nbsp;&nbsp;console.log("=".repeat(80));&nbsp; &nbsp;&nbsp;console.log(`问题:&nbsp;${input.question}`);&nbsp; &nbsp;&nbsp;console.log("=".repeat(80));&nbsp; &nbsp;&nbsp;console.log("\n【AI 流式回答】\n");&nbsp; &nbsp;&nbsp;const&nbsp;stream =&nbsp;await&nbsp;ragChain.stream(input);&nbsp; &nbsp;&nbsp;forawait&nbsp;(const&nbsp;chunk&nbsp;of&nbsp;stream) {&nbsp; &nbsp; &nbsp; process.stdout.write(chunk);&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;console.log("\n");&nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp;&nbsp;console.error("错误:", error.message);&nbsp; }}await&nbsp;main();

整个 chain 是这样的：

![image](../IMG/2026-02-26_实战练习LCEL组装chain/8_公众号_Yi昭.png)

检索 Milvus -> 构建带有文档片段的 prompt  -> 调用大模型 -> 打印结果

组装检索向量库的 chain：

这里用 StringOutputParser 把大模型返回结果变为字符串，然后用 stream 流式打印

> 🎬 视频演示（原公众号视频）

通过这两个案例，我们就知道怎么用 Runnable 的方式来写逻辑了：

- 分析整个流程，拆成原子步骤
- 根据步骤之间的关系选择组件（线性、分支、并行、自定义逻辑等）
- 统一调用（invoke、stream、batch）

而且用 chain 的方式来写有很多好处，可以在每个节点上加一些逻辑，比如重试、传入配置、回调等。

创建 src/runnables/RunnableWithRetry.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ RunnableLambda }&nbsp;from"langchain_core/runnables";let&nbsp;attempt =&nbsp;0;// 一个会随机失败的 Runnable，用来演示 withRetryconst&nbsp;unstableRunnable = RunnableLambda.from(async&nbsp;(input) =&gt; {&nbsp; attempt +=&nbsp;1;console.log(`第&nbsp;${attempt}&nbsp;次尝试，输入:&nbsp;${input}`);// 模拟 70% 概率失败的情况if&nbsp;(Math.random() &lt;&nbsp;0.7) {&nbsp; &nbsp;&nbsp;console.log("本次尝试失败，抛出错误。");&nbsp; &nbsp;&nbsp;thrownewError("模拟的随机错误");&nbsp; }console.log("本次尝试成功。");return`成功处理:&nbsp;${input}`;});// 使用 withRetry 为 runnable 加上重试逻辑const&nbsp;runnableWithRetry = unstableRunnable.withRetry({// 总共最多 5 次尝试stopAfterAttempt:&nbsp;5});try&nbsp;{const&nbsp;result =&nbsp;await&nbsp;runnableWithRetry.invoke("演示 withRetry");console.log("✅ 最终结果:", result);}&nbsp;catch&nbsp;(err) {console.error("❌ 重试多次后仍然失败:", err?.message ?? err);}

我们用 withRetry 给某个 Runnable 节点加上重试逻辑。

70% 的概率失败，最多尝试 5 次。

试一下：

> 🎬 视频演示（原公众号视频）

我们简单的调用下 withRetry 就可以给这个 Runnable 节点加上重试逻辑，不用自己实现。

除了重试外，还有回退，也就是备选的方案：

src/runnables/RunnableWithFallbacks.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ RunnableLambda }&nbsp;from"langchain_core/runnables";// 模拟三个"翻译服务"，优先级从高到低const&nbsp;premiumTranslator = RunnableLambda.from(async&nbsp;(text) =&gt; {console.log("[Premium] 尝试翻译...");// 模拟高级服务不可用thrownewError("Premium 服务超时");});const&nbsp;standardTranslator = RunnableLambda.from(async&nbsp;(text) =&gt; {console.log("[Standard] 尝试翻译...");// 模拟标准服务也挂了thrownewError("Standard 服务限流");});const&nbsp;localTranslator = RunnableLambda.from(async&nbsp;(text) =&gt; {console.log("[Local] 使用本地词典翻译...");const&nbsp;dict = {&nbsp;hello:&nbsp;"你好",&nbsp;world:&nbsp;"世界",&nbsp;goodbye:&nbsp;"再见"&nbsp;};const&nbsp;words = text.toLowerCase().split(" ");return&nbsp;words.map((w) =&gt;&nbsp;dict[w] ?? w).join("");});// withFallbacks：依次尝试 premium → standard → localconst&nbsp;translator = premiumTranslator.withFallbacks({fallbacks: [standardTranslator, localTranslator],});const&nbsp;result =&nbsp;await&nbsp;translator.invoke("hello world");console.log("翻译结果:", result);

通过 withFallbacks 传入几种备选方案。

当前面的报错时，会尝试后面的方案。

> 🎬 视频演示（原公众号视频）

这样通过 withFallbak 就可以给节点加上备选方案。

再比如配置：

src/runnables/RunnableWithConfig.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ RunnableLambda, RunnableSequence }&nbsp;from"langchain_core/runnables";// 模拟一个简单的"用户数据库"const&nbsp;mockUsers =&nbsp;newMap([&nbsp; [&nbsp; &nbsp;&nbsp;"user-123",&nbsp; &nbsp; {&nbsp; &nbsp; &nbsp;&nbsp;id:&nbsp;"user-123",&nbsp; &nbsp; &nbsp;&nbsp;name:&nbsp;"神光",&nbsp; &nbsp; &nbsp;&nbsp;email:&nbsp;"guang@example.com",&nbsp; &nbsp; },&nbsp; ],]);// 节点1：根据 config.configurable.userId 查用户const&nbsp;fetchUserFromConfig = RunnableLambda.from(async&nbsp;(input, config) =&gt; {const&nbsp;userId = config?.configurable?.userId;console.log("【节点1】收到了通知内容:", input);console.log("【节点1】从 config 里拿到 userId:", userId);const&nbsp;user = userId ? mockUsers.get(userId) :&nbsp;null;if&nbsp;(!user) {&nbsp; &nbsp;&nbsp;thrownewError("未找到用户，无法发送通知");&nbsp; }return&nbsp;{&nbsp; &nbsp; user,&nbsp; &nbsp;&nbsp;notification: input,&nbsp; };});// 节点2：根据 config.configurable.role 做权限判断const&nbsp;checkPermissionByRole = RunnableLambda.from(async&nbsp;(state, config) =&gt; {const&nbsp;role = config?.configurable?.role ??&nbsp;"普通用户";console.log("【节点2】当前角色:", role);const&nbsp;canSend =&nbsp; &nbsp; role ===&nbsp;"管理员"&nbsp;||&nbsp; &nbsp; role ===&nbsp;"运营"&nbsp;||&nbsp; &nbsp; role ===&nbsp;"系统";if&nbsp;(!canSend) {&nbsp; &nbsp;&nbsp;thrownewError(`角色「${role}」无权限发送系统通知`);&nbsp; }return&nbsp;{&nbsp; &nbsp; ...state,&nbsp; &nbsp; role,&nbsp; };});// 节点3：根据 locale 生成最终通知文案const&nbsp;formatNotificationByLocale = RunnableLambda.from(async&nbsp;(state, config) =&gt; {const&nbsp;locale = config?.configurable?.locale ??&nbsp;"zh-CN";console.log("【节点3】locale:", locale);let&nbsp;content;if&nbsp;(locale ===&nbsp;"en-US") {&nbsp; &nbsp; content =&nbsp;`Dear&nbsp;${state.user.name},\n\n${state.notification}\n\n(from role:&nbsp;${state.role})`;&nbsp; }&nbsp;else&nbsp;{&nbsp; &nbsp; content =&nbsp;`亲爱的&nbsp;${state.user.name}，\n\n${state.notification}\n\n（发送人角色：${state.role}）`;&nbsp; }return&nbsp;{&nbsp; &nbsp; ...state,&nbsp; &nbsp; locale,&nbsp; &nbsp;&nbsp;finalContent: content,&nbsp; };});// 把三个节点串起来const&nbsp;chain = RunnableSequence.from([&nbsp; fetchUserFromConfig,&nbsp; checkPermissionByRole,&nbsp; formatNotificationByLocale,]);// 使用 withConfig 为整个 chain 绑定统一的配置const&nbsp;chainWithConfig = chain.withConfig({tags: ["demo",&nbsp;"withConfig",&nbsp;"notification"],metadata: {&nbsp; &nbsp;&nbsp;demoName:&nbsp;"RunnableWithConfig",&nbsp; },configurable: {&nbsp; &nbsp;&nbsp;userId:&nbsp;"user-123",&nbsp; &nbsp;&nbsp;role:&nbsp;"管理员",&nbsp; &nbsp;&nbsp;locale:&nbsp;"zh-CN",&nbsp; },});// 再创建一个不同配置的 chainWithConfig2，使用英文 localeconst&nbsp;chainWithConfig2 = chain.withConfig({tags: ["demo",&nbsp;"withConfig",&nbsp;"notification-en"],metadata: {&nbsp; &nbsp;&nbsp;demoName:&nbsp;"RunnableWithConfig2",&nbsp; },configurable: {&nbsp; &nbsp;&nbsp;userId:&nbsp;"user-123",&nbsp; &nbsp;&nbsp;role:&nbsp;"运营",&nbsp; &nbsp;&nbsp;locale:&nbsp;"en-US",&nbsp; },});// 输入为"要发送的通知文案"const&nbsp;result =&nbsp;await&nbsp;chainWithConfig.invoke("你有一条新的系统通知，请及时查看。");console.log("✅ 最终通知内容:\n", result.finalContent);console.log("\n--- chainWithConfig2 ---\n");const&nbsp;result2 =&nbsp;await&nbsp;chainWithConfig2.invoke("System maintenance scheduled tonight.");console.log("✅ 最终通知内容:\n", result2.finalContent);

我们用 withConfig 给 chain 传入配置，它会在每个 Runable 节点的第二个参数拿到。

我们在第一个节点根据配置拿用户信息，第二个节点根据配置做权限判断，第三个节点根据配置返回不同语言的内容。

跑一下：

> 🎬 视频演示（原公众号视频）

通过 withConfig 可以给 chain 的每个节点加上配置信息，可以通过第二个参数取出来用。

再就是每个节点可以加上一些回调逻辑：

src/runnables/RunnableWithCallbacks.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ RunnableLambda, RunnableSequence }&nbsp;from"langchain_core/runnables";// 文本处理链：清洗 → 分词 → 统计const&nbsp;clean = RunnableLambda.from((text) =&gt;&nbsp;{return&nbsp;text.trim().replace(/\s+/g,&nbsp;" ");});const&nbsp;tokenize = RunnableLambda.from((text) =&gt;&nbsp;{return&nbsp;text.split(" ");});const&nbsp;count = RunnableLambda.from((tokens) =&gt;&nbsp;{return&nbsp;{ tokens,&nbsp;wordCount: tokens.length };});const&nbsp;chain = RunnableSequence.from([clean, tokenize, count]);// 用 callbacks 观测每一步的输出const&nbsp;callback = {&nbsp; handleChainStart(chain) {&nbsp; &nbsp;&nbsp;const&nbsp;step = chain?.id?.[chain.id.length -&nbsp;1] ??&nbsp;"unknown";&nbsp; &nbsp;&nbsp;console.log(`[START]&nbsp;${step}`);&nbsp; },&nbsp; handleChainEnd(output) {&nbsp; &nbsp;&nbsp;console.log(`[END] &nbsp; output=${JSON.stringify(output)}\n`);&nbsp; },&nbsp; handleChainError(err) {&nbsp; &nbsp;&nbsp;console.log(`[ERROR]&nbsp;${err.message}\n`);&nbsp; },};const&nbsp;result =&nbsp;await&nbsp;chain.invoke(" &nbsp;hello &nbsp; world &nbsp; from &nbsp; langchain &nbsp;", {callbacks: [callback],});console.log("结果:", result);

比如一条有三个节点的 chain，我们想知道每个节点的输出

但是直接加到节点逻辑里也不大好，这种就可以用 callback 来打印。

> 🎬 视频演示（原公众号视频）

所以，用 chain 的方式，可以给每个节点加很多逻辑，比之前的写法灵活很多。

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

前面学了 LCEL 的 Runnable api，这节我们综合用了一下

用 Runnable 的方式重写了之前的 MCP、RAG 的案例代码。

用 Runnable 的流程是这样的：

- 分析流程，拆分原子步骤
- 根据步骤之间的关系，选择对应 Runable api
- 统一调用（invoke、stream、batch）

并且写好这个 chain 之后，可以灵活的加一些逻辑：

- withConfig 加入一些配置，chain 的节点可以通过第二个参数拿到
- withRetry 加上重试逻辑
- withFallback 加上备选方案
- callbacks 可以加一些回调函数，比如打印节点的输出

LCEL 是 LangChain 的灵魂，通过 Runnable 把所有的节点变成组件，随意组合使用，而且可以加入很多额外的逻辑。

后面的 LangGraph、LangSmith 也是基于 Runnable 的，需要熟练掌握这种声明式的代码写法。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-02-26 23:20:48 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247484988&idx=1&sn=0b63b03d5caadd8ab99bce59094b190d&chksm=f189d8ce74ff04209d8f9e93404b901016596a4884d9f291ecdc21668e6df1bad159282efc62&scene=27#wechat_redirect
