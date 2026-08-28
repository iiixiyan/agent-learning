# 基于 Docker Compose 的本地开发提效和生产环境部署

> **Python 版** | 原课程基于 Node.js(Nest.js) + LangChain JS，本文转换为 Python(FastAPI) + LangChain Python 技术栈

---

神光的幸福生活 2026年4月25日 20:01

业务项目的 Agent 都是在后端跑的。

比如业务数据存在 MySQL，知识存在向量数据库 Milvus、短期记忆存 Redis、需要关键词检索的放在 ElasticSearch 等。

而且你在招聘软件上搜 Agent 岗位，基本都是后端岗，所以做 Agent 开发，必须得学后端技术。

这节开始，我们集中把后端的数据库与中间件过一遍。

数据库是业务的“压舱石”，负责持久化存储原始业务数据，比如 MySQL 存用户信息。核心要求是稳健、不丢失。

而中间件则是各类独立的辅助基础软件。如果说数据库是全能但笨重的“仓库”，中间件就是各怀绝技的“特种兵”，用来补足数据库和业务逻辑的短板：

- 检索补足：MySQL 不擅长全文模糊搜索，我们就引入 Elasticsearch 专门做高性能检索
- 性能补足：核心数据库读写磁盘太慢，我们就用 Redis 这种内存级中间件来做高速缓存
- 异步补足：业务逻辑处理太耗时，我们就用 RabbitMQ 或 BullMQ 这类消息队列中间件来做任务缓冲和解耦。

简单区分：

数据库：核心是持久化，存的是业务的“资产”，追求数据的绝对可靠。

中间件：核心是专项能力，它不负责通用的持久存储，而是提供单一的强力支持（如检索、缓存、消息调度）。

在全栈开发中，你的代码就像是“指挥官”。懂业务逻辑只是及格，能根据场景精准调度这些中间件去解决性能、并发和搜索痛点，才是真正迈向“后端架构师”的标志。

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/0_公众号_Yi昭.png)

**数据库是根，中间件是特种兵。**

如图，mysql存的是业务原始数据，是根，不能丢。

而 redis 专门做缓存、es 做全文检索、milvus 做语义检索、、bullmq 做消息队列，是用于专门的用途，各司其职、专精专用，它们不是原始数据，丢了也不影响数据完整性。

而**业务代码是数据库和中间件的调度者，整合所有底层组件，最终实现完整的业务功能，对外提供服务。**

理解了业务代码、中间件、数据库这三者的区别和联系，我们这些先把 docker 学一下。

因为数据库、中间件、业务代码，我们都会通过 docker 来跑。

Docker 将应用及其依赖环境统一封装为镜像，镜像运行后就成为容器。

一台服务器可以同时运行多个容器，容器之间相互隔离，拥有独立的文件系统、网络、端口等环境，互不干扰，专门用来运行各类服务。

这样整个环境都保存在这个镜像里，部署多个实例只要通过这个镜像跑多个容器就行。

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/1_公众号_Yi昭.png)

这也是为什么它的 logo 是这样的：

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/2_公众号_Yi昭.png)

Docker 提供了 Docker Hub 镜像仓库，可以把本地镜像 push 到仓库或者从仓库 pull 镜像到本地。

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/3_公众号_Yi昭.png)

我们之前在 docker desktop 里下载的镜像，就是从 docker hub 搜的。

当然，通过命令行执行 docker pull 也可以。

这些就是我们前面下载的镜像（image）

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/4_公众号_Yi昭.png)

这些是镜像跑起来的容器（container）实例：

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/5_公众号_Yi昭.png)

跑容器时的参数，基本都讲过：

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/6_公众号_Yi昭.png)

port 是映射宿主机的端口到容器内的端口。

下面是环境变量。

这个 volume 数据卷是挂载本地某个目录到容器内的。

虽然在容器内跑数据库，但我们希望数据能持久化保存到宿主机，这样下次跑其他容器，也能用这个目录下的数据。

这就是数据卷 volume 的作用，把它挂载到容器就好了。

上面这些用命令行就是这样：

    docker run -d \&nbsp; --name mysql-container2 \&nbsp; -p 3306:3306 \&nbsp; -e MYSQL_ROOT_PASSWORD=admin \&nbsp; -v /Users/guang/mysql:/var/lib/mysql \&nbsp; mysql:latest

在界面上填的参数本质上就是这行命令。

> 🎬 视频演示（原公众号视频）

前面是跑的 mysql、milvus 这种镜像，那如果我们想自己创建一个 docker 镜像呢？

比如把之前的 Nest 项目打包成镜像。

这种就要写 Dockerfile 了。

比如这样：

    # 指定基础镜像（必须第一行）FROM&nbsp;node:24.15-alpine# 设置容器内工作目录WORKDIR&nbsp;/app# 先复制 package.json 利用缓存加速COPY&nbsp;package*.json ./# 构建时执行：安装依赖RUN&nbsp;npm config&nbsp;set&nbsp;registry https://registry.npmmirror.com/RUN&nbsp;npm installRUN&nbsp;npm install -g @nestjs/cli# 复制项目所有代码到容器内COPY&nbsp;. .# 构建 Nest 项目（编译成 JS）RUN&nbsp;npm run build# 声明暴露端口（仅声明）EXPOSE3000# 容器启动时执行的命令（启动 Nest 服务）CMD&nbsp;["node",&nbsp;"dist/main.js"]

这些指令的含义如下：

- FROM：指定基础镜像，一切从这个镜像开始构建
- WORKDIR：指定容器内的工作目录，后续命令都在这个目录执行
- COPY：将宿主机的文件 / 目录复制到容器内部
- RUN：在构建镜像时执行命令，比如安装依赖、编译项目
- EXPOSE：声明容器要暴露的端口，仅作声明，方便阅读
- CMD：容器启动时执行的默认命令，一个 Dockerfile 只能有一个 CMD

我们创建个 nest 项目，打包成镜像试试：

    nest new nest-dockerfile-test

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/7_公众号_Yi昭.png)

创建一个增删改查模块：

    nest g res book --no-spec

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/8_公众号_Yi昭.png)

然后根目录创建 Dockerfile（刚才那个复制过来）

加一个 .dockerignore

    node_modules/.vscode/.git/

这些是复制的时候忽略的文件

打包成镜像：

    docker build -t nest-app .

-t 是指定镜像名字

然后跑一下：

    docker run -d \&nbsp; --name nest-container \&nbsp; -p 3006:3000 \&nbsp; nest-app

> 🎬 视频演示（原公众号视频）

现在这样是可以的，但是镜像里会多了一些无关代码

比如源码、非生产环境的依赖等

会导致镜像体积更大

所以我们一般用多阶段构建来写 Dockerfile：

    # 构建阶段：需要 devDependencies（含 @nestjs/cli、typescript）才能 nest buildFROM&nbsp;node:24.15-alpine AS builderWORKDIR&nbsp;/appCOPY&nbsp;package*.json ./RUN&nbsp;npm config&nbsp;set&nbsp;registry https://registry.npmmirror.com/RUN&nbsp;npm installCOPY&nbsp;. .RUN&nbsp;npm run build# 运行阶段：仅生产依赖 + 编译产物，镜像更小FROM&nbsp;node:24.15-alpineENV&nbsp;NODE_ENV=productionWORKDIR&nbsp;/appCOPY&nbsp;package*.json ./RUN&nbsp;npm config&nbsp;set&nbsp;registry https://registry.npmmirror.com/RUN&nbsp;npm install --productionCOPY&nbsp;--from=builder /app/dist ./distEXPOSE3000CMD&nbsp;["node",&nbsp;"dist/main.js"]

就是第一个阶段镜像只用于构建

之后再创建一个镜像，把前一个镜像构建出来的代码复制过去，跑起来

这样只保留最后一个镜像的文件，显然体积会更小

这就是多阶段构建

> 🎬 视频演示（原公众号视频）

镜像体积小了 400M

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/9_公众号_Yi昭.png)

现在有了 mysql、milvus 等镜像，有了 nest 服务的镜像

如果想让它们一起跑呢？

这就需要 Docker Compose 了

其实之前我们跑 Milvus，就是用 docker compose：

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/10_公众号_Yi昭.png)

它基于 3 个 docker 镜像来跑的。

当时我们就是基于一个 docker compose 的配置文件跑起来的：

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/11_公众号_Yi昭.png)

> 🎬 视频演示（原公众号视频）

**Docker Compose 用于编排多个容器，统一管理启动参数、依赖顺序与网络环境。**

**所有容器默认处于同一内网，天然互通，可直接用容器名互相调用。**

milvus 是这么跑的，我们自己的项目也是用这种方式来跑。

首先，本地开发我们要跑 mysql、milvus 等，之前都是手动在 docker desktop 里跑，其实可以用 docker compose 文件统一跑：

创建 docker-compose.dev.yml

    version:&nbsp;'3.8'services:# MySQLmysql:&nbsp; &nbsp;&nbsp;image:mysql:latest&nbsp; &nbsp;&nbsp;container_name:mysql-dev&nbsp; &nbsp;&nbsp;ports:&nbsp; &nbsp; &nbsp;&nbsp;-"3306:3306"&nbsp; &nbsp;&nbsp;environment:&nbsp; &nbsp; &nbsp;&nbsp;MYSQL_ROOT_PASSWORD:admin&nbsp; &nbsp;&nbsp;command:mysqld--character-set-server=utf8mb4--collation-server=utf8mb4_general_ci# 设置默认字符集&nbsp; &nbsp;&nbsp;volumes:&nbsp; &nbsp; &nbsp;&nbsp;-${DOCKER_VOLUME_DIRECTORY:-.}/volumes/mysql:/var/lib/mysql&nbsp; &nbsp;&nbsp;restart:always# Milvusetcd:&nbsp; &nbsp;&nbsp;container_name:milvus-etcd&nbsp; &nbsp;&nbsp;image:quay.io/coreos/etcd:v3.5.18&nbsp; &nbsp;&nbsp;environment:&nbsp; &nbsp; &nbsp;&nbsp;-ETCD_AUTO_COMPACTION_MODE=revision&nbsp; &nbsp; &nbsp;&nbsp;-ETCD_AUTO_COMPACTION_RETENTION=1000&nbsp; &nbsp; &nbsp;&nbsp;-ETCD_QUOTA_BACKEND_BYTES=4294967296&nbsp; &nbsp; &nbsp;&nbsp;-ETCD_SNAPSHOT_COUNT=50000&nbsp; &nbsp;&nbsp;volumes:&nbsp; &nbsp; &nbsp;&nbsp;-${DOCKER_VOLUME_DIRECTORY:-.}/volumes/etcd:/etcd&nbsp; &nbsp;&nbsp;command:etcd-advertise-client-urls=http://etcd:2379-listen-client-urlshttp://0.0.0.0:2379--data-dir/etcd&nbsp; &nbsp;&nbsp;healthcheck:&nbsp; &nbsp; &nbsp;&nbsp;test:["CMD","etcdctl","endpoint","health"]&nbsp; &nbsp; &nbsp;&nbsp;interval:30s&nbsp; &nbsp; &nbsp;&nbsp;timeout:20s&nbsp; &nbsp; &nbsp;&nbsp;retries:3minio:&nbsp; &nbsp;&nbsp;container_name:milvus-minio&nbsp; &nbsp;&nbsp;image:minio/minio:RELEASE.2024-05-28T17-19-04Z&nbsp; &nbsp;&nbsp;environment:&nbsp; &nbsp; &nbsp;&nbsp;MINIO_ACCESS_KEY:minioadmin&nbsp; &nbsp; &nbsp;&nbsp;MINIO_SECRET_KEY:minioadmin&nbsp; &nbsp;&nbsp;ports:&nbsp; &nbsp; &nbsp;&nbsp;-"9001:9001"&nbsp; &nbsp; &nbsp;&nbsp;-"9000:9000"&nbsp; &nbsp;&nbsp;volumes:&nbsp; &nbsp; &nbsp;&nbsp;-${DOCKER_VOLUME_DIRECTORY:-.}/volumes/minio:/minio_data&nbsp; &nbsp;&nbsp;command:minioserver/minio_data--console-address":9001"&nbsp; &nbsp;&nbsp;healthcheck:&nbsp; &nbsp; &nbsp;&nbsp;test:["CMD","curl","-f","http://localhost:9000/minio/health/live"]&nbsp; &nbsp; &nbsp;&nbsp;interval:30s&nbsp; &nbsp; &nbsp;&nbsp;timeout:20s&nbsp; &nbsp; &nbsp;&nbsp;retries:3standalone:&nbsp; &nbsp;&nbsp;container_name:milvus-standalone&nbsp; &nbsp;&nbsp;image:milvusdb/milvus:v2.5.25&nbsp; &nbsp;&nbsp;command:["milvus","run","standalone"]&nbsp; &nbsp;&nbsp;security_opt:&nbsp; &nbsp; &nbsp;&nbsp;-seccomp:unconfined&nbsp; &nbsp;&nbsp;environment:&nbsp; &nbsp; &nbsp;&nbsp;MINIO_REGION:us-east-1&nbsp; &nbsp; &nbsp;&nbsp;ETCD_ENDPOINTS:etcd:2379&nbsp; &nbsp; &nbsp;&nbsp;MINIO_ADDRESS:minio:9000&nbsp; &nbsp;&nbsp;volumes:&nbsp; &nbsp; &nbsp;&nbsp;-${DOCKER_VOLUME_DIRECTORY:-.}/volumes/milvus:/var/lib/milvus&nbsp; &nbsp;&nbsp;healthcheck:&nbsp; &nbsp; &nbsp;&nbsp;test:["CMD","curl","-f","http://localhost:9091/healthz"]&nbsp; &nbsp; &nbsp;&nbsp;interval:30s&nbsp; &nbsp; &nbsp;&nbsp;start_period:90s&nbsp; &nbsp; &nbsp;&nbsp;timeout:20s&nbsp; &nbsp; &nbsp;&nbsp;retries:3&nbsp; &nbsp;&nbsp;ports:&nbsp; &nbsp; &nbsp;&nbsp;-"19530:19530"&nbsp; &nbsp; &nbsp;&nbsp;-"9091:9091"&nbsp; &nbsp;&nbsp;depends_on:&nbsp; &nbsp; &nbsp;&nbsp;-"etcd"&nbsp; &nbsp; &nbsp;&nbsp;-"minio"networks:default:&nbsp; &nbsp;&nbsp;name:common-network

milvus 的部分复制之前那个 docker compose 配置文件的，我们加上了 mysql 的容器

重点是这里：

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/12_公众号_Yi昭.png)

这个 ${DOCKER\_VOLUME\_DIRECTORY:-.}  的意思是，如果我们指定了环境变量 DOCKER\_VOLUME\_DIRECTORY 是 /abc

那路径拼接起来就是：

/abc/volumes/mysql

没有指定就是：

./volumes/mysql

这样指定默认值，还支持环境变量来修改的方式更灵活。

在 package.json 里添加两个命令：

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/13_公众号_Yi昭.png)

    "docker:up":&nbsp;"DOCKER_VOLUME_DIRECTORY=/Users/guang/ docker compose -f docker-compose.dev.yml up -d","docker:down":&nbsp;"docker compose -f docker-compose.dev.yml down",

指定数据卷目录的环境变量，然后跑 docker compose up

以及停掉这些容器的 docker compose down

试一下：

> 🎬 视频演示（原公众号视频）

这样，本地环境就可以一键启动了，不用一个个跑 docker 容器。

接下来再写一下生产环境的 docker-compose.yml

首先，我们代码里用一下 mysql 做增删改查

安装 TypeORM 和 mysql 驱动包：

    pnpm install --save @nestjs/typeorm typeorm mysql2

在 AppModule 引入：

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/14_公众号_Yi昭.png)

    TypeOrmModule.forRoot({&nbsp;&nbsp;type:&nbsp;'mysql',host:&nbsp;'localhost',port:&nbsp;3306,username:&nbsp;'root',password:&nbsp;'admin',database:&nbsp;'book',synchronize:&nbsp;true,connectorPackage:&nbsp;'mysql2',logging:&nbsp;true,entities: []}),

改一下 book/entities/book.entity.ts

    import&nbsp;{&nbsp; Column,&nbsp; CreateDateColumn,&nbsp; Entity,&nbsp; PrimaryGeneratedColumn,&nbsp; UpdateDateColumn,}&nbsp;from'typeorm';@Entity({&nbsp;name:&nbsp;'books'&nbsp;})exportclass&nbsp;Book&nbsp;{&nbsp; @PrimaryGeneratedColumn()id: number;&nbsp; @Column({&nbsp;length:&nbsp;255&nbsp;})title: string;&nbsp; @Column({&nbsp;length:&nbsp;255&nbsp;})author: string;&nbsp; @Column({&nbsp;type:&nbsp;'text'&nbsp;})description: string;&nbsp; @Column({&nbsp;type:&nbsp;'decimal',&nbsp;precision:&nbsp;10,&nbsp;scale:&nbsp;2&nbsp;})price: number;&nbsp; @Column({&nbsp;type:&nbsp;'int',&nbsp;default:&nbsp;0&nbsp;})stock: number;&nbsp; @Column({&nbsp;type:&nbsp;'datetime'&nbsp;})publishedAt:&nbsp;Date;&nbsp; @CreateDateColumn({&nbsp;type:&nbsp;'datetime'&nbsp;})createdAt:&nbsp;Date;&nbsp; @UpdateDateColumn({&nbsp;type:&nbsp;'datetime'&nbsp;})updatedAt:&nbsp;Date;}

创建 book 的 entity，用 typeorm 做好和数据库表的映射。

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/15_公众号_Yi昭.png)

引入这个 Entity。

然后改下 BookService：

    import&nbsp;{ Inject, Injectable, NotFoundException }&nbsp;from'@nestjs/common';import&nbsp;{ EntityManager }&nbsp;from'typeorm';import&nbsp;{ CreateBookDto }&nbsp;from'./dto/create-book.dto';import&nbsp;{ UpdateBookDto }&nbsp;from'./dto/update-book.dto';import&nbsp;{ Book }&nbsp;from'./entities/book.entity';@Injectable()exportclass&nbsp;BookService&nbsp;{&nbsp; @Inject(EntityManager)&nbsp; private readonly entityManager: EntityManager;async&nbsp;create(createBookDto: CreateBookDto) {&nbsp; &nbsp;&nbsp;const&nbsp;book =&nbsp;this.entityManager.create(Book, {&nbsp; &nbsp; &nbsp; ...createBookDto,&nbsp; &nbsp; &nbsp;&nbsp;publishedAt:&nbsp;newDate(createBookDto.publishedAt),&nbsp; &nbsp; });&nbsp; &nbsp;&nbsp;returnthis.entityManager.save(Book, book);&nbsp; }async&nbsp;findAll() {&nbsp; &nbsp;&nbsp;returnthis.entityManager.find(Book, {&nbsp; &nbsp; &nbsp;&nbsp;order: {&nbsp;id:&nbsp;'DESC'&nbsp;},&nbsp; &nbsp; });&nbsp; }async&nbsp;findOne(id: number) {&nbsp; &nbsp;&nbsp;const&nbsp;book =&nbsp;awaitthis.entityManager.findOneBy(Book, { id });&nbsp; &nbsp;&nbsp;if&nbsp;(!book) {&nbsp; &nbsp; &nbsp;&nbsp;thrownew&nbsp;NotFoundException(`Book #${id}&nbsp;not found`);&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;return&nbsp;book;&nbsp; }async&nbsp;update(id: number,&nbsp;updateBookDto: UpdateBookDto) {&nbsp; &nbsp;&nbsp;const&nbsp;book =&nbsp;awaitthis.findOne(id);&nbsp; &nbsp;&nbsp;const&nbsp;{ publishedAt, ...restPayload } = updateBookDto;&nbsp; &nbsp;&nbsp;const&nbsp;updatePayload: Partial&lt;Book&gt; = { ...restPayload };&nbsp; &nbsp;&nbsp;if&nbsp;(publishedAt !==&nbsp;undefined) {&nbsp; &nbsp; &nbsp; updatePayload.publishedAt =&nbsp;newDate(publishedAt);&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;const&nbsp;mergedBook =&nbsp;this.entityManager.merge(Book, book, updatePayload);&nbsp; &nbsp;&nbsp;returnthis.entityManager.save(Book, mergedBook);&nbsp; }async&nbsp;remove(id: number) {&nbsp; &nbsp;&nbsp;const&nbsp;book =&nbsp;awaitthis.findOne(id);&nbsp; &nbsp;&nbsp;awaitthis.entityManager.remove(Book, book);&nbsp; &nbsp;&nbsp;return&nbsp;{&nbsp;deleted:&nbsp;true&nbsp;};&nbsp; }}

就是增删改查逻辑，不用细看。

对了，跑 docker 容器的时候，要让它自动创建 book 这个 database，然后 typeorm 才能在下面自动建表

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/16_公众号_Yi昭.png)

用 MYSQL\_DATABASE 这个环境变量指定。

跑一下：

> 🎬 视频演示（原公众号视频）

你可以用这个 curl 来测试：

    # 1) 新增（Create）curl -X POST&nbsp;"http://localhost:3000/book"&nbsp;\&nbsp; -H&nbsp;"Content-Type: application/json"&nbsp;\&nbsp; -d&nbsp;'{&nbsp; &nbsp; "title": "Clean Code",&nbsp; &nbsp; "author": "Robert C. Martin",&nbsp; &nbsp; "description": "A handbook of agile software craftsmanship",&nbsp; &nbsp; "price": 99.9,&nbsp; &nbsp; "stock": 50,&nbsp; &nbsp; "publishedAt": "2008-08-01"&nbsp; }'

    # 2) 查询全部（Read All）curl -X GET&nbsp;"http://localhost:3000/book"

我们加一个静态页面来测试：

安装依赖：

    pnpm install @nestjs/serve-static

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/17_公众号_Yi昭.png)

    ServeStaticModule.forRoot({&nbsp;&nbsp;rootPath: join(__dirname,&nbsp;'public'),&nbsp;&nbsp;serveRoot:&nbsp;'/books',}),

添加 public/index.html（ai 生成的，不用细看）

    &lt;!doctype&nbsp;html&gt;&lt;html&nbsp;lang="zh-CN"&gt;&lt;head&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;charset="UTF-8"&nbsp;/&gt;&nbsp; &nbsp;&nbsp;&lt;meta&nbsp;name="viewport"&nbsp;content="width=device-width, initial-scale=1.0"&nbsp;/&gt;&nbsp; &nbsp;&nbsp;&lt;title&gt;书籍管理&lt;/title&gt;&nbsp; &nbsp;&nbsp;&lt;style&gt;&nbsp; &nbsp; &nbsp;&nbsp;:root&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-family:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; Inter,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; -apple-system,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; BlinkMacSystemFont,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;'Segoe UI',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; sans-serif;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;body&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin:&nbsp;0;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#f7f8fa;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#222;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.container&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;max-width:&nbsp;980px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin:&nbsp;32px&nbsp;auto;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;016px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;h1&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-bottom:&nbsp;16px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.panel&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#fff;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;12px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;16px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;box-shadow:&nbsp;06px18pxrgba(0,&nbsp;0,&nbsp;0,&nbsp;0.08);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;margin-bottom:&nbsp;16px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;form&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;display: grid;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;grid-template-columns:&nbsp;repeat(2, minmax(0,&nbsp;1fr));&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;gap:&nbsp;12px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;label&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;display: flex;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;flex-direction: column;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;gap:&nbsp;4px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.full&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;grid-column:&nbsp;1&nbsp;/ -1;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;input,&nbsp; &nbsp; &nbsp;&nbsp;textarea&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;1px&nbsp;solid&nbsp;#d0d4dc;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;8px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;8px10px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;14px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;textarea&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;min-height:&nbsp;70px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.actions&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;display: flex;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;gap:&nbsp;8px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;button&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border:&nbsp;0;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-radius:&nbsp;8px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;9px12px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-weight:&nbsp;600;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;cursor: pointer;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.primary&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#2563eb;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#fff;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.muted&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#e5e7eb;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;table&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;width:&nbsp;100%;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-collapse: collapse;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;th,&nbsp; &nbsp; &nbsp;&nbsp;td&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;border-bottom:&nbsp;1px&nbsp;solid&nbsp;#eceff3;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;text-align: left;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;padding:&nbsp;10px8px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;14px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;vertical-align: top;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;.danger&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;background:&nbsp;#ef4444;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;color:&nbsp;#fff;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;#status&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;min-height:&nbsp;20px;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;font-size:&nbsp;14px;&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp;&nbsp;@media&nbsp;(max-width:700px) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;form&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;grid-template-columns:&nbsp;1fr;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;&lt;/style&gt;&lt;/head&gt;&lt;body&gt;&nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="container"&gt;&nbsp; &nbsp; &nbsp;&nbsp;&lt;h1&gt;书籍管理&lt;/h1&gt;&nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="panel"&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;form&nbsp;id="book-form"&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;input&nbsp;id="book-id"&nbsp;type="hidden"&nbsp;/&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;label&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 书名&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;input&nbsp;id="title"&nbsp;required&nbsp;/&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/label&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;label&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 作者&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;input&nbsp;id="author"&nbsp;required&nbsp;/&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/label&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;label&nbsp;class="full"&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 简介&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;textarea&nbsp;id="description"&nbsp;required&gt;&lt;/textarea&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/label&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;label&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 价格&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;input&nbsp;id="price"&nbsp;type="number"&nbsp;min="0"&nbsp;step="0.01"&nbsp;required&nbsp;/&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/label&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;label&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 库存&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;input&nbsp;id="stock"&nbsp;type="number"&nbsp;min="0"&nbsp;required&nbsp;/&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/label&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;label&nbsp;class="full"&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 出版日期&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;input&nbsp;id="publishedAt"&nbsp;type="date"&nbsp;required&nbsp;/&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/label&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="actions full"&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;button&nbsp;type="submit"&nbsp;class="primary"&gt;保存&lt;/button&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;button&nbsp;type="button"&nbsp;class="muted"&nbsp;id="reset-btn"&gt;清空&lt;/button&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/div&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/form&gt;&nbsp; &nbsp; &nbsp;&nbsp;&lt;/div&gt;&nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;class="panel"&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;div&nbsp;id="status"&gt;&lt;/div&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;table&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;thead&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;tr&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;th&gt;ID&lt;/th&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;th&gt;书名&lt;/th&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;th&gt;作者&lt;/th&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;th&gt;价格&lt;/th&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;th&gt;库存&lt;/th&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;th&gt;出版日期&lt;/th&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;th&gt;操作&lt;/th&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/tr&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/thead&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;tbody&nbsp;id="book-rows"&gt;&lt;/tbody&gt;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;&lt;/table&gt;&nbsp; &nbsp; &nbsp;&nbsp;&lt;/div&gt;&nbsp; &nbsp;&nbsp;&lt;/div&gt;&nbsp; &nbsp;&nbsp;&lt;script&gt;&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;form =&nbsp;document.getElementById('book-form');&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;rows =&nbsp;document.getElementById('book-rows');&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;statusNode =&nbsp;document.getElementById('status');&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;resetBtn =&nbsp;document.getElementById('reset-btn');&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;inputs = {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;id:&nbsp;document.getElementById('book-id'),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;title:&nbsp;document.getElementById('title'),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;author:&nbsp;document.getElementById('author'),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;description:&nbsp;document.getElementById('description'),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;price:&nbsp;document.getElementById('price'),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;stock:&nbsp;document.getElementById('stock'),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;publishedAt:&nbsp;document.getElementById('publishedAt'),&nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;setStatus =&nbsp;(text, isError =&nbsp;false) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; statusNode.textContent = text;&nbsp; &nbsp; &nbsp; &nbsp; statusNode.style.color = isError ?&nbsp;'#b91c1c'&nbsp;:&nbsp;'#2563eb';&nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;resetForm =&nbsp;()&nbsp;=&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; form.reset();&nbsp; &nbsp; &nbsp; &nbsp; inputs.id.value =&nbsp;'';&nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;mapFormData =&nbsp;()&nbsp;=&gt;&nbsp;({&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;title: inputs.title.value.trim(),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;author: inputs.author.value.trim(),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;description: inputs.description.value.trim(),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;price:&nbsp;Number(inputs.price.value),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;stock:&nbsp;Number(inputs.stock.value),&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;publishedAt: inputs.publishedAt.value,&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;createActionButton =&nbsp;(label, className, onClick) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;button =&nbsp;document.createElement('button');&nbsp; &nbsp; &nbsp; &nbsp; button.textContent = label;&nbsp; &nbsp; &nbsp; &nbsp; button.className = className;&nbsp; &nbsp; &nbsp; &nbsp; button.type =&nbsp;'button';&nbsp; &nbsp; &nbsp; &nbsp; button.addEventListener('click', onClick);&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;button;&nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;editBook =&nbsp;(book) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; inputs.id.value = book.id;&nbsp; &nbsp; &nbsp; &nbsp; inputs.title.value = book.title;&nbsp; &nbsp; &nbsp; &nbsp; inputs.author.value = book.author;&nbsp; &nbsp; &nbsp; &nbsp; inputs.description.value = book.description;&nbsp; &nbsp; &nbsp; &nbsp; inputs.price.value = book.price;&nbsp; &nbsp; &nbsp; &nbsp; inputs.stock.value = book.stock;&nbsp; &nbsp; &nbsp; &nbsp; inputs.publishedAt.value =&nbsp;newDate(book.publishedAt)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; .toISOString()&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; .split('T')[0];&nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;deleteBook =&nbsp;async&nbsp;(id) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!confirm(`确认删除书籍 #${id}&nbsp;吗？`))&nbsp;return;&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;response =&nbsp;await&nbsp;fetch(`/book/${id}`, {&nbsp;method:&nbsp;'DELETE'&nbsp;});&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!response.ok)&nbsp;thrownewError('删除失败');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus(`已删除书籍 #${id}`);&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;await&nbsp;loadBooks();&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus(error.message,&nbsp;true);&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;renderRows =&nbsp;(books) =&gt;&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; rows.innerHTML =&nbsp;'';&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!books.length) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; rows.innerHTML =&nbsp;'&lt;tr&gt;&lt;td colspan="7"&gt;暂无数据&lt;/td&gt;&lt;/tr&gt;';&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return;&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;for&nbsp;(const&nbsp;book&nbsp;of&nbsp;books) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;tr =&nbsp;document.createElement('tr');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; tr.innerHTML =&nbsp;`&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &lt;td&gt;${book.id}&lt;/td&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &lt;td&gt;${book.title}&lt;/td&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &lt;td&gt;${book.author}&lt;/td&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &lt;td&gt;${book.price}&lt;/td&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &lt;td&gt;${book.stock}&lt;/td&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &lt;td&gt;${new&nbsp;Date(book.publishedAt).toLocaleDateString()}&lt;/td&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &lt;td&gt;&lt;/td&gt;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; `;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;actionCell = tr.lastElementChild;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; actionCell.appendChild(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; createActionButton('编辑',&nbsp;'muted', () =&gt; editBook(book)),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; actionCell.appendChild(&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; createActionButton('删除',&nbsp;'danger', () =&gt; deleteBook(book.id)),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; );&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; rows.appendChild(tr);&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;loadBooks =&nbsp;async&nbsp;() =&gt; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;response =&nbsp;await&nbsp;fetch('/book');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!response.ok)&nbsp;thrownewError('加载书籍失败');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;books =&nbsp;await&nbsp;response.json();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; renderRows(books);&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus(error.message,&nbsp;true);&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; };&nbsp; &nbsp; &nbsp; form.addEventListener('submit',&nbsp;async&nbsp;(event) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp; event.preventDefault();&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;id = inputs.id.value.trim();&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;payload = mapFormData();&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;method = id ?&nbsp;'PATCH'&nbsp;:&nbsp;'POST';&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;url = id ?&nbsp;`/book/${id}`&nbsp;:&nbsp;'/book';&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;try&nbsp;{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;const&nbsp;response =&nbsp;await&nbsp;fetch(url, {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; method,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;headers: {&nbsp;'Content-Type':&nbsp;'application/json'&nbsp;},&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;body:&nbsp;JSON.stringify(payload),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;if&nbsp;(!response.ok) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;thrownewError(id ?&nbsp;'更新失败'&nbsp;:&nbsp;'创建失败');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus(id ?&nbsp;`已更新书籍 #${id}`&nbsp;:&nbsp;'已创建书籍');&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; resetForm();&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;await&nbsp;loadBooks();&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp;catch&nbsp;(error) {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; setStatus(error.message,&nbsp;true);&nbsp; &nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; resetBtn.addEventListener('click', resetForm);&nbsp; &nbsp; &nbsp; loadBooks();&nbsp; &nbsp;&nbsp;&lt;/script&gt;&lt;/body&gt;&lt;/html&gt;

看下效果：

> 🎬 视频演示（原公众号视频）

本地跑通之后，生产环境的 docker-compose.yml 怎么写呢？

创建 docker-compose.prod.yml

    services:&nbsp; mysql-prod:&nbsp; &nbsp; image: mysql:latest&nbsp; &nbsp; container_name: mysql-prod&nbsp; &nbsp; environment:&nbsp; &nbsp; &nbsp; MYSQL_ROOT_PASSWORD: admin&nbsp; &nbsp; &nbsp; MYSQL_DATABASE: book&nbsp; &nbsp; ports:&nbsp; &nbsp; &nbsp; -&nbsp;"3306:3306"&nbsp; &nbsp;&nbsp;command: mysqld --character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci&nbsp; &nbsp; volumes:&nbsp; &nbsp; &nbsp; -&nbsp;${DOCKER_VOLUME_DIRECTORY:-.}/volumes/mysql-prod:/var/lib/mysql&nbsp; &nbsp; restart: always&nbsp; nest-app:&nbsp; &nbsp; container_name: nest-app&nbsp; &nbsp; build:&nbsp; &nbsp; &nbsp; context: .&nbsp; &nbsp; &nbsp; dockerfile: Dockerfile&nbsp; &nbsp; ports:&nbsp; &nbsp; &nbsp; -&nbsp;"3000:3000"&nbsp; &nbsp; environment:&nbsp; &nbsp; &nbsp; NODE_ENV: production&nbsp; &nbsp; depends_on:&nbsp; &nbsp; &nbsp; - mysql-prod&nbsp; &nbsp; restart: always

这里 nest-app 的 docker 容器是从 Dockerfile 构建出来的

生成环境连接 mysql 是用容器名，也就是 mysql-prod

所以要改一下：

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/18_公众号_Yi昭.png)

此外，静态文件默认不会输出到 dist 目录，我们要配置下

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/19_公众号_Yi昭.png)

改下 nest-cli.json

    "compilerOptions": {&nbsp; &nbsp;&nbsp;"deleteOutDir":&nbsp;true,&nbsp; &nbsp;&nbsp;"assets": [&nbsp; &nbsp; &nbsp; {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"include":&nbsp;"../public/**/*",&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;"outDir":&nbsp;"dist/public"&nbsp; &nbsp; &nbsp; }&nbsp; &nbsp; ]&nbsp; }

加一个生产环境用的命令：

![image](../IMG/2026-04-25_基于DockerCompose的本地开发提效和生产环境部署/20_公众号_Yi昭.png)

    "docker:prod:up":&nbsp;"docker compose -f docker-compose.prod.yml up -d --build",

试一下：

> 🎬 视频演示（原公众号视频）

这样，我们就用 docker compose 实现了生产环境的部署。

> 代码上传了课程仓库： https://github.com/QuarkGluonPlasma/ai-agent-course-code

## 总结

Agent 开发离不开后端生态，这节我们开始学后端的技术。

首先我们明确了数据库、中间件、业务代码的区分。

数据库是根，存的是原始数据，中间件是特种兵，是完成特定用途的组件，比如缓存、全文检索、消息队列等。

业务代码调度数据库和中间件，实现完整的业务功能，对外提供服务

我们学了 docker 容器怎么跑，volume 数据卷的作用。

然后写了 Dockerfile，构建出了自己的 docker  镜像，并且基于多阶段构建实现了镜像大小的优化。

学了本地如何用 docker compose 一键启动多个容器

生产环境如何用 docker compose 来部署

后面我们本地开发、生产环境部署，都是基于 Docker Compose 的。

---

**公众号：** 神光的幸福生活 | **作者：** 神说要有光 | **发布时间：** 2026-04-25 20:01:46 | **文章地址：** http://mp.weixin.qq.com/s?__biz=MzYzNzI2MTI2Nw==&mid=2247485926&idx=1&sn=ba16ebb225c141d198e50510b45adcaf&chksm=f1ba9734a476866115f36b7876de2b0cf909e1aa1a30da87c0797d21f30b5cbbb94bf1cd75d4&scene=27#wechat_redirect
