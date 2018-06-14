# 无服务器架构（Serverless Architectures）

[![amio/serverless-zhcn](https://badgen.now.sh/badge/GitHub/amio%2Fserverless-zhcn/37F)](https://github.com/amio/serverless-zhcn)  
原文：[https://martinfowler.com/articles/serverless.html](https://martinfowler.com/articles/serverless.html)  
作者：[Mike Roberts](https://twitter.com/mikebroberts)

*无服务器架构（Serverless architectures）是指一个应用大量依赖第三方服务（后端即服务，Backend as a Service，简称“BaaS”），或者把代码交由托管的、短生命周期的容器中执行（函数即服务，Function as a Service，简称“FaaS”）。现在最知名的 FaaS 平台是 AWS Lambda。把这些技术和单页应用等相关概念相结合，这样的架构无需维护传统应用中永远保持在线的系统组件。Serverless 架构的长处是显著减少运维成本、复杂度、以及项目起步时间，劣势则在于更加依赖平台供应商和现阶段仍有待成熟的支持环境。*

## 引言

无服务器计算（Severless computing，简称 Serverless）现在是软件架构圈中的热门话题，三大云计算供应商（Amazon、Google 和 Microsoft）都在大力投入这个领域，涌现了不计其数的相关书籍、开源框架、商业产品、技术大会。到底什么是 Serverless？它有什么长处/短处？我希望通过本文对这些问题提供一些启发。

开篇我们先来看看 Serverless 是什么，之后我会尽我所能中立地谈谈它的优势和缺点。

## 什么是 Serverless

就像软件行业中的很多趋势一样，Serverless 的界限并不是特别清晰，尤其是它还涵盖了两个互相有重叠的概念：

- Serverless 最早用于描述那些大部分或者完全依赖于第三方（云端）应用或服务来管理服务器端逻辑和状态的应用，这些应用通常是富客户端应用（单页应用或者移动端 App），建立在云服务生态之上，包括数据库（Parse、Firebase）、账号系统（Auth0、AWS Cognito）等。这些服务最早被称为 [“(Mobile) Backend as a Service”](https://en.wikipedia.org/wiki/Mobile_backend_as_a_service)，下文将对此简称为 “**BaaS**”。
- Serverless 还可以指这种情况：应用的一部分服务端逻辑依然由开发者完成，但是和传统架构不同，它运行在一个无状态的计算容器中，由事件驱动、生命周期很短（甚至只有一次调用）、完全由第三方管理（感谢 ThoughtWorks 在他们最近的“[技术观察](https://www.thoughtworks.com/radar/techniques/serverless-architecture)”中对此所做的定义）。这种情况称为 [Functions as a service](https://twitter.com/marak/status/736357543598002176) / FaaS。[AWS Lambda](https://aws.amazon.com/lambda/) 是目前的热门 FaaS 实现之一，下文将对此简称为 “**FaaS**”。

【边栏注释：“Serverless” 术语的起源】

本文将主要聚焦于 FaaS，不仅仅因为它是 Serverless 中最新也最热门的领域，更重要的是它和我们传统技术架构思路的显著不同。

BaaS 和 FaaS 在运维层面有类似之处（都无需管理硬件资源），并且也经常配合使用。主流的云服务商都会提供一套“Serverless 全家桶”，囊括了 BaaS 和 FaaS 产品——例如 Amazon 的 Serverless 产品介绍，Google 的 Firebase 服务也紧密集成了 Google Cloud Functions。

对小公司而言这两个领域也有交叉，[Auth0](https://auth0.com/) 最初是一个 BaaS 产品，提供用户管理的各种服务，他们后来创建了配套的 FaaS 服务 [Webtask](https://webtask.io/)，并且将此概念进一步延伸推出了 [Extend](https://auth0.com/extend/)，支持其他 BaaS 和 SaaS 公司能够轻松地在现有产品中加入 FaaS 能力。

### 一些示例

#### 界面驱动的应用（UI-driven applications）

我们来设想一个传统的三层 C/S 架构，例如一个常见的电子商务应用（比如在线宠物商店），假设它服务端用 Java，客户端用 HTML/JavaScript：

![](http://martinfowler.com/articles/serverless/ps.svg)

在这个架构下客户端通常没什么功能，系统中的大部分逻辑——身份验证、页面导航、搜索、交易——都在服务端实现。

把它改造成 Serverless 架构的话会是这样：

![](http://martinfowler.com/articles/serverless/sps.svg)

这是张大幅简化的架构图，但还是有相当多变化之处：

1. 我们移除了最初应用中的身份验证逻辑，换用一个第三方的 BaaS 服务。
2. 另一个 BaaS 示例：我们允许客户端直接访问一部分数据库内容，这部分数据完全由第三方托管（如 AWS Dynamo），这里我们会用一些安全配置来管理客户端访问相应数据的权限。
3. 前面两点已经隐含了非常重要的第三点：先前服务器端的部分逻辑已经转移到了客户端，如保持用户 Session、理解应用的 UX 结构（做页面导航）、获取数据并渲染出用户界面等等。客户端实际上已经在逐步演变为[单页应用](https://en.wikipedia.org/wiki/Single-page_application)。
4. 还有一些任务需要保留在服务器上，比如繁重的计算任务或者需要访问大量数据的操作。这里以“搜索”为例，搜索功能可以从持续运行的服务端中拆分出来，以 FaaS 的方式实现，从 API 网关（后文做详细解释）接收请求返回响应。这个服务器端函数可以和客户端一样，从同一个数据库读取产品数据。
  我们原始的服务器端是用 Java 写的，而 AWS Lambda（假定我们用的这家 FaaS 平台）也支持 Java，那么原先的搜索代码略作修改就能实现这个搜索函数。
5. 最后我们还可以把“购买”功能改写为另一个 FaaS 函数，出于安全考虑它需要在服务器端，而非客户端实现。它同样经由 API 网关暴露给外部使用。

#### 消息驱动的应用（Message-driven applications）

再举一个后端数据处理服务的例子。假设你在做一个需要快速响应 UI 的用户中心应用，同时你又想捕捉记录所有的用户行为。设想一个在线广告系统，当用户点击了广告你需要立刻跳转到广告目标，同时你还需要记录这次点击以便向广告客户收费（这个例子并非虚构，我的一位前同事最近就在做这项重构）。

传统的架构会是这样：“广告服务器”同步响应用户的点击，同时发送一条消息给“点击处理应用”，异步地更新数据库（例如从客户的账户里扣款）。

![](http://martinfowler.com/articles/serverless/cp.svg)

在 Serverless 架构下会是这样：

![](http://martinfowler.com/articles/serverless/scp.svg)

这里两个架构的差异比我们上一个例子要小很多。我们把一个长期保持在内存中待命的任务替换为托管在第三方平台上以事件驱动的 FaaS 函数。注意这个第三方平台提供了消息代理和 FaaS 执行环境，这两个紧密相关的系统。

### 解构 “Function as a Service”

我们已经提到多次 FaaS 的概念，现在来挖掘下它究竟是什么含义。先来看看 Amazon 的 Lambda [产品简介](https://aws.amazon.com/cn/lambda/)：

_通过 AWS Lambda，无需配置或管理服务器**(1)**即可运行代码。您只需按消耗的计算时间付费 – 代码未运行时不产生费用。借助 Lambda，您几乎可以为任何类型的应用程序或后端服务**(2)**运行代码，而且全部无需管理。只需上传您的代码，Lambda 会处理运行**(3)**和扩展高可用性**(4)**代码所需的一切工作。您可以将您的代码设置为自动从其他 AWS 服务**(5)**触发，或者直接从任何 Web 或移动应用程序**(6)**调用。_

1. **本质上 FaaS 就是无需配置或管理你自己的服务器系统或者服务器应用即可运行后端代码**，其中第二项——服务器应用——是个关键因素，使其区别于现今其他一些流行的架构趋势如容器或者 PaaS（Platform as a Service）。
回顾前面点击处理的例子，FaaS 替换掉了点击处理服务器（可能跑在一台物理服务器或者容器中，但绝对是一个独立的应用程序），它不需要服务器，也没有一个应用程序在持续运行。

2. FaaS 不需要代码基于特定的库或框架，从语言或环境的层面来看 FaaS 就是一个普通的应用程序。例如 AWS Lambda 支持 JavaScript、Python 以及任意 JVM 语言（Java、Clojure、Scala 等），并且你的 FaaS 函数还可以调用任何一起部署的程序，也就是说实际上你可以用任何能编译为 Unix 程序的语言（稍后我们会讲到 Apex）。FaaS 也有一些不容忽视的局限，尤其是牵涉到状态和执行时长问题，这些我们稍后详谈。
再次回顾一下点击处理的例子——代码迁移到 FaaS 唯一需要修改的是 main 方法（启动）的部分，删掉即可，也许还会有一些上层消息处理的代码（实现消息监听界面），不过这很可能只是方法签名上的小改动。所有其他代码（比如那些访问数据库的）都可以原样用在 FaaS 中。

3. 既然我们没有服务器应用要执行，部署过程也和传统的方式大相径庭——把代码上传到 FaaS 平台，平台搞定所有其他事情。具体而言我们要做的就是上传新版的代码（zip 文件或者 jar 包）然后调用一个 API 来激活更新。

4. 横向扩展是完全自动化、弹性十足、由 FaaS 平台供应商管理的。如果你需要并行处理 100 个请求，不用做任何处理系统可以自然而然地支持。FaaS 的“运算容器”会在运行时按需启动执行函数，飞快地完成并结束。
回到我们的点击处理应用，假设某个好日子我们的客户点击广告的数量有平日的十倍之多，我们的点击处理应用能承载得住么？我们写的代码是否支持并行处理？支持的话，一个运行实例能够处理这么多点击量吗？如果环境允许多进程执行我们能自动支持或者手动配置支持吗？以 FaaS 实现你的代码需要一开始就以并行执行为默认前提，但除此之外就没有其他要求了，平台会完成所有的伸缩性需求。

5. FaaS 中的函数通常都由平台指定的一些事件触发。在 AWS 上有 S3（文件）更新、时间（定时任务）、消息总线（[Kinesis](https://aws.amazon.com/kinesis/)）消息等，你的函数需要指定监听某个事件源。在点击处理器的例子中我们有个假设是已经采用了支持 FaaS 订阅的消息代理，如果没有的话这部分也需要一些代码量。

6. 大部分的 FaaS 平台都支持 HTTP 请求触发函数执行，通常都是以某种 API 网关的形式实现（如 [AWS API Gateway](https://aws.amazon.com/api-gateway/)，[Webtask](https://webtask.io/)）。我们在宠物商店的例子中就以此来实现搜索和购买功能。

### 状态

当牵涉到本地（机器或者运行实例）状态时 FaaS 有个不能忽视的限制。简单点说就是你需要接受这么一个预设：函数调用中创建的所有中间状态或环境状态都不会影响之后的任何一次调用。这里的状态包括了内存数据和本地磁盘存储数据。从部署的角度换句话说就是 *FaaS 函数都是无状态的（Stateless）*。

这对于应用架构有重大的影响，无独有偶，“Twelve-Factor App” 的概念也有[一模一样的要求](http://12factor.net/processes)。

在此限制下的做法有多种，通常这个 FaaS 函数要么是天然无状态的——纯函数式地处理输入并且输出，要么使用数据库、跨应用缓存（如 Redis）或者网络文件系统（如 S3）来保存需要进一步处理的数据。

### 执行时长

FaaS 函数可以执行的时间通常都是受限的，目前 AWS Lambda 函数执行最长不能超过五分钟，否则会被强行终止。

这意味着某些需要长时间执行的任务需要调整实现方法才能用于 FaaS 平台，例如你可能需要把一个原先长时间执行的任务拆分成多个协作的 FaaS 函数来执行。

### 启动延迟

目前你的 FaaS 函数响应请求的时间会受到大量因素的影响，可能从 10 毫秒到 2 分钟不等。这听起来很糟糕，不过我们来看看具体的情况，以 AWS Lambda 为例。

如果你的函数是 JavaScript 或者 Python 的，并且代码量不是很大（千行以内），执行的消耗通常在 10 到 100 毫秒以内，大函数可能偶尔会稍高一些。

如果你的函数实现在 JVM 上，会偶尔碰到 10 秒以上的 JVM 启动时间，不过这只会在两种情况下发生：

- 你的函数调用触发比较稀少，两次调用间隔超过 10 分钟。
- 流量突发峰值，比如通常每秒处理 10 个请求的任务在 10 秒内飙升到每秒 100 个。

前一种情况可以用个 hack 来解决：每五分钟 ping 一次给函数保持热身。

这些问题严重么？这要看你的应用类型和流量特征。我先前的团队有一个 Java 的异步消息处理 Lambda 应用每天处理数亿条消息，他们就完全不担心启动延迟的问题。如果你要写的是一个低延时的交易程序，目前而言肯定不会考虑 FaaS 架构，无论你是用什么语言。

不论你是否认为你的应用会受此影响，都应该以生产环境级别的负载测试下实际性能情况。如果目前的情况还不能接受的话，可以几个月后再看看，因为这也是现在的 FaaS 平台供应商们主要集中精力在解决的问题。

### API 网关

![API Gateway](http://martinfowler.com/articles/serverless/ag.svg)

我们前面还碰到过一个 FaaS 的概念：“API 网关”。API 网关是一个配置了路由的 HTTP 服务器，每个路由对应一个 FaaS 函数，当 API 网关收到请求时它找到匹配请求的路由，调用相应的 FaaS 函数。通常 API 网关还会把请求参数转换成 FaaS 函数的调用参数。最后 API 网关把 FaaS 函数执行的结果返回给请求来源。

AWS 有自己的一套 API 网关，其他平台也大同小异。

除了纯粹的路由请求，API 网关还会负责身份认证、输入参数校验、响应代码映射等，你可能已经敏锐地意识到这是否合理，如果你有这个考虑的话，我们待会儿就谈。

另一个应用 API 网关加 FaaS 的场景是创建无服务器的 http 前端微服务，同时又具备了 FaaS 函数的伸缩性、管理便利等优势。

目前 API 网关的相关工具链还不成熟，尽管这是可行的但也要够大胆才能用。

### 工具链

前面关于工具链还不成熟的说法是指大体上 FaaS 无服务器架构平台的情况，也有例外，[Auth0 Webtask](https://webtask.io/) 就很重视改善开发者体验，[Tomasz Janczuk](https://twitter.com/tjanczuk) 在最近一届的 Serverless Conf 上做了精彩的展示。

无服务器应用的监控和调试还是有点棘手，我们会在本文未来的更新中进一步探讨这方面。

### 开源

无服务器 FaaS 的一个主要好处就是只需要近乎透明的运行时启动调度，所以这个领域不像 Docker 或者容器领域那么依赖开源实现。未来肯定会有一些流行的 FaaS / API 网关平台实现可以跑在私有服务器或者开发者工作站上，[IBM 的 OpenWhisk](http://martinfowler.com/articles/serverless.html) 就是一个这样的实现，不知道它是否能成为流行选择，接下来的时间里肯定会有更多竞争者出现。

除了运行时的平台实现，还是有不少开源工具用以辅助开发和部署的，例如 [Serverless Framework](https://github.com/serverless/serverless) 在 API 网关 + Lambda 的易用性上就比它的原创者 AWS 要好很多，这是一个 JS 为主的项目，如果你在写一个 JS 网关应用一定要去了解下。

再如 [Apex](https://github.com/apex/apex)——“轻松创建、部署及管理 AWS Lambda 函数”。Apex 有意思的一点是它允许你用 AWS 平台并不直接支持的语言来实现 Lambda 函数，比如 Go。

## 什么不是 Serverless

在前文中我定义了 “Serverless” 是两个概念的组合：“Backend as a Service” 和 “Function as a Service”，并且对后者的特性做了详细解释。

在我们开始探讨它的好处和弊端之前，我想再花点儿时间在它的定义上，或者说：区分开那些容易和 Serverless 混淆的概念。我看到一些人（包括我自己最近）对此都有困惑，我想值得对此做个澄清。

### 对比 PaaS

既然 Serverless FaaS 这么像 12-Factor 应用，那不就是另一种形式的 Platform as a Service 么？就像 Heroku？对此借用 Adrian Cockcroft 一句非常简明的话：

> 如果你的 PaaS 能在 20ms 内启动一个只运行半秒钟的实例，它就叫 Serverless。
> — [Adrian Cockcroft](https://twitter.com/adrianco/status/736553530689998848)

换句话说，大部分 PaaS 应用不会为了每个请求都启动并结束整个应用，而 FaaS 就是这么做的。

好吧，然而假设我是个娴熟的 12-Factor 应用开发者，写代码的方式还是没有区别对么？没错，但是你如何**运维**是有很大不同的。鉴于我们都是 DevOps 工程师我们会在开发阶段就充分考虑运维，对吧？

FaaS 和 PaaS 在运维方面的关键区别是**伸缩性**（Scaling）。对于大多数 PaaS 平台而言你需要考虑如何伸缩，例如在 Heroku 上你要用到多少 Dyno 实例？对于 FaaS 应用这一步骤是完全透明的。即便你将 PaaS 配置为自动伸缩，也无法精细到单个请求级别，除非你有一个非常明确稳定的流量曲线可以针对性地配置。所以 FaaS 应用在成本方面要高效得多。

既然如此，何必还用 PaaS？有很多原因，最主要的因素应该是工具链成熟度。另外像[Cloud Foundry](https://en.wikipedia.org/wiki/Cloud_Foundry) 能够给混合云和私有云的开发提供一致体验，在写就本文的时候 FaaS 还没有这么成熟的平台。

### 对比容器

使用 Serverless FaaS 的好处之一是避免在操作系统层面管理应用程序进程。有一些 PaaS 平台如 Heroku 也提供了这样的特性；另一种对进程的抽象是容器，这类技术的代表是 [Docker](https://www.docker.com/)。容器托管系统（Mesos、Kubernetes 等）把应用从系统级开发中抽象出来，这种做法日渐流行，甚至在此之上云服务商的容器平台（如 [Amazon ECS](https://aws.amazon.com/ecs/)、[EKS](https://aws.amazon.com/eks/)、[Google Cloud Engine](https://cloud.google.com/container-engine)）也像 Serverless FaaS 一样允许团队从管理主机中完全解放出来。在这股容器大潮中，FaaS 是否还有优势？

概念上来说前面对 PaaS 的论断仍旧适用于容器。Serverless FaaS 的伸缩性是**完全自动化、透明、良好组织**的，并且自动进行资源监控和分配；而容器平台仍旧需要你对容量和集群进行管理。

另外我还得说容器技术也处在不够成熟和稳定的阶段，尽管它越来越接近了。当然这不是说 Serverless 就成熟了，但你终究需要在两个都属前沿的技术方向中做出选择。

还有个值得一提的是不少容器平台支持了自动伸缩的容器集群，Kubernetes 有内建的 [Horizontal Pod Autoscaling](http://kubernetes.io/docs/user-guide/horizontal-pod-autoscaling/) 功能，[AWS Fargate](https://aws.amazon.com/fargate/) 则承诺会有“Serverless 容器”。

总的来说 Serverless FaaS 和托管容器在管理和伸缩性方面的差别已经不大，在它们之间的取舍更多看风格取向和应用的类型。例如事件驱动的应用组件更适合用 FaaS 实现，而同步请求驱动的应用组件更适合用容器实现。我预计很快就会有不少团队和应用同时采用这两种架构模式，期待看它们会擦出怎样的火花。

### 对比 NoOps

Serverless 并非“零运维”——尽管它可能是“无系统管理员”，也要看你在这个 Serverless 的兔子洞里走多深。

“运维”的意义远不止系统管理，它还包括并不限于监控、部署、安全、网络、支持、生产环境调试以及系统伸缩。这些事务同样存在于 Serverless 应用中，你仍旧需要相应的方法处理它们。某些情况下 Serverless 的运维会更难一些，毕竟它还是个崭新的技术。

系统管理的工作仍然要做，你只是把它外包给了 Serverless 环境。这既不能说坏也不能说好——我们外包了大量的内容，是好是坏要看具体情况。不论怎样，某些时候这层抽象也会发生问题，就会需要一个来自某个地方的人类系统管理员来支持你的工作了。

[Charity Majors](https://twitter.com/mipsytipsy) 在第一届 Serverless 大会上就这个主题做了个[非常不错的演讲](https://www.youtube.com/watch?v=wgT5f0eBhD8)，也可以看看她相关的两篇文章：[WTF is operations?](https://charity.wtf/2016/05/31/wtf-is-operations-serverless/) 和 [Operational Best Practices](https://charity.wtf/2016/05/31/operational-best-practices-serverless/)）。

### 对比存储过程即服务

还有一种说法把 Serverless FaaS 看做“存储过程即服务（Stored Procedures as a Service）”，我想原因是很多 FaaS 函数示范都用数据库访问作为例子。如果这就是它的主要用途，我想这个名字也不坏，但终究这只是 FaaS 的一种用例而已，这样去考虑 FaaS 局限了它的能力。

> 我好奇 Serverless 会不会最终变成类似存储过程那样的东西，开始是个好主意，然后迅速演变成大规模技术债务。
> — [Camille Fournier](https://twitter.com/skamille/status/719583067275403265)

但我们仍然值得考虑 FaaS 是否会导致跟存储过程类似的问题，包括 Camille 提到的技术债。有很多存储过程给我们的教训可以放在 FaaS 场景下重新审视，存储过程的问题在于：

1. 通常依赖于服务商指定的语言，或者至少是指定的语言框架/扩展
2. 因为必须在数据库环境中执行所以很难测试
3. 难以进行版本控制，或者作为应用包进行管理

尽管不是所有存储过程的实现都有这些问题，但它们都是常会碰到的。我们看看是否适用于 FaaS：

第一条就目前看来显然不是 FaaS 的烦恼，直接排除。

第二条，因为 FaaS 函数都是纯粹的代码，所以应该和其他任何代码一样容易测试。整合测试是另一个问题，我们稍后展开细说。

第三条，既然 FaaS 函数都是纯粹的代码，版本控制自然不成问题；最近大家开始关心的应用打包，相关工具链也在日趋成熟，比如 Amazon 的 [Serverless Application Model](https://docs.aws.amazon.com/lambda/latest/dg/serverless_app.html)（SAM）和前面提到的其他 Serverless 框架都提供了类似的功能。2018 年初 Amazon 还开放了 [Serverless Application Repository](https://aws.amazon.com/cn/serverless/serverlessrepo/)（SAR）服务，方便组织分发应用程序和组件，也是基于 AWS Serverless 服务构建的。关于 SAR 可以看看我的另一篇文章：[Examining the AWS Serverless Application Repository](https://blog.symphonia.io/examining-the-aws-serverless-application-repository-9ef316e2fd4)。

**未完待续**
