# 高德 MCP + 浏览器 MCP：LangChain 复用别人的 MCP Server 有多爽！

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2025年12月25日 13:12

上节我们学了 MCP。

自己实现了一个 MCP Server，然后在 Cursor 或者 LangChain 里连上这个 server，就可以用里面的 tools 了。

> 🎬 视频演示（原公众号视频）

![image](../IMG/2025-12-25_高德MCP浏览器MCP：LangChain复用别人的MCPServer有多爽/0_公众号_Yi昭.png)

它本质上还是 tool，只不过包了一层进程，可以通过 stdio 和 http 来访问。

![image](../IMG/2025-12-25_高德MCP浏览器MCP：LangChain复用别人的MCPServer有多爽/1_公众号_Yi昭.png)

有这一层协议之后，有个巨大的好处：

任何人都可以开发基于这个协议的 MCP Server，然后我们可以直接复用！

比如上节我们写的那个 MCP Server 就可以被别人用。

这节我们用一下别人写好的 MCP Server，感受下 MCP 有多爽！

我们用这三个 MCP Server：

- 高德 MCP：可以做位置查询、路线规划等
- Chrome DevTools MCP：控制浏览器，打开关闭页面、点击元素、截图等
- FileSystem MCP：读写文件、创建目录等

首先是高德 MCP，我们需要先获取一个 apikey：

https://developer.amap.com/

> 🎬 视频演示（原公众号视频）

创建应用，然后创建一个 api key

类型选 web 服务就行。

然后我们先在 cursor 里测试下这个 mcp 服务是否可用：

> 🎬 视频演示（原公众号视频）

可以看到，配好之后，就可以查到这个 mcp server 里的一堆 tool 了：

![image](../IMG/2025-12-25_高德MCP浏览器MCP：LangChain复用别人的MCPServer有多爽/2_公众号_Yi昭.png)

记得我们说过 mcp 有两种接入方式么？

![image](../IMG/2025-12-25_高德MCP浏览器MCP：LangChain复用别人的MCPServer有多爽/3_公众号_Yi昭.png)

这就是 http 的接入方式。

当然，高德也支持 stdio 的本地进程的接入方式，这样写：

    "amap-maps": {&nbsp;&nbsp;"command":&nbsp;"npx",&nbsp;&nbsp;"args": [&nbsp; &nbsp;&nbsp;"-y",&nbsp; &nbsp;&nbsp;"@amap/amap-maps-mcp-server"&nbsp; ],&nbsp;&nbsp;"env": {&nbsp; &nbsp;&nbsp;"AMAP_MAPS_API_KEY":&nbsp;"你的 api key"&nbsp; }},

> 🎬 视频演示（原公众号视频）

就是用 python -m 跑一个 npm 包，会创建一个支持 stdio 连接的进程，然后连上其中的 mcp server 就好了。

这个 mcp server 里肯定封装了和高德服务端的通信，本质上是一样的。

其实你的前端简历里就可以写一下这个：

我开发了一个 mcp server 的 npm 包，包含 xxx tool，支持 stdio 访问。可以在 cursor 或 langchain 里用 python -m 执行来连上这个 mcp server。

这样面试官一看就知道，这个人是真懂 MCP 的，而且还有实践经验。

说回正题，我们在 langchain 里用一下这个 mcp：

在 tool-test 项目里创建 src/mcp-test.mjs

    import&nbsp;'dotenv/config';import&nbsp;{ MultiServerMCPClient }&nbsp;from'langchain_mcp-adapters';import&nbsp;{ ChatOpenAI }&nbsp;from'langchain_openai';import&nbsp;chalk&nbsp;from'chalk';import&nbsp;{ HumanMessage, SystemMessage, ToolMessage }&nbsp;from'langchain_core/messages';const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({&nbsp;&nbsp; &nbsp;&nbsp;modelName:&nbsp;"qwen-plus",&nbsp; &nbsp;&nbsp;apiKey: process.env.OPENAI_API_KEY,&nbsp; &nbsp;&nbsp;configuration: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; &nbsp; },});const&nbsp;mcpClient =&nbsp;new&nbsp;MultiServerMCPClient({&nbsp; &nbsp;&nbsp;mcpServers: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'my-mcp-server': {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;command:&nbsp;"node",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;args: [&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"/Users/guang/code/tool-test/src/my-mcp-server.mjs"&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ]&nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"amap-maps-streamableHTTP": {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"url":&nbsp;"https://mcp.amap.com/mcp?key="&nbsp;+ process.env.AMAP_MAPS_API_KEY&nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; }});const&nbsp;tools =&nbsp;await&nbsp;mcpClient.getTools();const&nbsp;modelWithTools = model.bindTools(tools);asyncfunction&nbsp;runAgentWithTools(query, maxIterations =&nbsp;30)&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;messages = [&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;new&nbsp;HumanMessage(query)&nbsp; &nbsp; ];&nbsp; &nbsp;&nbsp;for&nbsp;(let&nbsp;i =&nbsp;0; i &lt; maxIterations; i++) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(chalk.bgGreen(`⏳ 正在等待 AI 思考...`));&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;response =&nbsp;await&nbsp;modelWithTools.invoke(messages);&nbsp; &nbsp; &nbsp; &nbsp; messages.push(response);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 检查是否有工具调用&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!response.tool_calls || response.tool_calls.length ===&nbsp;0) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(`\n✨ AI 最终回复:\n${response.content}\n`);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;response.content;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(chalk.bgBlue(`🔍 检测到&nbsp;${response.tool_calls.length}&nbsp;个工具调用`));&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(chalk.bgBlue(`🔍 工具调用:&nbsp;${response.tool_calls.map(t =&gt; t.name).join(', ')}`));&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 执行工具调用&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;for&nbsp;(const&nbsp;toolCall&nbsp;of&nbsp;response.tool_calls) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;foundTool = tools.find(t&nbsp;=&gt;&nbsp;t.name === toolCall.name);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(foundTool) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;toolResult =&nbsp;await&nbsp;foundTool.invoke(toolCall.args);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; messages.push(new&nbsp;ToolMessage({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content: toolResult,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;tool_call_id: toolCall.id,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }));&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;return&nbsp;messages[messages.length -&nbsp;1].content;}await&nbsp;runAgentWithTools("北京南站附近的酒店，以及去的路线");await&nbsp;mcpClient.close();

mcp client 的代码和上节一样，用 langchain_mcp-adapters

拿到其中的 tools 绑定给 model

然后调用 model，如果有 tool\_calls 就调用下，把工具调用结果封装为 ToolMessage 传给大模型继续处理。

这里的高德 api key 同样放到了 .env 里：

![image](../IMG/2025-12-25_高德MCP浏览器MCP：LangChain复用别人的MCPServer有多爽/4_公众号_Yi昭.png)

先注释掉高德 mcp server 跑一下：

> 🎬 视频演示（原公众号视频）

可以看到，大模型没法处理地理位置信息，让你用地图。

我们启用高德 MCP Server：

> 🎬 视频演示（原公众号视频）

现在，大模型就可以调用高德 mcp 里的 tool 给出酒店位置和路线了！

这就是 mcp 的好处，直接复用别人写好的 tool。

然后文件读写、创建目录这种，也不用自己写 tool，可以用现成 mcp：

![image](../IMG/2025-12-25_高德MCP浏览器MCP：LangChain复用别人的MCPServer有多爽/5_公众号_Yi昭.png)

mcp 官方维护的一个 mcp server

    "filesystem": {&nbsp; &nbsp;&nbsp;"command":&nbsp;"npx",&nbsp; &nbsp;&nbsp;"args": [&nbsp; &nbsp; &nbsp;&nbsp;"-y",&nbsp; &nbsp; &nbsp;&nbsp;"@modelcontextprotocol/server-filesystem",&nbsp; &nbsp; &nbsp; ...(process.env.ALLOWED_PATHS.split(',') ||&nbsp;'')&nbsp; &nbsp; ]}

后面是可访问的目录，我们配在 .env 里：

![image](../IMG/2025-12-25_高德MCP浏览器MCP：LangChain复用别人的MCPServer有多爽/6_公众号_Yi昭.png)

逗号分隔

我们可以先在 cursor 里配置下这个 mcp 服务，看下有哪些 tool：

> 🎬 视频演示（原公众号视频）

可以看到，有文件读写、目录创建、文件移动等 tool。

这样，配上这个 mcp，大模型就有文件读写能力了。

不过这里还有个坑注意下：

![image](../IMG/2025-12-25_高德MCP浏览器MCP：LangChain复用别人的MCPServer有多爽/7_公众号_Yi昭.png)

一般我们写 tool 都是直接返回字符串，但是 FileSystem MCP 封装的这些 tool 返回的是对象，有 text 属性，所以要处理下：

    // 确保 content 是字符串类型let&nbsp;contentStr;if&nbsp;(typeof&nbsp;toolResult ===&nbsp;'string') {&nbsp; &nbsp; contentStr = toolResult;}&nbsp;else&nbsp;if&nbsp;(toolResult &amp;&amp; toolResult.text) {&nbsp; &nbsp;&nbsp;// 如果返回对象有 text 字段，优先使用&nbsp; &nbsp; contentStr = toolResult.text;}messages.push(new&nbsp;ToolMessage({&nbsp; &nbsp;&nbsp;content: contentStr,&nbsp; &nbsp;&nbsp;tool_call_id: toolCall.id,}));

改下提示词：

    await&nbsp;runAgentWithTools("北京南站附近的5个酒店，以及去的路线，路线规划生成文档保存到 /Users/guang/Desktop 的一个 md 文件");

跑跑试试：

> 🎬 视频演示（原公众号视频）

可以看到，大模型首先调用高德 mcp 拿到了附近的酒店位置，然后规划了路线

最后调用 FileSystem MCP 写入了文件。

直接复用别人的 MCP，完全不用自己写。

你自己写的 tool 想给别人用，也可以封装成 MCP，最好发个 npm 包，这样还可以写到简历上去，让面试官用。

最后我们再来用一下 Chrome Devtools 的 MCP，它是可以用来做浏览器自动化的。

比如打开页面、点击元素、截图等。

在 cursor 配置下：

    "chrome-devtools": {&nbsp;&nbsp;"command":&nbsp;"npx",&nbsp;&nbsp;"args": [&nbsp; &nbsp;&nbsp;"-y",&nbsp; &nbsp;&nbsp;"chrome-devtools-mcp@latest"&nbsp; ]}

> 🎬 视频演示（原公众号视频）

可以看到很多工具。

在 langchain 里调用下：

![image](../IMG/2025-12-25_高德MCP浏览器MCP：LangChain复用别人的MCPServer有多爽/8_公众号_Yi昭.png)

改下提示词：

    await&nbsp;runAgentWithTools("北京南站附近的酒店，最近的 3 个酒店，拿到酒店图片，打开浏览器，展示每个酒店的图片，每个 tab 一个 url 展示，并且在把那个页面标题改为酒店名");

> 🎬 视频演示（原公众号视频）

可以看到，搜到了北京南站最近的 3 个酒店，并且浏览器打开了酒店图片。

只要配好 MCP，大模型就可以直接调用里面的 tools 了：

![image](../IMG/2025-12-25_高德MCP浏览器MCP：LangChain复用别人的MCPServer有多爽/9_公众号_Yi昭.png)

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code/tool-test

## 总结

这节我们使用了高德、FileSystem、Chrome Devtools 的 MCP，用它们结合来实现了一些功能。

这些 MCP Server 有的是 stdio 本地进程调用，有的是 http 远程进程调用。

MCP 的一大好处就是别人开发好的，可以直接用。

你全程不需要知道怎么用高德的 API 查询位置、路线，不需要知道怎么用 cdp 协议控制浏览器。

你只需要把这些 MCP 给到 AI，让它自己去调用。

你不需要知道这些 tool 里面的高德 API 怎么用、浏览器控制怎么用，大模型会自己读取 tool 描述来传入参数调用。是不是特别爽！

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2025-12-25 13:12:45 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247483989&idx=1&sn=1fdb1181382f1ae1c33083a5815e9459&chksm=f16fc32b9fbf48067f1b7aeafea622edbcca2a5391fae808bee0da2ff8488c1e074eedf2e229&scene=27#wechat_redirect
