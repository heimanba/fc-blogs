---
title: Springboot Serverless 实战 - 性能调优
description: 'Springboot Serverless 实战 - 性能调优'
position: 14
category: 'FC_Blog'
---

作者 | 西流

[SpringBoot](https://spring.io/projects/spring-boot) 是基于 Java Spring 框架的套件，它预装了 Spring 的一系列组件，让开发者只需要很少的配置就可以创建独立运行的应用程序。在云原生的世界，有大量的平台可以运行 SpringBoot 应用，例如虚拟机，容器等。但其中最有吸引力的，是以 Serverless 的方式运行 SpringBoot 应用。我将通过一系列文章，从架构，部署，监控、性能、安全等5个方面来分析 Serverless 平台运行 SpringBoot 应用的优劣。为了让分析更有代表性，我选择了 github 上 star 数超过 50k 的电商应用 [mall](https://github.com/macrozheng/mall) 作为示例。这是系列文章的第四篇， 向大家展示如何对 Serverless 应用性能调优。

## 1. 实例启动速度优化

在之前的教程中，相信大家都感受到 Serverless 的快捷，上传代码包和镜像，就上线了一个弹性高可用的 Web 应用。但其中一个不好的点是首次启动会很慢，因为一段时间没有请求后，Serverless 平台会回收函数实例。下一次再有请求，系统会实时拉起实例，这个过程称之为冷启动。Mall 应用实例的启动大约 30 秒左右，用户会感受较长时间的冷启动延时。

在优化冷启动之前，我们首先分析清楚冷启动各个阶段的耗时。在函数计算（FC） 控制台的服务配置界面，开启链路追踪功能。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1636529127016-4d7c8676-164f-4830-a42f-fa5f4c1e3ad8.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0#crop=0&crop=0&crop=1&crop=1&id=P0e5T&originHeight=798&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

对 mall-admin 服务发起请求，成功后查看 FC 控制台，我们能看到相应的请求信息。注意关闭“仅查看函数错误”，这样才会显示所有请求。指标监控和调用链路数据收集有一定延时，如果没有显示，请等待一会再刷新。找到冷启动标记的请求，点击 “更多” 下的 “请求详情”。

![](https://cdn.nlark.com/yuque/0/2022/png/995498/1641283516964-1e17dc64-1011-4a40-bb57-ea7370269e3e.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0#crop=0&crop=0&crop=1&crop=1&id=nyZEH&originHeight=721&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

调用链路会显示冷启动各个环节的耗时。冷启动包含以下几个环节：

- 代码准备（PrepareCode）：主要是下载代码包或者镜像。由于我们已经启用了镜像加速功能，不需要下载全部的镜像，因此这一步的延时非常短
- 运行时初始化（RuntimeInitialization）：从启动函数开始，到函数计算（FC）系统探测到应用端口就绪为止。这中间包含了应用启动时间。在命令行执行 `s mall-admin logs`查看相应的日志时间，我们也能看到 SpringBoot 应用的启动需要花大量的时间
- 应用初始化（Initialization）：函数计算提供了 Initializer 接口，用户可以将一些初始化逻辑放在 initializer 中执行
- 调用延时（Invocation）：处理请求的延时，这个延时非常短

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1636532656418-6570b6e9-b3ab-4923-b064-2ce8a408672e.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0#crop=0&crop=0&crop=1&crop=1&id=zWAMb&originHeight=390&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

从上述链路追踪图来看，实例启动时间是瓶颈，我们可以采取多种方式来优化。

### 1.1. 使用预留实例

Java 类应用普遍启动较慢。应用在初始化时，也需要和很多外部服务交互，耗时较长。这类流程是业务逻辑需要的，很难优化延时。因此函数计算提供了预留实例功能。预留实例的起停由用户自己控制，没有请求也会常驻在那，因此不会有冷启动的问题，当然用户需要为整个实例的运行付费，即便实例没有处理任何请求。

在函数计算控制台，我们可以在“弹性伸缩”页面为函数设置预留实例。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1636616176229-bde75cec-bad6-4801-a61d-6346ed0b1ea4.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0#crop=0&crop=0&crop=1&crop=1&id=OYNx0&originHeight=470&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

用户在控制台中配置最小和最大实例数。平台会预留最小实例数目的实例，最大实例是指该函数下实例的上限。用户也可以设置定时预留和按指标预留的规则。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1636616934812-6d21f0c2-2586-430d-89c1-6d8023d195d0.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0#crop=0&crop=0&crop=1&crop=1&id=nk1Hf&originHeight=618&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

创建预留规则后，系统就会创建预留实例。当预留实例就绪后，我们再访问函数就不会有冷启动。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1636618290293-d4c58351-6b1d-49c7-9826-67a46a28bea4.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0#crop=0&crop=0&crop=1&crop=1&id=BQCdJ&originHeight=469&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

### 1.2. 优化实例启动速度

#### 延迟初始化

在 Spring Boot 2.2 及更高版本中，可以开启一个全局延迟初始化标志。这将提高启动速度，但代价是第一个请求的延迟时间可能变长，因为需要等待组件首次初始化。

可在 s.yaml 中为相关应用配置以下环境变量

```shell
SPRING_MAIN_LAZY_INITIATIALIZATION=true
```

#### 关闭优化编译器

默认情况下，JVM 有多个阶段的 JIT 编译。虽然这些阶段可以逐渐提高应用的效率，但它们也会增加内存使用的开销，并增加启动时间。对于短期运行的 serverless 应用，请考虑关闭此优化，以牺牲长期效率换取更短的启动时间。

可在 s.yaml 中为相关应用配置以下环境变量：

```shell
JAVA_TOOL_OPTIONS="-XX:+TieredCompilation -XX:TieredStopAtLevel=1"
```

#### s.yaml 中设置环境变量示例：

如下图所示，对 mall-admin 函数配置环境变量。然后执行 `sudo -E s mall-admin deploy` 部署。

![](https://cdn.nlark.com/yuque/0/2022/png/995498/1642384597670-93d52126-1392-4077-add5-42e619a617a8.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0#crop=0&crop=0&crop=1&crop=1&id=aPGAW&originHeight=1433&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

#### 登录实例检查环境变量是否配置正确

在控制台函数详情页的请求列表中找到对应的请求，点击`更多`中的`实例详情`链接。

![](https://cdn.nlark.com/yuque/0/2022/png/995498/1642384850027-aac7716b-9bde-4866-841d-675ac6726a9c.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0#crop=0&crop=0&crop=1&crop=1&id=fNr3T&originHeight=700&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

在实例详情页中点击`登录实例`。

![](https://cdn.nlark.com/yuque/0/2022/png/995498/1642385045445-625b2fbc-3ff1-48a1-9de8-61c7f760a7a6.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0#crop=0&crop=0&crop=1&crop=1&id=IAPjf&originHeight=740&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

在 shell 界面中执行 echo 命令，查看对应的环境变量是否设置正确。

> 注意：对于非预留实例，一段时间没有请求后，函数计算系统会自动回收实例。此时无法再登入实例（上面的`实例详情`页面中的`登录实例`按钮会变灰）。所以请执行调用后，在实例被回收之前尽快登录。


![](https://cdn.nlark.com/yuque/0/2022/png/995498/1642385572234-8531e14c-0fe7-4a74-892b-d1885c40c1a5.png#crop=0&crop=0&crop=1&crop=1&id=fbrgA&originHeight=980&originWidth=2840&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

## 2. 配置合理的实例参数

当我们选择了应用实例规格，比如 2C4G 或者 4C8G，接下来我们希望知道一个实例处理多少请求是合适的，既能充分利用资源又能保证性能。当处理的请求超过一个限制后，系统能够快速弹出实例，保证应用性能平滑。如何度量实例过载有多个维度，例如 qps 超过一定阈值，或者实例 CPU/Memory/Network/Load 等指标超过阈值等等。函数计算使用实例并发度（Instance Concurrency）来作为实例负载的度量和实例伸缩的依据。实例并发度（Instance Concurrency）是指一个实例能同时执行的请求数。例如将实例并发度设置为20，则意味着一个实例在任意时刻最大能**同时**执行20个请求。

> 注意：请区分实例并发度和 QPS 的区别。


使用实例并发度来度量负载有如下优势：

- 系统能够迅速统计实例并发度指标值进行扩缩容。CPU/Memory/Network/Load 等实例级别的指标通常是后台统计，需要花费数十秒的指标统计后才能进行伸缩，难以满足在线应用的弹性伸缩要求
- 在各种条件下，实例并发度指标都能够稳定的反映系统负载高低。如果以请求延时作为指标，系统难以区分是实例过载导致延时变大，还是下游服务成为瓶颈导致延时变大。例如一个典型的 Web 应用，通常会访问 MySQL 数据库。如果数据库成为瓶颈，请求延时变大，此时扩容不但毫无意义，而且会压垮数据库，让情况更加恶化。QPS 和请求延时相关，也会有上述问题

实例并发度作为伸缩依据虽然有上述优点，但用户常常并不知道该设置多大的实例并发度。推荐按照下述流程确定合理的并发度：

1. 将应用函数的最大实例数设置为1，确保压测到单个实例的性能。
1. 使用负载压测工具对应用进行压测，查看 tps 和请求延时等指标
1. 逐步调大实例并发度，如果性能仍然良好，则继续调大；如果性能不符合预期，则调小并发度。
