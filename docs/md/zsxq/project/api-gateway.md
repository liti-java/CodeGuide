---
title: API网关：中间件设计和实践
lock: need
---

# API网关：中间件设计和实践

<div align="center">
    <img src="https://bugstack.cn/images/article/assembly/api-gateway/api-gateway-logo.png?raw=true">
    <div style="font-size: 12px;"><a href="https://t.zsxq.com/Ja27ujq">星球介绍：码农会锁 - 实战项目、专属小册、问题解答、简历指导、架构图稿、视频课程</a></div>
</div>

作者：小傅哥
<br/>博客：[https://bugstack.cn](https://bugstack.cn)

>沉淀、分享、成长，让自己和他人都能有所收获！😄

---

是滴，小傅哥又要准备搞事情了！这次准备下手**API网关**项目，因为这是所有互联网大厂都有的一个核心服务，承接着来自用户的滴滴打车、美团外卖、京东购物、微信支付，更是大促期间千万级访问量的核心系统。

🤔 那么它是一个什么样的项目呢？为什么会有它的存在？它是怎么设计实现的呢？都用到了哪些技术栈呢？

## 一、前言：网关是啥东西

在计算机网络中，[**网关**](https://zh.wikipedia.org/wiki/%E7%BD%91%E5%85%B3)（Gateway）是转发其他服务器通信数据的服务器，接收从客户端发送来的请求时，它就像自己拥有资源的源服务器一样对请求进行处理。

而**API网关**也是随着对传统庞大的单体应用（All in one）拆分为众多的微服务（Microservice）以后，所引入的统一通信管理系统。用于运行在外部http请求与内部rpc服务之间的一个流量入口，实现对外部请求的`协议转换`、`参数校验`、`鉴权`、`切量`、`熔断`、`限流`、`监控`、`风控`等各类共性的通用服务。

## 二、大厂：为啥都做网关

各大厂做网关，其实做的就是一套统一方案。将分布式微服务下的RPC到HTTP通信的同类共性的需求，凝练成通用的组件服务，减少在业务需求场景开发下，非业务需求的同类技术诉求的开发成本。

那么以往没有网关的时候怎么做，基本的做法就是再 RPC 服务之上再开发一个对应的 WEB 服务，这些 WEB 服务可以是 Spring MVC 工程，在 Spring MVC 工程中调用 RPC 服务，最终提供 HTTP 接口给到 H5、Web、小程序、APP 等应用中进行使用。如图 1-1 所示

![图 1-1 从传统方式到网关设计](https://bugstack.cn/images/article/assembly/api-gateway/api-gateway-220809-01.png)

传统开发 WEB 服务的几个问题：
- 问题1：每一个 WEB 应用，都需要与之匹配申请一套工程、域名、机器等资源，一直到部署，研发效率降低，维护成本增加。
- 问题2：每一个 WEB 应用，都会有所涉及共性需求，限流、熔断、降级、切量等诉求，维护代码成本增加。
- 问题3：每一个 WEB 应用，在整个使用生命周期内，都会涉及到文档的维护、工程的调试、联调的诉求，类似刀耕火种一样的开发势必降低研发效率。

**所以**：综上在微服务下的传统开发所遇到的这些问题，让各个大厂都有了自己自研网关的诉求，包括；`阿里`、`腾讯`、`百度`、`美团`、`京东`、`网易`、`亚马逊`等，都有自己成熟的 API 网关解决方案。毕竟这可以降低沟通成本、提升研发效率、提升资源利用率。

## 三、网关：系统架构设计

如果希望实现一个能支撑百亿级吞吐量的网关，那么它就应该是按照分布式架构思维做去中心化设计，支持横向扩展。让每一台网关服务都成为一个算力，把不同的微服务RPC接口，按照权重策略计算动态分配到各个算力组中，做到分布式运算的能力。

此外从设计实现上，要把网关的通信模块、管理服务、SDK、注册中心、运营平台等依次分开单独开发实现，这样才能进行独立的组合包装使用。

这就像为什么 ORM 框架在开发的时候不是与 Spring 强绑定在一起，而是开发一个独立的组件，当需要有 Spring 融合使用的时候，再单独开发一个 Mybatis-Spring 来整合服务。

所以在这里设计网关的时候也是同样的思路，就像官网的通信不应该一开始就把 Netty 相关的服务全部绑定到 Spring 容器，这样即增加了维护成本，也降低了系统的扩展性。

诸如此类的软件架构设计，都会在这套网关微服务架构中体现，整体架构如图 1-2 所示

![图 1-2 网关架构设计](https://bugstack.cn/images/article/assembly/api-gateway/api-gateway-220809-02.png)

整个**API网关**设计核心内容分为这么五块；
- `第一块`：是关于通信的协议处理，也是网关最本质的处理内容。这里需要借助 NIO 框架 Netty 处理 HTTP 请求，并进行协议转换泛化调用到 RPC 服务返回数据信息。
- `第二块`：是关于注册中心，这里需要把网关通信系统当做一个算力，每部署一个网关服务，都需要向注册中心注册一个算力。而注册中心还需要接收 RPC 接口的注册，这部分可以是基于 SDK 自动扫描注册也可以是人工介入管理。当 RPC 注册完成后，会被注册中心经过AHP权重计算分配到一组网关算力上进行使用。
- `第三块`：是关于路由服务，每一个注册上来的Netty通信服务，都会与他对应提供的分组网关相关联，例如：wg/(a/b/c)/user/... a/b/c 需要匹配到 Nginx 路由配置上，以确保不同的接口调用请求到对应的 Netty 服务上。PS：如果对应错误或者为启动，可能会发生类似B站事故。
- `第四块`：责任链下插件模块的调用，鉴权、授信、熔断、降级、限流、切量等，这些服务虽然不算是网关的定义下的内容，但作为共性通用的服务，它们通常也是被放到网关层统一设计实现和使用的。
- `第五块`：管理后台，作为一个网关项目少不了一个与之对应的管理后台，用户接口的注册维护、mock测试、日志查询、流量整形、网关管理等服务。

综上系统微服务模块结构如下：


| 序号 | 系统               | 描述                                                         |
| :----: | ------------------ | ------------------------------------------------------------ |
| 1    | api-gateway-core   | 网关核心系统：用于网络通信转换处理，承接http请求，调用RPC服务，责任链模块调用 |
| 2    | api-gateway-admin  | 网关管理系统：用于网关接口后台管理，注册下线停用控制         |
| 3    | api-gateway-sdk    | 网关注册组件：用于注解方式采集接口，发送消息注册接口         |
| 4    | api-gateway-center | 网关注册中心：提供网关注册中心服务，登记网关接口信息         |
| 5    | api-gateway-test-provider | 网关测试工程：提供RPC接口        |
| 6    | api-gateway-test-consumer | 网关测试工程：消费RPC接口         |

## 四、演示：网关运行效果

趁着周末假期小傅哥已经做了一部分的功能实现，就像小傅哥以前[《手写Spring》](https://bugstack.cn/md/spring/develop-mybatis/2022-03-20-%E7%AC%AC1%E7%AB%A0%EF%BC%9A%E5%BC%80%E7%AF%87%E4%BB%8B%E7%BB%8D%EF%BC%8C%E6%89%8B%E5%86%99Mybatis%E8%83%BD%E7%BB%99%E4%BD%A0%E5%B8%A6%E6%9D%A5%E4%BB%80%E4%B9%88%EF%BC%9F.html)、[《手写Mybatis》](https://bugstack.cn/md/spring/develop-spring/2021-05-16-%E7%AC%AC1%E7%AB%A0%EF%BC%9A%E5%BC%80%E7%AF%87%E4%BB%8B%E7%BB%8D%EF%BC%8C%E6%89%8B%E5%86%99Spring%E8%83%BD%E7%BB%99%E4%BD%A0%E5%B8%A6%E6%9D%A5%E4%BB%80%E4%B9%88%EF%BC%9F.html)一样，此项目也是渐进式的逐步完成各个模块功能的开发。并参照优秀源码级的项目架构设计，运用抽象和分治的设计技巧，解决功能间的耦合调用和服务设计。同时也结合设计原则和相应场景下的设计模式，开发出高质量易于迭代和维护的代码。部分代码实现和运行如图 1-3 所示

![图 1-3 网关运行效果](https://bugstack.cn/images/article/assembly/api-gateway/api-gateway-220809-03.png)

- 左侧是API网关核心通信模块，右侧是RPC(Dubbo)服务。通过对网页端发起的 http 请求，经过API网关的协议转换和对RPC的泛化调用包装结果数据并返回到页面，就是中间这张图的运行效果了。
- 左侧工程的实现，以渐进式分拆模块逐步完成，例如： core-01(Netty通信)、core-02(泛化调用)、core-03(执行器)等，让每一个对API网关感兴趣的读者都能从中学习到；架构的分层、功能的设计、代码的实现。

## 五、邀请：一起学习技术

💐以上关**API网关**的项目，也是小傅哥的星球【[码农会锁](https://t.zsxq.com/Ja27ujq)】准备带着读者一起利用`周末`和`假期`学习实践的内容。现在上车你将会通过小傅哥的`视频`+`小册`+`代码`+`直播`+`作业`，5方面来与你一起学习，帮助你提升技术实力，为你的职业生涯续期，也为你可以走的更远，可以多赚些钱。

### 1. 适合谁学

1. 一直使用 SpringMVC，想了解分布式架构。
2. 常年做CRUD开发，手里缺少硬核项目，面试张不开嘴。
3. 在校准备校招，市面都是流水账项目，面试没竞争力。
4. 希望多沉淀核心技术，让自己能公司留的下来，也能走的出去。

### 2. 能学到啥

1. 优秀架构：学习微服务架构设计方案
2. 整洁编码：学习源码级工程编程实现
3. 扩充技术：学习开发中间件所需技术（Netty、CGlib、SPI、GenericReference）
4. 打开思路：学习让自己开眼界的项目（技术并不难，但没人告诉你，就走的很难！）

### 3. 开发计划

1. 每周末和假期编写API网关，并同时录制设计和实现的视频、编写小册和定期组织直播解决读者疑惑以及分享。**稳定输出，可靠学习**
2. 整个系统的设计和实现，遵循分治和抽象的设计分层原则，按照微服务构建服务模块，以源码级体验学习。
3. 关于API网关会更注重骨架的结构和核心流程的功能，把最干净的部分交给读者，让读者更加易懂，也易扩展。

**目录：**

- [x] [第1章：HTTP请求会话协议处理](https://bugstack.cn/md/assembly/api-gateway/2022-08-13-%E7%AC%AC1%E7%AB%A0%EF%BC%9AHTTP%E8%AF%B7%E6%B1%82%E4%BC%9A%E8%AF%9D%E5%8D%8F%E8%AE%AE%E5%A4%84%E7%90%86.html)
    ![](https://bugstack.cn/images/article/assembly/api-gateway/api-gateway-220809-07.png)
- [x] [第2章：代理RPC泛化调用](https://bugstack.cn/md/assembly/api-gateway/2022-08-20-%E7%AC%AC2%E7%AB%A0%EF%BC%9A%E4%BB%A3%E7%90%86RPC%E6%B3%9B%E5%8C%96%E8%B0%83%E7%94%A8.html)
- [x] [第3章：分治处理会话流程](https://bugstack.cn/md/assembly/api-gateway/2022-08-27-%E7%AC%AC3%E7%AB%A0%EF%BC%9A%E5%88%86%E6%B2%BB%E5%A4%84%E7%90%86%E4%BC%9A%E8%AF%9D%E6%B5%81%E7%A8%8B.html)
- [ ] [第4章：方法执行器封装](#)
- [ ] 梳理中 ... 每周更新
