## Spring OAuth2 开发指南（一）：体系架构和开发概览

[TOC]

### 一、开篇

《Spring OAuth2 开发指南》是系列文章，详细介绍基于 Spring 生态（包括 Spring Cloud） OAuth2 的实战开发。本系列将由五篇文章组成：

1. （一）体系架构和开发概览：是系列文章的开篇，主要对 OAuth2 的体系架构和主要流程进行梳理剖析，并对当前 Spring OAuth2 开发做一个概括性、全局性介绍；
2. （二）OAuth2 密码模式开发实例
3. （三）OAuth2 客户端模式开发实例
4. （四）OAuth2 授权码模式开发实例
4. （五）OAuth2 微服务场景实例开发：以密码模式为例，介绍在微服务场景下使用 OAuth2 以及整合 JWT 的关键技术方法，主要围绕客户端、微服务网关、认证授权服务进行，不涉及微服务的其他模块等。

本系列第一篇主要聚焦 OAuth2 体系的概念介绍，其余各篇偏向于实战，主要实现 OAuth2 的三种授权模式：密码模式、客户端模式和授权码模式，包括展示授权服务器、资源服务器、客户端等几种角色的交互，以及 JWT 的整合。并且每个实例都提供两个代码版本：一个是基于旧的 Spring Security OAuth2 组件；一个是基于新的 Spring Authorization Server 组件。


> 需要注意的是 password 模式由于 OAuth2.1 不推荐使用所以只能提供旧的组件代码版本，具体请参见 https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-02

如果对认证和授权平台的相关理论感兴趣，可以移步本人早前的两篇文章：

1. 平台级 SaaS 架构的基础：统一身份管理系统， https://www.jianshu.com/p/990d8acfdb69
2. 微服务架构下的统一身份认证和授权， https://www.jianshu.com/p/2571f6a4e192

### 二、OAuth2 体系结构

OAuth 授权体系设计之初主要是为了解决第三方应用登录和授权的问题，但由于其严格规范的流程定义，广泛的授权通用性，且与具体技术平台无关等诸多优点，逐渐发展成为认证和授权领域的主流技术规范。但其实 OAuth2 规范归纳起来并不复杂，就四种主要的授权模式和五种角色。

四种授权模式，分别是：

1. 授权码模式（authorization code）
2. 简化模式（implicit）
3. 密码模式（resource owner password credentials）
4. 客户端模式（client credentials）

五种主要的角色，分别是：

1. 用户代理/浏览器 User Agent
2. 客户端 Client
3. 资源所有者 Resource Owner
4. 资源服务器 Resource Server （受保护资源）
5. 授权服务器 Authorization Server

建议熟练掌握以上概念，理清其中的关系，有助于对整个体系有一个全局性的了解。倘若一上来就撸代码，那造出来的系统必然不健康。须知身份管理体系是任何平台的重要基础设施，一旦建造的不牢固，必然会种下隐患，好似大厦地基埋下的一颗延迟炸弹。

回到主题，四种模式都有其特定的使用场景，但是在落地过程中，也可以根据实际情况自行取舍。为了方便接下来的介绍，我们假设两套演示案例：基于图像的物品分类系统（IBCS，Image-Based Classification System）、相册预览系统（PAPS，Photo Album Preview System）。

> 为什么假设两套？主要是为了分析对比各种授权模式的适用场景。

#### 场景假设 A：基于图像的物品分类系统（IBCS，Image-Based Classification System）

假设团队正在开发一套“基于图像的物品分类系统（Image-Based Classification System）”，该系统采用 RESTful API 对外交互，主要功能是允许用户通过 H5 应用或 APP 上传图片，系统分析后返回分类结果。为了表演 OAuth2 体系认证、授权、鉴权和权限控制的全流程，我们构建一个最简化的 IBCS 演示项目：

服务名 | 类别 | 描述 | 技术选型
- | - | - | -
ibc-service | 内部服务 | 资源服务器角色，图像分类服务 | Spring Boot 开发的 RESTful 服务
idp | 内部服务 | 授权服务器角色，具体指负责认证、授权和鉴权 | Spring Boot 开发
demo-h5 | 外部应用 | demo 应用的前端 | Antd-Pro 开发
demo-service | 外部应用 | demo 应用的后端 | Spring Boot 开发

这里设计了一个 demo 应用，对于 IBCS 来说，demo 应用就是一个外部应用，是一个消费者，包括前端 demo-h5 和后端 demo-service。但是从整个演示项目来说，ibc-service、idp、demo-h5 和 demo-service 都是自家应用，是相互信任的。

> 值得注意的是，在现实世界里，应用有两种类型：一是有 Server 端的应用；另一种是无 Server 端的应用。前者比较好理解，它可能是前后端分离的 MVC 单体应用，也有可能是 REST 型的前后端应用，这些都是常见的开发结构。除此之外，还有一些没有或者不需要后端的应用，比如 SinglePage H5，由 Vue 或 React 开发的简单 H5，整个应用仅由 JavaScript 代码组成，前端即应用的全部。这种类型的应用，有一个最大的安全问题，即 client_secret 如何安全存储，在无 Server 场景中无论是经典的授权码模式还是密码模式，都无法有效解决这个问题，因为一个全部由 JavaScript 写成的程序，本质上是完全透明的，client_secret 存在泄露的风险。client_secret 将导致客户端被伪造，即攻击者在拿到 client_id 和 client_secret 后伪造一个 App 发起虚假请求。当然许多平台会在 idp 端增加 redirect_uri 的绑定或 IP/域名白名单机制，从而有效降低伪造/劫持等安全风险。

> 有人说 OAuth 2.0 规范提出的 PKCE（Proof Key for Code Exchange by OAuth Public Clients）协议可以解决这个问题，这是一个错误的观点，PKCE 在技术原理上并不能解决 client_secret 泄露的风险，所以对安全性要求高的情况下最好不要采用无 Server 端方式。不过 PKCE 作为一种增强协议可以搭配 OAuth2 组合使用以提高整体安全性。其主要流程为：由暴露在外的“前端”自身生成一串随机的验证码，再根据验证码生成挑战码，然后用挑战码向授权服务器请求授权码，授权服务器保存挑战码后返回授权码给“前端”，“前端”拿到授权码后用第一步生成的随机验证码+授权码向授权服务器申请令牌，授权服务器拿到验证码+授权码后根据此验证码生成挑战码，如果生成的挑战码跟上一步保存的挑战码一致则颁发令牌。有兴趣的同学可以自行查阅相关资料。

#### 场景假设 B：相册预览系统（PAPS，Photo Album Preview System）

PAPS 是一个社交平台的子系统，与 IBCS 类似，采用 RESTful API 对外交互，主要功能是允许用户预览自己的相册，我们构建一个最简化的 PAPS 演示项目：

服务名 | 类别 | 描述 | 技术选型
- | - | - | -
photo-service | 内部服务 | 资源服务器角色，相册预览服务 | Spring Boot 开发的 RESTful 服务
idp | 内部服务 | 授权服务器角色，具体指负责认证、授权和鉴权 | Spring Boot 开发
demo-h5 | 外部应用 | demo 应用的前端 | Antd-Pro 开发
demo-service | 外部应用 | demo 应用的后端 | Spring Boot 开发

两个演示案例唯一的不同就在于核心服务不同，一个是图像分类服务，一个是相册预览服务。看起来似乎没什么差别，实际上他们的业务模型还很不同的，接下来会分析这一点。

#### 授权码模式 vs 密码模式 vs 客户端模式

大家思考一下，授权码模式、密码模式和客户端模式，在不同的业务场景下该如何选择？

无非有两种做法：

1. 不管三七二十几，整个项目只使用一种授权模式，简单明了；
2. 仔细分析各种业务场景，给予合适的模式选型。

第一种做法，简单粗暴，看起来好像不可取，但也不能一棍子打死，具体还是看项目的业务模型、设计目标和具体要求。但仅从科学严谨或者技术研究角度来说，肯定是第二种方式更加可取。接下来，我们就以第二种做法展开，详细分析下模式选型的问题。

##### 授权码模式和密码模式

我们先来看授权码模式和密码模式之间的比较，大家知道，授权码模式是 OAuth2 体系安全性最高的模式，密码模式与其相比，主要差别是少了一层用户确认授权的动作，缺乏这一动作就导致在授权阶段，用户需要把用户名密码告知客户端，造成潜在的密码泄露风险。我们看一下对比：

比较项 | 授权码模式 | 密码模式
- | - | - 
适用场景 | 不可信/第三方认证和授权、可信/内部服务认证和授权 | 可信/内部服务认证和授权
开发难度 | 较为复杂 | 相对简单
安全性 | 最高 | 只要不运用于不可信/第三方场景则同样安全

在这里，可信/内部服务场景的定义是相对的概念，指纳入同一套 OAuth2 体系的应用和服务，且这些应用和服务是由相同的或者相互信任的团队开发，也可以称作第一方应用。打个比方，微服务架构下的 B2C 商城系统，基本组成有前端 H5、无线端 APP、API 网关、认证授权服务、订单服务、商品服务等，由于上述所有组成部分都同属于一套 OAuth2 体系，且都由相同团队开发，那么他们全都归属于可信/内部服务场景。

那么为什么又说是相对概念？我们把视角聚焦到整个商城系统，毫无疑问，网关属于安全边界，网关以内的认证授权服务、订单服务、商品服务属于内部服务，而前端 H5、无线端 APP 则属于外部应用，如果这些外部应用是其他团队开发，我们也可以定义它们为第三方应用。这样就以网关为边界，划分出了内部服务和外部服务，这就是所说的相对概念。

那么，电商系统的前端 H5、无线端 APP 在认证和授权阶段，采用授权码模式和采用密码模式，有什么差别吗？授权码模式有一层用户确认授权的动作，从而避免泄露用户名和密码给第三方应用，除此之外，两者之间几乎提供了相同的安全流程。这里所指的第三方应用不正是前端 H5 和无线端 APP 吗？都是自家开发的应用，自然是可信的，因此无须担忧泄露不泄露的风险。

以此来看，在上述条件的限定下，两种模式的安全性打平了。遵循“简化原则”，采用密码模式即可，当然喜欢挑战复杂的同学选择授权码模式也是可以的。

但是也不是说授权码模式就可以被密码模式取代了，授权码模式主要的应用场景，是在第三方/不可信应用的登录和授权，主要解决在不泄露用户密码的情况下如何安全授权某个应用向另一个应用提供用户资源的问题，举例来说，某第三方应用（客户端）需要获得用户（资源所有者）在另一个不可信应用（资源服务器）上的该用户的用户数据（资源）的场景就特别适合采用授权码模式。

综上，选择授权码模式还是密码模式，具体要根据业务场景来确定，其中的关键决策点是应用或服务之间是否互相信任。

##### 客户端模式

> PAPS 相册预览系统演示案例应该采用何种授权模式？

回答这个问题之前，大家先思考一个问题：在 PAPS 中，资源所有者所指代的对象是什么？

首先要明确资源是什么，其次该资源是受保护的，最后资源归谁所有，谁就是资源所有者。在 PAPS 中，很明显受保护的资源是用户的相册，资源所有者自然是用户本人。

明确资源所有者的含义后，再根据前文的分析，毫无疑问：如果 PAPS 的 demo 应用是第三方的不可信应用，则应该采用授权码模式；如果是第一方可信应用，则可以采用密码模式，当然不怕麻烦也可以用授权码模式。

> 接下来，我们来分析 IBCS 图片分类系统演示案例该用何种模式。

同样地，回答这个问题之前，大家再思考一下：在 IBCS 中，资源所有者所指代的对象是什么？

首先资源所有者所指代的对象不是一成不变的。在 IBCS 演示案例中，demo 应用向 ibc-service 传送一张图片，并希望返回分类结果，那么这里面的受保护资源具体是什么呢？不似 PAPS 相册预览服务有实体资源概念，其受保护资源是用户的相册，而 IBCS 难以抽象出一个实体的资源来。

可以这么理解，IBCS 提供的核心能力是图片分类算法，这就是它的受保护资源，图片分类算法的所有权人显然是持有此算法的实体组织或个人，因此资源所有者是该实体组织或个人。那么矛盾点来了，以密码模式为例，按照 OAuth2 的设计，资源所有者向客户端提供用户名和密码，客户端将 client_id 和 client_secret 连同该用户名和密码，向授权服务器申请令牌，此处的资源所有者是 IBCS 的普通用户，但是按照刚才所述，资源所有者应该是持有图片分类算法的实体组织或个人，这不就矛盾了吗？

因此，从资源所有者的角度来分析，IBCS 演示案例不适合采用授权码模式，也不适合采用密码模式，客户端模式才是最佳选择。IBCS 并没有用户的资源，授权码模式和密码模式都是需要用户授权才能跑通的，而 IBCS 提供的资源或服务并不属于用户，所以法理上来说不需要用户的授权，IBCS 提供的分析服务是跟用户无关的。

以上就是选择何种授权模式的一种分析模型，从资源所有者的维度切入。总结一下：

案例 | 场景 | 适合模式
- | - | -
IBCS 图像分类系统案例 | 可信/内部服务 | 客户端模式
IBCS 图像分类系统案例 | 不可信/外部应用 | 客户端模式
PAPS 相册预览系统案例 | 可信/内部服务 | 密码模式
PAPS 相册预览系统案例 | 不可信/外部应用 | 授权码模式

> 其实资源所有者的具体指代对于实际开发来说，并不是一个重要的方面，关键是开发人员要有自己的理解，能够灵活巧用。说句题外话，虽然 OAuth2 是当今领域内的权威，但我们也不应盲目地无条件相信权威，技术发展一定是有漏洞和不完美之处的。

#### 密码模式的典型架构层次和主要流程

我们以 PAPS 相册预览系统为例，介绍密码模式的架构层次和主要流程。

![OAuth2 密码模式典型架构层次](https://mtide.net/assets/images/post/20210809/1.png)

如图所示，是密码模式的最精简架构层次，在实际开发中可以此作为基础进行扩展。密码模式涉及到五种主要角色，另外还有一个用户代理/浏览器角色：

1. 用户代理/浏览器：一般单体应用都是前后端分离的 MVC 结构，从这个角度看，这里具体可以将用户代理/浏览器理解为前端的 H5 应用或者无线端的 APP，换句话说 H5/APP 承载用户的交互操作而成为用户代理。具体到 PAPS 演示案例就是 demo-h5；
2. 客户端：具体指单体应用的后端服务，具体到 PAPS 演示案例就是 demo-service；
3. 授权服务器：具体指负责认证、授权和鉴权的 IDP（Identify Provider），也是 OAuth2 体系的核心服务，也可以简单叫做 auth-service。具体到 PAPS 演示案例就是 idp；
4. 受保护资源：即资源服务器，一般是内部服务，比如产品服务、订单服务等。具体到 PAPS 演示案例就是 photo-service；
5. 资源所有者：顾名思义，即受保护资源的所有者，一般具体指代用户自己。具体到 PAPS 演示案例就是用户。

整个流程分为两个阶段：

* 第一阶段：认证授权阶段

1. 用户代理（demo-h5）将用户输入的用户名和密码，发送给客户端（demo-service）；
2. 客户端（demo-service）将用户输入的用户名和密码，连同 client_id + client_secret (由 idp 分配）一起发送到 idp 以请求令牌，如果 idp 约定了 scope 则还需要带上 scope 参数；
3. idp 首先验证 client_id + client_secret  的合法性，再检查 scope 是否无误，最后验证用户名和密码是否正确，正确则生成 token。这一步也叫“认证”；
4. idp 返回认证结果给客户端，认证通过返回 token，认证失败返回 401。如果认证成功则此步骤也叫“授权”；
5. 客户端收到 token 后进行暂存，并创建对应的 session；
6. 客户端颁发 cookie 给用户代理/浏览器。

至此，认证授权阶段完成。其中步骤 5-6 也有其他会话方案，比如 REST 型应用可能会将 token 存储在浏览器端，但 session/cookie 方案无疑是最稳妥的选择。

> 在一般的 Web 应用中，可以将此阶段看作是一次用户登录过程。

* 第二阶段：授权后请求资源阶段

7. 用户通过用户代理（demo-h5）访问“我的相册”页面，用户代理携带 cookie 向客户端（demo—service）发起请求；
8. 客户端通过 session 找到对应的 token，携带此 token 向资源服务器（photo-service）发起请求；
9. 资源服务器（photo-service）向 idp 请求验证 token 有效性；
10. idp 校验 token 有效性，再根据 scope 判断客户端（demo-service）是否有权限调用此 API，最后返回校验结果给资源服务器。这一步也叫“鉴权”；
11. 资源服务器根据 idp 检验结果（true/false 或其他等效手段）决定是否返回用户相册数据给客户端。如果 token 校验失败则返回 401 给客户端，如果 scope 检查不通过则返回 403。这一步也叫“权限控制”。

至此，授权后请求资源阶段完成。

> 事实上 scope 参数不是核心的内容，实际工作中为了简化开发步骤甚至可以忽略它。scope 参数是用来约束客户端的权限的，跟用户权限（authorities）是不同的。比如可以在 idp 中利用 scope 参数约束某客户端只能发起读（GET）型请求，或只能调用指定的几个 API 等，具体业务逻辑自行编写。

#### 密码模式的微服务架构层次和主要流程

我们仍以 PAPS 相册预览系统为例，介绍密码模式在微服务场景下的架构层次和主要流程。

![OAuth2 密码模式微服务架构层次](https://mtide.net/assets/images/post/20210809/2.png)

微服务场景下，增加了一个网关，网关实际上是一个反向代理，将用户的请求转发到内部服务器。类似地，微服务场景下也分为两个阶段，而且第一阶段没什么变化，主要不同在于第二阶段：

7. 用户通过用户代理（demo-h5）访问“我的相册”页面，用户代理携带 cookie 向客户端（demo—service）发起请求；
8. 客户端通过 session 找到对应的 token，携带此 token 向网关发起对资源服务器（photo-service）的请求；
9. 网关截取 token 连同本次请求的细节，一并向 idp 请求校验；
10. idp 校验 token 有效性，再根据 scope 和请求细节判断客户端（demo-service）是否有权限调用此 API，最后返回校验结果给网关。如果校验全部通过，idp 生成 JWT 并返回给网关；如果 token 校验失败返回 401；如果 scope 检查不通过则返回 403；
11. 如果校验通过，网关将得到 JWT，携带此 JWT 转发请求到资源服务器；
12. 资源服务器解析 JWT 得到用户信息，查询用户相册数据后返回给网关；
13. 网关将用户相册数据返回给客户端。

此流程有两项重大变化：一是加入网关使得整个流程复杂了一些；二是网关以内使用 JWT 作为令牌。关于这两点的解读，本文不再赘述，感兴趣的同学可以参阅本人早前的文章《微服务架构下的统一身份认证和授权》。

值得注意的是，步骤 9-11，还有另一种处理方法，即将 scope 客户端权限检查放到网关进行：

9. 网关截取 token 后向 idp 请求校验；
10. idp 校验 token 有效性，通过校验则根据 token 查询用户信息和 scope，生成 JWT 返回给网关；如果不通过则返回 401；
11. 网关得到 JWT，解析后根据 scope 判断客户端是否有权限调用此 API，如有则携带 JWT 转发请求到资源服务器，否则返回 403 给客户端。

> 客户端权限检查放到网关，则网关要维护 scope 和客户端权限的逻辑。

#### 客户端模式的微服务架构层次和主要流程

我们以 IBCS 图片分类系统为例，介绍客户端模式在微服务场景下的架构层次和主要流程。

![OAuth2 客户端模式微服务架构层次](https://mtide.net/assets/images/post/20210809/3.png)

可以看到，客户端模式流程比较简单，这里就不再叙述具体流程了，不过请注意第 2 步：

2. 客户端用向 idp 申请令牌之前，应该先检查是否缓存了有效令牌，有的话直接跳到第 6 步发起服务访问请求。

#### 授权码模式的微服务架构层次和主要流程

我们仍以 PAPS 相册预览系统为例，介绍授权码模式在微服务场景下的架构层次和主要流程。

![OAuth2 授权码模式微服务架构层次](https://mtide.net/assets/images/post/20210809/4.png)

整个流程分为两个阶段：

* 第一阶段：认证授权阶段

1. 用户在用户代理（demo-h5）处点击登录按钮，或请求授权登录按钮，此操作将访问客户端的某个 URI；
2. 客户端（demo-service）将用户导向 idp 提供的认证授权页面，并在页面 ULR 参数中携带 client_id，response_type=code，redirect_uri（可选），scope（可选），state（可选）；
3. 用户通过用户代理（demo-h5），在 idp 的认证授权页面选择是否给予授权，如用户未登录，则需要先登录后再操作；
4. 用户给予授权，idp 将用户导向 redirect_uri 指定的页面，并附加授权码（code）；如果未指定 redirect_uri，则导向发起该请求时的 URI，同时附加授权码（code）；
5. 客户端收到授权码（code），向 idp 发起令牌申请，同时附上 client_id（必填） + client_secret（必填） + state（如有） + scope（如有）。注意这一步是客户端在后台发起的，用户层面无法感知；
6. idp 收到请求后，先核对 client_id + client_secret + scope（如有）是否无误，然后校验授权码（code），全部正确后颁发 token 给客户端。

* 第二阶段：授权后请求资源阶段

该阶段的流程，与密码模式的微服务场景流程一致，此处不在赘述。

##### 对于 REST 型 demo 应用

如果 demo 应用是一个 REST 型应用，则在第 1、2 步骤中，还可以这么处理：

1. 用户在用户代理（demo-h5）处点击登录按钮或请求授权登录按钮后，通知客户端（demo-service），客户端收到通知后返回重定向的指示，以及 scope（可选），state（可选）等；
2. demo-h5 收到响应后，直接将用户导向 idp 提供的认证授权页面，并在页面 URL 参数中携带客户端返回的参数（除了 state 参数外，其他参数可以在 demo-h5 中写死）client_id，response_type=code，redirect_uri（必选），scope（可选），state（可选），其中 redirect_uri 建议是必选的，而且必须是客户端提供的 URI（回调地址）。

还可以这么处理：

1. 用户在用户代理（demo-h5）处点击登录按钮或请求授权登录按钮后，直接将用户导向 idp 提供的认证授权页面，并在页面 URL 参数中携带 client_id，response_type=code，redirect_uri（必选），scope（可选），但是不需要携带 state 参数，因为客户端不知道此参数的存在，其中 redirect_uri 建议是必选的，而且必须是客户端提供的 URI（回调地址）。

至此，授权码模式的认证授权全流程完毕。

> 讨论：客户端第一次将用户导向 idp 提供的认证授权页面时，idp 是否需要验证客户端的身份呢？或者说需不需要提供 client_secret 呢？

#### 微服务网关的负载问题

在微服务场景下，大量的请求和响应都经过网关转发，我们设想一下，如果类似 PAPS 相册系统那样，返回的是图片数据，而且不采用 CDN 分发网络或文件存储服务等技术，那么图片流通过网关再返回给用户，网关的负载是不是将会非常庞大呢？

当然，网关本身可以做负载均衡，可以引入缓存，数据流可以做 CDN 处理等，这些都是非常好的高性能方案，除此之外，有没有其他办法呢？

这里引入一个网络技术领域的概念——负载均衡有三种模式：反向代理、透传模式和三角模式。这里简单介绍一下三角模式：

假设一种网络结构，包括 Web 服务器、PC 客户端和负载均衡器。PC 通过负载均衡器发起对 Web 服务器的访问。在三角模式下的访问流程如下：

1. PC 向负载均衡器发起对 Web 服务器的访问请求；
2. 负载均衡器将 PC 的请求转发给 Web 服务器；
3. Web 服务器收到请求后，直接将响应数据发送给 PC 端。

PC、Web 服务器和负载均衡器三者呈三角形状，因此叫做三角模式。这个模式的优点就是负载均衡器只负责转发请求，而响应数据包则不需要接收和转发，从而大副降低负载均衡器自身的数据流通压力。

按照这个思路，我们可以将三角模式引入网关，从而大副降低网关的负载压力。

#### 令牌的复用问题

我们设想一个场景，团队研发的平台同时包含了 IBCS 图片分类服务和 PAPS 相册预览服务，那么用户在登录平台（用密码模式认证授权）后，先访问“我的相册”，然后从中选择一张照片发起物品识别的请求。

根据文章的模式选型分析，IBCS 服务应采用客户端模式，PAPS 服务应采用密码模式，那么，客户端是否应该申请两套令牌？

回答这个问题，我们还是要从具体场景切入分析：

1. 如果用户需要登录平台后才能使用 IBCS 和 PAPS 的服务，那么，只需要用密码模式一种令牌即可；
2. 如果 PAPS 功能无需登录，游客也能使用，那么密码模式和客户端模式要分开处理。

授权码模式是最严格的，密码模式次之，客户端模式最差，因此一般情况下，授权码模式的令牌可以给其他模式使用，密码模式令牌可以给客户端模式使用，客户端模式只能自己使用。

### 三、Spring 家族 OAuth2 相关组件概览

好了，从本节开始我们脱离枯燥的理论环节，进入一样枯燥的实战开发频道。

目前构建 OAuth2 授权系统有三种方式：一是基于主流的开源组件构建；二是接入第三方授权服务（如 Google、GitHub OAuth2）；三是根据 OAuth2 的标准规范自行开发相关授权组件。

常用的开源组件有 RedHat Keycloak、Spring Security、Spring Security OAuth2，以及刚起步的 Spring Authorization Server 等。值得一提的是 RedHat Keycloak，它是一款开源、成熟的 IAM 解决方案，功能强大且可私有化部署。

在 Spring 的开发生态里，建议采用 Spring Security 方向的开源组件方案，可扩展性好，定制性强，用户广泛且社区活跃。

> Spring Security OAuth2 目前已停止更新，官方不推荐继续使用。相关功能已经迁移到 Spring Security，但授权服务器（Authorization Server）功能并未包含。Authorization Server 功能将由 Spring Security 团队主导开发的 Spring Authorization Server 开源组件提供，详细信息请查看官方通告：https://spring.io/blog/2020/04/15/announcing-the-spring-authorization-server

Spring Authorization Server 项目目前还在迭代中，该项目的开发计划托管在 ZenHub 上，感兴趣的同学可以自行了解： https://app.zenhub.com/workspaces/authorization-server-5e8f3182b5e8f5841bfc4902/board?repos=248032165

#### 一）基于 Spring Framework

* spring-security

> 核心组件，当前几乎所有 Spring OAuth2 开源技术方案都依赖于它。核心模块： spring-security-core。

```
<dependencies>
    <!-- ... other dependency elements ... -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-web</artifactId>
        <version>5.5.1</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-config</artifactId>
        <version>5.5.1</version>
    </dependency>
</dependencies>
```

* [已停止更新] spring-security-oauth2

> 依赖于 spring-security。该组件现已合并到 spring-security 中，官方已不建议使用。在此之前是 Spring Security 团队官方维护的唯一且被广泛使用的组件。

```
<!-- https://mvnrepository.com/artifact/org.springframework.security.oauth/spring-security-oauth2 -->
<dependency>
    <groupId>org.springframework.security.oauth</groupId>
    <artifactId>spring-security-oauth2</artifactId>
    <version>2.5.1.RELEASE</version>
</dependency>
```

* spring-security-oauth2-authorization-server

> 依赖于 spring-security。在 Spring Security 官方宣布停止 spring-security-oauth2 组件更新后，推出的授权服务器 Authorization Server 的替代组件。 spring-security 整合 spring-security-oauth2 后，并不包括 Authorization Server 功能，因此如果需要开发此功能，则要搭配 spring-security-oauth2-authorization-server 一起使用。

```
<!-- https://mvnrepository.com/artifact/org.springframework.security.experimental/spring-security-oauth2-authorization-server -->
<dependency>
    <groupId>org.springframework.security.experimental</groupId>
    <artifactId>spring-security-oauth2-authorization-server</artifactId>
    <version>0.1.2</version>
</dependency>
```

#### 二）基于 Spring Boot

* spring-boot-starter-security

> 依赖于 spring-security。相当于 spring-boot + spring-security ，默认包含 SecurityAutoConfiguration.class ，因此会执行一些自动化配置，可以简化开发步骤。如果想关闭自动配置，可以修改 Spring Boot 启动注解为 @SpringBootApplication(exclude = { SecurityAutoConfiguration.class })

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

#### 三）基于 Spring Cloud

* spring-cloud-starter-oauth2

> 依赖于 spring-boot-starter-security + spring-security-oauth2，并额外提供了许多功能实现，在微服务场景下比较实用。

```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Hoxton.SR12</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
    
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-oauth2</artifactId>
</dependency>
```

### 三、核心组件选型

#### 最佳方案： spring-boot-starter-security + spring-authorization-server

> 由于 spring-security-oauth2 已经迁移到 spring-security，而 spring-boot-starter-security 集成了 spring-security，且做了许多简化配置，特别适合于构建 spring-boot 程序。截至 2021年8月，spring-authorization-server 的最新版本是 0.1.2，最新的消息请关注官方动态 https://spring.io/blog/2020/04/15/announcing-the-spring-authorization-server

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.security.experimental/spring-security-oauth2-authorization-server -->
<dependency>
    <groupId>org.springframework.security.experimental</groupId>
    <artifactId>spring-security-oauth2-authorization-server</artifactId>
    <version>0.1.2</version>
</dependency>
```

Spring Authorization Server 项目遵守 OAuth2.1 规范而不支持 password 模式，因此需要是用此模式的项目可以考虑备选方案。

https://app.zenhub.com/workspaces/authorization-server-5e8f3182b5e8f5841bfc4902/issues/spring-projects/spring-authorization-server/459

![Spring Authorization Server 不支持 password 模式的官方答复截图](https://mtide.net/assets/images/post/20210809/5.png)

#### 备选方案： spring-security-oauth2 或 spring-cloud-starter-oauth2

> 此方案采用了 spring-security-oauth2，已被 Sprnig Security 官方停止更新，因此不再推荐。另外请注意的 spring-security-oauth2 2.4.0.RELEASE 及之后的版本会提示 deprecated。

* 如果采用 spring-security-oauth2，则 pom 引用如下：

```
<!-- https://mvnrepository.com/artifact/org.springframework.security.oauth/spring-security-oauth2 -->
<dependency>
    <groupId>org.springframework.security.oauth</groupId>
    <artifactId>spring-security-oauth2</artifactId>
    <version>2.3.8.RELEASE</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework.security.oauth.boot/spring-security-oauth2-autoconfigure -->
<dependency>
    <groupId>org.springframework.security.oauth.boot</groupId>
    <artifactId>spring-security-oauth2-autoconfigure</artifactId>
    <version>2.1.2.RELEASE</version>
</dependency>

<!-- spring-security-oauth2-autoconfigure 2.1.3.RELEASE 及之后的版本会提示 deprecated。
```

* 或采用 spring-cloud-starter-oauth2，则 pom 引用如下：

> spring-cloud-starter-oauth2 集成了 spring-security-oauth2，且做了许多简化配置，是在很长一段时间里基于开源软件构建 OAuth2 的主要方案之一。

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-oauth2</artifactId>
</dependency>
```

