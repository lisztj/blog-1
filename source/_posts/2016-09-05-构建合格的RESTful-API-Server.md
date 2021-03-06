---
title: 构建合格的 RESTful API Server
subtitle: build-qualified-restful-api-server
tags:
  - RESTful
  - API
  - Node.js
  - Express
categories: 一只代码狗的自我修养
date: 2016-09-05 16:11:49
---
这一篇我要把 References 写在前面：
- [再谈 API 的撰写 - 总览](http://mp.weixin.qq.com/s?__biz=MzA3NDM0ODQwMw==&mid=401902529&idx=1&sn=575ae8fdf163afa30604d712a73079fd&scene=21#wechat_redirect)；
- [再谈 API 的撰写 - 架构](http://mp.weixin.qq.com/s?__biz=MzA3NDM0ODQwMw==&mid=401924543&idx=1&sn=97de2e09c9fddfd905992c19aedb6182&scene=1&srcid=0427sAmgXKhksqURPXRj2cxv#wechat_redirect)；
- [再谈 API 的撰写 - 子系统](http://mp.weixin.qq.com/s?__biz=MzA3NDM0ODQwMw==&mid=402076898&idx=1&sn=32b7591a6385ab695d5070061bf18a0a&scene=1&srcid=04276Jyhm6g4QMOyPgfm8jxj#wechat_redirect)；
- [再谈 API 的撰写 - 契约](http://mp.weixin.qq.com/s?__biz=MzA3NDM0ODQwMw==&mid=402114651&idx=1&sn=a7b891f532e29b73afd83f17ae071023&scene=1&srcid=0427CHvTKeMIQsr5uT3x9nIN#wechat_redirect)；

通过这一系列文章，大神已经自顶向下的把构建一个合格的 RESTful API Server 的要点都涉及到了，并且基本都是最佳实践，值得反复咀嚼。这一篇我结合自己的实践做一些 localization 的总结和实践归纳。文中都以 Node.js 的 Express 框架来举例。
<!-- more -->

## 分层架构

> All problems in computer science can be solved by another level of indirection.
> —— David Wheeler

分层架构是最常见的软件架构，作为 API Server，一般我们也采用这样的架构，而在 Web 后端架构中最流行的当然是 MVC 的架构模式。不过，View 层对于 API server 是没有的。

先梳理一下操作流程，前端请求 URL 经过 Router 匹配，之后 Controller 层进行数据处理，不过数据处理一般是繁杂的，这里可以再分一层叫 Service 层，处理所有与数据库直接交互的部分，也便于 Controller 层对于相同功能进行再拆分和复用，最后返回。这样的分层模型我们不如叫 MRCS 比较精确。

具体来说：
- Model：与数据库的数据模型一一对应，定义了整个项目的所有数据操作的模型基础；
- Router：API Server 所有定义的路由；
- Controller：因为已经有 Service 层，所有这里仅进行一些输入参数的验证和解析以及结果数据的重新组织和返回，更底层的数据库交互交给 Service 层；
- Service：所有与数据库和缓存的数据交互都在这里，这里的函数不是与 Controller 层的函数一一对应的，而应该是更细小颗粒功能的划分，让 Controller 层来进行组织，从而实现对底层功能的充分复用。

## 文档与接口参数验证

一个普通的网站可以没有对外的文档，可是一个 API Server 却一定得有文档，没有文档的 API Server 毫无意义。

而且，文档和接口参数验证是紧密相关的，如果你把这两部分分开了，那说明你一定是在某处重复定义了接口的参数模型。所以这两部分放在一起来讲。这也就是说，不管具体是写在哪，接口参数模型只手写定义一次，文档生成和参数验证都以这同一个定义为依据来进行（这样也极大的增强了文档与代码一致性的可能）。接口参数验证的同时也可以进行部分参数解析的工作，比如还原参数的类型（前端传递过来的都是 String），甚至还可以顺手把数据组织成 Controller 层需要的格式。

程序员一般而言都是不喜欢写文档的，团队也通常没有更多的资源让专人来维护文档，所以如何花费最小的代价完成与代码一致的高质量的文档是一个很重要的课题。

一个好的文档系统应该具备的特性：
- 具有良好的机制尽可能保证代码和文档的一致性；
- 不同版本之间`diff`的功能；
- 每一个定义的接口下面可以直接在线进行类似于 Postman 的接口可用性测试；
- 需要重复定义的部分可以抽象出来，定义一次，多处复用；

如果你时间充裕，并且想做边际效应高的事情，那么你可以考虑使用 Swagger。这是一个庞大的文档框架体系，工具全面且强大，但有些门槛，需要学习一段时间（使用可以参看我的下一篇博文：[使用Swagger构建Express API Server的文档系统](http://maples7.com/2016/09/06/build-doc-system-of-express-api-server-with-swagger/)）。类似的还有RAML，这是一门基于 YAML 的文档建模语言，用法灵活，功能强大，但是目前工具还不是很靠谱，或许因为用的人也不是很多，项目显得有些缺乏维护，不过我觉得它对于构建文档系统的指导思想是很先进的。

如果是为了花费最小的成本，可以使用 apidoc，这就是一个普通的 npm 包，你可以直接使用它从函数注释生成 HTML 文档，支持版本对比，支持继承复用，支持接口在线可用性测试。另外，使用这个包的插件可以直接从 json-schema 中导入对参数模型的定义，这样只需要定义一次，就可以同时用 json-shcema 进行接口参数验证和生成文档。~~目前我一般选择这种方案。~~不过你也可以反过来，先在注释上定义所有的文档参数模型，然后用一个 Parser 解析从而验证参数（我相信 apidoc 中是有一个这样的 Parser 的，不过对外没有提供参数验证的功能，我也没有找到第三方的插件可以实现，或许有时间可以自己去写一个插件或者直接在 apidoc 上实现）。如果你采用后一种方案，那么分清程序的[「编译时」和「运行时」](http://mp.weixin.qq.com/s?__biz=MzA3NDM0ODQwMw==&mid=402003317&idx=1&sn=68dabd5cbf565ab3fd99f90641a01a9f&scene=21#wechat_redirect)很重要，因为如果你要从注释解析后验证参数，那么你必须在「编译时」就已经从注释获得了所有的接口参数的定义模型，只有这样在「运行时」才能快速进行接口参数模型的匹配与验证。

文档最后需要部署到一个外部可以查看到的地方供使用者查阅和测试，这时可以借助 gulp 等类似工具来尽可能自动化的实现。

## 测试

一般的软件可能需要开发书写单元测试，可是个人觉得 API Server 的单元测试和接口测试实际上做的是很多重复的工作。所以我觉得 API Server 直接对接口进行功能性测试就好。

不过接口测试是非常难写的，一个接口需要完整测试的 test cases 可能高达十几个甚至几十个。除了更好的对接口进行划分外，目前我还没想好如何缓解这个问题。

不管怎么说，需要持续维护的项目都应该写测试，API Server 依然可以用`Mocha/should/supertest/istanbul/gulp`这套技术栈来书写自动化的接口测试。

## 统一数据返回

好的 API Server 应该定义统一的数据返回格式，这应该成为与前端的固定约定，这样前端才能方便的对返回数据进行验证和进一步操作。这就好比浏览器通过 Status Code 来进行对应的后续操作一样。

为了实现这一点，所有的接口调用在返回给前端之前都应该经过至少同一个中间件进行数组格式的重新组织，并且匹配到合适的 HTTP Status Code 以及自定义的返回码（非正常的数据返回时自定义的返回码，这个也是与前端的约定之一）和明确的返回信息，JSON 化之后返回。

## 数据序列化 

数据序列化分为输入数据的序列化和输出数据的序列化。

输入数据的序列化可以在前文所述的接口参数验证时完成（不复杂的情况下），也可以单独在一个中间件中完成。其实这一步不是必要的，因为通常各个接口的 Controller 需求是各不一样的，这里只能进行一些通用化的序列化操作。

相比于输入数据的序列化，输出数据的序列化要重要得多，而且一般通用性更强。举例来说，你有很多个接口都需要返回数据库中的同一个数据实例，但是不同接口需要返回给前端的字段和内容可能是各不一样的，此时就可以在这一步把输出数据序列化成前端的要求。最后的返回数据都是最精简且语义化良好的。这一步的操作可以有效节省网络流量，对移动端和处女座程序员都很有意义。

## 缓存

缓存不是必需的，但却是对高性能服务的基本要求。一个好的缓存设计不仅对性能有影响，而且对后期的开发调试也有很大影响。毕竟，解决了缓存，你就已经解决的计算机科学中一半的难题（:D）：

> There are only two hard things in Computer Science: cache invalidation and naming things. 
> —— Phil Karlton

你可以进行路由层级的缓存设计，也可以进行数据库操作层级的缓存设计。但不管缓存怎样设计，除去基本的可用性和鲁棒性之外，最大的目标应该是对使用者尽可能友好：

> Simplicity is the ultimate sophistication.   - Leonardo Da Vinci

「对使用者尽可能友好」指的是：
1. 接口调用简单，甚至不需要手动调用；
2. 缓存可以自动过期；
3. 尽可能保证已经无效的缓存（缓存数据已经与真实数据不一致，也可以称为旧的缓存）可以无遗漏的尽快被删除；

第3点尤其重要，因为如果程序员手动控制缓存删除，那么对同一个数据块缓存的操作代码可能分散在项目各处，很难保证及时和没有遗漏。不过目前我也没有找到比较好的实现方案，初步的想法是如果能实现有一个 watcher 可以监听某一个数据块是否即将被改动（或者是否刚刚已经被改动）就好了。如果即将被更改，那在更改后立刻自动删除旧缓存。其实我觉得这个方案可以在 ORM 中实现，但目前没有发现有 ORM 支持这一点。

与「设计」一样，缓存系统的终极目标应该是使上层使用者根本感觉不到它的存在。这是指，当你调用获取数据的底层接口时，你无需知道数据是来源于真实数据库还是缓存。做到这一点无疑很难，不过你至少可以做到把对上层系统的影响降到最低，也就是说，使用缓存和不使用缓存只需要更改尽可能少的代码即可以轻易实现。

## 总结

本文没有提到标题中 RESTful 相关的东西，主要是路由设计的时候遵循 RESTful 的原则就可以了，无需多讲。

另外，很多东西还只是提供了一个基本的方向和原则，还有待更多的实践来验证和改进。

## 彩蛋

1. Facebook 提出了一种新的不同于 REST 的 API 数据查询标准——[GraphQL](http://graphql.org/)，这套标准可以让前端（广义的，包含移动端）来定义需要获取的数据模型，这样做可以极大的减轻后端对于前文所述的文档、接口参数验证、统一数据返回、数据序列化的工作，看起来很有意思。更多了解除了[官网](http://graphql.org/)之外，还可以参考这篇文章：[《新一代数据查询语言 GraphQL 来啦！》](http://imweb.io/topic/58499c299be501ba17b10a9e)。

2. Google 在这方面当然也毫不示弱，它家的 Google+ API 中也有类似的思想：[Partial Responses](https://developers.google.com/+/web/api/rest/#partial-response)。具体说，利用请求的参数`fields`来由客户端决定哪些返回参数是我这次请求所需要的。当然，这没有 Facebook 的 GraphQL 功能系统和强大，只能返回一个API全集数据的一个子集，不过它是完全基于REST的，这意味着你可能只需对你现有的系统做最小的改动即可实现类似的功能，减轻后端对于输出数据序列化的负担。
