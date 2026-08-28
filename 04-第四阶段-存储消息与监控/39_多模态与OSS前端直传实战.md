# 多模态与 OSS 前端直传实战：AI 画板

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年7月10日 10:38

之前我们的 Agent 都是输入文字、返回文字。

但平时用的很多 Agent 都支持输入图片、返回图片

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/0_公众号_Yi昭.png)

这是怎么实现的呢？

首先模型要用支持多模态的：

> 🎬 视频演示（原公众号视频）

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/1_公众号_Yi昭.png)

然后我们要用 OSS 来存储图片，拿到 url：

> 🎬 视频演示（原公众号视频）

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/2_公众号_Yi昭.png)

基于多模态的大模型 + OSS，我们就可以实现支持多模态的 Agent

创建项目：

    mkdir multi-modal-agentcd&nbsp;multi-modal-agentnpm init -y

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/3_公众号_Yi昭.png)

安装依赖：

    ppip install langchain_core langchain_openai dashscope-sdk-official dotenv ali-oss

创建 src/image-understanding.mjs

    /**&nbsp;* 图像理解 — qwen-vl-plus&nbsp;* DashScope OpenAI 兼容接口 + ChatOpenAI&nbsp;*/import'dotenv/config';import&nbsp;{ ChatOpenAI }&nbsp;from'langchain_openai';import&nbsp;{ HumanMessage }&nbsp;from'langchain_core/messages';const&nbsp;model =&nbsp;new&nbsp;ChatOpenAI({apiKey: process.env.OPENAI_API_KEY,model:&nbsp;'qwen-vl-plus',configuration: {&nbsp; &nbsp;&nbsp;baseURL: process.env.OPENAI_BASE_URL,&nbsp; },});const&nbsp;response =&nbsp;await&nbsp;model.invoke([new&nbsp;HumanMessage({&nbsp; &nbsp;&nbsp;content: [&nbsp; &nbsp; &nbsp; {&nbsp;type:&nbsp;'text',&nbsp;text:&nbsp;'详细描述这张图片的内容'&nbsp;},&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;'image_url',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;image_url: {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;url:&nbsp;'https://dashscope.oss-cn-beijing.aliyuncs.com/images/dog_and_girl.jpeg',&nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; &nbsp; },&nbsp; &nbsp; ],&nbsp; }),]);console.log('model: qwen-vl-plus');console.log(response.content);

其余案例代码从仓库复制。

跑一下：

> 🎬 视频演示（原公众号视频）

兼容 openai 协议的大模型就可以用 ChatOpenAI 来调用，其余的直接用 dashscope 的 SDK 来调。

这个过程涉及到了 OSS，传入的图片、视频、音频 url、生成的视频、音频、图片的保存等。

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/4_公众号_Yi昭.png)

我们来完整实现下这个流程。

生成图片传到 OSS 直接后端做就行，返回 oss 的 url

但是用户上传视频，有必要先传到我们服务器，再传到 oss 么？

没必要，这种可以用 OSS 直传。

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/5_公众号_Yi昭.png)

阿里云文档里有写：

https://help.aliyun.com/zh/oss/user-guide/uploading-objects-to-oss-directly-from-clients/

（微信最近文章内容不能复制了，代码部分可以直接从仓库复制，文字可以截图让豆包之类的提取）

> 🎬 视频演示（原公众号视频）

写一下生成 sts 的代码：

src/sts-gen.mjs

    import&nbsp;'dotenv/config';import&nbsp;OSS&nbsp;from'ali-oss';asyncfunction&nbsp;main()&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;config = {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;region:&nbsp;'oss-cn-beijing',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;bucket:&nbsp;'agent-bucket123',&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;accessKeyId: process.env.OSS_ACCESS_KEY_ID,&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;accessKeySecret: process.env.OSS_ACCESS_KEY_SECRET,&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;const&nbsp;client =&nbsp;new&nbsp;OSS(config);&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;const&nbsp;date =&nbsp;newDate();&nbsp; &nbsp;&nbsp;&nbsp; &nbsp; date.setDate(date.getDate() +&nbsp;1);&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;const&nbsp;res = client.calculatePostSignature({&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;expiration: date.toISOString(),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;conditions: [&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ["content-length-range",&nbsp;0,&nbsp;1048576000],&nbsp;//设置上传文件的大小限制。 &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; ]&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;console.log(res);&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;const&nbsp;location =&nbsp;await&nbsp;client.getBucketLocation();&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;const&nbsp;host =&nbsp;`http://${config.bucket}.${location.location}.aliyuncs.com`;&nbsp; &nbsp;&nbsp;console.log(host);}main();

创建一个前端的 html

public/index.html

    &lt;!DOCTYPE&nbsp;html&gt;&lt;html&nbsp;lang="en"&gt;&lt;head&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;charset="UTF-8"&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;name="viewport"&nbsp;content="width=device-width, initial-scale=1.0"&gt;&nbsp; &nbsp;&nbsp;&lt;title&gt;Document&lt;/title&gt;&nbsp; &nbsp;&nbsp;&lt;script&nbsp;src="https://unpkg.com/axios@1.6.5/dist/axios.min.js"&gt;&lt;/script&gt;&lt;/head&gt;&lt;body&gt;&nbsp; &nbsp;&nbsp;&lt;input&nbsp;id="fileInput"&nbsp;type="file"/&gt;&nbsp; &nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&lt;script&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;fileInput =&nbsp;document.getElementById('fileInput');&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;asyncfunctiongetOSSInfo()&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;await'请求应用服务器拿到临时凭证';&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;OSSAccessKeyId:&nbsp;'',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;Signature:&nbsp;'',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;policy:&nbsp;'',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;host:&nbsp;''&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; fileInput.onchange =&nbsp;async&nbsp;() =&gt; {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;file = fileInput.files[0];&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;ossInfo =&nbsp;await&nbsp;getOSSInfo();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;formdata =&nbsp;new&nbsp;FormData()&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; formdata.append('key', file.name);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; formdata.append('OSSAccessKeyId', ossInfo.OSSAccessKeyId)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; formdata.append('policy', ossInfo.policy)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; formdata.append('signature', ossInfo.Signature)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; formdata.append('success_action_status',&nbsp;'200')&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; formdata.append('file', file)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;res =&nbsp;await&nbsp;axios.post(ossInfo.host, formdata);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if(res.status ===&nbsp;200) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;img =&nbsp;document.createElement('img');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; img.src = ossInfo.host +&nbsp;'/'&nbsp;+ file.name&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;document.body.append(img);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; alert('上传成功');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;&lt;/script&gt;&lt;/body&gt;&lt;/html&gt;

跑一下：

> 🎬 视频演示（原公众号视频）

多模态大模型调用、前端直传 OSS 都跑通了，我们来做一个小实战：AI 画板。

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/6_公众号_Yi昭.png)

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/7_公众号_Yi昭.png)

先来写一下后端的接口：

    nest new ai-canvas

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/8_公众号_Yi昭.webp)

进入项目，创建个新模块：

    nest g res ai --no-spec

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/9_公众号_Yi昭.webp)

安装依赖：

    ppip install dashscope-sdk-official dotenv ali-oss &nbsp;fastapi.config fastapi.serve-static

把我们前面写的那个图片修改的逻辑拿过来，放到 service 里：

具体代码从仓库复制。

> 🎬 视频演示（原公众号视频）

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/10_公众号_Yi昭.webp)

![image](../IMG/2026-07-10_多模态与OSS前端直传实战：AI画板/11_公众号_Yi昭.webp)

这样我们就把前端直传 OSS，与多模态大模型，综合用了一遍。

## 总结

Agent 很多都支持多模态，比如上传图片识别、生成图片、视频等。

我们用了一下多模态的大模型，阿里的模型有的不支持 openai 协议，需要用 dashscope 的 sdk 来调用。

生成的图片、视频等会放到临时的 oss，有效期大概 24 小时，我们要传到自己的 oss 持久保存。

我们实现了前端直传 OSS，服务端只返回 sts 信息就可以了。

然后把多模态大模型与前端直传 oss 做了一个综合的小实战：AI 画板。

前端直传 OSS + 多模态大模型，会免回经常用到。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-07-10 10:38:13 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247486598&idx=1&sn=1bbef1dee3b95188619690c08c415b83&chksm=f18e6215a1bed7bc37831b789970e49a9204ff0fb9d1aeb7904ca0f5b1a9138499f4557779e8&scene=27#wechat_redirect
