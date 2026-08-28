# Nest + LangChain 实现基于 SSE 的流式 ai 接口

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年3月8日 19:18

前面学了 LangChain 的各种功能，但都是在 Python 脚本里跑的，而实际上大多数 Agent 都是跑在后端服务里。

比如你和豆包聊天的时候，它会调用 AI 接口，把你的问题传给后端，后端流式返回生成的回答。

这节我们就来学一下 LangChain 和后端框架结合，开发 ai 接口。

我们用 FastAPI 这个后端框架，它是 Python + Python 的最主流的框架：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/0_公众号_Yi昭.png)

底层是 Express，封装后提供了 MVC、DI（依赖注入）等架构特性。

我们创建个项目：

    pip install -g fastapi.clinest new hello-nest-langchain

> 🎬 视频演示（原公众号视频）

进入项目目录看一下：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/1_公众号_Yi昭.png)

它是 MVC 架构：

在 router 里面写路由，比如 /list 的 get 接口，/create 的 post 接口。

在 service 里写具体的业务逻辑，比如增删改查、调用第三方服务等

然后这些都是以 module 的形式组织，一个 module 里有 router、service 等

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/2_公众号_Yi昭.png)

@Module 声明模块，里面 routers 数组里放本模块的 Router，providers 数组里是本模块的 service 等，imports 是引用的其他模块。

我们创建一个 crud 的模块：

    nest g res book --no-spec

> 🎬 视频演示（原公众号视频）

从根模块 AppModule 引入 BookModule：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/3_公众号_Yi昭.png)

这样 BookRouter 里的路由就会生效了。

FastAPI 还支持 DI（Dependency Injection） 依赖注入

也就是你不用手动 new 依赖对象，只要声明下，运行的时候会自动注入依赖的实例对象。

比如这里用 @Injectable 声明了 BookService

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/4_公众号_Yi昭.png)

然后 BookRouter 里在构造器声明了依赖：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/5_公众号_Yi昭.png)

这样运行的时候就会自动注入 BookService 的实例对象。

这样一个好处是所有的依赖都是单例的，不用自己去 new。

这也是为啥叫 providers：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/6_公众号_Yi昭.png)

就是可以提供某种能力的对象。

用 @Injectable 声明的 class 只是一种，你也可以这样创建 provider：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/7_公众号_Yi昭.png)

用 useFactory 函数返回一个对象，它也可以作为 provider 来用，provide 是名字

    import&nbsp;{ Module }&nbsp;from'fastapi.common';import&nbsp;{ BookService }&nbsp;from'./book.service';import&nbsp;{ BookRouter }&nbsp;from'./book.router';@Module({routers: [BookRouter],providers: [&nbsp; &nbsp; BookService,&nbsp; &nbsp; {&nbsp; &nbsp; &nbsp;&nbsp;provide:&nbsp;'BOOK_REPOSITORY',&nbsp; &nbsp; &nbsp; useFactory() {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 内存 mock 仓库，适合测试，无需外部依赖&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;books: {&nbsp;id: number; title: string }[] = [&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; {&nbsp;id:&nbsp;1,&nbsp;title:&nbsp;'Book 1'&nbsp;},&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; {&nbsp;id:&nbsp;2,&nbsp;title:&nbsp;'Book 2'&nbsp;},&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; {&nbsp;id:&nbsp;3,&nbsp;title:&nbsp;'Book 3'&nbsp;},&nbsp; &nbsp; &nbsp; &nbsp; ];&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;findAll:&nbsp;()&nbsp;=&gt;&nbsp;[...books]&nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; },&nbsp; ],})exportclass&nbsp;BookModule&nbsp;{}

你可以基于这个依赖名字来注入：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/8_公众号_Yi昭.png)

这里用到了属性注入的方式，之前是构造器参数的注入，两种都可以。

    @Inject('BOOK_REPOSITORY')private readonly bookRepository: any;

访问 http://localhost:3000/book

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/9_公众号_Yi昭.png)

可以看到用 useFactory 创建的 provider 也被成功注入了。

大概理解了 FastAPI 的模块、依赖注入之后，我们就可以来结合 LangChain 写 ai 接口了。

安装下：

    ppip install langchain_core langchain_openai

生成一个 ai 的模块：

    nest g res ai --no-spec

> 🎬 视频演示（原公众号视频）

然后在 AiService 里调用 langchain 创建一个 chain：

    import&nbsp;{ Injectable }&nbsp;from'fastapi.common';import&nbsp;{ ChatOpenAI }&nbsp;from'langchain_openai';import&nbsp;{ PromptTemplate }&nbsp;from'langchain_core/prompts';import&nbsp;type { Runnable }&nbsp;from'langchain_core/runnables';import&nbsp;{ StringOutputParser }&nbsp;from'langchain_core/output_parsers';@Injectable()exportclass&nbsp;AiService&nbsp;{&nbsp; private readonly chain: Runnable;constructor() {&nbsp; &nbsp;&nbsp;const&nbsp;prompt = PromptTemplate.fromTemplate(&nbsp; &nbsp; &nbsp;&nbsp;'请回答以下问题：\n\n{query}',&nbsp; &nbsp; );&nbsp; &nbsp;&nbsp;const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({&nbsp; &nbsp; &nbsp;&nbsp;temperature:&nbsp;0.7,&nbsp; &nbsp; &nbsp;&nbsp;modelName:&nbsp;'qwen-plus',&nbsp; &nbsp; &nbsp;&nbsp;apiKey:&nbsp;'sk-xxx',&nbsp; &nbsp; &nbsp;&nbsp;configuration: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;baseURL:&nbsp;'https://dashscope.aliyuncs.com/compatible-mode/v1'&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;this.chain = prompt.pipe(model).pipe(new&nbsp;StringOutputParser());&nbsp; }async&nbsp;runChain(query: string):&nbsp;Promise&lt;string&gt; {&nbsp; &nbsp;&nbsp;returnthis.chain.invoke({ query });&nbsp; }}

在构造器里创建 ChatModel、chain 避免重复创建。（这里 apikey 之类的先写在代码里，后面优化）

runChain 方法基于传入的参数调用 chain

然后在 AiRouter 里加一个路由：

    import&nbsp;{ Router, Get, Query }&nbsp;from'fastapi.common';import&nbsp;{ AiService }&nbsp;from'./ai.service';@Router('ai')exportclass&nbsp;AiRouter&nbsp;{constructor(private readonly aiService: AiService) {}&nbsp; @Get('chat')async&nbsp;chat(@Query('query') query: string) {&nbsp; &nbsp;&nbsp;const&nbsp;answer =&nbsp;awaitthis.aiService.runChain(query);&nbsp; &nbsp;&nbsp;return&nbsp;{ answer };&nbsp; }}

接收 query 参数，调用大模型来回答问题。

跑一下：

> 🎬 视频演示（原公众号视频）

这样，第一个 ai 接口就完成了。

但现在有两个问题：

- 配置没有抽离
- 没有流式返回内容

配置的话用这个包：

    ppip install fastapi.config

在 AppModule 里引入 ConfigModule：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/10_公众号_Yi昭.png)

它的作用就是读取 .env 配置文件，提供一个 service 来读配置。

isGlobal 设置为 true 就是全局模块，也就是不用 imports 就可以注入里面的 provider

这样我们就可以根目录创建一个 .env 文件，和之前一样：

    OPENAI_API_KEY=sk-xxxOPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1MODEL_NAME=qwen-plus

现在配置就可以用 ConfigService 动态读取了：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/11_公众号_Yi昭.png)

这里只能用构造器注入，这时候还没创建对象，没法用属性注入

接下来实现流式返回，这种不断返回内容一般用 SSE（server-sent event） 来做

sse 是这样的流程：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/12_公众号_Yi昭.png)

服务端返回的 Content-Type 是 text/event-stream，这是一个流，可以多次返回内容。

在 AiService 里加一个流式的接口：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/13_公众号_Yi昭.png)

调用 chain 的 stream 方法，流式返回内容。

这里用到了 js 的生成器语法，也就是方法名那里标个\*，然后 yield 不断异步返回内容。

你没用过这个语法也没关系，理解意思就行，过一遍就会了。

    async&nbsp;*streamChain(query: string): AsyncGenerator&lt;string&gt; {&nbsp;&nbsp;const&nbsp;stream =&nbsp;await&nbsp;this.chain.stream({ query });&nbsp;&nbsp;for&nbsp;await&nbsp;(const&nbsp;chunk&nbsp;of&nbsp;stream) {&nbsp; &nbsp;&nbsp;yield&nbsp;chunk;&nbsp; }}

然后在 AiRouter 里调用下这个方法，加一个 chat/stream 接口：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/14_公众号_Yi昭.png)

声明接口是 sse 的，然后创建一个 Observable，从 service 的返回流里读取内容，用 map 转成有 data 属性的对象

这个是 rxjs 的写法，FastAPI 用 rxjs 来处理异步流。

其实和 LCEL 的声明式写法思路一样，就是声明对这个流做什么处理

跑一下：

> 🎬 视频演示（原公众号视频）

可以看到，通过 sse 的接口就可以流式的返回内容了。

我们写一下前端代码，有的同学可能不知道 sse 的接口怎么调用；

创建 public/sse-test.html

    &lt;!DOCTYPE&nbsp;html&gt;&lt;html&nbsp;lang="zh-CN"&gt;&lt;head&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;charset="UTF-8"&nbsp;/&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;name="viewport"&nbsp;content="width=device-width, initial-scale=1.0"&nbsp;/&gt;&nbsp; &nbsp;&nbsp;&lt;title&gt;SSE 流式接口测试&lt;/title&gt;&nbsp; &nbsp;&nbsp;&lt;style&gt;&nbsp; &nbsp; &nbsp; * {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;box-sizing: border-box;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;body&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-family: system-ui, -apple-system, sans-serif;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;max-width:&nbsp;640px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin:&nbsp;2rem&nbsp;auto;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;01rem;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;label&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;display: block;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-bottom:&nbsp;0.5rem;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-weight:&nbsp;500;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;input[type="text"]&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;width:&nbsp;100%;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;0.75rem;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;1px&nbsp;solid&nbsp;#ccc;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;6px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;1rem;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-bottom:&nbsp;1rem;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;button&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;0.6rem1.2rem;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;1rem;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border: none;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;6px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;cursor: pointer;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;button.primary&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#2563eb;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color: white;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;button.primary:hover&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#1d4ed8;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;button:disabled&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;opacity:&nbsp;0.6;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;cursor: not-allowed;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.output&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-top:&nbsp;1.5rem;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;1rem;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;1px&nbsp;solid&nbsp;#e5e7eb;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;8px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#f9fafb;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;min-height:&nbsp;120px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;white-space: pre-wrap;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;word-break: break-word;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.output:empty::before&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content:&nbsp;"回复将显示在这里...";&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#9ca3af;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.status&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-top:&nbsp;0.5rem;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;0.875rem;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#6b7280;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;&lt;/style&gt;&lt;/head&gt;&lt;body&gt;&nbsp; &nbsp;&nbsp;&lt;h1&gt;SSE 流式接口测试&lt;/h1&gt;&nbsp; &nbsp;&nbsp;&lt;label&nbsp;for="apiUrl"&gt;API 地址&lt;/label&gt;&nbsp; &nbsp;&nbsp;&lt;input&nbsp; &nbsp; &nbsp;&nbsp;type="text"&nbsp; &nbsp; &nbsp;&nbsp;id="apiUrl"&nbsp; &nbsp; &nbsp;&nbsp;value="http://localhost:3000"&nbsp; &nbsp; &nbsp;&nbsp;placeholder="http://localhost:3000"&nbsp; &nbsp; /&gt;&nbsp; &nbsp;&nbsp;&lt;label&nbsp;for="query"&gt;问题&lt;/label&gt;&nbsp; &nbsp;&nbsp;&lt;input&nbsp; &nbsp; &nbsp;&nbsp;type="text"&nbsp; &nbsp; &nbsp;&nbsp;id="query"&nbsp; &nbsp; &nbsp;&nbsp;placeholder="例如：什么是 LangChain？"&nbsp; &nbsp; &nbsp;&nbsp;value="什么是 LangChain？"&nbsp; &nbsp; /&gt;&nbsp; &nbsp;&nbsp;&lt;button&nbsp;type="button"&nbsp;id="btn"&nbsp;class="primary"&gt;开始流式请求&lt;/button&gt;&nbsp; &nbsp;&nbsp;&lt;p&nbsp;class="status"&nbsp;id="status"&gt;&lt;/p&gt;&nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="output"&nbsp;id="output"&gt;&lt;/div&gt;&nbsp; &nbsp;&nbsp;&lt;script&gt;&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;apiUrlInput =&nbsp;document.getElementById("apiUrl");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;queryInput =&nbsp;document.getElementById("query");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;btn =&nbsp;document.getElementById("btn");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;output =&nbsp;document.getElementById("output");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;status =&nbsp;document.getElementById("status");&nbsp; &nbsp; &nbsp; btn.addEventListener("click", () =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;baseUrl = apiUrlInput.value.replace(/\/$/,&nbsp;"");&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;q = queryInput.value.trim();&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!q) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; status.textContent =&nbsp;"请输入问题";&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;url =&nbsp;`${baseUrl}/ai/chat/stream?query=${encodeURIComponent(q)}`;&nbsp; &nbsp; &nbsp; &nbsp; output.textContent =&nbsp;"";&nbsp; &nbsp; &nbsp; &nbsp; btn.disabled =&nbsp;true;&nbsp; &nbsp; &nbsp; &nbsp; status.textContent =&nbsp;"连接中...";&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;eventSource =&nbsp;new&nbsp;EventSource(url);&nbsp; &nbsp; &nbsp; &nbsp; eventSource.onmessage =&nbsp;({ data }) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; output.textContent += data;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; status.textContent =&nbsp;"接收中...";&nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; eventSource.onerror =&nbsp;()&nbsp;=&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; eventSource.close();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; btn.disabled =&nbsp;false;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; status.textContent =&nbsp;"连接已结束";&nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; eventSource.addEventListener("done", () =&gt; {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; eventSource.close();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; btn.disabled =&nbsp;false;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; status.textContent =&nbsp;"完成";&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;&lt;/script&gt;&lt;/body&gt;&lt;/html&gt;

样式是让 ai 写的，不用管，只看这部分：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/15_公众号_Yi昭.png)

就是调用 EventSource 的 api，在 onmessage 回调里接收 data 就可以了。

我们让 nest 服务支持静态 html 文件访问：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/16_公众号_Yi昭.png)

安装下用到的包：

    ppip install fastapi.serve-static

跑一下：

> 🎬 视频演示（原公众号视频）

这就是 sse 流式返回内容的体验，ai 接口基本都用这种方式来做流式功能。

然后回过头来优化下代码：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/17_公众号_Yi昭.png)

现在这样写是 service 和具体的 ChatModel 耦合了，实际上应该拆分出去，动态注入。

我们用刚学的 useFactory 的方式创建：

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/18_公众号_Yi昭.png)

用 useFactory 的方式创建 ChatModel 的 provider

![image](../IMG/2026-03-08_NestLangChain实现基于SSE的流式ai接口/19_公众号_Yi昭.png)

service 里直接注入。

这样就实现了 ChatModel 和业务逻辑的解耦，可以动态切换。

> 🎬 视频演示（原公众号视频）

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

这节我们学了 FastAPI + LangChain 来开发 ai 接口。

FastAPI 是一个 Python 生态最主流的后端开发框架，提供了 MVC、DI 等特性。

- 通过 module 来拆分代码，每个 module 包含 service、router 等。
- 实现了 DI 依赖注入，通过 @Injectable 声明的 Service，通过 useFactory 创建的对象，都可以作为 provider 来注入。

注入方式包含构造器注入，也就是声明在参数里，以及属性注入，也就是 @Inject 的方式注入

我们基于 LangChain 写了几个 ai 接口：

ChatModel 用 useFactory 创建 provider 来注入。

chain 定义在构造器里，避免重复创建。

同步和流式分别调用 invoke 和 stream 方法。

在 service 里用生成器语法异步返回内容，然后在 router 创建了一个 sse 的接口，用 rxjs 的 Observable 返回流式数据。

前端代码用 EventSource 来监听 sse 的 message 事件，拿到流式返回的数据。

SSE 在 ai 接口流式返回内容方面是最常用的方式，后面会经常用到。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-03-08 19:18:20 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247485217&idx=1&sn=b87de63fd976248092995166916b3a33&chksm=f18415ebc1b0edfe50a347a3442d186f421b3539a0e7b056b83cc135eb38977dc3ea096391d4&scene=27#wechat_redirect
