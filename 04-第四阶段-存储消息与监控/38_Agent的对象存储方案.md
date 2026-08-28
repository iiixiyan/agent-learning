# Agent 的对象存储方案：MinIO、RustFS、阿里云 OSS

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年6月30日 14:20

AI Agent 在跑业务的时候，时时刻刻都要读写各种文件。

不管是上传的文档，还是 AI 自己生成的报表、图片视频

普通本地文件夹根本扛不住海量文件的生产场景。

所以要用对象存储（Object Storage）

比如这三类场景：

- 存放 RAG 知识库所有原始文件，给智能问答提供数据源
- 保存 Agent 自动运行产出的报表、图表、运行日志
- 统一存图片、音频、视频，支撑多模态 AI 处理任务

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/0_公众号_Yi昭.jpeg)

（MinIO 是常用的对象存储方案）

单独拿知识库场景来说：

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/1_公众号_Yi昭.png)

MinIO 承担着原始文件的存储重任。

各类文档、PDF、网页素材都会先进入数据处理环节，完成文件解析、文本切分、内容清洗与元数据提取。

处理完成后的原始文件，会完整存入 MinIO 对象存储中长久保存。

同时文件对应的名称、来源、切片等元数据写入关系型数据库（PostgreSQL）。

而切分后的文本分片会用嵌入模型向量化，存入向量数据库。

这就走完了知识库完整的数据入库流程。

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/2_公众号_Yi昭.png)

等到用户发起提问检索时，先把用户问题用嵌入模型向量化，去向量库做语义检索

向量数据库返回相似度高的文本片段，同时附带对应的文件 ID

检索服务拿着文件 ID 去元数据库，调取这份文件的基础信息

再根据文件 ID 从 MinIO 拉取完整原始文件、原文片段内容

最终把原文内容和检索结果一并返回给提问的用户。

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/3_公众号_Yi昭.png)

整套流程里 MinIO 对象存储的作用不可替代。

向量库只存文本向量，不会存放完整原始文件。

关系数据库仅保管元数据，无法承载大体积二进制附件。

只有 MinIO 能统一存放 PDF、图片、各类附件等大容量素材。

既能保障文件长期安全归档，又能随时按需调取原文溯源。

能和向量库、业务数据库无缝联动。

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/4_公众号_Yi昭.jpeg)

当然，对象存储不止有 MinIO

市面上主流可选方案主要分为三类：阿里云 OSS、MinIO、RustFS。

先说阿里云 OSS，属于公有云托管服务。

不用自己搭建服务器，零运维，开箱就能用。

完美适配云上业务，能和阿里云各类产品打通联动。

采用按量计费模式，自动扩容，业务规模越大扩容越省心。

适合线上 SaaS 平台、在线教育这类不想维护存储的团队。

再就是 MinIO，是轻量化私有化方案。

支持 Docker 一键部署，单机、小型集群都能快速搭建。

日常小批量文档存取流畅，搭建成本几乎为零。

但短板也很明显，大批量大文件并发时容易卡顿。

开源协议为 AGPL，如果商用落地会存在版权风险。

更适合中小企业小型知识库、本地测试环境使用。

最后是 RustFS，面向大型私有化集群设计。

支持多服务器分布式部署，海量文件并发场景稳定性更强。

底层基于 Rust 开发，运行时内存占用更低。

同时兼容 S3、POSIX、WebDAV 多种访问协议，适配更广。

开源协议是宽松的 Apache2.0，商用无任何版权约束。

专门匹配集团级多模态知识库、海量音视频、国产化政务国企项目。

综上：

- 如果业务跑在公有云上、想省去运维压力，直接选阿里云 OSS。
- 如果只是小型本地自建知识库、低成本快速落地，优先 MinIO。
- 如果是海量音视频存储、大型集团国产化项目，推荐 RustFS。

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/5_公众号_Yi昭.png)

这节我们把这三种都用一下：

我们本地文件存储是目录 - 文件的真实树状组织方式：

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/6_公众号_Yi昭.png)

而 OSS 对象存储底层是扁平化结构：

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/7_公众号_Yi昭.png)

所有文件都平铺在同一个桶内，不存在原生文件夹。

阿里云 OSS 官方文档也明确说明，对象存储底层没有真实目录层级：

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/8_公众号_Yi昭.png)

控制台里我们看到的文件夹视图，只是系统模拟出来的效果：

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/9_公众号_Yi昭.png)

这套虚拟目录的实现逻辑和文件元数据无关。

每个 Object 对象包含三部分核心信息：唯一 Key 标识、文件二进制内容、自定义元数据：

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/10_公众号_Yi昭.png)

OSS 只是解析文件 Key 里的/斜杠分隔符，渲染出目录分层视图。

用 Key 前缀做分组检索。

手动创建空文件夹时，本质是生成一个以/结尾的 0 字节占位对象。

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/11_公众号_Yi昭.png)

除了对象存储 OSS，阿里云也提供了文件存储和块存储的方式：

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/12_公众号_Yi昭.png)

块存储就是把整块磁盘给你用，你需要自己格式化，存储容量有限。

文件存储就是有目录层次结构，你可以上传下载文件，存储容量有限。

对象存储就是 key-value 存储，分布式的方式实现的，存储容量无限。

这些简单了解就行，绝大多数情况下，我们都是用 OSS 对象存储。

我们买一下阿里云 OSS 服务，5 块钱够用半年：

https://www.aliyun.com/product/oss

> 🎬 视频演示（原公众号视频）

进到控制台，创建 Bucket，上传文件：

> 🎬 视频演示（原公众号视频）

很多时候，我们需要在代码里上传，比如知识库里，用户上传的文件，要传到 OSS。

> 🎬 视频演示（原公众号视频）

创建项目：

    mkdir oss-testcd&nbsp;oss-testnpm init -y

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/13_公众号_Yi昭.png)

安装依赖：

    pnpm install ali-oss dotenv

创建 src/oss-upload.mjs

    import&nbsp;'dotenv/config';import&nbsp;OSS&nbsp;from'ali-oss';import&nbsp;fs&nbsp;from'fs';const&nbsp;client =&nbsp;new&nbsp;OSS({// yourRegion填写Bucket所在地域。以华东1（杭州）为例，Region填写为oss-cn-hangzhou。region: process.env.OSS_REGION,accessKeyId: process.env.OSS_ACCESS_KEY_ID,accessKeySecret: process.env.OSS_ACCESS_KEY_SECRET,authorizationV4:&nbsp;true,bucket: process.env.OSS_BUCKET,});asyncfunction&nbsp;putStream&nbsp;()&nbsp;{try&nbsp;{&nbsp; &nbsp;&nbsp;// 使用chunked encoding。使用putStream接口时，SDK默认会发起一个chunked encoding的HTTP PUT请求。&nbsp; &nbsp;&nbsp;let&nbsp;stream = fs.createReadStream('./zao.png');&nbsp; &nbsp;&nbsp;// 填写Object完整路径，例如exampledir/exampleobject.txt。Object完整路径中不能包含Bucket名称。&nbsp; &nbsp;&nbsp;let&nbsp;result =&nbsp;await&nbsp;client.putStream('aaa/bbb/first.png', stream); &nbsp; &nbsp;&nbsp; &nbsp;&nbsp;console.log(result);&nbsp;&nbsp; }&nbsp;catch&nbsp;(e) {&nbsp; &nbsp;&nbsp;console.log(e)&nbsp; }}putStream();

还有 .env

    OSS_REGION=OSS_ACCESS_KEY_ID=OSS_ACCESS_KEY_SECRET=OSS_BUCKET=

> 🎬 视频演示（原公众号视频）

这样，我们就通过代码完成了 OSS 文件上传。

直接用阿里云的 OSS 是挺方便，但是要花钱，而且企业内部有的资料也不希望上云。

这种情况就要自己搭 OSS 服务了：

比如 MinIO 或者 RustFS

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/14_公众号_Yi昭.png)

创建 docker-compose.yml

    version:&nbsp;"3.8"services:minio:&nbsp; &nbsp;&nbsp;image:minio/minio:RELEASE.2025-04-22T22-12-26Z&nbsp; &nbsp;&nbsp;container_name:minio-server&nbsp; &nbsp;&nbsp;restart:always&nbsp; &nbsp;&nbsp;ports:&nbsp; &nbsp; &nbsp;&nbsp;# S3 对象存储API端口（程序对接用）&nbsp; &nbsp; &nbsp;&nbsp;-"9000:9000"&nbsp; &nbsp; &nbsp;&nbsp;# Web图形控制台端口（浏览器访问UI）&nbsp; &nbsp; &nbsp;&nbsp;-"9001:9001"&nbsp; &nbsp;&nbsp;environment:&nbsp; &nbsp; &nbsp;&nbsp;# 登录控制台、S3接口的账号（至少3位）&nbsp; &nbsp; &nbsp;&nbsp;MINIO_ROOT_USER:admin&nbsp; &nbsp; &nbsp;&nbsp;# 登录密码（至少8位，数字+字母）&nbsp; &nbsp; &nbsp;&nbsp;MINIO_ROOT_PASSWORD:Admin@123456&nbsp; &nbsp;&nbsp;volumes:&nbsp; &nbsp; &nbsp;&nbsp;# 持久化数据到本地 ./minio-data 文件夹&nbsp; &nbsp; &nbsp;&nbsp;-./minio-data:/data&nbsp; &nbsp;&nbsp;command:server/data--console-address":9001"

跑一下：

> 🎬 视频演示（原公众号视频）

这样我们就在本地跑了一个 OSS 服务：

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/15_公众号_Yi昭.png)

然后我们在代码里用 sdk 上传

    pnpm install minio

创建 src/minio-upload.mjs

    import&nbsp;'dotenv/config';import&nbsp;fs&nbsp;from'fs';import&nbsp;*&nbsp;as&nbsp;Minio&nbsp;from'minio';const&nbsp;minioClient =&nbsp;new&nbsp;Minio.Client({endPoint:&nbsp;'localhost',port:&nbsp;9000,useSSL:&nbsp;false,accessKey: process.env.MINIO_ACCESS_KEY,secretKey: process.env.MINIO_SECRET_KEY,})asyncfunction&nbsp;putStream()&nbsp;{&nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;stream = fs.createReadStream('./zao.png');&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;result =&nbsp;await&nbsp;minioClient.putObject('aaa',&nbsp;'ccc/ddd/hello.png', stream);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(result);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log('上传成功');&nbsp; &nbsp; }&nbsp;catch&nbsp;(err) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;console.log(err);&nbsp; &nbsp; }}putStream();

> 🎬 视频演示（原公众号视频）

你会发现代码和之前阿里云 OSS 的差不多，为什么 OSS 服务都这么相似呢？

因为它们都是遵循 AWS 的 Simple Storage Service（S3）规范的，简称 S3 规范。

所以不管哪家的 OSS，用起来都是差不多的。

![image](../IMG/2026-06-30_Agent的对象存储方案：MinIO、RustFS、阿里云OSS/16_公众号_Yi昭.png)

简单试一下 minio 新版的改动：

> 🎬 视频演示（原公众号视频）

最后再来用一下 RustFS

改下配置文件：

    version:&nbsp;"3.8"services:rustfs:&nbsp; &nbsp;&nbsp;image:rustfs/rustfs:latest&nbsp; &nbsp;&nbsp;container_name:rustfs-server&nbsp; &nbsp;&nbsp;restart:always&nbsp; &nbsp;&nbsp;ports:&nbsp; &nbsp; &nbsp;&nbsp;-"9000:9000"&nbsp; &nbsp;&nbsp;# S3 API 端口&nbsp; &nbsp; &nbsp;&nbsp;-"9001:9001"&nbsp; &nbsp;&nbsp;# Web控制台端口&nbsp; &nbsp;&nbsp;environment:&nbsp; &nbsp; &nbsp;&nbsp;TZ:Asia/Shanghai&nbsp; &nbsp; &nbsp;&nbsp;# S3/后台登录账号密钥&nbsp; &nbsp; &nbsp;&nbsp;RUSTFS_ACCESS_KEY:admin&nbsp; &nbsp; &nbsp;&nbsp;RUSTFS_SECRET_KEY:Admin@123456&nbsp; &nbsp; &nbsp;&nbsp;# 开启Web管理控制台&nbsp; &nbsp; &nbsp;&nbsp;RUSTFS_CONSOLE_ENABLE:"true"&nbsp; &nbsp;&nbsp;volumes:&nbsp; &nbsp; &nbsp;&nbsp;-./volumes/rustfs-data:/data&nbsp; &nbsp; &nbsp;&nbsp;-./volumes/rustfs-logs:/logs&nbsp; &nbsp;&nbsp;command:server/data

跑一下：

> 🎬 视频演示（原公众号视频）

除了界面不大一样，功能都是差不多的。

然后在代码里上传个文件；

    pnpm install @aws-sdk/client-s3

因为都兼容 S3 协议，所以所有对象存储服务都可直接使用 AWS 官方 S3 SDK；

前面我们用 ali-oss、minio 写的代码也都可以换成这个 sdk

创建 src/s3-upload.mjs

    import&nbsp;'dotenv/config';import&nbsp;{ S3Client, PutObjectCommand }&nbsp;from'@aws-sdk/client-s3';import&nbsp;fs&nbsp;from'fs';// 初始化统一S3客户端（RustFS/MinIO/阿里云OSS通用）const&nbsp;s3Client =&nbsp;new&nbsp;S3Client({endpoint: process.env.S3_ENDPOINT,credentials: {&nbsp; &nbsp;&nbsp;accessKeyId: process.env.S3_ACCESS_KEY_ID,&nbsp; &nbsp;&nbsp;secretAccessKey: process.env.S3_SECRET_ACCESS_KEY,&nbsp; },forcePathStyle:&nbsp;true,signatureVersion:&nbsp;'v4',region:&nbsp;'aaa'// 本地私有存储随便填，不影响});/**&nbsp;* 文件流上传&nbsp;* @param {string} objectKey 对象路径 aaa/bbb/first.png&nbsp;* @param {ReadableStream} stream fs可读流&nbsp;* @param {string} contentType 文件类型（图片/pdf等）&nbsp;*/asyncfunction&nbsp;putStream(objectKey, stream, contentType =&nbsp;'image/png')&nbsp;{try&nbsp;{&nbsp; &nbsp;&nbsp;const&nbsp;uploadCmd =&nbsp;new&nbsp;PutObjectCommand({&nbsp; &nbsp; &nbsp;&nbsp;Bucket:&nbsp;'hello',&nbsp; &nbsp; &nbsp;&nbsp;Key: objectKey,&nbsp; &nbsp; &nbsp;&nbsp;Body: stream,&nbsp; &nbsp; &nbsp;&nbsp;ContentType: contentType&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;await&nbsp;s3Client.send(uploadCmd);&nbsp; &nbsp;&nbsp;console.log('上传成功');&nbsp; }&nbsp;catch&nbsp;(err) {&nbsp; &nbsp;&nbsp;console.error('上传失败', err);&nbsp; &nbsp;&nbsp;throw&nbsp;err;&nbsp; }}asyncfunction&nbsp;main()&nbsp;{const&nbsp;stream = fs.createReadStream('./zao.png');await&nbsp;putStream('aaa/bbb/first.png', stream,&nbsp;'image/png');}main();

改一下 .env

    S3_ENDPOINT=http://localhost:9000S3_ACCESS_KEY_ID=adminS3_SECRET_ACCESS_KEY=Admin@123456

跑一下：

> 🎬 视频演示（原公众号视频）

至此，我们阿里云 OSS、MinIO、RustFS 就都用了一遍了。

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

AI Agent 运行过程中会持续产生、读取大量各类文件。

用户上传的文档、程序自动生成的图表、音视频等都需要稳定存储。

普通本地文件夹无法支撑海量文件并发读写的业务场景

所以做 AI 知识库、多模态 Agent 项目，必须使用对象存储。

比如 RAG 知识库的完整流程：

用户上传的 PDF、网页素材先经过解析、清洗、切片处理。

原始文件会完整存入对象存储长期归档保存。

文件名称、来源、切片信息这类元数据存入 PostgreSQL 关系库。

文本切片经过向量化后，单独存入向量数据库用于语义检索。

用户提问时，向量库返回匹配片段并附带对应文件 ID

程序拿着 ID 去数据库读取文件基础信息

再通过对象存储拉取完整原始文档做溯源展示。

向量库只存向量、数据库只存文字信息，都存不了大体积二进制文件。

只有对象存储能统一承载图片、PDF、音视频等大容量素材。

目前主流可选三类对象存储方案，分别是阿里云 OSS、MinIO、RustFS

阿里云 OSS 是公有云托管服务，不用自己维护服务器，按量自动扩容

适合线上 SaaS、不想投入运维人力的业务团队

MinIO 可以 Docker 快速私有化部署，本地测试、小型知识库用着很方便

但新版社区版阉割了可视化管理功能，商用还存在 AGPL 开源版权风险

RustFS 专为私有化海量文件场景打造，Rust 底层内存占用低、并发稳定

商用无约束，适配多模态国产化项目

三类存储底层全部遵循 S3 标准协议，核心能力基本一致。

只是后台管理界面、商用约束存在区别。

安装 @aws-sdk/client-s3 这一个 aws 的包就能对接所有 OSS 服务，也可以分别用 ali-oss、minio 来对接。

对象存储是各类 AI Agent 存储文件的底层核心支撑，后面会大量用到。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-06-30 14:20:20 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247486481&idx=1&sn=a92727efcdfa0368156c30b6afce3829&chksm=f18cb11df9e3e999487c1bbd00f193e7bcb152ab00f8068cf3410a7c77881c4ccbb17b9b2211&scene=27#wechat_redirect
