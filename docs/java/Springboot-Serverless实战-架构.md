---
title: Springboot Serverless 实战 - 架构
description: 'Springboot Serverless 实战 - 架构'
position: 11
category: 'FC_Blog'
---

作者 | 西流

[SpringBoot](https://spring.io/projects/spring-boot) 是基于 Java Spring 框架的套件，它预装了 Spring 的一系列组件，让开发者只需要很少的配置就可以创建独立运行的应用程序。在云原生的世界，有大量的平台可以运行 SpringBoot 应用，例如虚拟机，容器等。但其中最有吸引力的，是以 Serverless 的方式运行 SpringBoot 应用。我将通过一系列文章，从架构，部署，监控、性能、安全等5个方面来分析 Serverless 平台运行 SpringBoot 应用的优劣。为了让分析更有代表性，我选择了 github 上 star 数超过 50k 的电商应用 [mall](https://github.com/macrozheng/mall) 作为示例。这是系列文章的第一篇，首先从架构角度分析 SpringBoot 应用的 Serverless 化。

## Mall 架构简介

Mall 是一套电商系统，包括前台商城系统及后台管理系统，基于 SpringBoot + MyBatis 实现。前台商城系统包含首页门户、商品推荐、商品搜索、商品展示、购物车、订单流程、会员中心、客户服务、帮助中心等模块。后台管理系统包含商品管理、订单管理、会员管理、促销管理、运营管理、内容管理、统计报表、财务管理、权限管理、设置等模块。

Mall 的架构如下图所示，分为网关层，应用层，数据存储层。请求首先通过网关到达 SpringBoot 应用服务。网关实现负载均衡，流量控制等功能。应用层包含3个 SpringBoot 应用和1个前端应用：

- mall-admin：后台商城管理系统
- mall-portal：前台商城系统
- mall-search：于Elasticsearch的商品搜索系统
- Mall-admin-web：mall-admin 的前端展示，基于 Vue+Element 实现

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635518177477-9483f633-440d-4015-9d4a-7b9d32e1efc3.png#id=hFSCs&originHeight=665&originWidth=1744&originalType=binary&ratio=1&status=done&style=none)

Mall 使用了 MySQL，Redis，MongoDB，ElaisticSearch 等多种数据库。主要业务数据存储在 MySQL，缓存数据存储在 Redis，用户行为分析数据存储在 MongoDB，搜索数据存储在 ElasticSearch 中。SpringBoot 应用服务间使用 RabbitMQ 实现异步通信。

## Serverless 计算平台-函数计算简介

Serverless 是指用户上传代码包或者容器镜像，平台会负责申请资源，可靠的运行代码。[函数计算](https://help.aliyun.com/product/50980.html)（Function as a Service，FaaS）是 Serverless 的一种计算模式，最突出的特点是内置了网关层能力，能够实现缩容到0，快速的自动伸缩。函数计算也提供了全面的可观测和问题诊断能力。函数计算这些特点，使其很适合 SpringBoot 这类 Web 应用。使用函数计算，开发者只需要专注于 SpringBoot 应用逻辑的实现，而不再费心应用运行环境的搭建、部署、监控等无差别的工作。

## Mall 应用 Serverless 架构总览

Mall 是一个非常标准的 3 层架构 Web 应用，改造为 Serverless 架构非常容易，架构如下所示。由于函数计算内置了网关服务，自动拉起实例运行应用，因此开发者只需要上传应用代码即可。

应用实例在函数计算平台上运行，能够自由的访问其他服务，因此和 MySQL，Redis，RabbitMQ 等服务的访问方式不变。

函数计算内置了日志收集和展示能力。开发者为函数计算指定[阿里云日志服务](https://help.aliyun.com/product/28958.html)的 LogStore，打到标准输出的日志会自动收集到日志服务查询、展示。开发者也可将日志投递到自己的日志处理系统中，但需要做一些额外的配置。在本示例中，我们采用阿里云日志服务来处理应用日志。

函数计算也提供了一系列工具，帮助开发者通过 Jenkins CICD 工具发布应用。我们将在后续的文章中一一展示。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635558315375-5d5ea2dd-abcb-4473-8477-e22817184a5c.png#id=nOX6T&originHeight=734&originWidth=1744&originalType=binary&ratio=1&status=done&style=none)

## 在函数计算平台运行 SpringBoot 应用

在阿里云函数计算平台上运行 Web 应用，需要了解以下几个概念：

- [服务](https://help.aliyun.com/document_detail/74925.htm)：函数计算的服务资源对应微服务。一个服务下可以创建多个函数，这些函数共享服务级别的配置，包括日志、权限、VPC 网络访问配置等等。一般来说，开发者根据业务场景设计微服务架构，然后为每一个微服务创建函数计算的服务。然后再根据需求，将微服务实现为更细粒度的函数。比如有些逻辑是计算密集型的，可以将它拆分为另一个函数，配置不同的实例规格，既满足性能要求，又优化了成本。按照微服务的理念，一个服务下的函数个数不宜太多
- [函数](https://help.aliyun.com/document_detail/52077.html)：函数是运行开发者代码的基本单位。函数的粒度可以很细，比如对应 1 个 API，也可以较粗，对应一组 API。不同的函数配置不同的实例规格。函数计算提供了各种语言的运行时，也提供 custom runtime/custom container 和语言无关的运行时。如果只是用函数计算实现片段代码，可以使用相关语言的运行时。在我们的场景下，因为要无缝迁移 SpringBoot 应用，我们选择 custom container 运行时。Mall 项目已经支持了将 mall 应用自动打包为容器镜像，因此只需要将镜像上传至阿里云容器镜像仓库，并在函数上指定相关信息即可
- [HTTP 触发器](https://help.aliyun.com/document_detail/71229.html)：为函数配置 HTTP 触发器后，函数可通过 HTTP 请求的方式调用。函数计算配套的 Serverless Devs 工具会为 HTTP 触发器生成测试域名，开发者可以方便的调试和运行 Web 应用

在了解了这些基本概念后，我们就可以开始构建和部署 Serverless 架构的 mall 应用了。请参考 SpringBoot on FC - 部署篇。
