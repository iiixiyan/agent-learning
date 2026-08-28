# 图编排引擎：LangGraph 和多 Agent 架构

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年4月12日 00:02

复杂的 Agent 产品基本都是多 Agent 架构。

为什么呢？

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/0_公众号_Yi昭.png)

单 Agent 架构下，所有 tool 的描述、每个功能的 prompt 都放到 system prompt 里。

实际上执行每个功能只需要其中一部分 prompt，但每次都全带上。

这样会导致 token 消耗更高，更重要的是很多无关信息干扰，思考效率低还更容易出错。

而如果你拆分成多个 Agent 呢？

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/1_公众号_Yi昭.png)

每个 Agent 只保留需要的 prompt，执行功能的时候，消耗的 token 更少，没有无关信息干扰，准确率也更高。

再就是单 Agent 只有一个大脑，需要一步步思考，调用 tool

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/2_公众号_Yi昭.png)

而多 Agent 的多个大脑当然是可以并行思考的，主 Agent 下发任务，子 Agent 并行处理完成后返回

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/3_公众号_Yi昭.png)

还有，单 Agent 虽然可以加上反思阶段，但相当于自己给自己纠错

而多 Agent 每个都是不同的角色，可以互相讨论纠错

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/4_公众号_Yi昭.png)

基于这三个原因：

- **决策准确率更高、token 消耗更低**：每个 Agent 只带必要的最少prompt，没有冗余信息干扰，虽然调用 LLM 次数多了，但更省 token、决策更准、更稳定
- **并行思考和任务处理**：主管分派任务，子 Agent 并行处理，整体效率更高
- **多角色互相讨论，纠错能力更强**：多 Agent 有不同角色，可以互相监督、互相纠错，比单个 Agent 自己反思更靠谱，复杂任务表现更强

现在复杂 Agent 产品基本都是多 Agent 架构的。

实现 Multi Agent 就需要学习 LangGraph 了。

用到的 api 还是 LangChain 那些，但它多了一套图编排引擎。

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/5_公众号_Yi昭.png)

我们学了 LangChain 的组件，学了 LCEL 的线性编排，今天来学一下 LangGraph 的图编排引擎。

我们直接通过代码来学一下：

    mkdir langgraph-testcd&nbsp;langgraph-testnpm init -y

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/6_公众号_Yi昭.png)

安装依赖：

    ppip install langchain_langgraph langchain_core langchain_openai dotenv zod

创建 .env

    OPENAI_API_KEY=sk-xxxOPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1MODEL_NAME=qwen-plus

创建 src/basic-graph.mjs

    import&nbsp;{ Annotation, END, START, StateGraph }&nbsp;from"langchain_langgraph";const&nbsp;StateAnnotation = Annotation.Root({text: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;"",&nbsp; }),});const&nbsp;step1 =&nbsp;(state) =&gt;&nbsp;({&nbsp;text:&nbsp;`${state.text}&nbsp;-&gt; step1`&nbsp;});const&nbsp;step2 =&nbsp;(state) =&gt;&nbsp;({&nbsp;text:&nbsp;`${state.text}&nbsp;-&gt; step2`&nbsp;});const&nbsp;graph =&nbsp;new&nbsp;StateGraph(StateAnnotation)&nbsp; .addNode("step1", step1)&nbsp; .addNode("step2", step2)&nbsp; .addEdge(START,&nbsp;"step1")&nbsp; .addEdge("step1",&nbsp;"step2")&nbsp; .addEdge("step2", END)&nbsp; .compile();// 导出为 Mermaid：可复制到 https://mermaid.live 或 Markdown 的 ```mermaid 代码块const&nbsp;drawable =&nbsp;await&nbsp;graph.getGraphAsync();const&nbsp;mermaid = drawable.drawMermaid({&nbsp;withStyles:&nbsp;true&nbsp;});console.log(mermaid);const&nbsp;result =&nbsp;await&nbsp;graph.invoke({&nbsp;text:&nbsp;"hello"&nbsp;});console.log("result:", result);

创建 StateGraph 图

添加两个节点（node），加上固定的 START、END 节点

然后用边（edge）连起来

编译后执行

> 🎬 视频演示（原公众号视频）

Annotation 用于创建 State，指定默认值（default）和合并逻辑（reducer）

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/7_公众号_Yi昭.png)

这样我们基于 LangGraph 的第一个图就完成了。

图中当然有分支和循环。

先试一下分支：

src/conditional-routing.mjs

    import&nbsp;{ Annotation, END, START, StateGraph }&nbsp;from"langchain_langgraph";const&nbsp;StateAnnotation = Annotation.Root({query: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;"",&nbsp; }),route: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;"chat",&nbsp; }),answer: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;"",&nbsp; }),});const&nbsp;router =&nbsp;(state) =&gt;&nbsp;{const&nbsp;isMath =&nbsp;/[+\-*/]/.test(state.query);return&nbsp;{&nbsp;route: isMath ?&nbsp;"math"&nbsp;:&nbsp;"chat"&nbsp;};};const&nbsp;mathNode =&nbsp;(state) =&gt;&nbsp;{try&nbsp;{&nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp;answer:&nbsp;String(eval(state.query)) };&nbsp; }&nbsp;catch&nbsp;{&nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp;answer:&nbsp;"表达式无法计算"&nbsp;};&nbsp; }};const&nbsp;chatNode =&nbsp;(state) =&gt;&nbsp;({&nbsp;answer:&nbsp;`你说的是：${state.query}`&nbsp;});const&nbsp;graph =&nbsp;new&nbsp;StateGraph(StateAnnotation)&nbsp; .addNode("router", router)&nbsp; .addNode("math", mathNode)&nbsp; .addNode("chat", chatNode)&nbsp; .addEdge(START,&nbsp;"router")&nbsp; .addConditionalEdges("router", (state) =&gt; state.route, {&nbsp; &nbsp;&nbsp;math:&nbsp;"math",&nbsp; &nbsp;&nbsp;chat:&nbsp;"chat",&nbsp; })&nbsp; .addEdge("math", END)&nbsp; .addEdge("chat", END)&nbsp; .compile();// 导出为 Mermaid：可复制到 https://mermaid.live 或 Markdown 的 ```mermaid 代码块const&nbsp;drawable =&nbsp;await&nbsp;graph.getGraphAsync();const&nbsp;mermaid = drawable.drawMermaid({&nbsp;withStyles:&nbsp;true&nbsp;});console.log(mermaid);console.log("result:",await&nbsp;graph.invoke({&nbsp;query:&nbsp;"你好"&nbsp;}));console.log(&nbsp; &nbsp;&nbsp;"result:",&nbsp; &nbsp;&nbsp;await&nbsp;graph.invoke({&nbsp;query:&nbsp;"10 * 8"&nbsp;}));

用 addConditionalEdges 添加分支

判断文本如果有+-\*/字符就走 math 分支，否则走 chat 分支

> 🎬 视频演示（原公众号视频）

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/8_公众号_Yi昭.png)

接下来试一下循环，其实它也是用分支来实现：

src/loop-retry.mjs

    import&nbsp;{ Annotation, END, START, StateGraph }&nbsp;from"langchain_langgraph";const&nbsp;StateAnnotation = Annotation.Root({tries: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;0,&nbsp; }),ok: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;false,&nbsp; }),message: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;"",&nbsp; }),});const&nbsp;attempt =&nbsp;(state) =&gt;&nbsp;{const&nbsp;tries = state.tries +&nbsp;1;const&nbsp;ok = tries &gt;=&nbsp;3;return&nbsp;{&nbsp; &nbsp; tries,&nbsp; &nbsp; ok,&nbsp; &nbsp;&nbsp;message: ok ?&nbsp;`第&nbsp;${tries}&nbsp;次成功`&nbsp;:&nbsp;`第&nbsp;${tries}&nbsp;次失败，继续重试`,&nbsp; };};const&nbsp;graph =&nbsp;new&nbsp;StateGraph(StateAnnotation)&nbsp; .addNode("attempt", attempt)&nbsp; .addEdge(START,&nbsp;"attempt")&nbsp; .addConditionalEdges("attempt", (state) =&gt; (state.ok ?&nbsp;"done"&nbsp;:&nbsp;"retry"), {&nbsp; &nbsp;&nbsp;retry:&nbsp;"attempt",&nbsp; &nbsp;&nbsp;done: END,&nbsp; })&nbsp; .compile();// 导出为 Mermaid：可复制到 https://mermaid.live 或 Markdown 的 ```mermaid 代码块const&nbsp;drawable =&nbsp;await&nbsp;graph.getGraphAsync();const&nbsp;mermaid = drawable.drawMermaid({&nbsp;withStyles:&nbsp;true&nbsp;});console.log(mermaid);const&nbsp;result =&nbsp;await&nbsp;graph.invoke({&nbsp;tries:&nbsp;0&nbsp;});console.log("result:", result);

同样用 addConditionalEdges 判断条件满足就到 END 节点，否则重新路由到之前的节点

这样就可以实现循环效果

> 🎬 视频演示（原公众号视频）

经过这几个例子，应该能看出节点之间是怎么通信的：

通过 state

那把 state 保存下来不就是把当前图的执行状态保存下来了么？

这个通过 ChekpointerSaver 的 api 就可以保存

创建 src/checkpointer-memory.mjs

    import&nbsp;{&nbsp; Annotation,&nbsp; END,&nbsp; MemorySaver,&nbsp; START,&nbsp; StateGraph,}&nbsp;from"langchain_langgraph";const&nbsp;StateAnnotation = Annotation.Root({visitCount: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;0,&nbsp; }),message: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;"",&nbsp; }),});/** 每跑一轮图，给「当前会话」访问次数 +1 */function&nbsp;recordVisit(state)&nbsp;{const&nbsp;visitCount = state.visitCount +&nbsp;1;const&nbsp;message =&nbsp; &nbsp; visitCount ===&nbsp;1&nbsp; &nbsp; &nbsp; ?&nbsp;"这是你在本会话里第 1 次进入。"&nbsp; &nbsp; &nbsp; :&nbsp;`这是你在本会话里第&nbsp;${visitCount}&nbsp;次进入`;return&nbsp;{ visitCount, message };}const&nbsp;graph =&nbsp;new&nbsp;StateGraph(StateAnnotation)&nbsp; .addNode("recordVisit", recordVisit)&nbsp; .addEdge(START,&nbsp;"recordVisit")&nbsp; .addEdge("recordVisit", END);const&nbsp;checkpointer =&nbsp;new&nbsp;MemorySaver();const&nbsp;app = graph.compile({ checkpointer });const&nbsp;user1 = {&nbsp;configurable: {&nbsp;thread_id:&nbsp;"用户-小张"&nbsp;} };const&nbsp;user2 = {&nbsp;configurable: {&nbsp;thread_id:&nbsp;"用户-小李"&nbsp;} };const&nbsp;res1 =&nbsp;await&nbsp;app.invoke({}, user1);const&nbsp;res2 =&nbsp;await&nbsp;app.invoke({}, user1);const&nbsp;res3 =&nbsp;await&nbsp;app.invoke({}, user1);const&nbsp;res4 &nbsp;=&nbsp;await&nbsp;app.invoke({}, user2);console.log(res1)console.log(res2);console.log(res3);console.log(res4);

我们用 MemorySaver 来把 state 保存到内存里，这样下次就会基于上次的 state 继续执行

当然，还可以保存到 sqlite、redis 等，分别用 SqliteSave、RedisSaver 等 api

> 🎬 视频演示（原公众号视频）

我们用 cursor 之类的 coding agent，它经常会让你确认，确认后再继续执行，这种打断功能咋做呢？

LangGraph 提供了 interrupt 的 api

创建 src/graph-interrupt.mjs

    import&nbsp;{ createInterface }&nbsp;from"node:readline/promises";import&nbsp;{&nbsp; Annotation,&nbsp; Command,&nbsp; END,&nbsp; MemorySaver,&nbsp; START,&nbsp; StateGraph,&nbsp; interrupt,}&nbsp;from"langchain_langgraph";const&nbsp;StateAnnotation = Annotation.Root({actionSummary: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;"",&nbsp; }),userInput: Annotation({&nbsp; &nbsp;&nbsp;reducer:&nbsp;(_prev, next) =&gt;&nbsp;next,&nbsp; &nbsp;&nbsp;default:&nbsp;()&nbsp;=&gt;"",&nbsp; }),});/** 展示一笔待确认的转账 */const&nbsp;showTransfer =&nbsp;()&nbsp;=&gt;&nbsp;({actionSummary:&nbsp;"向张三转账 ¥100（模拟，不会真扣款）",});/** 停在这里等人输入；resume 的值会写进 userInput */const&nbsp;waitConfirm =&nbsp;(state) =&gt;&nbsp;{const&nbsp;text = interrupt({&nbsp; &nbsp;&nbsp;hint:&nbsp;"终端里输入「确认」或备注后回车，图才会继续",&nbsp; &nbsp;&nbsp;actionSummary: state.actionSummary,&nbsp; });return&nbsp;{&nbsp;userInput:&nbsp;String(text) };};const&nbsp;graph =&nbsp;new&nbsp;StateGraph(StateAnnotation)&nbsp; .addNode("showTransfer", showTransfer)&nbsp; .addNode("waitConfirm", waitConfirm)&nbsp; .addEdge(START,&nbsp;"showTransfer")&nbsp; .addEdge("showTransfer",&nbsp;"waitConfirm")&nbsp; .addEdge("waitConfirm", END)&nbsp; .compile({&nbsp;checkpointer:&nbsp;new&nbsp;MemorySaver() });// 导出为 Mermaid：可复制到 https://mermaid.live 或 Markdown 的 ```mermaid 代码块const&nbsp;drawable =&nbsp;await&nbsp;graph.getGraphAsync();const&nbsp;mermaid = drawable.drawMermaid({&nbsp;withStyles:&nbsp;true&nbsp;});console.log(mermaid);const&nbsp;config = {&nbsp;configurable: {&nbsp;thread_id:&nbsp;"interrupt-demo"&nbsp;} };const&nbsp;paused =&nbsp;await&nbsp;graph.invoke({}, config);console.log("\n待你确认：", paused.__interrupt__?.[0]?.value);const&nbsp;rl = createInterface({&nbsp;input: process.stdin,&nbsp;output: process.stdout });const&nbsp;line = (await&nbsp;rl.question("&gt; ")).trim();await&nbsp;rl.close();if&nbsp;(!line) {console.error("未输入，退出。");&nbsp; process.exit(1);}const&nbsp;done =&nbsp;await&nbsp;graph.invoke(new&nbsp;Command({&nbsp;resume: line }), config);console.log("结果：", done);

用 interrupt 中断图的执行

等待用户输入之后再次 invoke，传入 new Command({resume: 'xxx'})

这样图就会在上次断点位置继续执行

这里用了 nodejs 的 readline 包读取键盘输入

> 🎬 视频演示（原公众号视频）

这样就可以实现图的中断、恢复了。

此外，有些常用的节点，langgrph 给封装好了，放到 prebuilt 下：

src/prebuilt-tool-node.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ HumanMessage }&nbsp;from"langchain_core/messages";import&nbsp;{ tool }&nbsp;from"langchain_core/tools";import&nbsp;{&nbsp; END,&nbsp; MessagesAnnotation,&nbsp; START,&nbsp; StateGraph,}&nbsp;from"langchain_langgraph";import&nbsp;{ ToolNode, toolsCondition }&nbsp;from"langchain_langgraph/prebuilt";import&nbsp;{ ChatOpenAI }&nbsp;from"langchain_openai";import&nbsp;{ z }&nbsp;from"zod";import&nbsp;{ getProductBySku }&nbsp;from"./inventory-mock.mjs";const&nbsp;getProductStock = tool(async&nbsp;({ sku }) =&gt; getProductBySku(sku),&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;"get_product_stock",&nbsp; &nbsp;&nbsp;description:&nbsp; &nbsp; &nbsp;&nbsp;"按 SKU 查商品名与库存，SKU 如 SKU-001。",&nbsp; &nbsp;&nbsp;schema: z.object({&nbsp; &nbsp; &nbsp;&nbsp;sku: z.string().describe("商品 SKU"),&nbsp; &nbsp; }),&nbsp; });const&nbsp;tools = [getProductStock];const&nbsp;llm =&nbsp;new&nbsp;ChatOpenAI({&nbsp;modelName: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp; &nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; },}).bindTools(tools);asyncfunction&nbsp;agent(state)&nbsp;{const&nbsp;response =&nbsp;await&nbsp;llm.invoke(state.messages);return&nbsp;{&nbsp;messages: response };}const&nbsp;toolNode =&nbsp;new&nbsp;ToolNode(tools);const&nbsp;graph =&nbsp;new&nbsp;StateGraph(MessagesAnnotation)&nbsp; .addNode("agent", agent)&nbsp; .addNode("tools", toolNode)&nbsp; .addEdge(START,&nbsp;"agent")&nbsp; .addConditionalEdges("agent", toolsCondition, ["tools", END])&nbsp; .addEdge("tools",&nbsp;"agent")&nbsp; .compile();const&nbsp;result =&nbsp;await&nbsp;graph.invoke({messages: [&nbsp; &nbsp;&nbsp;new&nbsp;HumanMessage(&nbsp; &nbsp; &nbsp;&nbsp;"查一下 SKU-001 的库存还有多少，回答里带上商品名和数字。"&nbsp; &nbsp; ),&nbsp; ],});// 导出为 Mermaid：可复制到 https://mermaid.live 或 Markdown 的 ```mermaid 代码块const&nbsp;drawable =&nbsp;await&nbsp;graph.getGraphAsync();const&nbsp;mermaid = drawable.drawMermaid({&nbsp;withStyles:&nbsp;true&nbsp;});console.log(mermaid);const&nbsp;last = result.messages.at(-1);console.log(last?.content ?? result.messages);

比如我们要调用 tool，用 graph 的写法怎么写呢？

创建 model 的节点，创建 tool 的节点

然后加一个 conditional 节点，判断如果有 tool call 就走 tool 节点，否则走 END

但不用自己写，langgraph 内置了 ToolNode 和 toolsCondition 的 api

用到的 inventory.mock.mjs

    /** 假数据，模拟「按 SKU 查库存」接口 */const&nbsp;rows = [&nbsp; {&nbsp;sku:&nbsp;"SKU-001",&nbsp;name:&nbsp;"无线鼠标",&nbsp;stock:&nbsp;42&nbsp;},&nbsp; {&nbsp;sku:&nbsp;"SKU-002",&nbsp;name:&nbsp;"机械键盘",&nbsp;stock:&nbsp;7&nbsp;},&nbsp; {&nbsp;sku:&nbsp;"SKU-003",&nbsp;name:&nbsp;"USB-C 线缆",&nbsp;stock:&nbsp;120&nbsp;},];exportfunction&nbsp;getProductBySku(sku)&nbsp;{const&nbsp;key =&nbsp;String(sku).trim().toUpperCase();const&nbsp;row = rows.find((r) =&gt;&nbsp;r.sku.toUpperCase() === key);if&nbsp;(!row)&nbsp;returnJSON.stringify({&nbsp;found:&nbsp;false,&nbsp;sku:&nbsp;String(sku).trim() });returnJSON.stringify({&nbsp;found:&nbsp;true, ...row });}

> 🎬 视频演示（原公众号视频）

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/9_公众号_Yi昭.png)

当然，像这么常用的 agent loop 自然也给封装好了，就是 createAgent 的 api：

prebuilt-agent.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ HumanMessage }&nbsp;from"langchain_core/messages";import&nbsp;{ ChatOpenAI }&nbsp;from"langchain_openai";import&nbsp;{ MemorySaver }&nbsp;from"langchain_langgraph";import&nbsp;{ createAgent, tool }&nbsp;from"langchain";import&nbsp;{ z }&nbsp;from"zod";import&nbsp;{ getProductBySku }&nbsp;from"./inventory-mock.mjs";const&nbsp;getProductStock = tool(async&nbsp;({ sku }) =&gt; getProductBySku(sku),&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;"get_product_stock",&nbsp; &nbsp;&nbsp;description:&nbsp; &nbsp; &nbsp;&nbsp;"按 SKU 查商品名与库存，SKU 如 SKU-001。",&nbsp; &nbsp;&nbsp;schema: z.object({&nbsp; &nbsp; &nbsp;&nbsp;sku: z.string().describe("商品 SKU"),&nbsp; &nbsp; }),&nbsp; });const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({&nbsp;modelName: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp; &nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; },});const&nbsp;agent = createAgent({&nbsp; model,tools: [getProductStock],systemPrompt:&nbsp; &nbsp;&nbsp;"你是仓库助手。问库存时必须调用 get_product_stock（模拟数据），禁止编造。",checkpointer:&nbsp;new&nbsp;MemorySaver(),});const&nbsp;result =&nbsp;await&nbsp;agent.invoke(&nbsp; {&nbsp;messages: [new&nbsp;HumanMessage("SKU-002 还剩多少库存？")] },&nbsp; {&nbsp;configurable: {&nbsp;thread_id:&nbsp;"demo-thread"&nbsp;} });// 导出为 Mermaid：可复制到 https://mermaid.live 或 Markdown 的 ```mermaid 代码块const&nbsp;drawable =&nbsp;await&nbsp;agent.graph.getGraphAsync();const&nbsp;mermaid = drawable.drawMermaid({&nbsp;withStyles:&nbsp;true&nbsp;});console.log(mermaid);const&nbsp;last = result.messages.at(-1);console.log(last?.content ?? result);

直接用 createAgent 来跑 agent loop

看一下它的图：

> 🎬 视频演示（原公众号视频）

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/10_公众号_Yi昭.png)

和刚才写的一样，这个 api 内部就是基于 LangGraph 构建的 agent loop 的图。

学完 LangGraph 的图，我们来写一个多 Agent 的架构

多 Agent 最常用的是 Supervisor - Worker 模式，也就是“主管 - 工人”模式

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/11_公众号_Yi昭.png)

langchain 提供了这种多 Agent 架构的包 langchain_langgraph-supervisor

安装下：

    ppip install langchain_langgraph-supervisor

创建 multi-agent-supervisor.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ HumanMessage }&nbsp;from"langchain_core/messages";import&nbsp;{ createSupervisor }&nbsp;from"langchain_langgraph-supervisor";import&nbsp;{ ChatOpenAI }&nbsp;from"langchain_openai";import&nbsp;{ createAgent, tool }&nbsp;from"langchain";import&nbsp;{ z }&nbsp;from"zod";import&nbsp;{ lookupCityTrivia, lookupWeather }&nbsp;from"./simple-mock.mjs";const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({modelName: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; },});const&nbsp;lookupWeatherTool = tool(async&nbsp;({ city }) =&gt; lookupWeather(city),&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;"lookup_weather",&nbsp; &nbsp;&nbsp;description:&nbsp;"查询某城市当日天气概况（气温区间、天气、空气质量等）。",&nbsp; &nbsp;&nbsp;schema: z.object({&nbsp; &nbsp; &nbsp;&nbsp;city: z.string().describe("城市名，如 杭州"),&nbsp; &nbsp; }),&nbsp; });const&nbsp;lookupCityTriviaTool = tool(async&nbsp;({ city }) =&gt; lookupCityTrivia(city),&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;"lookup_city_trivia",&nbsp; &nbsp;&nbsp;description:&nbsp;"查询与某城市相关的一句趣味知识。",&nbsp; &nbsp;&nbsp;schema: z.object({&nbsp; &nbsp; &nbsp;&nbsp;city: z.string().describe("城市名，如 杭州"),&nbsp; &nbsp; }),&nbsp; });/** 子代理 A：只回答「天气」类问题 */const&nbsp;weatherAgent = createAgent({name:&nbsp;"weather_agent",description:&nbsp;"专门查天气",&nbsp; model,tools: [lookupWeatherTool],systemPrompt:&nbsp;"你只处理天气。用户提到城市时，用 lookup_weather 查询后再用中文简短说明。",});/** 子代理 B：只回答「城市小知识」 */const&nbsp;triviaAgent = createAgent({name:&nbsp;"trivia_agent",description:&nbsp;"专门讲与城市相关的小知识；必须调用 lookup_city_trivia。",&nbsp; model,tools: [lookupCityTriviaTool],systemPrompt:&nbsp;"你只讲城市小知识。先 lookup_city_trivia，再用人话转述，不要编造工具里没有的内容。",});/**&nbsp;* Supervisor：根据用户问的是「天气」还是「小知识」切换子代理。&nbsp;* （真实业务里还可以再加更多子代理，思路一样。）&nbsp;*/const&nbsp;workflow = createSupervisor({agents: [weatherAgent.graph, triviaAgent.graph],llm: model,prompt:&nbsp;`你是调度员，只负责选人，不要自己报气温、也不要自己讲城市百科。- 问天气、气温、下不下雨、空气 → 用 weather_agent- 问小知识、名胜、历史、一句介绍 → 用 trivia_agent`,});const&nbsp;app = workflow.compile();const&nbsp;drawable =&nbsp;await&nbsp;app.getGraphAsync();console.log(drawable.drawMermaid({&nbsp;withStyles:&nbsp;true&nbsp;}));const&nbsp;input = {messages: [&nbsp; &nbsp;&nbsp;new&nbsp;HumanMessage("查一下杭州的天气，再讲一条和杭州有关的小知识。"),&nbsp; ],};const&nbsp;nodePath = [];let&nbsp;finalState =&nbsp;null;const&nbsp;stream =&nbsp;await&nbsp;app.stream(input, {&nbsp;streamMode: ["updates",&nbsp;"values"] });forawait&nbsp;(const&nbsp;event&nbsp;of&nbsp;stream) {const&nbsp;[mode, payload] = event;if&nbsp;(mode ===&nbsp;"updates"&nbsp;&amp;&amp; payload &amp;&amp;&nbsp;typeof&nbsp;payload ===&nbsp;"object") {&nbsp; &nbsp; nodePath.push(...Object.keys(payload));&nbsp; }&nbsp;elseif&nbsp;(mode ===&nbsp;"values") {&nbsp; &nbsp; finalState = payload;&nbsp; }}console.log("路径:", nodePath.join(" → "));const&nbsp;last = finalState?.messages?.at(-1);console.log(last?.content ?? finalState?.messages);

我们用 createAgent 创建了 2 个 子 Agent

然后用 createSupervisor 创建主管 Agent：

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/12_公众号_Yi昭.png)

子 Agent 一个查天气，一个查城市历史

用 stream 可以拿到整个图运行过程的状态

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/13_公众号_Yi昭.png)

它的 state 内容挺多的，所以支持几种模式

updates 是增量模式，就是过滤出这个节点增量修改的 state 来

values 是全量模式，给你所有的 state

我们这里用 updates 模式拿到经过的节点的名字

最后的回复用 values 模式拿

用到查询代码的实现：

    /** 假接口：演示 supervisor 如何把问题分给不同子代理 */function&nbsp;normCity(city)&nbsp;{returnString(city).trim();}const&nbsp;weatherTable = {&nbsp; 杭州: {&nbsp;summary:&nbsp;"多云转小雨",&nbsp;tempHighC:&nbsp;22,&nbsp;tempLowC:&nbsp;15,&nbsp;aqi:&nbsp;"良"&nbsp;},&nbsp; 北京: {&nbsp;summary:&nbsp;"晴",&nbsp;tempHighC:&nbsp;26,&nbsp;tempLowC:&nbsp;12,&nbsp;aqi:&nbsp;"轻度污染"&nbsp;},&nbsp; 上海: {&nbsp;summary:&nbsp;"阴",&nbsp;tempHighC:&nbsp;20,&nbsp;tempLowC:&nbsp;16,&nbsp;aqi:&nbsp;"良"&nbsp;},};const&nbsp;triviaTable = {&nbsp; 杭州:&nbsp;"西湖文化景观是世界文化遗产之一。",&nbsp; 北京:&nbsp;"故宫是世界上现存规模最大的古代宫殿建筑群之一。",&nbsp; 上海:&nbsp;"外滩万国建筑博览群是近代城市历史的缩影。",};/** 查某地当日天气摘要（模拟） */exportfunction&nbsp;lookupWeather(city)&nbsp;{const&nbsp;c = normCity(city);const&nbsp;w = weatherTable[c];if&nbsp;(!w) {&nbsp; &nbsp;&nbsp;returnJSON.stringify({&nbsp; &nbsp; &nbsp;&nbsp;city: c,&nbsp; &nbsp; &nbsp;&nbsp;summary:&nbsp;"暂无该城市数据，以下为占位",&nbsp; &nbsp; &nbsp;&nbsp;tempHighC:&nbsp;20,&nbsp; &nbsp; &nbsp;&nbsp;tempLowC:&nbsp;12,&nbsp; &nbsp; &nbsp;&nbsp;aqi:&nbsp;"—",&nbsp; &nbsp; });&nbsp; }returnJSON.stringify({&nbsp;city: c, ...w });}/** 查与某城市相关的一句小知识（模拟） */exportfunction&nbsp;lookupCityTrivia(city)&nbsp;{const&nbsp;c = normCity(city);const&nbsp;line = triviaTable[c];returnJSON.stringify({&nbsp; &nbsp;&nbsp;city: c,&nbsp; &nbsp;&nbsp;trivia: line ??&nbsp;`没有为「${c}」准备内置小知识，可换杭州/北京/上海试试。`,&nbsp; });}

跑一下：

> 🎬 视频演示（原公众号视频）

这样，我们第一个多 Agent 的代码就跑通了。

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/14_公众号_Yi昭.png)

虽然用 stream 的 values 模式可以打印 state，但是它内容太多了。

如果想看一下执行过程，最好的方式是断点调试。

> 🎬 视频演示（原公众号视频）

通过调试，就可以清晰的看到整个 graph 的流转过程。

也可以看到 langchain_langgraph-supervisor 的多 Agent 架构的实现原理，就是在 state 里保存了 messages 数组来传递信息

回头看下这张图：

![image](../IMG/2026-04-12_图编排引擎：LangGraph和多Agent架构/15_公众号_Yi昭.)

我们学 LangChain 的组件层花了比较多时间，学编排层的 LCEL、LangGraph 都是很快的，一两节搞定。

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

这节我们学了 LangGraph 和多 Agent 架构。

我们理清了 3 个用多 Agent 架构的理由：

- prompt 拆分到多个 Agent 中去，更纯净，token 消耗少，不容易决策出错
- 多个 Agent 可以并行思考和执行任务
- 多个 Agent 基于各自的角色可以相互讨论、纠错

复杂的 Agent 产品基本都是多 Agent 架构。

我们学了 LangGraph 的图怎么创建：

- state 用 Annotation 创建，包括 default（默认值）、reducer（值怎么合并）
- 图用 StateGraph 创建，可以添加 node（节点）、edge（边）
- 边可以用 addConditionalEdges 添加路由分支，基于这个也可以实现循环
- 可以用 MemorySaver 等 checkpointer 保存节点的 state，这样就可以恢复上次执行状态了
- 用 interupt 可以做图执行过程的打断，之后再次 invoke 传入 resume Command 即可恢复执行

还学了 prebuilt 的 ToolNode、toolsCondition 以及 createAgent 这些内置的节点、图

学完 LangGraph 的图之后，我们学了多 Agent

多 Agent 一般是 Supervisor - Worker 的架构

直接用 langchain_langgraph-supervisor 这个包就行，它封装了这套架构。

用 stream 可以看到图执行过程中的 state，分别用 updates、values 可以增量、全量看到节点输出的 state

当然，打印太多的话可以直接用断点调试来看多 Agent 的流转过程。

Supervisor 主管节点只负责任务分发，Worker 来做具体的任务执行。

后面我们的项目实战都是基于这种多 Agent 的架构来写。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-04-12 00:02:29 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247485805&idx=1&sn=24678fcc899c925d821daf3826a1597e&chksm=f1cc422c60f86b1045d39a0cc3e33bee8aa06b49bb74b1881eb54ea8a525a65d4b9c5051c59e&scene=27#wechat_redirect
