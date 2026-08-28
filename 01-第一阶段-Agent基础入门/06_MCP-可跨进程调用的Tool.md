# MCP：可跨进程调用的 Tool

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2025年12月24日 08:10

我们已经写了一些 tool 了：读写文件和目录、执行命令

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/0_公众号_Yi昭.png)

只要声明 tool 的名字、描述、参数格式，模型会在发现需要用 tool 的时候自动解析出参数传入来调用，然后把执行结果封装成 ToolMessage 传入 chat。

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/1_公众号_Yi昭.png)

比如上节我们实现了简易的 cursor，就是声明了读写文件和目录、执行命令的 tool，这样你让大模型创建 react + vite 项目，它就会自动判断什么时候调用哪个 tool，自动实现目录、文件的创建，以及 ppip install 和 pnpn run dev 的执行。

> 🎬 视频演示（原公众号视频）

我们只是告诉他要创建的项目，然后安装依赖跑起来。

这些 tool 怎么调用、参数是什么都是大模型自己决定的。

tool 给大模型扩展了做事情的能力，本来它只能思考，不能做事情，但是现在可以自己调用 tool 来帮你做事情了。

但你有没有发现 tool 有个问题：

python 写的 ai agent 的代码，你的 tool 也得是 python 写。

如果你之前有一些工具是 java、python、rust 写的呢？

你想封装成 tool 怎么办呢？

有的同学说：现在不是可以执行命令么，通过单独进程把这些其他语言写的代码跑一下就行啊。

确实，也就是这样：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/2_公众号_Yi昭.png)

这里的 stdio 就是标准输入输出流，也就是键盘输入、控制台输出。当你进程跑一个子进程，就可以用这种方式通信。

还有的同学说：简单，用 http 啊！本地跑个服务就好了。

也就是这样：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/3_公众号_Yi昭.png)

现在是解决了跨语言调用工具的问题。

那如果每个人都这样搞，它们提供的服务都不一样，我想接入别的 tool，是不是要了解每个服务都是怎么定义的呢？

能不能定义一个统一的通信协议，我们都按照这个格式来沟通，这样所有的跨进程工具调用就都可以接入了。

也就是这样：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/4_公众号_Yi昭.png)

想跨进程调用某个工具，通过这个协议通信就行。

不管是本地工具，直接跑那个进程，然后 stdio 通信。

还是远程工具，通过 http 连接远程服务进程。

这个协议叫什么呢？

是给 Model 扩展 Context 上下文，让它能做的更多，知道的更多的 Protocal 协议。

就叫 MCP 吧。

恭喜你，你发明了 MCP！

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/5_公众号_Yi昭.png)

MCP 最大的特点就是可以**跨进程调用工具**。

跨本地的进程调用，就是用 stdio。

跨远程的进程调用，就是用 http。

提到 MCP 都会提到这张图：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/6_公众号_Yi昭.png)

你的 ai agent 就是 MCP 客户端，可以通过 MCP 协议调用各种 MCP Server，实现跨进程的工具调用。

当然，在 langchain 里，它也是 tool ，只不过是 tool 的一种而已：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/7_公众号_Yi昭.png)

MCP 是由 AI 巨头 Anthropic 公司发起并开发，但是 2025 年 12 月交给了 Linux 基金会维护。

也就是说它现在是完全中立于任何一个模型的行业通用协议。

大概知道 MCP 是啥就行，我们自己来写个 MCP 服务就明白了。

继续在 tool-test 这个项目里写：

安装 mcp 的包：

    ppip install @modelcontextprotocol/sdk

从包名就可以看出来是中立于任何一家公司的。

创建 src/my-mcp-server.mjs

    import&nbsp;{ McpServer }&nbsp;from'@modelcontextprotocol/sdk/server/mcp.js';import&nbsp;{ StdioServerTransport }&nbsp;from'@modelcontextprotocol/sdk/server/stdio.js';import&nbsp;{ z }&nbsp;from'zod';// 数据库const&nbsp;database = {users: {&nbsp; &nbsp;&nbsp;'001': {&nbsp;id:&nbsp;'001',&nbsp;name:&nbsp;'张三',&nbsp;email:&nbsp;'zhangsan@example.com',&nbsp;role:&nbsp;'admin'&nbsp;},&nbsp; &nbsp;&nbsp;'002': {&nbsp;id:&nbsp;'002',&nbsp;name:&nbsp;'李四',&nbsp;email:&nbsp;'lisi@example.com',&nbsp;role:&nbsp;'user'&nbsp;},&nbsp; &nbsp;&nbsp;'003': {&nbsp;id:&nbsp;'003',&nbsp;name:&nbsp;'王五',&nbsp;email:&nbsp;'wangwu@example.com',&nbsp;role:&nbsp;'user'&nbsp;},&nbsp; }};const&nbsp;server =&nbsp;new&nbsp;McpServer({name:&nbsp;'my-mcp-server',version:&nbsp;'1.0.0',});// 注册工具：查询用户信息server.registerTool('query_user', {description:&nbsp;'查询数据库中的用户信息。输入用户 ID，返回该用户的详细信息（姓名、邮箱、角色）。',inputSchema: {&nbsp; &nbsp;&nbsp;userId: z.string().describe('用户 ID，例如: 001, 002, 003'),&nbsp; },},&nbsp;async&nbsp;({ userId }) =&gt; {const&nbsp;user = database.users[userId];if&nbsp;(!user) {&nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp; &nbsp; &nbsp;&nbsp;content: [&nbsp; &nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;'text',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;text:&nbsp;`用户 ID&nbsp;${userId}&nbsp;不存在。可用的 ID: 001, 002, 003`,&nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; ],&nbsp; &nbsp; };&nbsp; }return&nbsp;{&nbsp; &nbsp;&nbsp;content: [&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;'text',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;text:&nbsp;`用户信息：\n- ID:&nbsp;${user.id}\n- 姓名:&nbsp;${user.name}\n- 邮箱:&nbsp;${user.email}\n- 角色:&nbsp;${user.role}`,&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; ],&nbsp; };});server.registerResource('使用指南',&nbsp;'docs://guide', {description:&nbsp;'MCP Server 使用文档',mimeType:&nbsp;'text/plain',},&nbsp;async&nbsp;() =&gt; {return&nbsp;{&nbsp; &nbsp;&nbsp;contents: [&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;uri:&nbsp;'docs://guide',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;mimeType:&nbsp;'text/plain',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;text:&nbsp;`MCP Server 使用指南功能：提供用户查询等工具。使用：在 Cursor 等 MCP Client 中通过自然语言对话，Cursor 会自动调用相应工具。`,&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; ],&nbsp; };});const&nbsp;transport =&nbsp;new&nbsp;StdioServerTransport();await&nbsp;server.connect(transport); &nbsp; &nbsp;

代码很容易看懂：

- new McpServer 创建了 mcp server 实例
- server.registerTool 注册了一个工具，声明 name、description、schema
- server.registerResource 注册了一个资源，就是静态数据

和我们写 tool 的时候差不多，只不过这里分了 resource 和 tool，resouce 一般返回静态数据，tool 来做一些事情。

最后，可以提供 stdio 的本地进程的调用方式，也可以提供 http 的远程调用方式。

这里是 stdio 的传输方式（Transport）

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/8_公众号_Yi昭.png)

这样，我们的 MCP 服务就创建好了！

是不是很简单。

其实就是 tool，加上了协议而已。

我们在 cursor 里配置下这个 mcp server：

> 🎬 视频演示（原公众号视频）

配置好之后测试下：

> 🎬 视频演示（原公众号视频）

我特意换了个项目来测。

可以看到，确实检测到了这个 mcp 然后调用了！

这里 cursor 有个坑注意下：

> 🎬 视频演示（原公众号视频）

点一下 tool 是禁用，再点一下是启用。

但是 cursor 这个状态颜色区分不明显，没有调用 mcp 工具，可能你关掉了。

**这就是 mcp 的好处，写好之后可以插拔到任何地方当 tool 用。**

那 resource 呢？

它其实不是用来作为 tool 触发的，主要是你可以引用用来写 prompt 之类的。

比如这样：

> 🎬 视频演示（原公众号视频）

resource 主要是查询信息用的（read）， 而 tool 是执行功能用的（call）

当然，因为有了 mcp，除了 cursor，别的软件同样可以调用这个服务：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/9_公众号_Yi昭.png)

我们在 langchain 代码里调用下 mcp server：

用这个包：

    ppip install langchain_mcp-adapters

创建 src/langchain-mcp-test.mjs

    import&nbsp;'dotenv/config';import&nbsp;{ MultiServerMCPClient }&nbsp;from'langchain_mcp-adapters';import&nbsp;{ ChatOpenAI }&nbsp;from'langchain_openai';import&nbsp;chalk&nbsp;from'chalk';import&nbsp;{ HumanMessage, ToolMessage }&nbsp;from'langchain_core/messages';const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({&nbsp;&nbsp; &nbsp;&nbsp;modelName:&nbsp;"qwen-plus",&nbsp; &nbsp;&nbsp;apiKey: process.env.OPENAI_API_KEY,&nbsp; &nbsp;&nbsp;configuration: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; &nbsp; },});const&nbsp;mcpClient =&nbsp;new&nbsp;MultiServerMCPClient({&nbsp; &nbsp;&nbsp;mcpServers: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'my-mcp-server': {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;command:&nbsp;"node",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;args: [&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"/Users/guang/code/tool-test/src/my-mcp-server.mjs"&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ]&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; }});const&nbsp;tools =&nbsp;await&nbsp;mcpClient.getTools();const&nbsp;modelWithTools = model.bindTools(tools);asyncfunction&nbsp;runAgentWithTools(query, maxIterations =&nbsp;30)&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;messages = [&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;new&nbsp;HumanMessage(query)&nbsp; &nbsp; ];&nbsp; &nbsp;&nbsp;for&nbsp;(let&nbsp;i =&nbsp;0; i &lt; maxIterations; i++) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(chalk.bgGreen(`⏳ 正在等待 AI 思考...`));&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;response =&nbsp;await&nbsp;modelWithTools.invoke(messages);&nbsp; &nbsp; &nbsp; &nbsp; messages.push(response);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 检查是否有工具调用&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!response.tool_calls || response.tool_calls.length ===&nbsp;0) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(`\n✨ AI 最终回复:\n${response.content}\n`);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;response.content;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(chalk.bgBlue(`🔍 检测到&nbsp;${response.tool_calls.length}&nbsp;个工具调用`));&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(chalk.bgBlue(`🔍 工具调用:&nbsp;${response.tool_calls.map(t =&gt; t.name).join(', ')}`));&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 执行工具调用&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;for&nbsp;(const&nbsp;toolCall&nbsp;of&nbsp;response.tool_calls) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;foundTool = tools.find(t&nbsp;=&gt;&nbsp;t.name === toolCall.name);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(foundTool) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;toolResult =&nbsp;await&nbsp;foundTool.invoke(toolCall.args);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; messages.push(new&nbsp;ToolMessage({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content: toolResult,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;tool_call_id: toolCall.id,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }));&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;return&nbsp;messages[messages.length -&nbsp;1].content;}await&nbsp;runAgentWithTools("查一下用户 002 的信息");

我们用 langchain_mcp-adapters 创建了 mcp client

写法和 cursor 里配置一样：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/10_公众号_Yi昭.png)

就是用命令行启动这个进程，之后用 stdio 的方式做通信。

拿到 tools 之后绑定到模型。

模型调用返回 tool\_calls 消息需要自己调用 tool，调用完通过 ToolMessage 封装返回的消息，继续调用。

这个循环我们写过很多次了。

调用下试试：

> 🎬 视频演示（原公众号视频）

可以看到，你让大模型查询用户，它识别到了工具调用，然后调用了 mcp 的工具。

这里进程没退出，因为你跑了一个子进程作为 mcp server，需要把那个关掉才可以：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/11_公众号_Yi昭.png)

    await&nbsp;mcpClient.close();

> 🎬 视频演示（原公众号视频）

那 resource 怎么用呢？

那种静态信息可以放到 system message 里。

我们先查一下 resource：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/12_公众号_Yi昭.png)

    const&nbsp;res =&nbsp;await&nbsp;mcpClient.listResources();console.log(res);

> 🎬 视频演示（原公众号视频）

遍历依次读取 uri 内容

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/13_公众号_Yi昭.png)

    const&nbsp;res =&nbsp;await&nbsp;mcpClient.listResources();for&nbsp;(const&nbsp;[serverName, resources]&nbsp;of&nbsp;Object.entries(res)) {&nbsp; &nbsp;&nbsp;for&nbsp;(const&nbsp;resource&nbsp;of&nbsp;resources) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;content =&nbsp;await&nbsp;mcpClient.readResource(serverName, resource.uri);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(content);&nbsp; &nbsp; }}

> 🎬 视频演示（原公众号视频）

然后只要把它放到 system message 里作为上下文就好了：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/14_公众号_Yi昭.png)

    const&nbsp;res =&nbsp;await&nbsp;mcpClient.listResources();let&nbsp;resourceContent =&nbsp;'';for&nbsp;(const&nbsp;[serverName, resources]&nbsp;of&nbsp;Object.entries(res)) {&nbsp; &nbsp;&nbsp;for&nbsp;(const&nbsp;resource&nbsp;of&nbsp;resources) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;content =&nbsp;await&nbsp;mcpClient.readResource(serverName, resource.uri);&nbsp; &nbsp; &nbsp; &nbsp; resourceContent += content[0].text;&nbsp; &nbsp; }}

    const&nbsp;messages = [&nbsp; &nbsp;&nbsp;new&nbsp;SystemMessage(resourceContent),&nbsp; &nbsp;&nbsp;new&nbsp;HumanMessage(query)];

调用下：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/15_公众号_Yi昭.png)

    await&nbsp;runAgentWithTools("MCP Server 的使用指南是什么");

跑一下：

> 🎬 视频演示（原公众号视频）

现在，大模型就知道这个 resource 的信息，可以用来回答问题了。

resource 可以用在 system message 里，也可以用在 human message 里，总之，是作为信息引用的。

我们主要还是用 mcp 的 tools。

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/16_公众号_Yi昭.png)

这样，我们就写了一个 mcp server，并分别在 cursor、langchain 里用了这个 mcp server。

mcp 本质上还是 tool，和之前的 tool 的区别只不过是可以跨进程调用：

![image](../IMG/2025-12-24_MCP：可跨进程调用的Tool/17_公众号_Yi昭.png)

当你不需要跨进程用的时候，还是之前那样写更好，还少了进程通信的成本。

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code/tool-test

## 总结

这节我们学了 MCP，它是可跨进程调用的 Tool。

可以是本地进程，用 stdio 进程通信。

可以是远程进程，用 http 通信。

在 langchain 里用 langchain_mcp-adapters 封装成 tools 来用，其实和其他 tool 没区别。

跨进程就意味着不限语言，开发好之后，可以被任意 mcp client 调用，比如 cursor、langchain 等。

除了自己写 mcp server，现在也有很多现成的 mcp server 可以直接用，下节我们来用一下。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2025-12-24 08:10:00 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247483924&idx=1&sn=f8329779692e50d669b7386eb613a8b4&chksm=f147744560593c4a0f98c8f3117428c21bb3de89ecfec078bcc3a8121431e20a959f964c3e73&scene=27#wechat_redirect
