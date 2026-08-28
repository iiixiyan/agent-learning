# LangChain 全部 Splitter，其实只需要其中的一个

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年1月10日 19:13

上节我们学了 loader 和 splitter

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/0_公众号_Yi昭.png)

知识可能有各种来源，比如一个视频、一个 pdf、一个网页、一个 word 文档

这时候就需要通过各种 loader 从中提取信息，把它们转换成 Document

但是 Document 可能会很大，需要用 Splitter 分割成一个个的比较小的 Document（chunk）

之后用嵌入模型，把分块的文档向量化后存入向量数据库。

上节跑通了这个流程：

> 🎬 视频演示（原公众号视频）

这节我们把所有的 Splitter 过一遍。

首先要区分 sperator 和 chunk size 的概念：

比如上节我们这样分割的 Document：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/1_公众号_Yi昭.png)

首先按照 。的 sperator 来分割字符串，然后按照 chunk size 放入一个个 Document，也就是这样：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/2_公众号_Yi昭.png)

如果分割后还是大于 chunk size，就需要按照后面的 sperator 继续分割，然后加上 overlap：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/3_公众号_Yi昭.png)

注意，**overloap 只有文本超过 chunk size，文本被打断了才会加**，不是所有的块都会有 overlap

比如上面那段话超过了 chunk size，分割到两个 chunk 里，第二个 chunk 就会按照设置重复一部分内容

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/4_公众号_Yi昭.png)

设置这个是为了保证语义连贯性

通常设置为 chunkSize 的 10% - 20%

牺牲了一点存储空间（因为数据重复了），换取了模型对上下文理解的完整性。

那 langchain 都有哪些 splitter 呢？

我们看下 langchain_textsplitters 这个包：

> 🎬 视频演示（原公众号视频）

可以看到这个包导出的 splitter，以及它们的继承关系：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/5_公众号_Yi昭.png)

所有的 Splitter 都继承自 TextSplitter，包括 RecursiveCharacterTextSplitter 等。

而 MarkdownTextSplitter、LatexTextSplitter 又继承自 RecursiveCharacterTextSplitter。

其实很容易理解：

CharacterTextSplitter 是按照某个字符来分割，比如按照句号

RecursiveCharacterTextSplitter 是递归分割，比如“ 。 ？ ！”就是先尝试按照 。 分割，如果分割后大于 chunk 剩余空间再按照 ？ 分割，是一个递归过程。

而 MarkdownTextSplitter 自然就是按照 #、##、### 等一级级标题来递归分割，所以是 RecursiveCharacterTextSplitter 的子类。

Latex 是写数学公式的语法，比如这样：

https://www.latexlive.com/

> 🎬 视频演示（原公众号视频）

它就是一种描述数学公式的语法，和 markdown 一样，自然也是递归按照某些字符分割的。

所以也是继承自 RecursiveCharacterTextSplitter

那 TokenTextSplitter 呢？

这个是另一种分割策略。

我们按照字符分割，分割出来的文档的 token 大小是不一定的。

token 是大模型输入的一个单位，可能一个单词是 1 到 2 个 token：

apple 是 1 个 token

pineapple 是 2 个 token

苹果是 1-2 个 token

我们试一下就知道了，用 js-tiktoken 这个包，它是 openai 模型的分词器

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/6_公众号_Yi昭.png)

安装下：

    ppip install js-tiktoken

创建 src/tiktoken-test.mjs

    import&nbsp;{ getEncodingNameForModel }&nbsp;from&nbsp;"js-tiktoken";&nbsp;const&nbsp;modelName =&nbsp;"gpt-4";&nbsp;const&nbsp;encodingName = getEncodingNameForModel(modelName);console.log(encodingName);

我们打印下 gpt-4 这个模型的编码的名字

> 🎬 视频演示（原公众号视频）

然后可以用这个编码来计算下 token 数量：

    import&nbsp;{ getEncoding, getEncodingNameForModel }&nbsp;from&nbsp;"js-tiktoken";&nbsp;const&nbsp;modelName =&nbsp;"gpt-4";&nbsp;const&nbsp;encodingName = getEncodingNameForModel(modelName);console.log(encodingName);const&nbsp;enc = getEncoding("cl100k_base");console.log('apple', enc.encode("apple").length);console.log('pineapple', enc.encode("pineapple").length);console.log('苹果', enc.encode("苹果").length);console.log('吃饭', enc.encode("吃饭").length);console.log('一二三', enc.encode("一二三").length);

> 🎬 视频演示（原公众号视频）

可以看到，字符和 token 数量并没有一个确定的关系，与不同模型的分词器有关。

这样我们按照字符数来计算 chunk size 就没法准确估算 token 大小。

对于需要精准控制 token 数量的场景就不大合适了。

这时候就可以用 TokenTextSplitter，它是按照 token 数来分割的。

回过头来再看下所有的 Splitter：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/7_公众号_Yi昭.png)

关系就比较清晰了。

先用一下 CharacterTextSplitter

创建 src/CharacterTextSplitter-test.mjs

    import&nbsp;"dotenv/config";import"cheerio";import&nbsp;{ CharacterTextSplitter }&nbsp;from"langchain_textsplitters";import&nbsp;{ Document }&nbsp;from"langchain_core/documents";import&nbsp;{ getEncoding }&nbsp;from"js-tiktoken";&nbsp;const&nbsp;logDocument =&nbsp;new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`[2024-01-15 10:00:00] INFO: Application started[2024-01-15 10:00:05] DEBUG: Loading configuration file[2024-01-15 10:00:10] INFO: Database connection established[2024-01-15 10:00:15] WARNING: Rate limit approaching[2024-01-15 10:00:20] ERROR: Failed to process request[2024-01-15 10:00:25] INFO: Retrying operation[2024-01-15 10:00:30] SUCCESS: Operation completed`});const&nbsp;logTextSplitter =&nbsp;new&nbsp;CharacterTextSplitter({&nbsp; &nbsp;&nbsp;separator:&nbsp;'\n',&nbsp; &nbsp;&nbsp;chunkSize:&nbsp;200,&nbsp; &nbsp;&nbsp;chunkOverlap:&nbsp;20});const&nbsp;splitDocuments =&nbsp;await&nbsp;logTextSplitter.splitDocuments([logDocument]);// console.log(splitDocuments);const&nbsp;enc = getEncoding("cl100k_base");splitDocuments.forEach(document&nbsp;=&gt;&nbsp;{&nbsp; &nbsp;&nbsp;console.log(document);&nbsp; &nbsp;&nbsp;console.log('charater length:',document.pageContent.length);&nbsp; &nbsp;&nbsp;console.log('token length:',enc.encode(document.pageContent).length);});

一段日志文本，按照换行符来分割，每个块 200 字符。

打印下每个块的字符数，然后用 js-tiktoken 看一下 token数

跑一下：

> 🎬 视频演示（原公众号视频）

可以看到，按照换行符分割文本，然后按照 chunk size 放到了 3 个块里。

有同学可能会问，chunk 的大小也没有到 200 啊？

因为 splitter 会优先保证语义完整，宁愿 chunk 小一点。

这里到了 160 左右字符的时候，发现加上下一个文本就超过 200 了，所以会放到下一个块。

这里因为没有被断开的文本，所以就没有需要加 overlap 重复的，**只有被断开的文本才有 overlap**

我们加一个长的文本试一下：

    const&nbsp;logDocument =&nbsp;new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`[2024-01-15 10:00:00] INFO: Application started[2024-01-15 10:00:05] DEBUG: Loading configuration file[2024-01-15 10:00:10] INFO: Database connection established[2024-01-15 10:00:15] WARNING: Rate limit approaching[2024-01-15 10:00:20] ERROR: Failed to process request[2024-01-15 10:00:25] INFO: Retrying operation[2024-01-15 10:00:30] SUCCESS: Operation completed[2026-01-10 14:30:00] INFO: 系统开始执行大规模数据迁移任务，本次迁移涉及核心业务数据库中的用户表、订单表、商品库存表、物流信息表、支付记录表、评论数据表等共计十二个关键业务表，预计处理数据量约500万条记录，数据总大小预估为280GB，迁移过程将采用分批次增量更新策略以减少对生产环境的影响，同时启用双写机制确保数据一致性，任务预计总耗时约3小时15分钟，迁移完成后将自动触发全面的数据一致性校验流程以及性能基准测试，请相关运维人员和DBA团队密切关注系统资源使用情况、网络带宽占用率以及任务执行进度，如遇异常情况请立即启动应急预案并通知技术负责人`});

> 🎬 视频演示（原公众号视频）

看到问题了么？

CharacterTextSplitter 非常死板，你告诉它按照换行符分割，它就会严格按照这个，就算超过了 chunk size 也不拆分。

所以一般还是用 RecursiveCharacterTextSplitter

创建 src/RecursiveCharacterTextSplitter-test.mjs

    import&nbsp;"dotenv/config";import"cheerio";import&nbsp;{ RecursiveCharacterTextSplitter }&nbsp;from"langchain_textsplitters";import&nbsp;{ Document }&nbsp;from"langchain_core/documents";import&nbsp;{ getEncoding }&nbsp;from"js-tiktoken";&nbsp;const&nbsp;logDocument =&nbsp;new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`[2024-01-15 10:00:00] INFO: Application started[2024-01-15 10:00:05] DEBUG: Loading configuration file[2024-01-15 10:00:10] INFO: Database connection established[2024-01-15 10:00:15] WARNING: Rate limit approaching[2024-01-15 10:00:20] ERROR: Failed to process request[2024-01-15 10:00:25] INFO: Retrying operation[2024-01-15 10:00:30] SUCCESS: Operation completed[2026-01-10 14:30:00] INFO: 系统开始执行大规模数据迁移任务，本次迁移涉及核心业务数据库中的用户表、订单表、商品库存表、物流信息表、支付记录表、评论数据表等共计十二个关键业务表，预计处理数据量约500万条记录，数据总大小预估为280GB，迁移过程将采用分批次增量更新策略以减少对生产环境的影响，同时启用双写机制确保数据一致性，任务预计总耗时约3小时15分钟，迁移完成后将自动触发全面的数据一致性校验流程以及性能基准测试，请相关运维人员和DBA团队密切关注系统资源使用情况、网络带宽占用率以及任务执行进度，如遇异常情况请立即启动应急预案并通知技术负责人`});const&nbsp;logTextSplitter =&nbsp;new&nbsp;RecursiveCharacterTextSplitter({&nbsp; &nbsp;&nbsp;chunkSize:&nbsp;150,&nbsp; &nbsp;&nbsp;chunkOverlap:&nbsp;20,&nbsp; &nbsp;&nbsp;separators: ['\n',&nbsp;'。',&nbsp;'，']});const&nbsp;splitDocuments =&nbsp;await&nbsp;logTextSplitter.splitDocuments([logDocument]);// console.log(splitDocuments);const&nbsp;enc = getEncoding("cl100k_base");splitDocuments.forEach(document&nbsp;=&gt;&nbsp;{&nbsp; &nbsp;&nbsp;console.log(document);&nbsp; &nbsp;&nbsp;console.log('charater length:',document.pageContent.length);&nbsp; &nbsp;&nbsp;console.log('token length:',enc.encode(document.pageContent).length);});

它可以指定多个分隔符：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/8_公众号_Yi昭.png)

当 “\n” 分割后还是大，就会用 “。” 还是不行再尝试用 “，”

> 🎬 视频演示（原公众号视频）

这样就明显好很多：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/9_公众号_Yi昭.png)

这两段文本是用换行符分割的。

按照换行符分割后下面的文本超过 chunk size，就会尝试按照句号逗号分割，然后加上 overlap：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/10_公众号_Yi昭.png)

最后这个是按照逗号分隔的，也没超过 chunk size，就没有 overlap了：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/11_公众号_Yi昭.png)

所以说 RecursiveCharacterTextSplitter 这种递归的方式灵活太多了。

绝大多数情况下，用这个就可以了。

然后我们再来试一下 TokenTextSplitter：

创建 src/TokenTextSplitter-test.mjs

    import&nbsp;"dotenv/config";import"cheerio";import&nbsp;{ TokenTextSplitter }&nbsp;from"langchain_textsplitters";import&nbsp;{ Document }&nbsp;from"langchain_core/documents";import&nbsp;{ getEncoding }&nbsp;from"js-tiktoken";&nbsp;const&nbsp;logDocument =&nbsp;new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent:&nbsp;`[2024-01-15 10:00:00] INFO: Application started[2024-01-15 10:00:05] DEBUG: Loading configuration file[2024-01-15 10:00:10] INFO: Database connection established[2024-01-15 10:00:15] WARNING: Rate limit approaching[2024-01-15 10:00:20] ERROR: Failed to process request[2024-01-15 10:00:25] INFO: Retrying operation[2024-01-15 10:00:30] SUCCESS: Operation completed`});const&nbsp;logTextSplitter =&nbsp;new&nbsp;TokenTextSplitter({&nbsp; &nbsp;&nbsp;chunkSize:&nbsp;50, &nbsp; &nbsp; &nbsp; &nbsp;// 每个块最多 50 个 Token&nbsp; &nbsp;&nbsp;chunkOverlap:&nbsp;10, &nbsp; &nbsp;// 块之间重叠 10 个 Token&nbsp; &nbsp;&nbsp;encodingName:&nbsp;'cl100k_base', &nbsp;// OpenAI 使用的编码方式&nbsp; });const&nbsp;splitDocuments =&nbsp;await&nbsp;logTextSplitter.splitDocuments([logDocument]);// console.log(splitDocuments);const&nbsp;enc = getEncoding("cl100k_base");splitDocuments.forEach(document&nbsp;=&gt;&nbsp;{&nbsp; &nbsp;&nbsp;console.log(document);&nbsp; &nbsp;&nbsp;console.log('charater length:',document.pageContent.length);&nbsp; &nbsp;&nbsp;console.log('token length:',enc.encode(document.pageContent).length);});

用这个 splitter，然后指定下编码：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/12_公众号_Yi昭.png)

跑一下：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/13_公众号_Yi昭.png)

可以看到，它优先保证 token 正好是 50，为了这个不惜强行打断文本。

当然，打断后也加了 overlap：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/14_公众号_Yi昭.png)

RecursiveCharacterTextSplitter 分出的 chunk 可能大于 chunk size，也可以小，优先保证语义完整，是按照分割符来分割。

但是 TokenTextSplitter 不是，它会只会保证 token 数量

这种不管不顾的分割显然不靠谱，不一定在什么地方就断开了。

还是 RecursiveCharacterTextSplitter 那种更科学。

那能不能用 RecursiveCharacterTextSplitter 的分割方式，然后按照 token 长度来设置 chunk size 呢？

可以的。重写一下它的长度计算函数就可以了：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/15_公众号_Yi昭.png)

这样，chunk size 指的就是 token 的长度

    const&nbsp;enc = getEncoding("cl100k_base");const&nbsp;logTextSplitter =&nbsp;new&nbsp;RecursiveCharacterTextSplitter({&nbsp; &nbsp;&nbsp;chunkSize:&nbsp;150,&nbsp; &nbsp;&nbsp;chunkOverlap:&nbsp;20,&nbsp; &nbsp;&nbsp;separators: ['\n',&nbsp;'。',&nbsp;'，'],&nbsp; &nbsp;&nbsp;lengthFunction:&nbsp;(text) =&gt;&nbsp;enc.encode(text).length,});

现在就是按照现在的 token 数量作为分割依据了：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/16_公众号_Yi昭.png)

这样就完全不需要用 TokenTextSplitter。

最后再来看一下 markdown、latex、代码的分割。

其实这些很明显，都是 RecursiveCharacterTextSplitter 实现的。

比如 markdown 是按照 #、##、### 的子标题来递归分割

latex 是按照那些数学公式的语法来分割

代码则是分语言来用不同的分割符。

但总体来说都是递归分割，所以他们都是用 RecursiveCharacterTextSplitter 实现的。

我们快速测一下：

创建 recursive-splitter-markdown.mjs

    import&nbsp;"dotenv/config";import"cheerio";import&nbsp;{ Document }&nbsp;from"langchain_core/documents";import&nbsp;{ MarkdownTextSplitter }&nbsp;from"langchain_textsplitters";const&nbsp;readmeText =&nbsp;`# Project Name&gt; A brief description of your project[

![License](https://img.shields.io/badge/license-MIT-blue.svg)

](LICENSE)## Features- ✨ Feature 1- 🚀 Feature 2- 💡 Feature 3## Installation\`\`\`bashpip install project-name\`\`\`## Usage### Basic Usage\`\`\`javascriptimport { Project } from 'project-name';const project = new Project();project.init();\`\`\`### Advanced Usage\`\`\`javascriptconst project = new Project({&nbsp; config: {&nbsp; &nbsp; apiKey: 'your-api-key',&nbsp; &nbsp; timeout: 5000,&nbsp; }});await project.run();\`\`\`## API Reference### \`Project\`Main class for the project.#### Methods- \`init()\`: Initialize the project- \`run()\`: Run the project- \`stop()\`: Stop the project## ContributingContributions are welcome! Please read our [contributing guide](CONTRIBUTING.md).## LicenseMIT License`;const&nbsp;readmeDoc =&nbsp;new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent: readmeText});const&nbsp;markdownTextSplitter =&nbsp;new&nbsp;MarkdownTextSplitter({&nbsp; &nbsp;&nbsp;chunkSize:&nbsp;400,&nbsp; &nbsp;&nbsp;chunkOverlap:&nbsp;80});const&nbsp;splitDocuments =&nbsp;await&nbsp;markdownTextSplitter.splitDocuments([readmeDoc]);// console.log(splitDocuments);splitDocuments.forEach(document&nbsp;=&gt;&nbsp;{&nbsp; &nbsp;&nbsp;console.log(document);&nbsp; &nbsp;&nbsp;console.log('charater length:',document.pageContent.length);});

创建 MarkdownTextSplitter，不用指定分割符，内置了。

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/17_公众号_Yi昭.png)

跑一下：

> 🎬 视频演示（原公众号视频）

可以看到，都是从标题处断开的，也就是根据语法分割的。

再来试下 letex

创建 src/recursive-splitter-latex.mjs

    import&nbsp;"dotenv/config";import"cheerio";import&nbsp;{ Document }&nbsp;from"langchain_core/documents";import&nbsp;{ LatexTextSplitter }&nbsp;from"langchain_textsplitters";const&nbsp;latexText =&nbsp;`\int x^{\mu}\mathrm{d}x=\frac{x^{\mu +1}}{\mu +1}+C, \left({\mu \neq -1}\right) \int \frac{1}{\sqrt{1-x^{2}}}\mathrm{d}x= \arcsin x +C \int \frac{1}{\sqrt{1-x^{2}}}\mathrm{d}x= \arcsin x +C \begin{pmatrix} &nbsp;&nbsp; a_{11} &amp; a_{12} &amp; a_{13} \\ &nbsp;&nbsp; a_{21} &amp; a_{22} &amp; a_{23} \\ &nbsp;&nbsp; a_{31} &amp; a_{32} &amp; a_{33} &nbsp;\end{pmatrix} `;const&nbsp;latexDoc =&nbsp;new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent: latexText});const&nbsp;markdownTextSplitter =&nbsp;new&nbsp;LatexTextSplitter({&nbsp; &nbsp;&nbsp;chunkSize:&nbsp;200,&nbsp; &nbsp;&nbsp;chunkOverlap:&nbsp;40});const&nbsp;splitDocuments =&nbsp;await&nbsp;markdownTextSplitter.splitDocuments([latexDoc]);// console.log(splitDocuments);splitDocuments.forEach(document&nbsp;=&gt;&nbsp;{&nbsp; &nbsp;&nbsp;console.log(document);&nbsp; &nbsp;&nbsp;console.log('charater length:',document.pageContent.length);});

跑一下：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/18_公众号_Yi昭.png)

也是按照正确的语法分割的。

最后来试下代码的：

创建 src/recursive-splitter-code.mjs

    import&nbsp;"dotenv/config";import"cheerio";import&nbsp;{ Document }&nbsp;from"langchain_core/documents";import&nbsp;{ LatexTextSplitter, RecursiveCharacterTextSplitter }&nbsp;from"langchain_textsplitters";const&nbsp;jsCode =&nbsp;`// Complete shopping cart implementationclass Product {&nbsp; constructor(id, name, price, description) {&nbsp; &nbsp; this.id = id;&nbsp; &nbsp; this.name = name;&nbsp; &nbsp; this.price = price;&nbsp; &nbsp; this.description = description;&nbsp; }&nbsp; getFormattedPrice() {&nbsp; &nbsp; return '$' + this.price.toFixed(2);&nbsp; }}class ShoppingCart {&nbsp; constructor() {&nbsp; &nbsp; this.items = [];&nbsp; &nbsp; this.discountCode = null;&nbsp; &nbsp; this.taxRate = 0.08;&nbsp; }&nbsp; addItem(product, quantity = 1) {&nbsp; &nbsp; const existingItem = this.items.find(item =&gt; item.product.id === product.id);&nbsp; &nbsp; if (existingItem) {&nbsp; &nbsp; &nbsp; existingItem.quantity += quantity;&nbsp; &nbsp; } else {&nbsp; &nbsp; &nbsp; this.items.push({ product, quantity, addedAt: new Date() });&nbsp; &nbsp; }&nbsp; &nbsp; return this;&nbsp; }&nbsp; removeItem(productId) {&nbsp; &nbsp; this.items = this.items.filter(item =&gt; item.product.id !== productId);&nbsp; &nbsp; return this;&nbsp; }&nbsp; calculateSubtotal() {&nbsp; &nbsp; return this.items.reduce((total, item) =&gt; {&nbsp; &nbsp; &nbsp; return total + (item.product.price * item.quantity);&nbsp; &nbsp; }, 0);&nbsp; }&nbsp; calculateTotal() {&nbsp; &nbsp; const subtotal = this.calculateSubtotal();&nbsp; &nbsp; const discount = this.calculateDiscount();&nbsp; &nbsp; const tax = (subtotal - discount) * this.taxRate;&nbsp; &nbsp; return subtotal - discount + tax;&nbsp; }&nbsp; calculateDiscount() {&nbsp; &nbsp; if (!this.discountCode) return 0;&nbsp; &nbsp; const discounts = { 'SAVE10': 0.10, 'SAVE20': 0.20, 'WELCOME': 0.15 };&nbsp; &nbsp; return this.calculateSubtotal() * (discounts[this.discountCode] || 0);&nbsp; }}// Usage exampleconst product1 = new Product(1, 'Laptop', 999.99, 'High-performance laptop');const product2 = new Product(2, 'Mouse', 29.99, 'Wireless mouse');const cart = new ShoppingCart();cart.addItem(product1, 1).addItem(product2, 2);console.log('Total:', cart.calculateTotal());`;const&nbsp;jsCodeDoc =&nbsp;new&nbsp;Document({&nbsp; &nbsp;&nbsp;pageContent: jsCode});const&nbsp;codeSplitter = RecursiveCharacterTextSplitter.fromLanguage('js', {&nbsp; &nbsp;&nbsp;chunkSize:&nbsp;300,&nbsp; &nbsp;&nbsp;chunkOverlap:&nbsp;60,})const&nbsp;splitDocuments =&nbsp;await&nbsp;codeSplitter.splitDocuments([jsCodeDoc]);// console.log(splitDocuments);splitDocuments.forEach(document&nbsp;=&gt;&nbsp;{&nbsp; &nbsp;&nbsp;console.log(document);&nbsp; &nbsp;&nbsp;console.log('charater length:',document.pageContent.length);});

用 RecursiveCharacterTextSplitter.fromLanguage 这个方法，指定语言，就会按照对应的语法来分割。

支持的语言有很多，包括： java、go、js、html、python、rust、swift、markdown 等

我们用最熟悉的 js 来测试下这个分割。

> 🎬 视频演示（原公众号视频）

可以看到，完全没有破坏代码完整性，确实是按照语法分割的。

这样，我们就把所有 splitter 过了一遍：

![image](../IMG/2026-01-10_LangChain全部Splitter，其实只需要其中的一个/19_公众号_Yi昭.png)

其实看到这里你应该也有答案了，基本就用 RecursiveCharacterTextSplitter 就行。

另外两个都有很明显的缺点：

CharacterTextSplitter 功能 RecursiveCharacterTextSplitter 里都有

TokenTextSplitter 严格按照 token，会破坏文档语义，不如 RecursiveCharacterTextSplitter 重写 lengthFunction

另外两个则是 RecursiveCharacterTextSplitter 的子功能。

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

这节我们把所有 splitter 过了一遍。

结论是直接用 RecursiveCharacterTextSplitter 就行。

splitter 是先按照 sperator 来分割，然后按照 chunk size 放到一个个 chunk 里。

chunk 的实际大小可能小于 chunk size 也可以大于。

如果分割后文本长度大于 chunk size，会继续按照后面的 sperator 拆分，然后放到两个 chunk 里，加上 overlap 来保证语义连贯。

如果从前到后尝试 sperator，尝试到最后一个，拆分完还是大于 chunk size 就不会再拆分了。

默认是按照字符计数，如果你想严格控制 token 大小，比如需要计费的场景，就可以实现 lengthFunction 用 token 的方式计算长度。

RecursiveCharacterTextSplitter 还支持代码分割，用 fromLanguage 的静态方法，这个在处理代码文档的时候很有用。

虽然这节讲了很多，但是结论很简单，就是用 RecursiveCharacterTextSplitter 就好了。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-01-10 19:13:59 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247484159&idx=1&sn=67fb03746d4c8f9cd66173d72f73e199&chksm=f117f9cc986be49887dc8d2a112d8d55869f6bc5d7c9e77252a7248e8447bec7c669ccd2d6ae&scene=27#wechat_redirect
