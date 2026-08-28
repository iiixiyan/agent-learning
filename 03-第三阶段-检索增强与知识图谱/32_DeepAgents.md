# DeepAgents：开箱即用的 skill、上下文压缩等 middleware

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年5月23日 22:45

我们学了 LangChain、LangGraph，可以基于它们实现各种 Agent。

但如果想做一个复杂的 Agent，全部从头自己实现还是比较麻烦。

有没有基于 LangGraph 再封装一层，也就是半成品的 Agent 框架呢？

有的，就是 DeepAgents。

LangChain 是给你一堆 AI 开发积木，LangGraph 是搭建复杂工作流的底层蓝图，那 DeepAgents 就是提前搭好主体结构的半成品房子。

底层依赖 LangGraph 的状态管理（state）、循环路由、持久化执行能力（checkpointer），上层直接内置了任务规划、长期记忆、子 Agent 调度、上下文压缩等核心能力。

它最大的优势，就是大幅降低复杂 Agent 的开发门槛。

原生 LangGraph 适合极致自定义、底层深度开发

DeepAgents 适合快速落地复杂 Agent 应用，比如深度调研、代码开发、多步骤业务执行、多智能体协作等场景。

它帮我们跳过重复的底层基建，直接聚焦 Agent 的业务逻辑与能力迭代，是 LangGraph 生态里面向生产落地的高阶封装方案。

![image](../IMG/2026-05-23_DeepAgents：开箱即用的skill、上下文压缩等middleware/0_公众号_Yi昭.png)

接下来我们就来学一下 DeepAgents：

创建项目：

    mkdir deepagents-testcd&nbsp;deepagents-testnpm init -y

![image](../IMG/2026-05-23_DeepAgents：开箱即用的skill、上下文压缩等middleware/1_公众号_Yi昭.png)

安装依赖：

    ppip install deepagents langchain langchain_langgraph langchain_openai zod dotenv&nbsp;

创建 .env

    OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1MODEL_NAME=qwen-plusOPENAI_API_KEY=sk-xxx# 用于身份验证，实现链路上报 &nbsp;LANGCHAIN_API_KEY=xxx# 指定LangSmith中的项目，追踪结果会归类到该项目下LANGCHAIN_PROJECT=deepagents-test# 开启LangSmith追踪功能LANGCHAIN_TRACING_V2=true

然后先来试一下 middleware，这个是 langchain 的功能：

创建 src/middleware-test.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ z }&nbsp;from"zod";import&nbsp;{ ChatOpenAI }&nbsp;from"langchain_openai";import&nbsp;{&nbsp; createAgent,&nbsp; createMiddleware,&nbsp; HumanMessage,&nbsp; AIMessage,}&nbsp;from"langchain";// --- 自定义 Middleware ---/** 日志 + 模型调用次数统计 */const&nbsp;loggingMiddleware = createMiddleware({name:&nbsp;"LoggingMiddleware",stateSchema: z.object({&nbsp; &nbsp;&nbsp;modelCallCount: z.number().default(0),&nbsp; }),beforeAgent:&nbsp;(state) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;console.log("\n[Logging] agent 开始，消息数:", state.messages.length);&nbsp; },beforeModel:&nbsp;(state) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;console.log(&nbsp; &nbsp; &nbsp;&nbsp;`[Logging] 即将调用模型，当前消息数:&nbsp;${state.messages.length}，已调用:&nbsp;${state.modelCallCount}&nbsp;次`&nbsp; &nbsp; );&nbsp; },afterModel:&nbsp;(state) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;last = state.messages.at(-1);&nbsp; &nbsp;&nbsp;const&nbsp;preview =&nbsp; &nbsp; &nbsp;&nbsp;typeof&nbsp;last?.content ===&nbsp;"string"&nbsp; &nbsp; &nbsp; &nbsp; ? last.content.slice(0,&nbsp;80)&nbsp; &nbsp; &nbsp; &nbsp; :&nbsp;JSON.stringify(last?.content)?.slice(0,&nbsp;80);&nbsp; &nbsp;&nbsp;console.log(`[Logging] 模型返回:&nbsp;${preview}...`);&nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp;modelCallCount: state.modelCallCount +&nbsp;1&nbsp;};&nbsp; },afterAgent:&nbsp;(state) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;console.log(&nbsp; &nbsp; &nbsp;&nbsp;`[Logging] agent 结束，累计模型调用:&nbsp;${state.modelCallCount}&nbsp;次\n`&nbsp; &nbsp; );&nbsp; },});/** 在每次模型调用前追加 system 上下文 */const&nbsp;addContextMiddleware = createMiddleware({name:&nbsp;"AddContextMiddleware",wrapModelCall:&nbsp;async&nbsp;(request, handler) =&gt; {&nbsp; &nbsp;&nbsp;console.log("[AddContext] 注入额外 system 上下文");&nbsp; &nbsp;&nbsp;return&nbsp;handler({&nbsp; &nbsp; &nbsp; ...request,&nbsp; &nbsp; &nbsp;&nbsp;systemMessage: request.systemMessage.concat(&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"\n\n 请用一句话简洁回答。"&nbsp; &nbsp; &nbsp; ),&nbsp; &nbsp; });&nbsp; },});/** 拦截敏感词，直接结束 agent */const&nbsp;blockedContentMiddleware = createMiddleware({name:&nbsp;"BlockedContentMiddleware",beforeModel: {&nbsp; &nbsp;&nbsp;canJumpTo: ["end"],&nbsp; &nbsp;&nbsp;hook:&nbsp;(state) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;last = state.messages.at(-1);&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;text =&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;typeof&nbsp;last?.content ===&nbsp;"string"&nbsp;? last.content :&nbsp;String(last?.content ??&nbsp;"");&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(text.includes("BLOCKED")) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log("[Blocked] 检测到 BLOCKED，短路结束");&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;messages: [new&nbsp;AIMessage("该请求已被 middleware 拦截，无法处理。")],&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;jumpTo:&nbsp;"end",&nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; },&nbsp; },});// --- Agent ---const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({model: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; },temperature:&nbsp;0,});const&nbsp;agent = createAgent({&nbsp; model,tools: [],systemPrompt:&nbsp;"你是一个助手。",middleware: [&nbsp; &nbsp; loggingMiddleware,&nbsp; &nbsp; addContextMiddleware,&nbsp; &nbsp; blockedContentMiddleware,&nbsp; ],});for&nbsp;(const&nbsp;text&nbsp;of&nbsp;["用中文说：middleware 是什么？","这句话包含 BLOCKED 关键词",]) {console.log("\n用户:", text);const&nbsp;{ messages, modelCallCount } =&nbsp;await&nbsp;agent.invoke({&nbsp; &nbsp;&nbsp;messages: [new&nbsp;HumanMessage(text)],&nbsp; });console.log("回复:", messages.at(-1)?.content);console.log("modelCallCount:", modelCallCount);}

createAgent 这个 api 提供了 middleware 的扩展机制：

![image](../IMG/2026-05-23_DeepAgents：开箱即用的skill、上下文压缩等middleware/2_公众号_Yi昭.png)

可以在 agent 运行前后、model 调用前后加一些逻辑，以及控制 model 要不要调用，可以提前结束流程

> 🎬 视频演示（原公众号视频）

此外，中间件还可以扩展 tool，以及 wrapToolCall

创建 src/middleware-test2.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ Command }&nbsp;from"langchain_langgraph";import&nbsp;{ z }&nbsp;from"zod";import&nbsp;{ ChatOpenAI }&nbsp;from"langchain_openai";import&nbsp;{&nbsp; createAgent,&nbsp; createMiddleware,&nbsp; HumanMessage,&nbsp; ToolMessage,&nbsp; tool,}&nbsp;from"langchain";const&nbsp;getCurrentTime = tool(()&nbsp;=&gt;newDate().toISOString(), {name:&nbsp;"get_current_time",description:&nbsp;"返回当前 UTC 时间的 ISO 8601 字符串",schema: z.object({}),});/** 通过 middleware 注册工具，并用 wrapToolCall 包装执行 */const&nbsp;extendedToolsMiddleware = createMiddleware({name:&nbsp;"ExtendedToolsMiddleware",stateSchema: z.object({&nbsp; &nbsp;&nbsp;toolInvocationCount: z.number().default(0),&nbsp; }),tools: [getCurrentTime],wrapToolCall:&nbsp;async&nbsp;(request, handler) =&gt; {&nbsp; &nbsp;&nbsp;const&nbsp;toolName = request.tool?.name ?? request.toolCall.name;&nbsp; &nbsp;&nbsp;console.log(&nbsp; &nbsp; &nbsp;&nbsp;`[Tools] 即将执行:&nbsp;${toolName}`,&nbsp; &nbsp; &nbsp;&nbsp;"args:",&nbsp; &nbsp; &nbsp; request.toolCall.args ?? {}&nbsp; &nbsp; );&nbsp; &nbsp;&nbsp;const&nbsp;result =&nbsp;await&nbsp;handler(request);&nbsp; &nbsp;&nbsp;if&nbsp;(!ToolMessage.isInstance(result))&nbsp;return&nbsp;result;&nbsp; &nbsp;&nbsp;const&nbsp;wrapped =&nbsp;new&nbsp;ToolMessage({&nbsp; &nbsp; &nbsp;&nbsp;content:&nbsp;`${result.content}\n[wrapToolCall] 已由 ExtendedToolsMiddleware 包装`,&nbsp; &nbsp; &nbsp;&nbsp;tool_call_id: result.tool_call_id,&nbsp; &nbsp; &nbsp;&nbsp;name: result.name,&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;console.log(&nbsp; &nbsp; &nbsp;&nbsp;`[Tools] 执行完成:&nbsp;${toolName}`,&nbsp; &nbsp; &nbsp;&nbsp;typeof&nbsp;wrapped.content ===&nbsp;"string"&nbsp; &nbsp; &nbsp; &nbsp; ? wrapped.content.slice(0,&nbsp;120)&nbsp; &nbsp; &nbsp; &nbsp; : wrapped&nbsp; &nbsp; );&nbsp; &nbsp;&nbsp;returnnew&nbsp;Command({&nbsp; &nbsp; &nbsp;&nbsp;update: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;toolInvocationCount: request.state.toolInvocationCount +&nbsp;1,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;messages: [wrapped],&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; });&nbsp; },afterAgent:&nbsp;(state) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;console.log(&nbsp; &nbsp; &nbsp;&nbsp;`[Tools] agent 结束，middleware 统计工具调用:&nbsp;${state.toolInvocationCount}&nbsp;次`&nbsp; &nbsp; );&nbsp; },});const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({model: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; },temperature:&nbsp;0,});const&nbsp;agent = createAgent({&nbsp; model,tools: [],systemPrompt:&nbsp; &nbsp;&nbsp;"你是一个助手。",middleware: [extendedToolsMiddleware],});for&nbsp;(const&nbsp;text&nbsp;of&nbsp;["给我当前时间",]) {console.log("\n用户:", text);const&nbsp;{ messages, toolInvocationCount } =&nbsp;await&nbsp;agent.invoke({&nbsp; &nbsp;&nbsp;messages: [new&nbsp;HumanMessage(text)],&nbsp; });console.log("回复:", messages.at(-1)?.content);console.log("toolInvocationCount:", toolInvocationCount);}

> 🎬 视频演示（原公众号视频）

这样我们就通过中间件给 agent 扩展了 tools 并且修改了 tool call 返回的结果

deepagents 里就有很多现成的中间件可以用：

![image](../IMG/2026-05-23_DeepAgents：开箱即用的skill、上下文压缩等middleware/3_公众号_Yi昭.png)

先试一下 FilesystemMiddleware，这个中间件可以指定一个 backend 作为文件系统，然后提供了读写、修改、搜索文件的命令。

创建 src/deepagents/filesystem-agent.mjs

    import&nbsp;"dotenv/config";import&nbsp;fs&nbsp;from"node:fs";import&nbsp;path&nbsp;from"node:path";import&nbsp;{ fileURLToPath }&nbsp;from"node:url";import&nbsp;{ ChatOpenAI }&nbsp;from"langchain_openai";import&nbsp;{ createAgent, HumanMessage }&nbsp;from"langchain";import&nbsp;{ createFilesystemMiddleware, FilesystemBackend }&nbsp;from"deepagents";const&nbsp;workspaceDir = path.join(&nbsp; path.dirname(fileURLToPath(import.meta.url)),"workspace");/** 先匹配先生效；未命中任何规则则默认允许 */const&nbsp;permissions = [&nbsp; {&nbsp;operations: ["read"],&nbsp;paths: ["/secret.txt"],&nbsp;mode:&nbsp;"deny"&nbsp;},&nbsp; {&nbsp;operations: ["write"],&nbsp;paths: ["/todo.md"],&nbsp;mode:&nbsp;"allow"&nbsp;},&nbsp; {&nbsp;operations: ["write"],&nbsp;paths: ["/**"],&nbsp;mode:&nbsp;"deny"&nbsp;},];fs.rmSync(workspaceDir, {&nbsp;recursive:&nbsp;true,&nbsp;force:&nbsp;true&nbsp;});fs.mkdirSync(workspaceDir);fs.writeFileSync(path.join(workspaceDir,&nbsp;"secret.txt"),&nbsp;"机密：不得读取",&nbsp;"utf8");const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({model: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp;baseURL: process.env.OPENAI_BASE_URL },temperature:&nbsp;0,});const&nbsp;agent = createAgent({&nbsp; model,tools: [],systemPrompt:&nbsp; &nbsp;&nbsp;"工作区根路径为 /。用 ls、read_file、write_file、edit_file 操作文件，路径以 / 开头。中文回答。",middleware: [&nbsp; &nbsp; createFilesystemMiddleware({&nbsp; &nbsp; &nbsp;&nbsp;backend:&nbsp;new&nbsp;FilesystemBackend({&nbsp;rootDir: workspaceDir,&nbsp;virtualMode:&nbsp;true&nbsp;}),&nbsp; &nbsp; &nbsp; permissions,&nbsp; &nbsp; }),&nbsp; ],});console.log("工作区:", workspaceDir);console.log("权限:",&nbsp;JSON.stringify(permissions,&nbsp;null,&nbsp;2));asyncfunction&nbsp;run(label, prompt)&nbsp;{console.log(`\n===&nbsp;${label}&nbsp;===\n`, prompt,&nbsp;"\n");const&nbsp;{ messages } =&nbsp;await&nbsp;agent.invoke(&nbsp; &nbsp; {&nbsp;messages: [new&nbsp;HumanMessage(prompt)] },&nbsp; &nbsp; {&nbsp;recursionLimit:&nbsp;20&nbsp;}&nbsp; );for&nbsp;(const&nbsp;m&nbsp;of&nbsp;messages) {&nbsp; &nbsp;&nbsp;for&nbsp;(const&nbsp;t&nbsp;of&nbsp;m.tool_calls ?? [])&nbsp;console.log("→", t.name);&nbsp; }console.log("回复:", messages.at(-1)?.content);}asyncfunction&nbsp;expectDenied(label, prompt)&nbsp;{console.log(`\n===&nbsp;${label}（预期拒绝）===\n`, prompt,&nbsp;"\n");try&nbsp;{&nbsp; &nbsp;&nbsp;await&nbsp;agent.invoke({&nbsp;messages: [new&nbsp;HumanMessage(prompt)] }, {&nbsp;recursionLimit:&nbsp;5&nbsp;});&nbsp; &nbsp;&nbsp;console.log("未触发拒绝（异常）");&nbsp; }&nbsp;catch&nbsp;(e) {&nbsp; &nbsp;&nbsp;const&nbsp;msg = e.cause?.message ?? e.message;&nbsp; &nbsp;&nbsp;console.log("✗", msg);&nbsp; }}await&nbsp;run("允许的操作","write_file 创建 /todo.md（三条待办），edit_file 把第一条标为完成，ls /，一句话总结。");await&nbsp;expectDenied("禁止读",&nbsp;"只调用 read_file，路径 /secret.txt。");await&nbsp;expectDenied("禁止写",&nbsp;"只调用 write_file，路径 /hack.txt，内容 test。");

> 🎬 视频演示（原公众号视频）

只要加上 deepagents 这个 FileSystem 中间件，agent 就有了一个文件系统，并且有了读写搜索文件的各种 tool，还做了权限控制。

超级方便，不用自己写！

大家应该都用过 skill，如果我们的 Agent 也要支持 skill 呢？

直接用 deepagents 的 Skill sMiddleware

创建 src/deepagents/skills-agent.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ existsSync, mkdirSync }&nbsp;from"node:fs";import&nbsp;{ ChatOpenAI }&nbsp;from"langchain_openai";import&nbsp;{ createAgent, HumanMessage }&nbsp;from"langchain";import&nbsp;{&nbsp; LocalShellBackend,&nbsp; createFilesystemMiddleware,&nbsp; createSkillsMiddleware,}&nbsp;from"deepagents";const&nbsp;skills =&nbsp;"/.agents/skills/";const&nbsp;output =&nbsp;"src/deepagents/output/deepagents-skills-flow.excalidraw";if&nbsp;(!existsSync(".agents/skills/excalidraw-diagram-generator/SKILL.md")) {thrownewError(&nbsp; &nbsp;&nbsp;"未找到 excalidraw-diagram-generator，请先: python -m skills add github/awesome-copilot --skill excalidraw-diagram-generator -y"&nbsp; );}mkdirSync("src/deepagents/output", {&nbsp;recursive:&nbsp;true&nbsp;});const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({model: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp;baseURL: process.env.OPENAI_BASE_URL },temperature:&nbsp;0,streaming:&nbsp;true,});const&nbsp;backend =&nbsp;await&nbsp;LocalShellBackend.create({rootDir:&nbsp;".",virtualMode:&nbsp;true,inheritEnv:&nbsp;true,});const&nbsp;agent = createAgent({&nbsp; model,tools: [],systemPrompt:&nbsp;"按 skills 库完成任务，需要时 read_file 对应 SKILL.md。中文回答。",middleware: [&nbsp; &nbsp; createSkillsMiddleware({ backend,&nbsp;sources: [skills] }),&nbsp; &nbsp; createFilesystemMiddleware({ backend }),&nbsp; ],});const&nbsp;prompt = ["画一张流程图，描述本项目的 skills-agent 工作流：","用户 Prompt → createAgent → createSkillsMiddleware → createFilesystemMiddleware → 模型回复。",`保存为&nbsp;${output}。要求：`,"- 顶部大标题 + 副标题","- 每个主节点 numbered（①②…）且框内 2～3 行中文说明","- 右侧一列「说明：…」补充细节","- 箭头上标注阶段名（如 invoke、wrapModelCall）","- 底部图例（颜色含义 + 如何运行 demo）",].join("\n");console.log("用户:", prompt);function&nbsp;chunkText(chunk)&nbsp;{if&nbsp;(!chunk?.content)&nbsp;return"";if&nbsp;(typeof&nbsp;chunk.content ===&nbsp;"string")&nbsp;return&nbsp;chunk.content;if&nbsp;(Array.isArray(chunk.content)) {&nbsp; &nbsp;&nbsp;return&nbsp;chunk.content&nbsp; &nbsp; &nbsp; .map((p) =&gt;&nbsp;(typeof&nbsp;p ===&nbsp;"string"&nbsp;? p : (p?.text ??&nbsp;"")))&nbsp; &nbsp; &nbsp; .join("");&nbsp; }return"";}const&nbsp;stream =&nbsp;await&nbsp;agent.streamEvents(&nbsp; {&nbsp;messages: [new&nbsp;HumanMessage(prompt)] },&nbsp; {&nbsp;recursionLimit:&nbsp;100&nbsp;});let&nbsp;skillsMetadata;console.log("\n--- 流式输出 ---\n");try&nbsp;{forawait&nbsp;(const&nbsp;event&nbsp;of&nbsp;stream) {&nbsp; &nbsp;&nbsp;if&nbsp;(event.event ===&nbsp;"on_chat_model_stream") {&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;text = chunkText(event.data?.chunk);&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(text) process.stdout.write(text);&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;if&nbsp;(event.event ===&nbsp;"on_tool_start") {&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;name = event.name?.split("/").pop() ?? event.name;&nbsp; &nbsp; &nbsp; process.stdout.write(`\n\n→&nbsp;${name}\n\n`);&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;if&nbsp;(event.event ===&nbsp;"on_chain_end"&nbsp;&amp;&amp; event.data?.output?.skillsMetadata) {&nbsp; &nbsp; &nbsp; skillsMetadata = event.data.output.skillsMetadata;&nbsp; &nbsp; }&nbsp; }}&nbsp;catch&nbsp;(e) {console.error("\n\n[错误]", e.cause?.message ?? e.message);throw&nbsp;e;}console.log("\n");console.log("skills:", skillsMetadata?.map((s) =&gt;&nbsp;s.name));if&nbsp;(existsSync(output)) {console.log("图表:", output);console.log("打开: https://excalidraw.com → Open → 选择该文件");}&nbsp;else&nbsp;{console.log("未生成:", output);}await&nbsp;backend.close();

从 https://www.skills.sh/ 查找 skill

![image](../IMG/2026-05-23_DeepAgents：开箱即用的skill、上下文压缩等middleware/4_公众号_Yi昭.png)

> 🎬 视频演示（原公众号视频）

这样，我们的 Agent 就支持 skill 了！

再来试一下 deepagents 其他中间件：

SubAgentMiddleware 这个是创建多 Agent 用的

创建 src/deepagents/subagent-agent.mjs

    import&nbsp;"dotenv/config";import&nbsp;{ z }&nbsp;from"zod";import&nbsp;{ ChatOpenAI }&nbsp;from"langchain_openai";import&nbsp;{ createAgent, HumanMessage, tool }&nbsp;from"langchain";import&nbsp;{ createSubAgentMiddleware }&nbsp;from"deepagents";/** 四则运算 */const&nbsp;calc = tool(({ a, b, op }) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;ops = {&nbsp; &nbsp; &nbsp;&nbsp;add: a + b,&nbsp; &nbsp; &nbsp;&nbsp;subtract: a - b,&nbsp; &nbsp; &nbsp;&nbsp;multiply: a * b,&nbsp; &nbsp; &nbsp;&nbsp;divide: b ===&nbsp;0&nbsp;?&nbsp;NaN&nbsp;: a / b,&nbsp; &nbsp; };&nbsp; &nbsp;&nbsp;const&nbsp;result = ops[op];&nbsp; &nbsp;&nbsp;if&nbsp;(Number.isNaN(result)) {&nbsp; &nbsp; &nbsp;&nbsp;returnJSON.stringify({&nbsp;error:&nbsp;"除数不能为 0"&nbsp;});&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;const&nbsp;symbols = {&nbsp;add:&nbsp;"+",&nbsp;subtract:&nbsp;"-",&nbsp;multiply:&nbsp;"×",&nbsp;divide:&nbsp;"÷"&nbsp;};&nbsp; &nbsp;&nbsp;returnJSON.stringify({&nbsp; &nbsp; &nbsp;&nbsp;expression:&nbsp;`${a}&nbsp;${symbols[op]}&nbsp;${b}`,&nbsp; &nbsp; &nbsp; result,&nbsp; &nbsp; });&nbsp; },&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;"calc",&nbsp; &nbsp;&nbsp;description:&nbsp;"计算两个数的加减乘除",&nbsp; &nbsp;&nbsp;schema: z.object({&nbsp; &nbsp; &nbsp;&nbsp;a: z.number().describe("左操作数"),&nbsp; &nbsp; &nbsp;&nbsp;b: z.number().describe("右操作数"),&nbsp; &nbsp; &nbsp;&nbsp;op: z.enum(["add",&nbsp;"subtract",&nbsp;"multiply",&nbsp;"divide"]).describe("运算类型"),&nbsp; &nbsp; }),&nbsp; });/** 平均分：总数 ÷ 份数 */const&nbsp;divideEvenly = tool(({ total, parts }) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;if&nbsp;(parts &lt;=&nbsp;0) {&nbsp; &nbsp; &nbsp;&nbsp;returnJSON.stringify({&nbsp;error:&nbsp;"份数须大于 0"&nbsp;});&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;const&nbsp;each = total / parts;&nbsp; &nbsp;&nbsp;const&nbsp;exact =&nbsp;Number.isInteger(each);&nbsp; &nbsp;&nbsp;returnJSON.stringify({&nbsp; &nbsp; &nbsp; total,&nbsp; &nbsp; &nbsp; parts,&nbsp; &nbsp; &nbsp; each,&nbsp; &nbsp; &nbsp; exact,&nbsp; &nbsp; &nbsp;&nbsp;note: exact&nbsp; &nbsp; &nbsp; &nbsp; ?&nbsp;`每人&nbsp;${each}（整除）`&nbsp; &nbsp; &nbsp; &nbsp; :&nbsp;`每人&nbsp;${each}（不能整除，应用题可说明余数）`,&nbsp; &nbsp; });&nbsp; },&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;"divide_evenly",&nbsp; &nbsp;&nbsp;description:&nbsp;"把总数平均分成若干份，求每份多少",&nbsp; &nbsp;&nbsp;schema: z.object({&nbsp; &nbsp; &nbsp;&nbsp;total: z.number().nonnegative().describe("总数"),&nbsp; &nbsp; &nbsp;&nbsp;parts: z.number().int().positive().describe("分成几份"),&nbsp; &nbsp; }),&nbsp; });/** 按模板生成同类练习题（只改数字） */const&nbsp;makeSimilarProblem = tool(({ template, seed }) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;n = (seed %&nbsp;7) +&nbsp;3;&nbsp; &nbsp;&nbsp;const&nbsp;problems = {&nbsp; &nbsp; &nbsp;&nbsp;divide_then_add: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;stem:&nbsp;`小红有&nbsp;${n *&nbsp;6}&nbsp;张贴纸，平均分给&nbsp;${n}&nbsp;个小组，又买了 2 包每包&nbsp;${n +&nbsp;2}&nbsp;张的。每个小组现在一共有多少张？`,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;hint:&nbsp;"先平均分，再加上后来买的，注意单位是「每个小组」",&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp;&nbsp;share_candy: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;stem:&nbsp;`小刚有&nbsp;${n *&nbsp;4}&nbsp;块糖，要分给&nbsp;${n}&nbsp;位同学，妈妈又买了 3 袋每袋&nbsp;${n}&nbsp;块的。每位同学现在能分到多少块？`,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;hint:&nbsp;"与分糖题类似：先平分，再加上新增",&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp;&nbsp;group_buy: {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;stem:&nbsp;`班里有&nbsp;${n}&nbsp;个小组，每组先分到&nbsp;${n *&nbsp;5}&nbsp;支铅笔，老师又补了 2 盒每盒&nbsp;${n +&nbsp;1}&nbsp;支。每个小组现在有多少支？`,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;hint:&nbsp;"先算每组原有，再加上后来补的",&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; };&nbsp; &nbsp;&nbsp;const&nbsp;picked = problems[template] ?? problems.share_candy;&nbsp; &nbsp;&nbsp;returnJSON.stringify({ template, ...picked });&nbsp; },&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;"make_similar_problem",&nbsp; &nbsp;&nbsp;description:&nbsp; &nbsp; &nbsp;&nbsp;"生成一道同类应用题。template: divide_then_add | share_candy | group_buy",&nbsp; &nbsp;&nbsp;schema: z.object({&nbsp; &nbsp; &nbsp;&nbsp;template: z&nbsp; &nbsp; &nbsp; &nbsp; .enum(["divide_then_add",&nbsp;"share_candy",&nbsp;"group_buy"])&nbsp; &nbsp; &nbsp; &nbsp; .describe("题目模板"),&nbsp; &nbsp; &nbsp;&nbsp;seed: z.number().int().describe("随机种子，用于变换数字"),&nbsp; &nbsp; }),&nbsp; });const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({model: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp;baseURL: process.env.OPENAI_BASE_URL },temperature:&nbsp;0,streaming:&nbsp;true,});const&nbsp;subagents = [&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;"math-solver",&nbsp; &nbsp;&nbsp;description:&nbsp; &nbsp; &nbsp;&nbsp;"解小学应用题：用 calc、divide_evenly 列式计算，给出最终答案与算式。有具体数字时先用此 Agent。",&nbsp; &nbsp;&nbsp;systemPrompt: [&nbsp; &nbsp; &nbsp;&nbsp;"你是解题子 Agent。",&nbsp; &nbsp; &nbsp;&nbsp;"必须用 calc、divide_evenly 完成计算，不要心算。",&nbsp; &nbsp; &nbsp;&nbsp;"输出：题目理解、分步算式、最终答案（带单位「块/人」等）。",&nbsp; &nbsp; ].join("\n"),&nbsp; &nbsp;&nbsp;tools: [calc, divideEvenly],&nbsp; },&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;"kid-tutor",&nbsp; &nbsp;&nbsp;description:&nbsp; &nbsp; &nbsp;&nbsp;"把 math-solver 的解法讲给家长听，方便辅导孩子。description 里会有完整解题过程。",&nbsp; &nbsp;&nbsp;systemPrompt: [&nbsp; &nbsp; &nbsp;&nbsp;"你是辅导讲解子 Agent，面向小学生家长。",&nbsp; &nbsp; &nbsp;&nbsp;"根据 description 中的解题过程，用短句、比喻或分步提问方式讲解（不要堆公式）。",&nbsp; &nbsp; &nbsp;&nbsp;"说明：先想什么、再算什么、怎么检查答案。不使用工具。",&nbsp; &nbsp; ].join("\n"),&nbsp; &nbsp;&nbsp;tools: [],&nbsp; },&nbsp; {&nbsp; &nbsp;&nbsp;name:&nbsp;"practice-maker",&nbsp; &nbsp;&nbsp;description:&nbsp; &nbsp; &nbsp;&nbsp;"出 2 道同类练习题。用 make_similar_problem 生成题干，可换不同 template 或 seed。",&nbsp; &nbsp;&nbsp;systemPrompt: [&nbsp; &nbsp; &nbsp;&nbsp;"你是出题子 Agent。",&nbsp; &nbsp; &nbsp;&nbsp;"调用 make_similar_problem 至少 2 次（不同 template 或不同 seed），",&nbsp; &nbsp; &nbsp;&nbsp;"每道题给出：题干、解题提示（一句话）。",&nbsp; &nbsp; ].join("\n"),&nbsp; &nbsp;&nbsp;tools: [makeSimilarProblem],&nbsp; },];const&nbsp;agent = createAgent({&nbsp; model,tools: [],systemPrompt: [&nbsp; &nbsp;&nbsp;"你是小学数学辅导主 Agent，通过 task 委派子 Agent，自己不解题、不讲题、不出题。",&nbsp; &nbsp;&nbsp;"按顺序：① math-solver ② kid-tutor（把 solver 完整过程写进 description）③ practice-maker。",&nbsp; &nbsp;&nbsp;"最后向家长汇总：答案、辅导要点、两道练习题。中文。",&nbsp; ].join("\n"),middleware: [&nbsp; &nbsp; createSubAgentMiddleware({&nbsp; &nbsp; &nbsp;&nbsp;defaultModel: model,&nbsp; &nbsp; &nbsp; subagents,&nbsp; &nbsp; &nbsp;&nbsp;generalPurposeAgent:&nbsp;false,&nbsp; &nbsp; }),&nbsp; ],});const&nbsp;prompt = ["孩子遇到这道题：","「小明有 24 块糖，平均分给 6 个同学；","妈妈又买了 3 包糖，每包 5 块。每个同学现在一共有多少块？」","请先 math-solver 解题，再 kid-tutor 教家长怎么讲，","最后 practice-maker 出 2 道类似练习题，并汇总给我。",].join("");function&nbsp;chunkText(chunk)&nbsp;{if&nbsp;(!chunk?.content)&nbsp;return"";if&nbsp;(typeof&nbsp;chunk.content ===&nbsp;"string")&nbsp;return&nbsp;chunk.content;if&nbsp;(Array.isArray(chunk.content)) {&nbsp; &nbsp;&nbsp;return&nbsp;chunk.content&nbsp; &nbsp; &nbsp; .map((p) =&gt;&nbsp;(typeof&nbsp;p ===&nbsp;"string"&nbsp;? p : (p?.text ??&nbsp;"")))&nbsp; &nbsp; &nbsp; .join("");&nbsp; }return"";}console.log("场景: 小学应用题辅导（解题 → 讲题 → 出题）");console.log("子 Agent:");console.log(" &nbsp;math-solver &nbsp; &nbsp; → calc, divide_evenly");console.log(" &nbsp;kid-tutor &nbsp; &nbsp; &nbsp; → （讲解，无工具）");console.log(" &nbsp;practice-maker &nbsp;→ make_similar_problem");console.log();console.log("用户:", prompt,&nbsp;"\n");console.log("--- 流式输出 ---\n");const&nbsp;stream =&nbsp;await&nbsp;agent.streamEvents(&nbsp; {&nbsp;messages: [new&nbsp;HumanMessage(prompt)] },&nbsp; {&nbsp;recursionLimit:&nbsp;60&nbsp;});try&nbsp;{forawait&nbsp;(const&nbsp;event&nbsp;of&nbsp;stream) {&nbsp; &nbsp;&nbsp;if&nbsp;(event.event ===&nbsp;"on_chat_model_stream") {&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;t = chunkText(event.data?.chunk);&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(t) process.stdout.write(t);&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;if&nbsp;(event.event ===&nbsp;"on_tool_start") {&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;name = event.name?.split("/").pop() ?? event.name;&nbsp; &nbsp; &nbsp; process.stdout.write(`\n\n→&nbsp;${name}\n\n`);&nbsp; &nbsp; }&nbsp; }}&nbsp;catch&nbsp;(e) {console.error("\n\n[错误]", e.cause?.message ?? e.message);throw&nbsp;e;}console.log("\n");

> 🎬 视频演示（原公众号视频）

用 SubAgent 的 middleware 创建子 Agent 更简单了，声明就行，不用自己去实现。

此外，长期记忆也是 Agent 必备的功能，deepagents 提供了 MemoryMiddleware

可以把记忆存储在 markdown 文件里，可以读取、更新，持久化存储

创建 src/deepagents/memory-agent.mjs

    import&nbsp;"dotenv/config";import&nbsp;fs&nbsp;from"node:fs";import&nbsp;path&nbsp;from"node:path";import&nbsp;{ fileURLToPath }&nbsp;from"node:url";import&nbsp;{ ChatOpenAI }&nbsp;from"langchain_openai";import&nbsp;{ createAgent, HumanMessage }&nbsp;from"langchain";import&nbsp;{&nbsp; createFilesystemMiddleware,&nbsp; createMemoryMiddleware,&nbsp; FilesystemBackend,}&nbsp;from"deepagents";const&nbsp;__dirname = path.dirname(fileURLToPath(import.meta.url));const&nbsp;workspaceDir = path.join(__dirname,&nbsp;"workspace-memory");const&nbsp;projectMemoryPath =&nbsp;"/AGENTS.md";const&nbsp;preferencesMemoryPath =&nbsp;"/memory/preferences.md";const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({model: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp;baseURL: process.env.OPENAI_BASE_URL },temperature:&nbsp;0,});const&nbsp;backend =&nbsp;new&nbsp;FilesystemBackend({rootDir: workspaceDir,virtualMode:&nbsp;true,});const&nbsp;agent = createAgent({&nbsp; model,tools: [],systemPrompt: [&nbsp; &nbsp;&nbsp;"你是项目助手。工作区根路径为 /，可用 ls、read_file、write_file、edit_file。",&nbsp; &nbsp;&nbsp;"根据 &lt;agent_memory&gt; 回答；用户要求记住时，必须立刻 edit_file，且按类型写入对应文件：",&nbsp; &nbsp;&nbsp;`-&nbsp;${projectMemoryPath}：项目说明、技术栈、架构、仓库约定等`,&nbsp; &nbsp;&nbsp;`-&nbsp;${preferencesMemoryPath}：用户个人偏好（语言、包管理器、回答风格等）`,&nbsp; &nbsp;&nbsp;"不要混写：项目事实不要写入 preferences，个人偏好不要写入 AGENTS.md。",&nbsp; ].join("\n"),middleware: [&nbsp; &nbsp; createFilesystemMiddleware({ backend }),&nbsp; &nbsp; createMemoryMiddleware({&nbsp; &nbsp; &nbsp; backend,&nbsp; &nbsp; &nbsp;&nbsp;sources: [projectMemoryPath, preferencesMemoryPath],&nbsp; &nbsp; }),&nbsp; ],});const&nbsp;prompts = ["根据记忆，这个项目是做什么的？只答一句。",`请记住：我常用的包管理器是 pnpm。`,`请记住：本仓库主入口脚本是 src/deepagents/memory-agent.mjs。`,"我常用什么包管理器？本 demo 主入口脚本路径是什么？各用一行回答。",];let&nbsp;messages = [];for&nbsp;(const&nbsp;prompt&nbsp;of&nbsp;prompts) {console.log("\n用户:", prompt);&nbsp; ({ messages } =&nbsp;await&nbsp;agent.invoke(&nbsp; &nbsp; {&nbsp;messages: [...messages,&nbsp;new&nbsp;HumanMessage(prompt)] },&nbsp; &nbsp; {&nbsp;recursionLimit:&nbsp;30&nbsp;}&nbsp; ));console.log("回复:", messages.at(-1)?.content);}for&nbsp;(const&nbsp;p&nbsp;of&nbsp;[projectMemoryPath, preferencesMemoryPath]) {const&nbsp;file = path.join(workspaceDir, p.replace(/^\//,&nbsp;""));console.log(`\n---&nbsp;${p}&nbsp;---\n`, fs.readFileSync(file,&nbsp;"utf8"));}

> 🎬 视频演示（原公众号视频）

这样，agent 就可以从 md 文件读取长期记忆，并且你让他记住的信息也会更新到 md 文件里。

最后，还有一个 SummarizationMiddleware 的中间件，它的作用是如果当前对话上下文长度超过预设阈值，就自动对历史对话进行摘要压缩，剔除冗余信息，只保留关键上下文摘要，再传入大模型进行后续续写 / 问答。

这样可以控制 Token 消耗、避免上下文溢出，同时保证核心对话语义不丢失。

我们自己做这个压缩还是比较麻烦的，这个 middleware 可以帮我们完成。

创建 src/deepagents/summarization-agent.mjs

    import&nbsp;"dotenv/config";import&nbsp;fs&nbsp;from"node:fs";import&nbsp;path&nbsp;from"node:path";import&nbsp;{ fileURLToPath }&nbsp;from"node:url";import&nbsp;{ ChatOpenAI }&nbsp;from"langchain_openai";import&nbsp;{ createAgent, HumanMessage }&nbsp;from"langchain";import&nbsp;{ createSummarizationMiddleware, FilesystemBackend }&nbsp;from"deepagents";const&nbsp;__dirname = path.dirname(fileURLToPath(import.meta.url));const&nbsp;workspaceDir = path.join(__dirname,&nbsp;"workspace-summarization");const&nbsp;historyPathPrefix =&nbsp;"/conversation_history";const&nbsp;summaryPrompt =&nbsp;`你是对话摘要助手。请用中文总结以下对话，包含：1. 讨论的主要话题2. 达成的关键结论或决定3. 继续对话所需的重要上下文保持简洁，不要罗列无关细节。待摘要的对话：{conversation}摘要：`;fs.rmSync(workspaceDir, {&nbsp;recursive:&nbsp;true,&nbsp;force:&nbsp;true&nbsp;});fs.mkdirSync(workspaceDir, {&nbsp;recursive:&nbsp;true&nbsp;});const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({model: process.env.MODEL_NAME,apiKey: process.env.OPENAI_API_KEY,configuration: {&nbsp;baseURL: process.env.OPENAI_BASE_URL },temperature:&nbsp;0,});const&nbsp;backend =&nbsp;new&nbsp;FilesystemBackend({rootDir: workspaceDir,virtualMode:&nbsp;true,});const&nbsp;agent = createAgent({&nbsp; model,tools: [],systemPrompt:&nbsp; &nbsp;&nbsp;"你是会话助手。记住用户提到的关键事实，中文简短回答。若看到「此前对话摘要」，请据此继续对话。",middleware: [&nbsp; &nbsp; createSummarizationMiddleware({&nbsp; &nbsp; &nbsp; model,&nbsp; &nbsp; &nbsp; backend,&nbsp; &nbsp; &nbsp; historyPathPrefix,&nbsp; &nbsp; &nbsp; summaryPrompt,&nbsp; &nbsp; &nbsp;&nbsp;// 低阈值便于 demo 触发摘要；生产环境可省略 trigger/keep，由模型 profile 自动推断&nbsp; &nbsp; &nbsp;&nbsp;trigger: {&nbsp;type:&nbsp;"messages",&nbsp;value:&nbsp;8&nbsp;},&nbsp; &nbsp; &nbsp;&nbsp;keep: {&nbsp;type:&nbsp;"messages",&nbsp;value:&nbsp;4&nbsp;},&nbsp; &nbsp; }),&nbsp; ],});const&nbsp;prompts = ["请记住：我的宠物猫叫小橘。","请记住：我住在北京。","请记住：我喜欢喝拿铁。","请记住：我的生日是 5 月 1 日。","根据我们聊过的内容，我的猫叫什么、住哪、喜欢喝什么、生日是哪天？每项一行。",];const&nbsp;historyDir = path.join(workspaceDir, historyPathPrefix.replace(/^\//,&nbsp;""));function&nbsp;listHistoryFiles()&nbsp;{if&nbsp;(!fs.existsSync(historyDir))&nbsp;return&nbsp;[];return&nbsp;fs.readdirSync(historyDir);}let&nbsp;messages = [];let&nbsp;knownHistory =&nbsp;newSet(listHistoryFiles());for&nbsp;(const&nbsp;prompt&nbsp;of&nbsp;prompts) {console.log("\n用户:", prompt);&nbsp; ({ messages } =&nbsp;await&nbsp;agent.invoke(&nbsp; &nbsp; {&nbsp;messages: [...messages,&nbsp;new&nbsp;HumanMessage(prompt)] },&nbsp; &nbsp; {&nbsp;recursionLimit:&nbsp;30&nbsp;}&nbsp; ));console.log("回复:", messages.at(-1)?.content);console.log("当前消息数:", messages.length);const&nbsp;historyFiles = listHistoryFiles();for&nbsp;(const&nbsp;file&nbsp;of&nbsp;historyFiles) {&nbsp; &nbsp;&nbsp;if&nbsp;(!knownHistory.has(file)) {&nbsp; &nbsp; &nbsp; knownHistory.add(file);&nbsp; &nbsp; &nbsp;&nbsp;console.log("已触发摘要，历史已写入:",&nbsp;`${historyPathPrefix}/${file}`);&nbsp; &nbsp; }&nbsp; }}if&nbsp;(knownHistory.size &gt;&nbsp;0) {for&nbsp;(const&nbsp;file&nbsp;of&nbsp;knownHistory) {&nbsp; &nbsp;&nbsp;const&nbsp;filePath = path.join(historyDir, file);&nbsp; &nbsp;&nbsp;console.log(`\n---&nbsp;${historyPathPrefix}/${file}&nbsp;---\n`, fs.readFileSync(filePath,&nbsp;"utf8"));&nbsp; }}&nbsp;else&nbsp;{console.log("\n未生成 conversation_history（可能未触发摘要阈值）");}

我们按照条数来摘要，达到 8 条触发摘要，保留 4 条，前面的变成摘要

> 🎬 视频演示（原公众号视频）

这样摘要后聊的再多也只保留最新的几条，更前面的都变成摘要了：

![image](../IMG/2026-05-23_DeepAgents：开箱即用的skill、上下文压缩等middleware/5_公众号_Yi昭.png)

有三种触发摘要的方式：

![image](../IMG/2026-05-23_DeepAgents：开箱即用的skill、上下文压缩等middleware/6_公众号_Yi昭.png)

这个 middleware 也是很有用的。

至此，我们就把 deepagents 提供的 middleware 过了一遍。

> 🎬 视频演示（原公众号视频）

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

DeepAgents 提供了很多开箱即用的功能，做 Agent 可以直接用。

这节我们学了它的各种 middleware。

middleware 是 createAgent 提供的机制，可以在大模型调用前后、tool 调用前后加一些逻辑，修改 state、参数、扩展 tool 等。

DeepAgents 提供了 skill、上下文压缩、长期记忆（md）、文件系统、subagent 的 middleware，直接用很方便。

当然，DeepAgents 不只有中间件，下节我们继续来学习其他功能。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-05-23 22:45:08 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247486117&idx=1&sn=c66273b7af6f0fbeceeec8baa565dfff&chksm=f1f036c0e52d14e1ce4c005d7dbaee177f8a6fc4b4c8f48ad98525ef64f6a07390ec9b164a2b&scene=27#wechat_redirect
