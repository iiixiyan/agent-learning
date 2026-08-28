# 给 Agent 加上语音交互：ASR + 流式 TTS

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年3月28日 11:08

我们常用的 Agent 都有语音功能。

比如你用豆包的时候：

> 🎬 视频演示（原公众号视频）

语音输入会转成文字，大模型的回答会通过语音朗读。可以切换音色。

这种 STT（Speech To Text）语音转文字，TTS（Text To Speech）文字转语音基本是 Agent 开发必备技术了。

这节我们就来学一下语音相关技术，实现豆包同款功能。

创建项目：

    mkdir tts-stt-testcd&nbsp;tts-stt-testnpm init -y

我们用腾讯云的语音（各家用法都差不多）。

https://console.cloud.tencent.com/tts

> 🎬 视频演示（原公众号视频）

拿到 secretId、secretKey 之后，就可以调用 api 了。

创建 src/tts-test.mjs

    import&nbsp;"dotenv/config";import&nbsp;tencentcloud&nbsp;from"tencentcloud-sdk-nodejs-tts";import&nbsp;fs&nbsp;from"node:fs";const&nbsp;secretId = process.env.SECRET_ID;const&nbsp;secretKey = process.env.SECRET_KEY;const&nbsp;TtsClient = tencentcloud.tts.v20190823.Client;const&nbsp;client =&nbsp;new&nbsp;TtsClient({credential: {&nbsp; &nbsp; secretId,&nbsp; &nbsp; secretKey,&nbsp; },region:&nbsp;"ap-beijing",profile: {&nbsp; &nbsp;&nbsp;httpProfile: {&nbsp; &nbsp; &nbsp;&nbsp;endpoint:&nbsp;"tts.tencentcloudapi.com",&nbsp; &nbsp; },&nbsp; },});const&nbsp;params = {Text:&nbsp;"下班路上，我还在为晚霞开心。突然电话响起：系统崩了。我的心一下揪紧，冲进办公室时几乎要绝望。可当大家一起排查、重启，屏幕终于恢复正常，我长长松了口气，笑着说：还好，我们没放弃。", &nbsp;// 要合成的文本SessionId:&nbsp;"session-001",VoiceType:&nbsp;502006, &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;// 101007：智瑜（女声）Codec:&nbsp;"mp3", &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 指定输出格式为 mp3};client.TextToVoice(params).then((data) =&gt;&nbsp;{&nbsp; &nbsp;&nbsp;// 返回的 Audio 字段是 Base64 编码的音频数据&nbsp; &nbsp;&nbsp;const&nbsp;audioBuffer = Buffer.from(data.Audio,&nbsp;"base64");&nbsp; &nbsp;&nbsp;const&nbsp;outputPath =&nbsp;"./output.mp3";&nbsp; &nbsp; fs.writeFile(outputPath, audioBuffer, (err) =&gt; {&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(err) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.error("保存文件失败：", err);&nbsp; &nbsp; &nbsp; }&nbsp;else&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log("MP3 已保存至：", outputPath);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; });&nbsp; },&nbsp; (err) =&gt; {&nbsp; &nbsp;&nbsp;console.error("合成失败：", err);&nbsp; });

调用文字转语音 tts 功能，传入参数，返回的 base64 字符串转为 buffer 写入文件。

这个音色 id 从这里找：

https://cloud.tencent.com/document/product/1073/92668

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/0_公众号_Yi昭.png)

安装用到的包：

    ppip install dotenv tencentcloud-sdk-nodejs-tts

创建 .env 配置文件：

    SECRET_ID=替换成你的SECRET_KEY=替换成你的

跑一下：

> 🎬 视频演示（原公众号视频）

但这种直接传入全部文本生成语音的方式，显然不太适合我们的场景。

比如豆包流式返回回答，语音也是流式播放的。

这种就需要用流式语音合成接口了，它是 websocket 的

https://cloud.tencent.com/document/product/1073/108595

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/1_公众号_Yi昭.png)

创建 src/streaming-tts-test.mjs

    import&nbsp;"dotenv/config";import&nbsp;WebSocket&nbsp;from"ws";import&nbsp;crypto&nbsp;from"node:crypto";import&nbsp;fs&nbsp;from"node:fs";const&nbsp;SECRET_ID = process.env.SECRET_ID;const&nbsp;SECRET_KEY = process.env.SECRET_KEY;const&nbsp;APP_ID = process.env.APP_ID;const&nbsp;VOICE_TYPE =&nbsp;101001;const&nbsp;OUTPUT_FILE =&nbsp;"output3.mp3";const&nbsp;TEXT_INTERVAL_MS =&nbsp;3000;const&nbsp;TEXTS = ["傍晚我还在为晚霞开心，","突然接到电话说系统崩了，","我心里一沉冲回办公室，","好在大家一起排查后终于恢复，","我长长松了口气。",];const&nbsp;sleep =&nbsp;(ms) =&gt;newPromise((resolve) =&gt;&nbsp;setTimeout(resolve, ms));function&nbsp;buildWsUrl()&nbsp;{const&nbsp;now =&nbsp;Math.floor(Date.now() /&nbsp;1000);const&nbsp;sessionId =&nbsp;`session_${now}_${Math.random().toString(36).slice(2)}`;const&nbsp;params = {&nbsp; &nbsp;&nbsp;Action:&nbsp;"TextToStreamAudioWSv2",&nbsp; &nbsp;&nbsp;AppId:&nbsp;parseInt(APP_ID),&nbsp; &nbsp;&nbsp;Codec:&nbsp;"mp3",&nbsp; &nbsp;&nbsp;Expired: now +&nbsp;3600,&nbsp; &nbsp;&nbsp;SampleRate:&nbsp;16000,&nbsp; &nbsp;&nbsp;SecretId: SECRET_ID,&nbsp; &nbsp;&nbsp;SessionId: sessionId,&nbsp; &nbsp;&nbsp;Speed:&nbsp;0,&nbsp; &nbsp;&nbsp;Timestamp: now,&nbsp; &nbsp;&nbsp;VoiceType: VOICE_TYPE,&nbsp; &nbsp;&nbsp;Volume:&nbsp;5,&nbsp; };const&nbsp;sortedKeys =&nbsp;Object.keys(params).sort();const&nbsp;signStr = sortedKeys.map((k) =&gt;`${k}=${params[k]}`).join("&amp;");const&nbsp;rawStr =&nbsp;`GETtts.cloud.tencent.com/stream_wsv2?${signStr}`;const&nbsp;signature = crypto&nbsp; &nbsp; .createHmac("sha1", SECRET_KEY)&nbsp; &nbsp; .update(rawStr)&nbsp; &nbsp; .digest("base64");const&nbsp;searchParams =&nbsp;new&nbsp;URLSearchParams({&nbsp; &nbsp; ...params,&nbsp; &nbsp;&nbsp;Signature: signature,&nbsp; });return&nbsp;{&nbsp; &nbsp; sessionId,&nbsp; &nbsp;&nbsp;url:&nbsp;`wss://tts.cloud.tencent.com/stream_wsv2?${searchParams.toString()}`,&nbsp; };}asyncfunction&nbsp;sendTexts(ws, sessionId)&nbsp;{for&nbsp;(let&nbsp;i =&nbsp;0; i &lt; TEXTS.length; i++) {&nbsp; &nbsp; ws.send(JSON.stringify({&nbsp;session_id: sessionId,&nbsp;message_id:&nbsp;`msg_${i}`,&nbsp;action:&nbsp;"ACTION_SYNTHESIS",&nbsp;data: TEXTS[i] }));&nbsp; &nbsp;&nbsp;console.log(`[文本] 已发送:&nbsp;${TEXTS[i]}`);&nbsp; &nbsp;&nbsp;if&nbsp;(i &lt; TEXTS.length -&nbsp;1)&nbsp;await&nbsp;sleep(TEXT_INTERVAL_MS);&nbsp; }&nbsp; ws.send(JSON.stringify({&nbsp;session_id: sessionId,&nbsp;action:&nbsp;"ACTION_COMPLETE"&nbsp;}));console.log("[文本] 已发送 ACTION_COMPLETE");}function&nbsp;streamTTS()&nbsp;{if&nbsp;(!SECRET_ID || !SECRET_KEY || !APP_ID) {&nbsp; &nbsp;&nbsp;thrownewError("请先在 .env 配置 SECRET_ID、SECRET_KEY、APP_ID");&nbsp; }const&nbsp;{ url, sessionId } = buildWsUrl();const&nbsp;ws =&nbsp;new&nbsp;WebSocket(url);const&nbsp;writeStream = fs.createWriteStream(OUTPUT_FILE, {&nbsp;flags:&nbsp;"w"&nbsp;});let&nbsp;totalBytes =&nbsp;0;let&nbsp;closed =&nbsp;false;let&nbsp;sent =&nbsp;false;const&nbsp;closeAll =&nbsp;()&nbsp;=&gt;&nbsp;{&nbsp; &nbsp;&nbsp;if&nbsp;(closed)&nbsp;return;&nbsp; &nbsp; closed =&nbsp;true;&nbsp; &nbsp; writeStream.end(()&nbsp;=&gt;&nbsp;{&nbsp; &nbsp; &nbsp;&nbsp;console.log(`[保存] 音频已保存至&nbsp;${OUTPUT_FILE}，共&nbsp;${totalBytes}&nbsp;字节`);&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;if&nbsp;(ws.readyState &lt; WebSocket.CLOSING) ws.close();&nbsp; };&nbsp; ws.on("open", () =&gt; {&nbsp; &nbsp;&nbsp;console.log("[连接] WebSocket 已建立，等待服务端就绪...");&nbsp; });&nbsp; ws.on("message",&nbsp;async&nbsp;(data, isBinary) =&gt; {&nbsp; &nbsp;&nbsp;if&nbsp;(isBinary) {&nbsp; &nbsp; &nbsp; writeStream.write(data);&nbsp; &nbsp; &nbsp; totalBytes += data.length;&nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;msg =&nbsp;JSON.parse(data.toString());&nbsp; &nbsp; &nbsp;&nbsp;console.log("[消息]",&nbsp;JSON.stringify(msg));&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(msg.ready ===&nbsp;1&nbsp;&amp;&amp; !sent) {&nbsp; &nbsp; &nbsp; &nbsp; sent =&nbsp;true;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;await&nbsp;sendTexts(ws, sessionId);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(msg.code &amp;&amp; msg.code !==&nbsp;0) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.error(`[错误] code=${msg.code}, message=${msg.message}`);&nbsp; &nbsp; &nbsp; &nbsp; closeAll();&nbsp; &nbsp; &nbsp; }&nbsp;elseif&nbsp;(msg.final ===&nbsp;1) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log("[完成] 合成结束。");&nbsp; &nbsp; &nbsp; &nbsp; closeAll();&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; }&nbsp;catch&nbsp;(e) {&nbsp; &nbsp; &nbsp;&nbsp;console.error("[解析错误]", e.message);&nbsp; &nbsp; }&nbsp; });&nbsp; ws.on("error", (err) =&gt; {&nbsp; &nbsp;&nbsp;console.error("[WebSocket 错误]", err.message);&nbsp; &nbsp; closeAll();&nbsp; });&nbsp; ws.on("close", (code, reason) =&gt; {&nbsp; &nbsp;&nbsp;console.log(`[断开] 连接已关闭，code=${code}, reason=${reason}`);&nbsp; &nbsp; closeAll();&nbsp; });}streamTTS();

我们先构造了 url，用 WebSocket 连上

每 3s 发送一次消息

然后用 fs.createWriteStream 异步写入文件

appid 从这里拿：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/2_公众号_Yi昭.png)

加到 .env 里：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/3_公众号_Yi昭.png)

跑一下：

> 🎬 视频演示（原公众号视频）

因为文本是流式返回的，所以语音一般也要流式生成，用 streaming tts 的接口。

接下来试一下语音识别 ASR（Automatic Speech Recognition），叫 STT （Speech To Text） 也可以，但 ASR 用的多一些。

这个就不用流式了。你平时用豆包的时候，都是说完一段话才转成的文本

创建 src/asr-test.mjs

    import&nbsp;"dotenv/config";import&nbsp;tencentcloud&nbsp;from"tencentcloud-sdk-nodejs";import&nbsp;fs&nbsp;from"node:fs";const&nbsp;SECRET_ID = process.env.SECRET_ID;const&nbsp;SECRET_KEY = process.env.SECRET_KEY;const&nbsp;AsrClient = tencentcloud.asr.v20190614.Client;const&nbsp;AUDIO_FILE =&nbsp;'./output.mp3';const&nbsp;client =&nbsp;new&nbsp;AsrClient({credential: {&nbsp; &nbsp;&nbsp;secretId: SECRET_ID,&nbsp; &nbsp;&nbsp;secretKey: SECRET_KEY,&nbsp; },region:&nbsp;"ap-shanghai",profile: {&nbsp; &nbsp;&nbsp;httpProfile: {&nbsp; &nbsp; &nbsp;&nbsp;reqMethod:&nbsp;"POST",&nbsp; &nbsp; &nbsp;&nbsp;reqTimeout:&nbsp;30,&nbsp; &nbsp; },&nbsp; },});asyncfunction&nbsp;run()&nbsp;{const&nbsp;audioBase64 = fs.readFileSync(AUDIO_FILE).toString("base64");const&nbsp;params = {&nbsp; &nbsp;&nbsp;EngSerViceType:&nbsp;"16k_zh",&nbsp; &nbsp;&nbsp;SourceType:&nbsp;1,&nbsp; &nbsp;&nbsp;Data: audioBase64,&nbsp; &nbsp;&nbsp;DataLen: Buffer.byteLength(audioBase64),&nbsp; &nbsp;&nbsp;VoiceFormat:&nbsp;"mp3",&nbsp; };try&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;data =&nbsp;await&nbsp;client.SentenceRecognition(params);&nbsp; &nbsp;&nbsp;console.log("识别结果：", data.Result);&nbsp; }&nbsp;catch&nbsp;(err) {&nbsp; &nbsp;&nbsp;console.error("识别失败：", err);&nbsp; }}run();

传入音频 mp3 文件，调用接口来识别，返回文本

安装下依赖：

    ppip install tencentcloud-sdk-nodejs

跑一下：

> 🎬 视频演示（原公众号视频）

这样，我们就可以来实现豆包同款的语音交互了：

点击录音，输入一段语音，服务端提供接口来转文字，之后用大模型生成回答。

流式 SSE 返回文字，同时用 WebSocket 返回流式语音。

这样就可以实现语音输入，流式的文字、语音输出。

为啥不直接用 SSE 返回音频数据呢？

因为 SSE 是基于 http 的文本协议，需要转 Base64 才行，传这种二进制数据还是 WebSocket 更合适。

思路理清了，接下来按照这个实现下豆包同款交互：

先创建后端项目：

    nest new asr-and-tts-nest-service

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/4_公众号_Yi昭.png)

先写一下调用大模型回答的 SSE 接口

创建 ai 模块：

    nest g module ainest g router ai --no-specnest g service ai --no-spec

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/5_公众号_Yi昭.png)

改下 AiService：

    import&nbsp;{ Inject, Injectable }&nbsp;from'fastapi.common';import&nbsp;{ ChatOpenAI }&nbsp;from'langchain_openai';import&nbsp;{ PromptTemplate }&nbsp;from'langchain_core/prompts';import&nbsp;type { Runnable }&nbsp;from'langchain_core/runnables';import&nbsp;{ StringOutputParser }&nbsp;from'langchain_core/output_parsers';@Injectable()exportclass&nbsp;AiService&nbsp;{&nbsp; private readonly chain: Runnable;constructor(&nbsp; &nbsp; @Inject('CHAT_MODEL') model: ChatOpenAI&nbsp; ) {&nbsp; &nbsp;&nbsp;const&nbsp;prompt = PromptTemplate.fromTemplate(&nbsp; &nbsp; &nbsp;&nbsp;'请回答以下问题：\n\n{query}',&nbsp; &nbsp; );&nbsp; &nbsp;&nbsp;this.chain = prompt.pipe(model).pipe(new&nbsp;StringOutputParser());&nbsp; }async&nbsp;*streamChain(query: string): AsyncGenerator&lt;string&gt; {&nbsp; &nbsp;&nbsp;const&nbsp;stream =&nbsp;awaitthis.chain.stream({ query });&nbsp; &nbsp;&nbsp;forawait&nbsp;(const&nbsp;chunk&nbsp;of&nbsp;stream) {&nbsp; &nbsp; &nbsp;&nbsp;yield&nbsp;chunk;&nbsp; &nbsp; }&nbsp; }}

还有 AiRouter：

    import&nbsp;{ Router, Get, Query, Sse }&nbsp;from'fastapi.common';import&nbsp;{&nbsp;from, map, Observable }&nbsp;from'rxjs';import&nbsp;{ AiService }&nbsp;from'./ai.service';@Router('ai')exportclass&nbsp;AiRouter&nbsp;{constructor(private readonly aiService: AiService) {}&nbsp; @Sse('chat/stream')&nbsp; chatStream(@Query('query') query: string): Observable&lt;{&nbsp;data: string }&gt; {&nbsp; &nbsp;&nbsp;returnfrom(this.aiService.streamChain(query)).pipe(&nbsp; &nbsp; &nbsp; map((chunk) =&gt;&nbsp;({&nbsp;data: chunk }))&nbsp; &nbsp; );&nbsp; }}

和 AiModule：

    import&nbsp;{ Module }&nbsp;from'fastapi.common';import&nbsp;{ AiService }&nbsp;from'./ai.service';import&nbsp;{ AiRouter }&nbsp;from'./ai.router';import&nbsp;{ ConfigService }&nbsp;from'fastapi.config';import&nbsp;{ ChatOpenAI }&nbsp;from'langchain_openai';@Module({routers: [AiRouter],providers: [AiService,&nbsp; &nbsp; {&nbsp; &nbsp; &nbsp;&nbsp;provide:&nbsp;'CHAT_MODEL',&nbsp; &nbsp; &nbsp;&nbsp;useFactory:&nbsp;(configService: ConfigService) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;returnnew&nbsp;ChatOpenAI({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;model: configService.get('MODEL_NAME'),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;apiKey: configService.get('OPENAI_API_KEY'),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;configuration: {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;baseURL: configService.get('OPENAI_BASE_URL'),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp;&nbsp;inject: [ConfigService],&nbsp; &nbsp; }&nbsp; ],})exportclass&nbsp;AiModule&nbsp;{}

就是基于用 langchain 创建一个 chain 来回答用户的问题，流式返回

安装用到的包：

    ppip install fastapi.config langchain_openai langchain_core

创建配置文件 .env

    OPENAI_API_KEY=sk-xxxOPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1MODEL_NAME=qwen-plus

在 AppModule 里引入下：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/6_公众号_Yi昭.png)

这些前面写过，就是再熟悉一遍。

跑一下：

> 🎬 视频演示（原公众号视频）

我们先接入语音转文字，实现一个接口：

创建 speech 模块：

    nest g module speechnest g service speech --no-specnest g router speech --no-spec

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/7_公众号_Yi昭.png)

把之前 asr 的逻辑拿过来，放到 service 里：

    import&nbsp;{ Inject, Injectable }&nbsp;from'fastapi.common';import&nbsp;type *&nbsp;as&nbsp;tencentcloud&nbsp;from'tencentcloud-sdk-nodejs';type UploadedAudio = {buffer: Buffer;&nbsp; originalname: string;&nbsp; mimetype: string;&nbsp; size: number;};type AsrClient = InstanceType&lt;typeof&nbsp;tencentcloud.asr.v20190614.Client&gt;;@Injectable()exportclass&nbsp;SpeechService&nbsp;{constructor(@Inject('ASR_CLIENT') private readonly asrClient: AsrClient) {}async&nbsp;recognizeBySentence(file: UploadedAudio):&nbsp;Promise&lt;string&gt; {&nbsp; &nbsp;&nbsp;const&nbsp;audioBase64 = file.buffer.toString('base64');&nbsp; &nbsp;&nbsp;const&nbsp;result =&nbsp;awaitthis.asrClient.SentenceRecognition({&nbsp; &nbsp; &nbsp;&nbsp;EngSerViceType:&nbsp;'16k_zh',&nbsp; &nbsp; &nbsp;&nbsp;SourceType:&nbsp;1,&nbsp; &nbsp; &nbsp;&nbsp;Data: audioBase64,&nbsp; &nbsp; &nbsp;&nbsp;DataLen: file.buffer.length,&nbsp; &nbsp; &nbsp;&nbsp;VoiceFormat:&nbsp;'ogg-opus',&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;return&nbsp;result.Result ??&nbsp;'';&nbsp; }}

把传过来的 buffer 转成 base64 字符串，用 asrClient 的 SentenceRecognition 方法来识别成文字返回。

SpeechModule 里创建 AsrClient：

    import&nbsp;{ Module }&nbsp;from'fastapi.common';import&nbsp;{ ConfigService }&nbsp;from'fastapi.config';import&nbsp;{ SpeechService }&nbsp;from'./speech.service';import&nbsp;{ SpeechRouter }&nbsp;from'./speech.router';import&nbsp;*&nbsp;as&nbsp;tencentcloud&nbsp;from'tencentcloud-sdk-nodejs';const&nbsp;AsrClient = tencentcloud.asr.v20190614.Client;@Module({providers: [&nbsp; &nbsp; SpeechService,&nbsp; &nbsp; {&nbsp; &nbsp; &nbsp;&nbsp;provide:&nbsp;'ASR_CLIENT',&nbsp; &nbsp; &nbsp;&nbsp;useFactory:&nbsp;(configService: ConfigService) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;returnnew&nbsp;AsrClient({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;credential: {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;secretId: configService.get&lt;string&gt;('SECRET_ID'),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;secretKey: configService.get&lt;string&gt;('SECRET_KEY'),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;region:&nbsp;'ap-shanghai',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;profile: {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;httpProfile: {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;reqMethod:&nbsp;'POST',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;reqTimeout:&nbsp;30,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp;&nbsp;inject: [ConfigService],&nbsp; &nbsp; },&nbsp; ],routers: [SpeechRouter],})exportclass&nbsp;SpeechModule&nbsp;{}

然后在 SpeechRouter 里加一个接口：

    import&nbsp;{&nbsp; BadRequestException,&nbsp; Router,&nbsp; Post,&nbsp; UploadedFile,&nbsp; UseInterceptors,}&nbsp;from'fastapi.common';import&nbsp;{ FileInterceptor }&nbsp;from'fastapi.platform-express';import&nbsp;{ SpeechService }&nbsp;from'./speech.service';@Router('speech')exportclass&nbsp;SpeechRouter&nbsp;{constructor(private readonly speechService: SpeechService) {}&nbsp; @Post('asr')&nbsp; @UseInterceptors(FileInterceptor('audio'))async&nbsp;recognize(&nbsp; &nbsp; @UploadedFile()&nbsp; &nbsp; file?: {&nbsp; &nbsp; &nbsp;&nbsp;buffer: Buffer;&nbsp; &nbsp; &nbsp; originalname: string;&nbsp; &nbsp; &nbsp; mimetype: string;&nbsp; &nbsp; &nbsp; size: number;&nbsp; &nbsp; },&nbsp; ) {&nbsp; &nbsp;&nbsp;if&nbsp;(!file?.buffer?.length) {&nbsp; &nbsp; &nbsp;&nbsp;thrownew&nbsp;BadRequestException(&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'请通过 FormData 的 audio 字段上传音频文件',&nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;const&nbsp;text =&nbsp;awaitthis.speechService.recognizeBySentence(file);&nbsp; &nbsp;&nbsp;return&nbsp;{ text };&nbsp; }}

这里 @UseInterceptors 装饰器是使用 FileInterceptor 这个拦截器取表单的 audio 字段。

然后通过 @UploadedFile 取出来作为参数传入 handler

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/8_公众号_Yi昭.png)

Router 里有很多 handler 方法

拦截器 interceptor 是可以动态的添加一些 handler 前后的处理逻辑。

比如这里 FileInterceptor 就是解析表单里的文件二进制数据，转成 File 对象

接口写完了，我们来测一下。

配置放到 .env 里

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/9_公众号_Yi昭.png)

加个页面：

public/asr.html

    &lt;!doctype&nbsp;html&gt;&lt;html&nbsp;lang="zh-CN"&gt;&lt;head&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;charset="UTF-8"&nbsp;/&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;name="viewport"&nbsp;content="width=device-width, initial-scale=1.0"&nbsp;/&gt;&nbsp; &nbsp;&nbsp;&lt;title&gt;ASR 录音测试&lt;/title&gt;&nbsp; &nbsp;&nbsp;&lt;style&gt;&nbsp; &nbsp; &nbsp;&nbsp;body&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-family: -apple-system, BlinkMacSystemFont,&nbsp;"Segoe UI", sans-serif;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;max-width:&nbsp;720px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin:&nbsp;40px&nbsp;auto;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;016px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;line-height:&nbsp;1.6;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;button&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-right:&nbsp;8px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-bottom:&nbsp;8px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;8px14px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.status&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin:&nbsp;12px0;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#444;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;pre&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#f6f8fa;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;12px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;6px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;white-space: pre-wrap;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;&lt;/style&gt;&lt;/head&gt;&lt;body&gt;&nbsp; &nbsp;&nbsp;&lt;h1&gt;ASR 录音上传测试&lt;/h1&gt;&nbsp; &nbsp;&nbsp;&lt;p&gt;点击开始录音，结束后自动上传到&nbsp;&lt;code&gt;/speech/asr&lt;/code&gt;。&lt;/p&gt;&nbsp; &nbsp;&nbsp;&lt;button&nbsp;id="startBtn"&gt;开始录音&lt;/button&gt;&nbsp; &nbsp;&nbsp;&lt;button&nbsp;id="stopBtn"&nbsp;disabled&gt;停止并上传&lt;/button&gt;&nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="status"&nbsp;id="status"&gt;状态：未开始&lt;/div&gt;&nbsp; &nbsp;&nbsp;&lt;h3&gt;识别结果&lt;/h3&gt;&nbsp; &nbsp;&nbsp;&lt;pre&nbsp;id="result"&gt;（暂无）&lt;/pre&gt;&nbsp; &nbsp;&nbsp;&lt;script&gt;&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;startBtn =&nbsp;document.getElementById("startBtn");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;stopBtn =&nbsp;document.getElementById("stopBtn");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;statusEl =&nbsp;document.getElementById("status");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;resultEl =&nbsp;document.getElementById("result");&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;mediaRecorder =&nbsp;null;&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;chunks = [];&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;recordFilename =&nbsp;"record.ogg";&nbsp; &nbsp; &nbsp;&nbsp;function&nbsp;setStatus(text)&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; statusEl.textContent =&nbsp;"状态："&nbsp;+ text;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; startBtn.addEventListener("click",&nbsp;async&nbsp;() =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;stream =&nbsp;await&nbsp;navigator.mediaDevices.getUserMedia({&nbsp;audio:&nbsp;true&nbsp;});&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; chunks = [];&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;preferredMimeType =&nbsp;"audio/ogg;codecs=opus";&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mediaRecorder = MediaRecorder.isTypeSupported(preferredMimeType)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ?&nbsp;new&nbsp;MediaRecorder(stream, {&nbsp;mimeType: preferredMimeType })&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; :&nbsp;new&nbsp;MediaRecorder(stream);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mediaRecorder.ondataavailable =&nbsp;(event) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(event.data &amp;&amp; event.data.size &gt;&nbsp;0) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; chunks.push(event.data);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mediaRecorder.onstop =&nbsp;async&nbsp;() =&gt; {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("录音结束，正在上传...");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;blob =&nbsp;new&nbsp;Blob(chunks, {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type: mediaRecorder.mimeType ||&nbsp;"audio/webm",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!blob.size) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;thrownewError("录音数据为空，请至少录制 1 秒再上传");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;formData =&nbsp;new&nbsp;FormData();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; formData.append("audio", blob, recordFilename);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;response =&nbsp;await&nbsp;fetch("/speech/asr", {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;method:&nbsp;"POST",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;body: formData,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!response.ok) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;text =&nbsp;await&nbsp;response.text();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;thrownewError(text ||&nbsp;"请求失败");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;data =&nbsp;await&nbsp;response.json();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; resultEl.textContent = data.text ||&nbsp;"（空结果）";&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("上传完成");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("上传失败");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; resultEl.textContent =&nbsp;"错误："&nbsp;+ (error.message ||&nbsp;String(error));&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;finally&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; stream.getTracks().forEach((t) =&gt;&nbsp;t.stop());&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mediaRecorder.start(250);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("录音中...");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; startBtn.disabled =&nbsp;true;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; stopBtn.disabled =&nbsp;false;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("无法开始录音");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; resultEl.textContent =&nbsp;"错误："&nbsp;+ (error.message ||&nbsp;String(error));&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; stopBtn.addEventListener("click", () =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!mediaRecorder)&nbsp;return;&nbsp; &nbsp; &nbsp; &nbsp; mediaRecorder.stop();&nbsp; &nbsp; &nbsp; &nbsp; startBtn.disabled =&nbsp;false;&nbsp; &nbsp; &nbsp; &nbsp; stopBtn.disabled =&nbsp;true;&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;&lt;/script&gt;&lt;/body&gt;&lt;/html&gt;

这里就是用 MediaRecorder 录音

把 chunks 数组转成 Blob 对象，作为 FormData 的表单项发送。

在 AppModule 里支持下静态文件访问：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/10_公众号_Yi昭.png)

安装用到的依赖：

    ppip install tencentcloud-sdk-nodejs fastapi.serve-static

跑一下：

> 🎬 视频演示（原公众号视频）

语音识别出文字，之后可以自动调用 /ai/chat/stream 接口拿到回答。

创建一个新的 html，这里用 ai 生成和豆包类似的界面。

（不用看样式，就是录音 + 调用 SSE 接口）

    &lt;!doctype&nbsp;html&gt;&lt;html&nbsp;lang="zh-CN"&gt;&lt;head&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;charset="UTF-8"&nbsp;/&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;name="viewport"&nbsp;content="width=device-width, initial-scale=1.0"&nbsp;/&gt;&nbsp; &nbsp;&nbsp;&lt;title&gt;AI 助手&lt;/title&gt;&nbsp; &nbsp;&nbsp;&lt;style&gt;&nbsp; &nbsp; &nbsp;&nbsp;:root&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;--bg:&nbsp;#f3f4f7;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;--card:&nbsp;#ffffff;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;--text:&nbsp;#1f2329;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;--muted:&nbsp;#6b7280;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;--primary:&nbsp;#3b82f6;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;--primary-soft:&nbsp;#e8f1ff;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;--assistant:&nbsp;#f8fafc;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;--border:&nbsp;#e5e7eb;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;--shadow:&nbsp;014px40pxrgba(15,&nbsp;23,&nbsp;42,&nbsp;0.08);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; * {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;box-sizing: border-box;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;body&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin:&nbsp;0;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;min-height:&nbsp;100vh;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-family: -apple-system, BlinkMacSystemFont,&nbsp;"Segoe UI",&nbsp;"PingFang SC",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"Hiragino Sans GB",&nbsp;"Microsoft YaHei", sans-serif;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;var(--text);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;radial-gradient(circle at top,&nbsp;#ffffff&nbsp;0%, var(--bg)&nbsp;45%);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.page&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;max-width:&nbsp;920px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin:&nbsp;28px&nbsp;auto;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;014px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.chat-shell&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;1px&nbsp;solid&nbsp;var(--border);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;22px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;var(--card);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;box-shadow:&nbsp;var(--shadow);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;overflow: hidden;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.header&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;18px20px14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-bottom:&nbsp;1px&nbsp;solid&nbsp;var(--border);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;linear-gradient(180deg,&nbsp;#ffffff&nbsp;0%,&nbsp;#fafbfd&nbsp;100%);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.title&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin:&nbsp;0;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;20px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-weight:&nbsp;700;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.subtitle&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-top:&nbsp;6px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;var(--muted);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;13px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.status-pill&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;display: inline-flex;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-top:&nbsp;10px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;5px10px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;999px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#f5f7fb;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#4b5563;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;12px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.messages&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;22px16px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;min-height:&nbsp;430px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;max-height:&nbsp;62vh;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;overflow-y: auto;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;linear-gradient(transparent&nbsp;95%, rgba(0,&nbsp;0,&nbsp;0,&nbsp;0.02)&nbsp;100%),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;#fcfdff;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.empty&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;text-align: center;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;var(--muted);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-top:&nbsp;38px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;14px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.msg-row&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;display: flex;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-bottom:&nbsp;14px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.msg-row.user&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;justify-content: flex-end;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.msg-row.assistant&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;justify-content: flex-start;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.bubble&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;max-width:&nbsp;min(680px,&nbsp;84%);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;16px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;12px14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;white-space: pre-wrap;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;line-height:&nbsp;1.55;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;1px&nbsp;solid&nbsp;var(--border);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.msg-row.user.bubble&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;var(--primary-soft);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-color:&nbsp;#cfe2ff;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.msg-row.assistant.bubble&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;var(--assistant);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.meta&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-top:&nbsp;5px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;12px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#8a93a1;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.composer&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-top:&nbsp;1px&nbsp;solid&nbsp;var(--border);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#ffffff;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.toolbar&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;display: flex;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;gap:&nbsp;10px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;align-items: flex-end;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.input-wrap&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;flex:&nbsp;1;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;1px&nbsp;solid&nbsp;var(--border);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;10px12px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#fff;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.prompt-input&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;width:&nbsp;100%;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border: none;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;outline: none;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;resize: none;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;min-height:&nbsp;44px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;max-height:&nbsp;130px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;line-height:&nbsp;1.55;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-family: inherit;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.btn&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;1px&nbsp;solid&nbsp;var(--border);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#fff;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;var(--text);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;10px14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;11px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;cursor: pointer;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.btn:disabled&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;opacity:&nbsp;0.5;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;cursor: not-allowed;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.btn-primary&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;var(--primary);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-color:&nbsp;var(--primary);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#fff;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.btn-voice&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;min-width:&nbsp;96px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.hint&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-top:&nbsp;10px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;var(--muted);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;12px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.typing::after&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;content:&nbsp;"● ● ●";&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-left:&nbsp;8px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;letter-spacing:&nbsp;2px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;11px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#94a3b8;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;animation: pulse&nbsp;1s&nbsp;ease-in-out infinite;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;@keyframes&nbsp;pulse {&nbsp; &nbsp; &nbsp; &nbsp; 0%, 100% {&nbsp;opacity:&nbsp;0.3; }&nbsp; &nbsp; &nbsp; &nbsp; 50% {&nbsp;opacity:&nbsp;1; }&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;&lt;/style&gt;&lt;/head&gt;&lt;body&gt;&nbsp; &nbsp;&nbsp;&lt;main&nbsp;class="page"&gt;&nbsp; &nbsp; &nbsp;&nbsp;&lt;section&nbsp;class="chat-shell"&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;header&nbsp;class="header"&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;h1&nbsp;class="title"&gt;AI 助手&lt;/h1&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="subtitle"&gt;录音后自动识别，再调用 AI 流式回复&lt;/div&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="status-pill"&nbsp;id="status"&gt;状态：未开始&lt;/div&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/header&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;section&nbsp;class="messages"&nbsp;id="messages"&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="empty"&nbsp;id="emptyTip"&gt;点击下方开始录音，体验语音问答。&lt;/div&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/section&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;footer&nbsp;class="composer"&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="toolbar"&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="input-wrap"&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;textarea&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;class="prompt-input"&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;id="promptInput"&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;placeholder="输入问题，回车发送（Shift+Enter 换行）；也可以用语音按钮说话"&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &gt;&lt;/textarea&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/div&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;button&nbsp;class="btn btn-voice"&nbsp;id="recordBtn"&gt;语音输入&lt;/button&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;button&nbsp;class="btn btn-primary"&nbsp;id="sendBtn"&gt;发送&lt;/button&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/div&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="hint"&gt;文字直问：/ai/chat/stream；语音链路：/speech/asr -&gt; /ai/chat/stream&lt;/div&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/footer&gt;&nbsp; &nbsp; &nbsp;&nbsp;&lt;/section&gt;&nbsp; &nbsp;&nbsp;&lt;/main&gt;&nbsp; &nbsp;&nbsp;&lt;script&gt;&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;promptInput =&nbsp;document.getElementById("promptInput");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;sendBtn =&nbsp;document.getElementById("sendBtn");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;recordBtn =&nbsp;document.getElementById("recordBtn");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;statusEl =&nbsp;document.getElementById("status");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;messagesEl =&nbsp;document.getElementById("messages");&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;emptyTipEl =&nbsp;document.getElementById("emptyTip");&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;mediaRecorder =&nbsp;null;&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;chunks = [];&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;activeStream =&nbsp;null;&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;activeAssistantContentEl =&nbsp;null;&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;activeAssistantMetaEl =&nbsp;null;&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;activeRecordStream =&nbsp;null;&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;isRecording =&nbsp;false;&nbsp; &nbsp; &nbsp;&nbsp;function&nbsp;setStatus(text, isTyping = false)&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; statusEl.textContent =&nbsp;"状态："&nbsp;+ text;&nbsp; &nbsp; &nbsp; &nbsp; statusEl.classList.toggle("typing", isTyping);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;function&nbsp;nowTime()&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;returnnewDate().toLocaleTimeString("zh-CN", {&nbsp;hour12:&nbsp;false&nbsp;});&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;function&nbsp;scrollToBottom()&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; messagesEl.scrollTop = messagesEl.scrollHeight;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;function&nbsp;hideEmptyTip()&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(emptyTipEl) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; emptyTipEl.style.display =&nbsp;"none";&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;function&nbsp;appendMessage(role, text, metaText)&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; hideEmptyTip();&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;row =&nbsp;document.createElement("div");&nbsp; &nbsp; &nbsp; &nbsp; row.className =&nbsp;"msg-row "&nbsp;+ role;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;bubble =&nbsp;document.createElement("div");&nbsp; &nbsp; &nbsp; &nbsp; bubble.className =&nbsp;"bubble";&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;content =&nbsp;document.createElement("div");&nbsp; &nbsp; &nbsp; &nbsp; content.textContent = text;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;meta =&nbsp;document.createElement("div");&nbsp; &nbsp; &nbsp; &nbsp; meta.className =&nbsp;"meta";&nbsp; &nbsp; &nbsp; &nbsp; meta.textContent = metaText || nowTime();&nbsp; &nbsp; &nbsp; &nbsp; bubble.appendChild(content);&nbsp; &nbsp; &nbsp; &nbsp; bubble.appendChild(meta);&nbsp; &nbsp; &nbsp; &nbsp; row.appendChild(bubble);&nbsp; &nbsp; &nbsp; &nbsp; messagesEl.appendChild(row);&nbsp; &nbsp; &nbsp; &nbsp; scrollToBottom();&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{ row, bubble, content, meta };&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;function&nbsp;closeActiveStream()&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(activeStream) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeStream.close();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeStream =&nbsp;null;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;function&nbsp;setRecordingUI(recording)&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; isRecording = recording;&nbsp; &nbsp; &nbsp; &nbsp; recordBtn.textContent = recording ?&nbsp;"停止录音"&nbsp;:&nbsp;"语音输入";&nbsp; &nbsp; &nbsp; &nbsp; recordBtn.classList.toggle("btn-primary", recording);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;asyncfunction&nbsp;uploadAndRecognize(blob)&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;formData =&nbsp;new&nbsp;FormData();&nbsp; &nbsp; &nbsp; &nbsp; formData.append("audio", blob,&nbsp;"record.ogg");&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;response =&nbsp;await&nbsp;fetch("/speech/asr", {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;method:&nbsp;"POST",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;body: formData,&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!response.ok) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;text =&nbsp;await&nbsp;response.text();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;thrownewError(text ||&nbsp;"ASR 请求失败");&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;data =&nbsp;await&nbsp;response.json();&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;data.text ||&nbsp;"";&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;function&nbsp;streamAiReply(query)&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;returnnewPromise((resolve) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; closeActiveStream();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;aiMsg = appendMessage("assistant",&nbsp;"",&nbsp;"AI 正在回答...");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeAssistantContentEl = aiMsg.content;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeAssistantMetaEl = aiMsg.meta;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;url =&nbsp;"/ai/chat/stream?query="&nbsp;+&nbsp;encodeURIComponent(query);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;es =&nbsp;new&nbsp;EventSource(url);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;aiResult =&nbsp;"";&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeStream = es;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; es.onmessage =&nbsp;(event) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; aiResult += event.data ||&nbsp;"";&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeAssistantContentEl.textContent = aiResult ||&nbsp;"（空结果）";&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; scrollToBottom();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; es.onerror =&nbsp;()&nbsp;=&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; es.close();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(activeStream === es) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeStream =&nbsp;null;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(activeAssistantMetaEl) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeAssistantMetaEl.textContent =&nbsp;"AI 回复完成 "&nbsp;+ nowTime();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; resolve(aiResult);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;asyncfunction&nbsp;askWithQuery(query, source)&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;trimmed = query.trim();&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!trimmed) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("请输入问题");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; appendMessage("user", trimmed, source +&nbsp;" "&nbsp;+ nowTime());&nbsp; &nbsp; &nbsp; &nbsp; promptInput.value =&nbsp;"";&nbsp; &nbsp; &nbsp; &nbsp; sendBtn.disabled =&nbsp;true;&nbsp; &nbsp; &nbsp; &nbsp; setStatus("AI 正在流式回答...",&nbsp;true);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;await&nbsp;streamAiReply(trimmed);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("对话完成");&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; appendMessage(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"assistant",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"处理失败："&nbsp;+ (error.message ||&nbsp;String(error)),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"异常 "&nbsp;+ nowTime(),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("处理失败");&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;finally&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; sendBtn.disabled =&nbsp;false;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; sendBtn.addEventListener("click",&nbsp;async&nbsp;() =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;await&nbsp;askWithQuery(promptInput.value,&nbsp;"文字提问");&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; promptInput.addEventListener("keydown",&nbsp;async&nbsp;(event) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(event.key ===&nbsp;"Enter"&nbsp;&amp;&amp; !event.shiftKey) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; event.preventDefault();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;await&nbsp;askWithQuery(promptInput.value,&nbsp;"文字提问");&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; recordBtn.addEventListener("click",&nbsp;async&nbsp;() =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(isRecording) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(mediaRecorder) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mediaRecorder.stop();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("已停止录音，正在识别...");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; closeActiveStream();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeRecordStream =&nbsp;await&nbsp;navigator.mediaDevices.getUserMedia({&nbsp;audio:&nbsp;true&nbsp;});&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; chunks = [];&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;preferredMimeType =&nbsp;"audio/ogg;codecs=opus";&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mediaRecorder = MediaRecorder.isTypeSupported(preferredMimeType)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ?&nbsp;new&nbsp;MediaRecorder(activeRecordStream, {&nbsp;mimeType: preferredMimeType })&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; :&nbsp;new&nbsp;MediaRecorder(activeRecordStream);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mediaRecorder.ondataavailable =&nbsp;(event) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(event.data &amp;&amp; event.data.size &gt;&nbsp;0) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; chunks.push(event.data);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mediaRecorder.onstop =&nbsp;async&nbsp;() =&gt; {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;blob =&nbsp;new&nbsp;Blob(chunks, {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type: mediaRecorder.mimeType ||&nbsp;"audio/webm",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!blob.size) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;thrownewError("录音数据为空，请至少录制 1 秒再上传");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("语音识别中...");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;recognized = (await&nbsp;uploadAndRecognize(blob)).trim();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; promptInput.value = recognized;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!recognized) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("识别为空，请重新录音");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;await&nbsp;askWithQuery(recognized,&nbsp;"语音提问");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; appendMessage(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"assistant",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"语音处理失败："&nbsp;+ (error.message ||&nbsp;String(error)),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"异常 "&nbsp;+ nowTime(),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("语音处理失败");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;finally&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(activeRecordStream) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeRecordStream.getTracks().forEach((t) =&gt;&nbsp;t.stop());&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; activeRecordStream =&nbsp;null;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setRecordingUI(false);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mediaRecorder.start(250);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setRecordingUI(true);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("录音中，点击“停止录音”完成提问");&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; appendMessage(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"assistant",&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"无法开始录音："&nbsp;+ (error.message ||&nbsp;String(error)),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"异常 "&nbsp;+ nowTime(),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus("无法开始录音");&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setRecordingUI(false);&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;&lt;/script&gt;&lt;/body&gt;&lt;/html&gt;

跑一下：

> 🎬 视频演示（原公众号视频）

接下来做一下流式语音朗读就可以了。

大概是这样的思路：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/11_公众号_Yi昭.png)

/ai/chat/stream 接口就是 SSE 流式返回文本的接口。

但是 SSE 是 http 的文本协议，二进制数据需要转成 base64 才行，但这样体积又会很大，所以不用 SSE 传二进制数据，我们单独一个 WebSocket 做语音的流式推送。

腾讯云的 streaming tts 接口是流式往那边推文本，流式返回语音数据。

我们在 SSE 接口生成流式文本的时候，通过事件的方式推送给 ws 接口，这里用腾讯云的流式语音接口生成语音数据后，推送给前端代码来播放语音。

总之，就是一个 SSE 通道，一个 WS 通道，SSE 返回流式文本，同时用 WS 流式返回语音。

事件通知用 fastapi.event-emitter 这个包。

安装下：

    ppip install fastapi.event-emitter

用法是这样：

AppModule 引入这个模块：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/12_公众号_Yi昭.png)

需要 emit 事件的地方这样：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/13_公众号_Yi昭.png)

注入这个 EventEmitter2 的实例，emit 一个事件。

然后需要处理这个事件的地方，用 OnEvent 监听下这个事件名：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/14_公众号_Yi昭.png)

那边 emit 这个事件的时候就会自动调用这个方法。

用起来超级简单。

然后我们来实现下流式语音的功能。

创建 src/speech/tts-relay.service.ts

这里主要是连接腾讯云的 streaming tts 接口来做语音合成。

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/15_公众号_Yi昭.png)

这个构造 ws 的 url 的方法之前讲过，不用细看。

然后用这个 url 连接上腾讯云的 tts 的 ws 服务

如果那边传过来的是二进制，就直接通过 websocket  发送给前端

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/16_公众号_Yi昭.png)

当然非二进制就作为 json 来处理：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/17_公众号_Yi昭.png)

非二进制用 JSON.parse 处理下，根据不同的类型，给前端返回不同的 json，比如 tts\_error、tts\_final 等。

然后收到事件的时候，根据事件类型做不同处理：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/18_公众号_Yi昭.png)

如果收到的是 start 事件，就和腾讯云的 tts 服务建立连接。

如果收到的是 chunk 事件，就把这段文本发送给 tts 服务

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/19_公众号_Yi昭.png)

这样，流程就走通了。

完整代码如下（不用细看，理解思路就行）：

speech/tts-relay.service.ts

    import&nbsp;{ Inject, Injectable, Logger, OnModuleDestroy }&nbsp;from'fastapi.common';import&nbsp;{ ConfigService }&nbsp;from'fastapi.config';import&nbsp;{ createHmac, randomUUID }&nbsp;from'node:crypto';import&nbsp;{ OnEvent }&nbsp;from'fastapi.event-emitter';import&nbsp;{ AI_TTS_STREAM_EVENT, type AiTtsStreamEvent }&nbsp;from'../common/stream-events';import&nbsp;WebSocket&nbsp;from'ws';type ClientSession = {sessionId: string;&nbsp; clientWs: WebSocket;&nbsp; tencentWs?: WebSocket;&nbsp; ready: boolean;&nbsp; pendingChunks: string[];&nbsp; closed: boolean;};@Injectable()exportclass&nbsp;TtsRelayService&nbsp;implements&nbsp;OnModuleDestroy&nbsp;{&nbsp; private readonly logger =&nbsp;new&nbsp;Logger(TtsRelayService.name);&nbsp; private readonly sessions =&nbsp;newMap&lt;string, ClientSession&gt;();&nbsp; private readonly secretId: string;&nbsp; private readonly secretKey: string;&nbsp; private readonly appId: number;&nbsp; private readonly voiceType: number;constructor(@Inject(ConfigService) configService: ConfigService) {&nbsp; &nbsp;&nbsp;this.secretId = configService.get&lt;string&gt;('SECRET_ID') ??&nbsp;'';&nbsp; &nbsp;&nbsp;this.secretKey = configService.get&lt;string&gt;('SECRET_KEY') ??&nbsp;'';&nbsp; &nbsp;&nbsp;this.appId =&nbsp;Number(configService.get&lt;string&gt;('APP_ID') ??&nbsp;0);&nbsp; &nbsp;&nbsp;this.voiceType =&nbsp;Number(configService.get&lt;string&gt;('TTS_VOICE_TYPE') ??&nbsp;101001);&nbsp; }&nbsp; onModuleDestroy():&nbsp;void&nbsp;{&nbsp; &nbsp;&nbsp;for&nbsp;(const&nbsp;session&nbsp;ofthis.sessions.values()) {&nbsp; &nbsp; &nbsp;&nbsp;this.closeSession(session.sessionId,&nbsp;'module destroy');&nbsp; &nbsp; }&nbsp; }&nbsp; registerClient(clientWs: WebSocket, wantedSessionId?: string): string {&nbsp; &nbsp;&nbsp;const&nbsp;sessionId = wantedSessionId?.trim() || randomUUID();&nbsp; &nbsp;&nbsp;const&nbsp;existing =&nbsp;this.sessions.get(sessionId);&nbsp; &nbsp;&nbsp;if&nbsp;(existing) {&nbsp; &nbsp; &nbsp;&nbsp;this.closeSession(sessionId,&nbsp;'client reconnected');&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;this.sessions.set(sessionId, {&nbsp; &nbsp; &nbsp; sessionId,&nbsp; &nbsp; &nbsp; clientWs,&nbsp; &nbsp; &nbsp;&nbsp;ready:&nbsp;false,&nbsp; &nbsp; &nbsp;&nbsp;pendingChunks: [],&nbsp; &nbsp; &nbsp;&nbsp;closed:&nbsp;false,&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;this.sendClientJson(clientWs, {&nbsp;type:&nbsp;'session', sessionId });&nbsp; &nbsp;&nbsp;this.logger.log(`TTS client connected:&nbsp;${sessionId}`);&nbsp; &nbsp;&nbsp;return&nbsp;sessionId;&nbsp; }&nbsp; unregisterClient(sessionId: string):&nbsp;void&nbsp;{&nbsp; &nbsp;&nbsp;this.closeSession(sessionId,&nbsp;'client disconnected');&nbsp; }&nbsp; @OnEvent(AI_TTS_STREAM_EVENT)&nbsp; handleAiStreamEvent(event: AiTtsStreamEvent):&nbsp;void&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;session =&nbsp;this.sessions.get(event.sessionId);&nbsp; &nbsp;&nbsp;if&nbsp;(!session)&nbsp;return;&nbsp; &nbsp;&nbsp;switch&nbsp;(event.type) {&nbsp; &nbsp; &nbsp;&nbsp;case'start': {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;this.ensureTencentConnection(session);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;this.sendClientJson(session.clientWs, {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;'tts_started',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;sessionId: session.sessionId,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;query: event.query,&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;break;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;case'chunk': {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;chunk = event.chunk?.trim();&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!chunk)&nbsp;return;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!session.ready || !session.tencentWs || session.tencentWs.readyState !== WebSocket.OPEN) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; session.pendingChunks.push(chunk);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;this.sendTencentChunk(session, chunk);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;break;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;case'end': {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;this.flushPendingChunks(session);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(session.tencentWs &amp;&amp; session.tencentWs.readyState === WebSocket.OPEN) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; session.tencentWs.send(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;JSON.stringify({&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;session_id: session.sessionId,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;action:&nbsp;'ACTION_COMPLETE',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;break;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;case'error': {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;this.sendClientJson(session.clientWs, {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;'tts_error',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;message: event.error,&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;this.closeSession(session.sessionId,&nbsp;'ai stream error');&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;break;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; }&nbsp; }&nbsp; private ensureTencentConnection(session: ClientSession):&nbsp;void&nbsp;{&nbsp; &nbsp;&nbsp;if&nbsp;(session.tencentWs &amp;&amp; session.tencentWs.readyState &lt;= WebSocket.OPEN) {&nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;if&nbsp;(!this.secretId || !this.secretKey || !this.appId) {&nbsp; &nbsp; &nbsp;&nbsp;this.sendClientJson(session.clientWs, {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;'tts_error',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;message:&nbsp;'TTS 凭证缺失，请检查 SECRET_ID/SECRET_KEY/APP_ID',&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;const&nbsp;url =&nbsp;this.buildTencentTtsWsUrl(session.sessionId);&nbsp; &nbsp;&nbsp;const&nbsp;tencentWs =&nbsp;new&nbsp;WebSocket(url);&nbsp; &nbsp; session.tencentWs = tencentWs;&nbsp; &nbsp; session.ready =&nbsp;false;&nbsp; &nbsp; tencentWs.on('open', () =&gt; {&nbsp; &nbsp; &nbsp;&nbsp;this.logger.log(`Tencent TTS ws opened:&nbsp;${session.sessionId}`);&nbsp; &nbsp; });&nbsp; &nbsp; tencentWs.on('message', (data, isBinary) =&gt; {&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(session.closed)&nbsp;return;&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(isBinary) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(session.clientWs.readyState === WebSocket.OPEN) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; session.clientWs.send(data, {&nbsp;binary:&nbsp;true&nbsp;});&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;raw = data.toString();&nbsp; &nbsp; &nbsp;&nbsp;let&nbsp;msg: Record&lt;string, unknown&gt; |&nbsp;undefined;&nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; msg =&nbsp;JSON.parse(raw)&nbsp;as&nbsp;Record&lt;string, unknown&gt;;&nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(Number(msg.ready) ===&nbsp;1) {&nbsp; &nbsp; &nbsp; &nbsp; session.ready =&nbsp;true;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;this.flushPendingChunks(session);&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(Number(msg.code) &amp;&amp;&nbsp;Number(msg.code) !==&nbsp;0) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;this.sendClientJson(session.clientWs, {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;'tts_error',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;message:&nbsp;String(msg.message ??&nbsp;'Tencent TTS error'),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;code:&nbsp;Number(msg.code),&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;this.closeSession(session.sessionId,&nbsp;'tencent error');&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(Number(msg.final) ===&nbsp;1) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;this.sendClientJson(session.clientWs, {&nbsp;type:&nbsp;'tts_final'&nbsp;});&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; });&nbsp; &nbsp; tencentWs.on('error', (error) =&gt; {&nbsp; &nbsp; &nbsp;&nbsp;this.sendClientJson(session.clientWs, {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;'tts_error',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;message:&nbsp;`Tencent ws error:&nbsp;${error.message}`,&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; });&nbsp; &nbsp; tencentWs.on('close', () =&gt; {&nbsp; &nbsp; &nbsp; session.tencentWs =&nbsp;undefined;&nbsp; &nbsp; &nbsp; session.ready =&nbsp;false;&nbsp; &nbsp; });&nbsp; }&nbsp; private flushPendingChunks(session: ClientSession):&nbsp;void&nbsp;{&nbsp; &nbsp;&nbsp;if&nbsp;(!session.ready || !session.tencentWs || session.tencentWs.readyState !== WebSocket.OPEN) {&nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;while&nbsp;(session.pendingChunks.length &gt;&nbsp;0) {&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;chunk = session.pendingChunks.shift();&nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!chunk)&nbsp;continue;&nbsp; &nbsp; &nbsp;&nbsp;this.sendTencentChunk(session, chunk);&nbsp; &nbsp; }&nbsp; }&nbsp; private sendTencentChunk(session: ClientSession,&nbsp;text: string):&nbsp;void&nbsp;{&nbsp; &nbsp;&nbsp;if&nbsp;(!session.tencentWs || session.tencentWs.readyState !== WebSocket.OPEN) {&nbsp; &nbsp; &nbsp; session.pendingChunks.push(text);&nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; }&nbsp; &nbsp; session.tencentWs.send(&nbsp; &nbsp; &nbsp;&nbsp;JSON.stringify({&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;session_id: session.sessionId,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;message_id:&nbsp;`msg_${Date.now()}_${Math.random().toString(36).slice(2,&nbsp;8)}`,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;action:&nbsp;'ACTION_SYNTHESIS',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;data: text,&nbsp; &nbsp; &nbsp; }),&nbsp; &nbsp; );&nbsp; }&nbsp; private closeSession(sessionId: string,&nbsp;reason: string):&nbsp;void&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;session =&nbsp;this.sessions.get(sessionId);&nbsp; &nbsp;&nbsp;if&nbsp;(!session)&nbsp;return;&nbsp; &nbsp; session.closed =&nbsp;true;&nbsp; &nbsp;&nbsp;if&nbsp;(session.tencentWs &amp;&amp; session.tencentWs.readyState &lt; WebSocket.CLOSING) {&nbsp; &nbsp; &nbsp; session.tencentWs.close();&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;if&nbsp;(session.clientWs.readyState &lt; WebSocket.CLOSING) {&nbsp; &nbsp; &nbsp;&nbsp;this.sendClientJson(session.clientWs, {&nbsp;type:&nbsp;'tts_closed', reason });&nbsp; &nbsp; &nbsp; session.clientWs.close();&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;this.sessions.delete(sessionId);&nbsp; &nbsp;&nbsp;this.logger.log(`TTS session closed:&nbsp;${sessionId}, reason:&nbsp;${reason}`);&nbsp; }&nbsp; private sendClientJson(clientWs: WebSocket,&nbsp;payload: Record&lt;string, unknown&gt;):&nbsp;void&nbsp;{&nbsp; &nbsp;&nbsp;if&nbsp;(clientWs.readyState !== WebSocket.OPEN)&nbsp;return;&nbsp; &nbsp; clientWs.send(JSON.stringify(payload));&nbsp; }&nbsp; private buildTencentTtsWsUrl(sessionId: string): string {&nbsp; &nbsp;&nbsp;const&nbsp;now =&nbsp;Math.floor(Date.now() /&nbsp;1000);&nbsp; &nbsp;&nbsp;const&nbsp;params: Record&lt;string, string | number&gt; = {&nbsp; &nbsp; &nbsp;&nbsp;Action:&nbsp;'TextToStreamAudioWSv2',&nbsp; &nbsp; &nbsp;&nbsp;AppId:&nbsp;this.appId,&nbsp; &nbsp; &nbsp;&nbsp;Codec:&nbsp;'mp3',&nbsp; &nbsp; &nbsp;&nbsp;Expired: now +&nbsp;3600,&nbsp; &nbsp; &nbsp;&nbsp;SampleRate:&nbsp;16000,&nbsp; &nbsp; &nbsp;&nbsp;SecretId:&nbsp;this.secretId,&nbsp; &nbsp; &nbsp;&nbsp;SessionId: sessionId,&nbsp; &nbsp; &nbsp;&nbsp;Speed:&nbsp;0,&nbsp; &nbsp; &nbsp;&nbsp;Timestamp: now,&nbsp; &nbsp; &nbsp;&nbsp;VoiceType:&nbsp;this.voiceType,&nbsp; &nbsp; &nbsp;&nbsp;Volume:&nbsp;5,&nbsp; &nbsp; };&nbsp; &nbsp;&nbsp;const&nbsp;signStr =&nbsp;Object.keys(params)&nbsp; &nbsp; &nbsp; .sort()&nbsp; &nbsp; &nbsp; .map((k) =&gt;`${k}=${params[k]}`)&nbsp; &nbsp; &nbsp; .join('&amp;');&nbsp; &nbsp;&nbsp;const&nbsp;rawStr =&nbsp;`GETtts.cloud.tencent.com/stream_wsv2?${signStr}`;&nbsp; &nbsp;&nbsp;const&nbsp;signature = createHmac('sha1',&nbsp;this.secretKey).update(rawStr).digest('base64');&nbsp; &nbsp;&nbsp;const&nbsp;searchParams =&nbsp;new&nbsp;URLSearchParams({&nbsp; &nbsp; &nbsp; ...Object.fromEntries(Object.entries(params).map(([k, v]) =&gt;&nbsp;[k,&nbsp;String(v)])),&nbsp; &nbsp; &nbsp;&nbsp;Signature: signature,&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;return`wss://tts.cloud.tencent.com/stream_wsv2?${searchParams.toString()}`;&nbsp; }}

在 SpeechModule 导出这个 service：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/20_公众号_Yi昭.png)

用到的一些常量、类型定义在 common/stream-events.ts

    export&nbsp;const&nbsp;AI_TTS_STREAM_EVENT =&nbsp;'ai.tts.stream';export&nbsp;type AiTtsStreamEvent =&nbsp; | {&nbsp;type:&nbsp;'start'; sessionId: string; query: string }&nbsp; | {&nbsp;type:&nbsp;'chunk'; sessionId: string; chunk: string }&nbsp; | {&nbsp;type:&nbsp;'end'; sessionId: string }&nbsp; | {&nbsp;type:&nbsp;'error'; sessionId: string; error: string };

然后在 SSE 接口那边，发事件来触发这边的语音生成：

首先在 AiRouter 里需要传入 sessionId

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/21_公众号_Yi昭.png)

用 eventEmitter 发事件，建立连接。

在 AiService 里，发送具体的文本：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/22_公众号_Yi昭.png)

这样，当用户传入文本，生成回答的时候，就会和腾讯云 tts 服务建立链接，发送文本

接下来只要再搞一个 ws 服务，让前端可以连就可以了。

改下 main.ts:

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/23_公众号_Yi昭.png)

用 ws 创建一个 WebSocketServer

把 socket 注册到 ttsRelayService，这样那边就可以用这个 socket 给客户端发消息了。

最后来改下前端的 html：

首先是和后端的 ws 建立连接：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/24_公众号_Yi昭.png)

和刚才的 ws 接口建立连接

根据返回的是字符串，还是二进制 ArrayBuffer 做不同处理

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/25_公众号_Yi昭.png)

字符串就是作为 JSON 来 parse，根据不同的类型做不同处理

二进制就是作为语音播放。

语音播放用到 Audio 的标签，然后它的 url 是 MediaSource

MediaSource 通过 SourceBuffer 动态添加流式的语音数据，就可以实现流式播放

原理是这样：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/26_公众号_Yi昭.png)

具体代码如下：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/27_公众号_Yi昭.png)

给 audio 的 element 设置 MediaSource 的 object url

然后添加一个 SourceBuffer

之后 ws 返回的服务，往这个 sourceBuffer.appendBuffer 就好了。

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/28_公众号_Yi昭.png)

具体代码从仓库复制吧，就不贴了。

配置下音色 id：

![image](../IMG/2026-03-28_给Agent加上语音交互：ASR流式TTS/29_公众号_Yi昭.png)

改了配置需要重启服务才生效。

安装下 ws 的包：

    ppip install wsppip install --save-dev @types/ws

我们跑一下：

> 🎬 视频演示（原公众号视频）

这样，我们就实现了豆包同款语音功能。

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

我们实现了豆包同款的语音功能。

首先是语音识别 ASR：

这个不需要流式，前端用 MediaRecorder 录音完成后，放到 FormData 里 post 传给后端，后端调用腾讯云的 asr 接口转成文本返回

之后前端用文本调用 ai 接口，通过 SSE 流式返回文本回答。

然后是语音合成 TTS：

这个一般都是要流式的，因为文本是流式生成的，不可能等文本全生成再播放语音，所以需要用流式语音合成的接口。

文字用 SSE 流式返回，但这个是基于 http 的文本协议，不适合返回二进制数据，所以需要再做一个 WebSocket 服务来推送语音数据。

SSE 那边流式生成文本之后，通过事件传给 WebSocket 服务，把文本推给腾讯云 tts 服务，那边返回语音数据之后用 ws 推给前端。

前端用 Audio 标签 + MediaSource + SourceBuffer 来实现流式的播放。

Audio 标签设置 MediaSource 为 object url 的 src，MediaSource 添加 SourceBuffer，然后就可以不断 push 二进制数据 ArrayBuffer 实现流式播放了。

这个流式语音功能有技术难点，可以作为简历的一个亮点，把思路理清，试着自己复述一下。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-03-28 11:08:08 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247485581&idx=1&sn=8a3a727bc55a5531493e9098ec7ea5e1&chksm=f137f01b7ce68f4b3bc4d81ab9b164bba2b0d8c0cc74ea1990d640d680df02dd4949549d0f55&scene=27#wechat_redirect
