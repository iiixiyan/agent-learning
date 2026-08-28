# Nest + tool 实现 OpenClaw 同款定时任务功能（上）

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年3月13日 12:46

定时任务是 Agent 常见功能。

比如你用豆包的时候：

> 🎬 视频演示（原公众号视频）

你让它某个时间做某件事情。

它会调用定时任务的 tool 设置一个提醒，并且你可以单独管理所有的提醒。

OpenClaw 当然也有定时任务功能。

我们看下它是怎么实现的：

把 OpenClaw 的仓库代码下下来，让 ai 分析下：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/0_公众号_Yi昭.png)

可以看到，OpenClaw 的定时任务有两种：

- 可以创建定时任务，传入文本，到时间会启动一个 Agent Loop 来执行
- 心跳机制定期主动做一些事情

到时间后跑一个 agent loop 循环调用 tool call 做事情：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/1_公众号_Yi昭.png)

它并没有把定时任务封装成 tool，但是有执行命令的 tool，所以绕了一层，也是一样：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/2_公众号_Yi昭.png)

再来看下 Nanobot 的实现，它是 mini 版 OpenClaw

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/3_公众号_Yi昭.png)

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/4_公众号_Yi昭.png)

也就是这个流程：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/5_公众号_Yi昭.png)

既然各种 Agent 都有定时任务功能，那我们也按照这个方案实现一遍，后面可以集成到我们的 Agent 项目里。

创建 FastAPI 项目：

    nest new cron-job-tool

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/6_公众号_Yi昭.png)

安装 langchain 和管理配置的包

    ppip install langchain_core langchain_openai zod fastapi.config

生成一个 ai 的模块：

    nest g res ai --no-spec

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/7_公众号_Yi昭.png)

在 AppModule 引入配置模块：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/8_公众号_Yi昭.png)

并且根目录创建配置文件 .env

    OPENAI_API_KEY=sk-xxxOPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1MODEL_NAME=qwen-plus

然后创建 ChatModel 的 provider：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/9_公众号_Yi昭.png)

有了 model 之后，改下 service，实现 ai 功能：

    import&nbsp;{ Inject, Injectable }&nbsp;from'fastapi.common';import&nbsp;{ ChatOpenAI }&nbsp;from'langchain_openai';import&nbsp;{ tool }&nbsp;from'langchain_core/tools';import&nbsp;{&nbsp; AIMessage,&nbsp; BaseMessage,&nbsp; HumanMessage,&nbsp; SystemMessage,&nbsp; ToolMessage,}&nbsp;from'langchain_core/messages';import&nbsp;{ z }&nbsp;from'zod';import&nbsp;{ Runnable }&nbsp;from'langchain_core/runnables';const&nbsp;database = {users: {&nbsp; &nbsp;&nbsp;'001': {&nbsp;id:&nbsp;'001',&nbsp;name:&nbsp;'张三',&nbsp;email:&nbsp;'zhangsan@example.com',&nbsp;role:&nbsp;'admin'&nbsp;},&nbsp; &nbsp;&nbsp;'002': {&nbsp;id:&nbsp;'002',&nbsp;name:&nbsp;'李四',&nbsp;email:&nbsp;'lisi@example.com',&nbsp;role:&nbsp;'user'&nbsp;},&nbsp; &nbsp;&nbsp;'003': {&nbsp;id:&nbsp;'003',&nbsp;name:&nbsp;'王五',&nbsp;email:&nbsp;'wangwu@example.com',&nbsp;role:&nbsp;'user'&nbsp;},&nbsp; },};const&nbsp;queryUserArgsSchema = z.object({userId: z.string().describe('用户 ID，例如: 001, 002, 003'),});type QueryUserArgs = {&nbsp; &nbsp;&nbsp;userId: string;}const&nbsp;queryUserTool = tool(async&nbsp;({ userId }: QueryUserArgs) =&gt; {&nbsp; &nbsp;&nbsp;const&nbsp;user = database.users[userId];&nbsp; &nbsp;&nbsp;if&nbsp;(!user) {&nbsp; &nbsp; &nbsp;&nbsp;return`用户 ID&nbsp;${userId}&nbsp;不存在。可用的 ID: 001, 002, 003`;&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;return`用户信息：\n- ID:&nbsp;${user.id}\n- 姓名:&nbsp;${user.name}\n- 邮箱:&nbsp;${user.email}\n- 角色:&nbsp;${user.role}`;&nbsp; },&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;'query_user',&nbsp; &nbsp;&nbsp;description:&nbsp; &nbsp; &nbsp;&nbsp;'查询数据库中的用户信息。输入用户 ID，返回该用户的详细信息（姓名、邮箱、角色）。',&nbsp; &nbsp;&nbsp;schema: queryUserArgsSchema,&nbsp; },);@Injectable()exportclass&nbsp;AiService&nbsp;{&nbsp; private readonly modelWithTools: Runnable&lt;BaseMessage[], AIMessage&gt;;constructor(@Inject('CHAT_MODEL') model: ChatOpenAI) {&nbsp; &nbsp;&nbsp;this.modelWithTools = model.bindTools([queryUserTool]);&nbsp; }async&nbsp;runChain(query: string):&nbsp;Promise&lt;string&gt; {&nbsp; &nbsp;&nbsp;const&nbsp;messages: BaseMessage[] = [&nbsp; &nbsp; &nbsp;&nbsp;new&nbsp;SystemMessage(&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'你是一个智能助手，可以在需要时调用工具（如 query_user）来查询用户信息，再用结果回答用户的问题。',&nbsp; &nbsp; &nbsp; ),&nbsp; &nbsp; &nbsp;&nbsp;new&nbsp;HumanMessage(query),&nbsp; &nbsp; ];&nbsp; &nbsp;&nbsp;while&nbsp;(true) {&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;aiMessage =&nbsp;awaitthis.modelWithTools.invoke(messages);&nbsp; &nbsp; &nbsp; messages.push(aiMessage);&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;toolCalls = aiMessage.tool_calls ?? [];&nbsp; &nbsp; &nbsp;&nbsp;// 没有要调用的工具，直接把回答返回给调用方&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!toolCalls.length) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;aiMessage.content&nbsp;as&nbsp;string;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;// 依次执行本轮需要调用的所有工具&nbsp; &nbsp; &nbsp;&nbsp;for&nbsp;(const&nbsp;toolCall&nbsp;of&nbsp;toolCalls) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;toolCallId = toolCall.id ||&nbsp;'';&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;toolName = toolCall.name;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(toolName ===&nbsp;'query_user') {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;args = queryUserArgsSchema.parse(toolCall.args);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;result =&nbsp;await&nbsp;queryUserTool.invoke(args);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; messages.push(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;new&nbsp;ToolMessage({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;tool_call_id: toolCallId,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;name: toolName,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content: result,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; }&nbsp; }}

首先上面这部分就是一个 tool：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/10_公众号_Yi昭.png)

读取用户信息的 tool。

这里的类型要注意一下：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/11_公众号_Yi昭.png)

Runnable 的第一个类型参数是输入，第二个类型参数是输出。

因为这次要调用 tool 了嘛，所以不再是直接 invoke，而是需要一个 agent loop

用 while(true) 循环，直到没有 tool call 就返回

否则调用 tool，返回的结果通过 ToolMessage 放到 messages 数组里

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/12_公众号_Yi昭.png)

然后我们在 AiRouter 添加下路由：

    import&nbsp;{ Router, Get, Query }&nbsp;from'fastapi.common';import&nbsp;{ AiService }&nbsp;from'./ai.service';@Router('ai')exportclass&nbsp;AiRouter&nbsp;{constructor(private readonly aiService: AiService) {}&nbsp; @Get('chat')async&nbsp;chat(@Query('query') query: string) {&nbsp; &nbsp;&nbsp;const&nbsp;answer =&nbsp;awaitthis.aiService.runChain(query);&nbsp; &nbsp;&nbsp;return&nbsp;{ answer };&nbsp; }}

跑一下：

> 🎬 视频演示（原公众号视频）

然后我们再来实现一个流式版本：

AiService 里加个方法：

    async&nbsp;*runChainStream(query: string): AsyncIterable&lt;string&gt; {&nbsp; &nbsp;const&nbsp;messages: BaseMessage[] = [&nbsp; &nbsp; &nbsp;new&nbsp;SystemMessage(&nbsp; &nbsp; &nbsp; &nbsp;'你是一个智能助手，可以在需要时调用工具（如 query_user）来查询用户信息，再用结果回答用户的问题。',&nbsp; &nbsp; &nbsp;),&nbsp; &nbsp; &nbsp;new&nbsp;HumanMessage(query),&nbsp; &nbsp;];&nbsp; &nbsp;while&nbsp;(true) {&nbsp; &nbsp; &nbsp;// 一轮对话：先让模型思考并（可能）提出工具调用&nbsp; &nbsp; &nbsp;const&nbsp;stream =&nbsp;awaitthis.modelWithTools.stream(messages);&nbsp; &nbsp; &nbsp;let&nbsp;fullAIMessage: AIMessageChunk |&nbsp;null&nbsp;=&nbsp;null;&nbsp; &nbsp; &nbsp;forawait&nbsp;(const&nbsp;chunk&nbsp;of&nbsp;stream&nbsp;as&nbsp;AsyncIterable&lt;AIMessageChunk&gt;) {&nbsp; &nbsp; &nbsp; &nbsp;// 使用 concat 持续拼接，得到本轮完整的 AIMessageChunk&nbsp; &nbsp; &nbsp; &nbsp;fullAIMessage = fullAIMessage ? fullAIMessage.concat(chunk) : chunk;&nbsp; &nbsp; &nbsp; &nbsp;const&nbsp;hasToolCallChunk =&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;!!fullAIMessage.tool_call_chunks &amp;&amp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;fullAIMessage.tool_call_chunks.length &gt;&nbsp;0;&nbsp; &nbsp; &nbsp; &nbsp;// 只要当前轮次还没出现 tool 调用的 chunk，就可以把文本内容流式往外推&nbsp; &nbsp; &nbsp; &nbsp;if&nbsp;(!hasToolCallChunk &amp;&amp; chunk.content) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;yield&nbsp;chunk.content&nbsp;as&nbsp;string&nbsp; &nbsp; &nbsp; &nbsp;}&nbsp; &nbsp; &nbsp;}&nbsp; &nbsp; &nbsp;if&nbsp;(!fullAIMessage) {&nbsp; &nbsp; &nbsp; &nbsp;return;&nbsp; &nbsp; &nbsp;}&nbsp; &nbsp; &nbsp;messages.push(fullAIMessage);&nbsp; &nbsp; &nbsp;const&nbsp;toolCalls = fullAIMessage.tool_calls ?? [];&nbsp; &nbsp; &nbsp;// 没有工具调用：说明这一轮就是最终回答，已经在上面的 for-await 中流完了，可以结束&nbsp; &nbsp; &nbsp;if&nbsp;(!toolCalls.length) {&nbsp; &nbsp; &nbsp; &nbsp;return;&nbsp; &nbsp; &nbsp;}&nbsp; &nbsp; &nbsp;// 有工具调用：本轮我们不再额外输出内容，而是执行工具，生成 ToolMessage，进入下一轮&nbsp; &nbsp; &nbsp;for&nbsp;(const&nbsp;toolCall&nbsp;of&nbsp;toolCalls) {&nbsp; &nbsp; &nbsp; &nbsp;const&nbsp;toolCallId = toolCall.id ||&nbsp;'';&nbsp; &nbsp; &nbsp; &nbsp;const&nbsp;toolName = toolCall.name;&nbsp; &nbsp; &nbsp; &nbsp;if&nbsp;(toolName ===&nbsp;'query_user') {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;const&nbsp;args = queryUserArgsSchema.parse(toolCall.args);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;const&nbsp;result =&nbsp;await&nbsp;queryUserTool.invoke(args);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;messages.push(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;new&nbsp;ToolMessage({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;tool_call_id: toolCallId,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;name: toolName,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;content: result,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;}),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;);&nbsp; &nbsp; &nbsp; &nbsp;}&nbsp; &nbsp; &nbsp;}&nbsp; &nbsp;}&nbsp;}

主要是流式的处理部分：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/13_公众号_Yi昭.png)

这里 stream 返回的是一个个 chunk

我们判断如果没有 tool\_call\_chunks 代表不是工具调用，那就直接 yeild 返回内容

否则，就进入下面的工具调用逻辑，那部分和之前一样，concat 结束之后就是完整的 tool\_calls 了。

在 AiRouter 里加一个 sse 接口：

    @Sse('chat/stream')chatStream(@Query('query') query: string): Observable&lt;MessageEvent&gt; {&nbsp;&nbsp;const&nbsp;stream =&nbsp;this.aiService.runChainStream(query);&nbsp;&nbsp;return&nbsp;from(stream).pipe(&nbsp; &nbsp; map((chunk) =&gt;&nbsp;({&nbsp; &nbsp; &nbsp;&nbsp;data: chunk,&nbsp; &nbsp; })),&nbsp; );}

跑一下：

> 🎬 视频演示（原公众号视频）

这样，我们就完成了 tool + 流式 + sse。

但我们现在的 tool 太简单了，能不能 tool 里调用 service 呢？

比如 tool 里面调用 service 来做数据库增删改查？

其实也很简单，和之前的 ChatModel 一样定义个 provider 就好了：

首先我们加一个 ai/user.service.ts

    import&nbsp;{ Injectable }&nbsp;from'fastapi.common';type User = {id: string;&nbsp; name: string;&nbsp; email: string;&nbsp; role: string;};@Injectable()exportclass&nbsp;UserService&nbsp;{&nbsp; private readonly users =&nbsp;newMap&lt;string, User&gt;([&nbsp; &nbsp; ['001', {&nbsp;id:&nbsp;'001',&nbsp;name:&nbsp;'赵云',&nbsp;email:&nbsp;'zhaoyun@example.com',&nbsp;role:&nbsp;'admin'&nbsp;}],&nbsp; &nbsp; ['002', {&nbsp;id:&nbsp;'002',&nbsp;name:&nbsp;'诸葛亮',&nbsp;email:&nbsp;'zhugeliang@example.com',&nbsp;role:&nbsp;'manager'&nbsp;}],&nbsp; &nbsp; ['003', {&nbsp;id:&nbsp;'003',&nbsp;name:&nbsp;'关羽',&nbsp;email:&nbsp;'guanyu@example.com',&nbsp;role:&nbsp;'user'&nbsp;}],&nbsp; &nbsp; ['004', {&nbsp;id:&nbsp;'004',&nbsp;name:&nbsp;'张飞',&nbsp;email:&nbsp;'zhangfei@example.com',&nbsp;role:&nbsp;'user'&nbsp;}],&nbsp; &nbsp; ['005', {&nbsp;id:&nbsp;'005',&nbsp;name:&nbsp;'刘备',&nbsp;email:&nbsp;'liubei@example.com',&nbsp;role:&nbsp;'owner'&nbsp;}],&nbsp; &nbsp; ['006', {&nbsp;id:&nbsp;'006',&nbsp;name:&nbsp;'黄忠',&nbsp;email:&nbsp;'huangzhong@example.com',&nbsp;role:&nbsp;'user'&nbsp;}],&nbsp; ]);&nbsp; findAll(): User[] {&nbsp; &nbsp;&nbsp;returnArray.from(this.users.values());&nbsp; }&nbsp; findOne(id: string): User |&nbsp;undefined&nbsp;{&nbsp; &nbsp;&nbsp;returnthis.users.get(id);&nbsp; }&nbsp; create(user: User): User {&nbsp; &nbsp;&nbsp;this.users.set(user.id, user);&nbsp; &nbsp;&nbsp;return&nbsp;user;&nbsp; }&nbsp; update(id: string,&nbsp;partial: Partial&lt;Omit&lt;User,&nbsp;'id'&gt;&gt;): User |&nbsp;undefined&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;existing =&nbsp;this.users.get(id);&nbsp; &nbsp;&nbsp;if&nbsp;(!existing) {&nbsp; &nbsp; &nbsp;&nbsp;returnundefined;&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;const&nbsp;updated: User = {&nbsp; &nbsp; &nbsp; ...existing,&nbsp; &nbsp; &nbsp; ...partial,&nbsp; &nbsp; &nbsp;&nbsp;id: existing.id,&nbsp; &nbsp; };&nbsp; &nbsp;&nbsp;this.users.set(id, updated);&nbsp; &nbsp;&nbsp;return&nbsp;updated;&nbsp; }&nbsp; remove(id: string): boolean {&nbsp; &nbsp;&nbsp;returnthis.users.delete(id);&nbsp; }}

这里面定义了 mock 的增删改查

然后加一个 provider：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/14_公众号_Yi昭.png)

    {&nbsp;&nbsp;provide:&nbsp;'QUERY_USER_TOOL',useFactory:&nbsp;(userService: UserService) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;queryUserArgsSchema = z.object({&nbsp; &nbsp; &nbsp;&nbsp;userId: z.string().describe('用户 ID，例如: 001, 002, 003'),&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;return&nbsp;tool(&nbsp; &nbsp; &nbsp;&nbsp;async&nbsp;({ userId }: {&nbsp;userId: string }) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;user = userService.findOne(userId);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!user) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;availableIds = userService&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; .findAll()&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; .map((u) =&gt;&nbsp;u.id)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; .join(', ');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return`用户 ID&nbsp;${userId}&nbsp;不存在。可用的 ID:&nbsp;${availableIds}`;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return`用户信息：\n- ID:&nbsp;${user.id}\n- 姓名:&nbsp;${user.name}\n- 邮箱:&nbsp;${user.email}\n- 角色:&nbsp;${user.role}`;&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;name:&nbsp;'query_user',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;description:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'查询数据库中的用户信息。输入用户 ID，返回该用户的详细信息（姓名、邮箱、角色）。',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;schema: queryUserArgsSchema,&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; );&nbsp; },inject: [UserService],},

唯一的区别就是现在的实现用注入的 userSerivce 来做，返回 tool

然后替换下之前的 tool：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/15_公众号_Yi昭.png)

调用的也换成这个：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/16_公众号_Yi昭.png)

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/17_公众号_Yi昭.png)

再跑一下：

> 🎬 视频演示（原公众号视频）

这样我们就打通了 tool 里调用 service

那自然就可以实现数据库增删改查的 tool、发送邮件的 tool

我们用 qq 邮箱的 smtp 服务发送邮件

> 🎬 视频演示（原公众号视频）

拿到授权码之后，我们安装下 nodemailer

    ppip install nodemailer @nestjs-modules/mailer

在 AppModule 引入下：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/18_公众号_Yi昭.png)

    MailerModule.forRootAsync({&nbsp;&nbsp;inject: [ConfigService],useFactory:&nbsp;(configService: ConfigService) =&gt;&nbsp;({&nbsp; &nbsp;&nbsp;transport: {&nbsp; &nbsp; &nbsp;&nbsp;host: configService.get&lt;string&gt;('MAIL_HOST'),&nbsp; &nbsp; &nbsp;&nbsp;port:&nbsp;Number(configService.get&lt;string&gt;('MAIL_PORT')),&nbsp; &nbsp; &nbsp;&nbsp;secure: configService.get&lt;string&gt;('MAIL_SECURE') ===&nbsp;'true',&nbsp; &nbsp; &nbsp;&nbsp;auth: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;user: configService.get&lt;string&gt;('MAIL_USER'),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;pass: configService.get&lt;string&gt;('MAIL_PASS'),&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; },&nbsp; &nbsp;&nbsp;defaults: {&nbsp; &nbsp; &nbsp;&nbsp;from:&nbsp; &nbsp; &nbsp; &nbsp; configService.get&lt;string&gt;('MAIL_FROM')&nbsp; &nbsp; },&nbsp; }),}),

这里的配置也是放在 .env 里：

    MAIL_HOST=smtp.qq.comMAIL_PORT=587MAIL_SECURE=falseMAIL_USER=你的邮箱MAIL_PASS=你的授权码MAIL_FROM="No Reply"&nbsp;&lt;你的邮箱&gt;

我们把它封装成 tool

在 AiModule 加上这个 provider：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/19_公众号_Yi昭.png)

    {&nbsp;&nbsp;provide:&nbsp;'SEND_MAIL_TOOL',useFactory:&nbsp;(mailerService: MailerService, configService: ConfigService) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;sendMailArgsSchema = z.object({&nbsp; &nbsp; &nbsp;&nbsp;to: z&nbsp; &nbsp; &nbsp; &nbsp; .email()&nbsp; &nbsp; &nbsp; &nbsp; .describe('收件人邮箱地址，例如：someone@example.com'),&nbsp; &nbsp; &nbsp;&nbsp;subject: z.string().describe('邮件主题'),&nbsp; &nbsp; &nbsp;&nbsp;text: z.string().optional().describe('纯文本内容，可选'),&nbsp; &nbsp; &nbsp;&nbsp;html: z.string().optional().describe('HTML 内容，可选'),&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;return&nbsp;tool(&nbsp; &nbsp; &nbsp;&nbsp;async&nbsp;({to, subject, text, html}: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;to: string;&nbsp; &nbsp; &nbsp; &nbsp; subject: string;&nbsp; &nbsp; &nbsp; &nbsp; text?: string;&nbsp; &nbsp; &nbsp; &nbsp; html?: string;&nbsp; &nbsp; &nbsp; }) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;fallbackFrom =&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; configService.get&lt;string&gt;('MAIL_FROM')&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;await&nbsp;mailerService.sendMail({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; to,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; subject,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;text: text ??&nbsp;'（无文本内容）',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;html: html ??&nbsp;`&lt;p&gt;${text ??&nbsp;'（无 HTML 内容）'}&lt;/p&gt;`,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;from: fallbackFrom,&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return`邮件已发送到&nbsp;${to}，主题为「${subject}」`;&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;name:&nbsp;'send_mail',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;description:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'发送电子邮件。需要提供收件人邮箱、主题，可选文本内容和 HTML 内容。',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;schema: sendMailArgsSchema,&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; );&nbsp; },inject: [MailerService, ConfigService],},

在 AiService 里注入下：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/20_公众号_Yi昭.png)

tool 调用的地方也要加一下：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/21_公众号_Yi昭.png)

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/22_公众号_Yi昭.png)

这样，我们就可以用自然语言调用这个工具了：

测一下：

> 🎬 视频演示（原公众号视频）

这样，邮件发送的 tool 就跑通了。

接下来实现网络搜索的 tool。

用博查的 api：

https://open.bochaai.com/

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/23_公众号_Yi昭.png)

deepseek 的搜索就是用的这个：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/24_公众号_Yi昭.png)

挺靠谱的。

我们先搞一个 api key：

> 🎬 视频演示（原公众号视频）

添加到 .env 文件里

    BOCHA_API_KEY=sk-xxx

然后在 AiModule 添加一个 tool 的 provider：

    {&nbsp;&nbsp;provide:&nbsp;'WEB_SEARCH_TOOL',useFactory:&nbsp;(configService: ConfigService) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;webSearchArgsSchema = z.object({&nbsp; &nbsp; &nbsp;&nbsp;query: z&nbsp; &nbsp; &nbsp; &nbsp; .string()&nbsp; &nbsp; &nbsp; &nbsp; .min(1)&nbsp; &nbsp; &nbsp; &nbsp; .describe('搜索关键词，例如：公司年报、某个事件等'),&nbsp; &nbsp; &nbsp;&nbsp;count: z&nbsp; &nbsp; &nbsp; &nbsp; .number()&nbsp; &nbsp; &nbsp; &nbsp; .int()&nbsp; &nbsp; &nbsp; &nbsp; .min(1)&nbsp; &nbsp; &nbsp; &nbsp; .max(20)&nbsp; &nbsp; &nbsp; &nbsp; .optional()&nbsp; &nbsp; &nbsp; &nbsp; .describe('返回的搜索结果数量，默认 10 条'),&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;return&nbsp;tool(&nbsp; &nbsp; &nbsp;&nbsp;async&nbsp;({ query, count }: {&nbsp;query: string; count?: number }) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;apiKey = configService.get&lt;string&gt;('BOCHA_API_KEY');&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!apiKey) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return'Bocha Web Search 的 API Key 未配置（环境变量 BOCHA_API_KEY），请先在服务端配置后再重试。';&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;url =&nbsp;'https://api.bochaai.com/v1/web-search';&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;body = {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; query,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;freshness:&nbsp;'noLimit',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;summary:&nbsp;true,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;count: count ??&nbsp;10,&nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;response =&nbsp;await&nbsp;fetch(url, {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;method:&nbsp;'POST',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;headers: {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;Authorization:&nbsp;`Bearer&nbsp;${apiKey}`,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'Content-Type':&nbsp;'application/json',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;body:&nbsp;JSON.stringify(body),&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!response.ok) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;errorText =&nbsp;await&nbsp;response.text();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return`搜索 API 请求失败，状态码:&nbsp;${response.status}, 错误信息:&nbsp;${errorText}`;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;json: any;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; json =&nbsp;await&nbsp;response.json();&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;(e) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return`搜索 API 请求失败，原因是：搜索结果解析失败&nbsp;${(e&nbsp;as&nbsp;Error).message}`;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(json.code !==&nbsp;200&nbsp;|| !json.data) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return`搜索 API 请求失败，原因是:&nbsp;${json.msg ??&nbsp;'未知错误'}`;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;webpages = json.data.webPages?.value ?? [];&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!webpages.length) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return'未找到相关结果。';&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;formatted = webpages&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; .map(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;(page: any, idx: number) =&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;`引用:&nbsp;${idx +&nbsp;1}标题:&nbsp;${page.name}URL:&nbsp;${page.url}摘要:&nbsp;${page.summary}网站名称:&nbsp;${page.siteName}网站图标:&nbsp;${page.siteIcon}发布时间:&nbsp;${page.dateLastCrawled}`,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; )&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; .join('\n\n');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;formatted;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;(e) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return`搜索 API 请求失败，原因是：搜索结果解析失败&nbsp;${(e&nbsp;as&nbsp;Error).message}`;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;name:&nbsp;'web_search',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;description:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'使用 Bocha Web Search API 搜索互联网网页。输入为搜索关键词（可选 count 指定结果数量），返回包含标题、URL、摘要、网站名称、图标和时间等信息的结果列表。',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;schema: webSearchArgsSchema,&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; );&nbsp; },inject: [ConfigService],},

就是从配置文件拿到 apikey，通过 http 调用搜索接口，把结果格式化后给大模型。

然后在 AiService 里用一下：

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/25_公众号_Yi昭.png)

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/26_公众号_Yi昭.png)

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/27_公众号_Yi昭.png)

然后来测一下

当然，sse 还是用界面测更好，我们加一个 html

public/ai-sse-test.html

    &lt;!doctype&nbsp;html&gt;&lt;html&nbsp;lang="zh-CN"&gt;&lt;head&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;charset="UTF-8"&nbsp;/&gt;&nbsp; &nbsp;&nbsp;&lt;title&gt;AI SSE Chat 测试&lt;/title&gt;&nbsp; &nbsp;&nbsp;&lt;style&gt;&nbsp; &nbsp; &nbsp; * {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;box-sizing: border-box;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;body&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin:&nbsp;0;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;min-height:&nbsp;100vh;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-family: system-ui, -apple-system, BlinkMacSystemFont,&nbsp;'SF Pro Text',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'Segoe UI', sans-serif;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;display: flex;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;align-items: center;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;justify-content: center;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#f5f5f5;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;24px16px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.shell&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;width:&nbsp;100%;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;max-width:&nbsp;720px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;h1&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;20px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin:&nbsp;0012px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.card&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#ffffff;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;10px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;1px&nbsp;solid&nbsp;#e5e7eb;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;box-shadow:&nbsp;04px12pxrgba(15,&nbsp;23,&nbsp;42,&nbsp;0.06);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;16px18px18px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;label&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;display: block;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;13px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-bottom:&nbsp;6px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#6b7280;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;textarea&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;width:&nbsp;100%;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;min-height:&nbsp;80px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;8px10px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;8px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;1px&nbsp;solid&nbsp;#d1d5db;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;resize: vertical;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-family: inherit;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;outline: none;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;textarea::placeholder&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#9ca3af;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;textarea:focus&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-color:&nbsp;#3b82f6;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;box-shadow:&nbsp;0001pxrgba(59,&nbsp;130,&nbsp;246,&nbsp;0.3);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.controls&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;display: flex;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;align-items: center;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-top:&nbsp;10px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;gap:&nbsp;10px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;button&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;6px14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;999px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;1px&nbsp;solid&nbsp;#2563eb;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#3b82f6;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#ffffff;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;13px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;cursor: pointer;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;button:disabled&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;opacity:&nbsp;0.7;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;cursor: not-allowed;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.status&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;12px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#6b7280;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.output&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-top:&nbsp;16px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;10px10px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;8px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#111827;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#e5e7eb;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'Liberation Mono',&nbsp;'Courier New', monospace;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;white-space: pre-wrap;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;max-height:&nbsp;360px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;overflow-y: auto;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;&lt;/style&gt;&lt;/head&gt;&lt;body&gt;&nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="shell"&gt;&nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="card"&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;h1&gt;AI SSE Chat 测试&lt;/h1&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;label&nbsp;for="query"&gt;输入你的问题：&lt;/label&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;textarea&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;id="query"&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;placeholder="请输入要发送给 AI 的问题..."&nbsp; &nbsp; &nbsp; &nbsp; &gt;&lt;/textarea&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="controls"&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;button&nbsp;id="sendBtn"&gt;开始对话（SSE）&lt;/button&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="status"&nbsp;id="status"&gt;状态：待机&lt;/div&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/div&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="output"&nbsp;id="output"&gt;&lt;/div&gt;&nbsp; &nbsp; &nbsp;&nbsp;&lt;/div&gt;&nbsp; &nbsp;&nbsp;&lt;/div&gt;&nbsp; &nbsp;&nbsp;&lt;script&gt;&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;sendBtn =&nbsp;document.getElementById('sendBtn');&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;queryInput =&nbsp;document.getElementById('query');&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;outputEl =&nbsp;document.getElementById('output');&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;statusEl =&nbsp;document.getElementById('status');&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;es =&nbsp;null;&nbsp; &nbsp; &nbsp;&nbsp;function&nbsp;closeEventSource()&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(es) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; es.close();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; es =&nbsp;null;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; sendBtn.disabled =&nbsp;false;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; sendBtn.onclick =&nbsp;()&nbsp;=&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;query = queryInput.value.trim();&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!query) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; alert('请输入问题');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; closeEventSource();&nbsp; &nbsp; &nbsp; &nbsp; outputEl.textContent =&nbsp;'';&nbsp; &nbsp; &nbsp; &nbsp; sendBtn.disabled =&nbsp;true;&nbsp; &nbsp; &nbsp; &nbsp; statusEl.textContent =&nbsp;'状态：连接中…';&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;url =&nbsp;`/ai/chat/stream?query=${encodeURIComponent(query)}`;&nbsp; &nbsp; &nbsp; &nbsp; es =&nbsp;new&nbsp;EventSource(url);&nbsp; &nbsp; &nbsp; &nbsp; es.onopen =&nbsp;()&nbsp;=&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; statusEl.textContent =&nbsp;'状态：已连接，流式接收中…';&nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; es.onmessage =&nbsp;(event) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 后端每个 chunk 用 data 发过来&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; outputEl.textContent += event.data;&nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; es.onerror =&nbsp;()&nbsp;=&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; statusEl.textContent =&nbsp;'状态：连接结束或发生错误';&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; closeEventSource();&nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp;&nbsp;window.addEventListener('beforeunload', closeEventSource);&nbsp; &nbsp;&nbsp;&lt;/script&gt;&lt;/body&gt;&lt;/html&gt;

同样是 ai 写的页面，主要是用 EventSource 对接 sse 接口。

在 AppModule 加一下静态文件的访问

![image](../IMG/2026-03-13_Nesttool实现OpenClaw同款定时任务功能（上）/28_公众号_Yi昭.png)

安装用到的包

    ppip install fastapi.serve-static

跑一下：

> 🎬 视频演示（原公众号视频）

网络搜索和发送邮件的 tool 都跑通了。

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

我们梳理了豆包、OpenClaw 定时任务的实现思路。

创建了 FastAPI 后端项目，基于 LangChain + tool 实现了工具调用。

在 service 里加上了 Agent Loop，并且用 stream 方法实现了流式，提供 sse 接口。

然后我们把 tool 封装到 provider 实现了 tool 里调用 service。

之后分别封装了邮件发送 tool、网络搜索 tool。

综合测试了下，可以通过自然语言调用这些 tool。

下篇我们继续来实现数据库增删改查的 tool、定时任务的 tool，然后实现完整定时任务机制。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-03-13 12:46:05 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247485406&idx=1&sn=e97b50d911bbfb81cb4aaae10ab01b3f&chksm=f1b3a32e5b1ae960b74ccae3175f583bec57b383be309807e061636ccbc75a3de142d595fdb1&scene=27#wechat_redirect
