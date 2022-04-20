---
title: Springboot Serverless 实战 - 监控调试
description: 'Springboot Serverless 实战 - 监控调试'
position: 13
category: 'FC_Blog'
---

作者 | 西流

[SpringBoot](https://spring.io/projects/spring-boot) 是基于 Java Spring 框架的套件，它预装了 Spring 的一系列组件，让开发者只需要很少的配置就可以创建独立运行的应用程序。在云原生的世界，有大量的平台可以运行 SpringBoot 应用，例如虚拟机，容器等。但其中最有吸引力的，是以 Serverless 的方式运行 SpringBoot 应用。我将通过一系列文章，从架构，部署，监控、性能、安全等5个方面来分析 Serverless 平台运行 SpringBoot 应用的优劣。为了让分析更有代表性，我选择了 github 上 star 数超过 50k 的电商应用 [mall](https://github.com/macrozheng/mall) 作为示例。通过“SpringBoot Serverless 实战 - 部署”这篇文章，相信大家已经感受到 Serverless 应用上线的便捷。这是系列文章的第三篇， 向大家展示如何监控和调试 Serverless 应用。

## 实时日志

对运行在远端云平台上的应用而言，日志是调试的主要手段。分布式应用下有多个实例，日志的收集和分析是很有挑战的。虽然有很多成熟的开源产品，但要搭建和持续运维这些软件，成本不小。函数计算内置了对日志收集/分析的完整支持。用户只需要在应用逻辑中输出日志，这些日子被自动收集，并可以实时查看，通过多种方式聚合，查询。

在之前的文章中，我们通过 Serverless Devs 工具已经为应用自动创建创建了日志仓库，可以在函数计算控制台查看请求，应用级别的日志，也可使用 SQL 语言进行高级查询。除此之外，Serverless Devs 工具还提供了实时日志功能，对于应用调试非常有帮助。

在项目根目录下，即 s.yaml 所在的目录，执行下面的命令，将输出 s.yaml 中定义的所有服务的日志。

```shell
sudo -E s logs
```

用户也可查看指定服务的日志。

```shell
sudo -E s mall-admin logs
```

通过 `-t` 参数，用户也可以进入观察模式实时查看日志。

```shell
sudo -E s mall-admin logs -t
```

此时 Serverless Devs 工具会实时监听 mall-admin 应用下所有实例的日志，将新产生的日志实时展示。此后，当我们通过浏览器或者 curl 等方式给 mall-admin 应用发送请求，就能看到对应的请求处理日志输出。

![](https://img.alicdn.com/imgextra/i4/O1CN01EUftQy1PE6q4HSaW5_!!6000000001808-2-tps-1500-851.png)

Serverless Devs 也支持根据关键词查询日志。比如我们可以执行下面的命令，查看 mall-admin 应用 ERROR 级别的日志。

```shell
s mall-admin logs -t --keyword=ERROR
```

## 指标多维查询展示

除了 Serverless Devs 的命令行工具，用户也可以在函数计算控制台上从函数、实例、请求等多个维度查看日志。

以 `mall-admin` 为例，在函数计算控制台左侧导航栏，点击“服务及函数”，选择 `mall-admin` 服务，再选择该服务下的同名函数，进入 `mall-admin` 函数详情页。点击 `监控指标` 标签页。

如下图所示，请求列表展示了请求的执行情况，包括成功/失败，是在什么函数版本上执行的，执行时长，内存用量，在哪个实例上执行等等。也可以方便的查询请求相关的日志。

![](https://img.alicdn.com/imgextra/i1/O1CN01xuUtfp1yYGpGXJY52_!!6000000006590-2-tps-1500-743.png)

下图展示了实例维度的信息。除了指标，用户也可以到滚动到页面下方，查看对应的实例列表，以及登录到实例上执行相关的操作。

> 注意：函数计算的**按量实例**完全由系统管理，实例在闲置一段时间后就会被系统回收。被回收的实例不再被使用，不能登录。在下图中以灰色显示。


![](https://img.alicdn.com/imgextra/i1/O1CN011aqeRO1VCArNZhYGt_!!6000000002616-2-tps-1500-724.png)

![](https://img.alicdn.com/imgextra/i4/O1CN01TSdPTH1pgD9KQkOjj_!!6000000005389-2-tps-1500-460.png)

通过函数计算平台提供的日志收集和查询能力，用户的开发流程被无缝衔接起来。修改代码，使用 Serverless Devs 工具部署应用，查看日志，整个流程丝般顺滑。

## 本地调试

在将应用部署到云平台之前，我们通常希望能在本地部署应用，进行调试。Serverless Devs 工具提供了本地运行应用的功能。

在项目根目录（s.yaml 所在目录），执行命令，即可启动对应的服务。`auto` 参数是指自动为实例生成和 Web 框架兼容的测试域名。例如执行下述命令：

```shell
sudo -E s mall-admin local start auto
```

工具会在本地启动函数实例，并提供一个可供调用的 url。这样我们可以在本地调试 Web 应用，提高效率。

> 注意：每次启动本地实例，监听端口是随机生成的。


![](https://img.alicdn.com/imgextra/i1/O1CN012AUKLA1rrmNz55z7O_!!6000000005685-2-tps-1892-446.png)

## 端云联调

很多时候，构成应用的微服务/函数需要和其他服务相互调用。除了在本地进行简单的单元测试，联调或者集成测试必须要把代码部署到云端。这样的方式使得开发调试的流程比较长，云端的复杂环境也增大了问题诊断的难度。比如：

- 要平迁原有的应用，函数实例需要访问云端环境中的其他服务，遇到实例启动不起来时，该怎么排查原因？
- 应用采用微服务架构，涉及到多个服务。能否在本地代码开发完成后快速进行端对端测试？
- 事件驱动的应用，通过事件源触发函数，环节多，链路长，能不能在本地快速测试整个链路？
- ……

为了解决上述问题，Serverless Devs 提供了端云联调功能。开发者通过端云联调能在本地启动实例，和云端环境无缝连通，快速进行测试和问题调试。端云联调能帮助开发者：

1.  变更代码，实时查看结果，调试迭代的闭环最短。 
1.  能够复用本地丰富的开发调试工具，效率最高。 

端云联调在本地开发机和云端应用的 VPC 环境间建立一条安全的隧道连接。访问云端应用的流量将自动转发到本地开发机上；同时本地实例对外访问的网络流量也被自动转发到云端应用的 VPC 环境中。比如在本地实例访问云端的 RDS 数据库实例，传统方式开发者只能放开 RDS 实例的公网访问。而使用端云联调，不需要任何配置的改变，可以直接以内网的方式访问 RDS 实例。

以 mall 应用为例，整个应用由 mall-admin-web，mall-admin，mall-portal，mall-search 等多个服务构成。服务之间有上下游依赖，比如 mall-admin-web 会向下游的 mall-admin 服务发送请求。假设我们已经在测试环境部署了一整套 mall 应用的服务，现在想在开发机全链路调试 mall-admin 服务，需要把 mall-admin-web 等整套服务和数据库都部署到开发机，或者通过公网与云端 VPC 内的服务和数据库交互，这是非常繁琐甚至不现实的。端云联调能让我们在本地开发机环境启动 mall-admin 服务的实例，安全的与云端 VPC 环境的其他服务和数据库无缝交互。用户不需要做任何设置。

首先在 s.yaml 所在的目录执行下述命令，针对 mall-admin 服务启动端云联调。

```shell
sudo -E s mall-admin proxied setup
```

然后在控制台访问 mall-admin-web 应用，可以看到相关的请求已经被转发到了本地的 mall-admin 函数实例上。而且本地实例可以无缝的访问云端 VPC 内的数据库或者其他服务。

> 注意：当使用了端云联调后，所有的流量都会发送到本地的实例上。要让流量恢复到函数计算上的实例，需要执行 s deploy 重新部署相关的函数。


![](https://img.alicdn.com/imgextra/i4/O1CN01Sotv3i1p61rWEyYqu_!!6000000005310-2-tps-1500-933.png)

## 总结
从下图的两个报告中， 我们可以看出， 在 Serverless 领域， 调试和可观测一直是 Serverless 开发实践者最大的两个痛点。
![image.png](https://img.alicdn.com/imgextra/i1/O1CN01s7Q4bc1QdaU7AOfBw_!!6000000001999-2-tps-2096-914.png)

函数计算这个  Serverless 产品始终践行开发者第一的理念， 在调试和可观测方面探索和落地走在所有云厂商的前面， 在调试方面，Serverless Devs 工具支持本地调试、端云联调甚至是远程调试； 而在可观测方面， 推出了其他厂商不具备的秒级监控、实例指标以及实例登录等，极大提高了 Serverless 开发者的工作效率和幸福感， Server Less, Value More!
