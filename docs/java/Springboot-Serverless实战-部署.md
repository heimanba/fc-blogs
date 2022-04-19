---
title: Springboot Serverless 实战 - 部署
description: 'Springboot Serverless 实战 - 部署'
position: 12
category: 'FC_Blog'
---

作者 | 西流

[SpringBoot](https://spring.io/projects/spring-boot) 是基于 Java Spring 框架的套件，它预装了 Spring 的一系列组件，让开发者只需要很少的配置就可以创建独立运行的应用程序。在云原生的世界，有大量的平台可以运行 SpringBoot 应用，例如虚拟机，容器等。但其中最有吸引力的，是以 Serverless 的方式运行 SpringBoot 应用。我将通过一系列文章，从架构，部署，监控、性能、安全等5个方面来分析 Serverless 平台运行 SpringBoot 应用的优劣。为了让分析更有代表性，我选择了 github 上 star 数超过 50k 的电商应用 [mall](https://github.com/macrozheng/mall) 作为示例。这是系列文章的第二篇， 将 mall 应用部署到函数计算平台上。

在阅读本文之前，请参考“SpringBoot on FC - 架构”文章，对 mall 应用架构以及 Serverless 平台有一个基本的了解。

## 0. 前置条件

- 您需要有阿里云的账户
- 您需要有一台能通过公网 ip 访问的机器，安装 MySQL，Redis 等 mall 应用依赖的软件。
- 您需要在运行依赖软件的机器上安装 git， docker，java 和 maven软件
- 您需要[安装并配置](http://serverless-devs.com/zh-cn/docs/installed/cliinstall.html) Serverless Devs 工具

注意，如果您使用了云主机，请先检查主机对应的安全组配置是否允许入方向的网络请求。一般的主机在创建后，对于入方向的网络端口访问做了严格限制。我们需要手动允许访问 MySQL 的 3306 端口，Redis 的 6379 端口等。如下图所示，我手动设置了安全组，允许所有入方向的网络请求。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635933259703-b4d900d9-127b-49a8-8256-e440cd43bda7.png#crop=0&crop=0&crop=1&crop=1&id=j3k5E&originHeight=1406&originWidth=2872&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

## 1. 部署依赖软件

Mall 应用依赖 MySQL，Redis，MongoDB，ElasticSearch，RabbitMQ 等软件。这些软件在云上都有对应的云产品。在生产环境，推荐使用云产品获得更好的性能和可用性。在个人开发或者 POC 原型演示场景下，我们选择一台 VM 来容器化部署所有依赖的软件。

### 1.1. Clone 代码仓库

```shell
git clone https://github.com/hryang/mall
```

国内访问 github 网络不太好，如果 clone 太慢，可使用 gitee 地址。

```shell
git clone https://gitee.com/aliyunfc/mall.git
```

### 1.2. 构建和运行 docker 镜像

在代码根目录的 `docker` 文件夹下，有每个依赖软件对应的 dockerfile。运行代码根目录下的 `run.sh`脚本，会自动构建所有依赖软件的 docker 镜像，并在本机运行。

```shell
sudo bash docker.sh
```

### 1.3. 验证依赖软件运行状态

执行 docker ps 命令，检查依赖软件是否正常运行。

```shell
sudo docker ps
```

## 2. 部署 mall 应用

### 2.1. 修改 mall 应用配置

修改 `mall-admin/src/main/resources/application-prod.yml`，`mall-portal/src/main/resources/application-prod.yml`，`mall-search/src/main/resources/application-prod.yml` 3个yaml 文件，将其中的 `host` 字段改成您第1步安装 MySQL 等软件的节点的公网 ip。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635584433134-1e2c6ca4-bf79-4249-8c91-29c907fb5683.png#crop=0&crop=0&crop=1&crop=1&id=gd767&originHeight=882&originWidth=1928&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

### 2.2. 生成 mall 应用容器镜像

执行 maven 打包命令，生成 docker 镜像。
> 本地是 java8 或者 java11 环境均可

```shell
sudo -E mvn package
```

成功后，将显示如下成功信息。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635584822610-fee1412f-74b8-497e-8e8d-ae38032ffbfe.png#crop=0&crop=0&crop=1&crop=1&id=YlNbz&originHeight=654&originWidth=1730&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

执行 `sudo docker images`，应该能看到 mall/mall-admin，mall/mall-portal，mall/mall-search 的 1.0-SNAPSHOT 版本的镜像。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635584979178-f3472300-dd5b-4c49-8c21-449cd4aec882.png#crop=0&crop=0&crop=1&crop=1&id=m0rAS&originHeight=194&originWidth=2006&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

### 2.3. 将镜像推送到阿里云镜像仓库

首先登录阿里云镜像仓库控制台，选择个人版实例，根据提示让 docker 登录阿里云镜像仓库。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635645578826-66c185c4-f2ad-4735-94cf-387f449e085d.png#crop=0&crop=0&crop=1&crop=1&id=BXTKP&originHeight=928&originWidth=2832&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

然后创建命名空间。如下图所示，我们创建了名为 `quanxi-hryang` 的命名空间。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635646107309-d30eb097-17c4-4124-9ae7-3c4591e7caff.png#crop=0&crop=0&crop=1&crop=1&id=Tj6o0&originHeight=720&originWidth=2860&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

根据之前的步骤，我们已经在本地生成了 mall/mall-admin, mall/mall-portal, mall/mall-search 的镜像。执行下面的命令，将 mall-admin 镜像推送到杭州区域，`quanxi-hryang` 命名空间下的镜像仓库。请将下面命令中的 `cn-hangzhou` 和 `quanxi-hryang` 修改为您自己的镜像仓库地域和命名空间。mall/mall-portal，mall/mall-search 以此类推。

```shell
sudo docker tag mall/mall-admin:1.0-SNAPSHOT registry.cn-hangzhou.aliyuncs.com/quanxi-hryang/mall-admin:1.0-SNAPSHOT

sudo docker push registry.cn-hangzhou.aliyuncs.com/quanxi-hryang/mall-admin:1.0-SNAPSHOT
```

### 2.4. 修改 Serverless Devs 工具的应用定义

我们使用 Serverless Devs 工具来定义和部署应用。在项目根目录下，有 `s.yaml`文件，这是 Serverless Devs 工具的项目定义文件。这里面定义了函数计算的资源。如下图所示，我们在函数计算上定义了名为 mall-admin 服务及其下的 mall-admin 函数。函数中定义了 port，内存大小，超时时间，运行时等属性。红框中的内容是您需要根据自己的配置修改的。

- `access` 是您使用 `s config` 配置的身份，默认是 default。如果您采用默认设置，那么这里不需要更改
- `region` 是您要部署的地域，有 cn-hangzhou，cn-shanghai，cn-beijing，cn-shenzheng 等选项。
- 函数使用了 custom-container 运行时，需要指定镜像地址。请将 s.yaml 中的镜像地址改为您上一步推送的 mall-admin 镜像地址。同理，也需要在 s.yaml 中更改 mall-portal，mall-search 的镜像地址。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635935496224-6a64e56a-064a-437d-86d5-a34e7fd28a0d.png#crop=0&crop=0&crop=1&crop=1&id=hGc56&originHeight=2600&originWidth=1866&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
> 建议：上面的镜像地址最好使用  registry-vpc.cn-hangzhou.aliyuncs.com/fc-demo/mall-admin:1.0-SNAPSHOT 这么形式

### 2.5. 部署 mall 应用到函数计算平台

执行 `s deploy` 命令，部署成功后，您将看到对应的访问网址。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635651108855-d720bf44-08df-46ef-99e0-5a75a4809280.png#crop=0&crop=0&crop=1&crop=1&id=dqdl5&originHeight=916&originWidth=1566&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

在浏览器中输入生成的网址，如果显示 “暂未登录或token已经过期”，说明服务部署成功。

> 注意：Serverless 的特点是系统默认在请求到达后才创建实例，所以第一次启动时间比较长，称之为冷启动。Mall 应用启动一般需要30秒左右。后面我们将在性能调优文章中来回顾这个问题，使用一系列手段优化。


![](/Users/hryang/Documents/%E5%87%BD%E6%95%B0%E8%AE%A1%E7%AE%97/%E4%BA%A7%E5%93%81%E8%BF%90%E8%90%A5/image-20211031113426539.png#crop=0&crop=0&crop=1&crop=1&id=i6Ety&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

访问对应的 swagger api 调试页面 host/swagger-ui.html，就能调试相关的后端 API 了。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635651600454-0598ac3b-1b53-4d0c-89eb-fdd60e557523.png#crop=0&crop=0&crop=1&crop=1&id=ZI2LD&originHeight=1344&originWidth=2878&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

### 2.6. 查看应用日志

我们在 `s.yaml` 中为每个服务都设置了 `logConfig:auto`，代表 serverless-devs 工具会自动为服务创建日志库（LogStore），所有的服务都共享一个日志库。应用所有的日志都输出到。您可以使用 `s logs` 命令查看所有服务某个时间点的日志；也可以使用 `s mall-admin logs` 查看 mall-admin 函数的日志；也可以使用 `s mall-admin logs -t`以跟随模式实时显示当前时间点之后的日志；也可以使用 `s mall-admin logs --keyword=abc` 查看包含关键词 abc 的日志。s logs 对于您了解服务运行情况和问题诊断非常有用。例如我们执行 `s mall-admin logs -t` 进入跟随模式，然后在浏览器中访问 `mall-admin` 服务的 endpoint，就能看到整个应用的启动和请求处理日志。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635937205843-d578e89a-088c-4199-9b83-fe2625cb6f15.png#crop=0&crop=0&crop=1&crop=1&id=ww5w4&originHeight=1366&originWidth=2098&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

### 2.7. 部署 mall 前端项目（可选）

Mall 也提供了一个前端界面，基于 Vue+Element 实现。主要包括商品管理、订单管理、会员管理、促销管理、运营管理、内容管理、统计报表、财务管理、权限管理、设置等功能。该项目同样可以无缝运行在函数计算上。

首先在所在机器上安装 nodejs12 和 npm，并下载项目源代码。

```shell
git clone https://github.com/hryang/mall-admin-web
```

国内访问 github 网络不太好，如果 clone 太慢，可使用下面的代理地址。

```shell
git clone https://gitee.com/aliyunfc/mall-admin-web.git
```

> 注意：**必须是 nodejs 12 或者 14**，太新的 node 版本会编译失败！


修改 `config/prod.env.js`，将其中的 BASE_API 改为之前在函数计算上部署成功的 mall-admin 的 endpoint。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635679887062-7ce95c5d-9828-4b61-bc20-c42a3b560a14.png#crop=0&crop=0&crop=1&crop=1&id=uRgaC&originHeight=504&originWidth=2042&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

在项目根目录执行下面的命令，构建前端项目。

```shell
npm install
npm run build
```

运行成功后，会生成 dist 目录。运行项目根目录下的 `docker.sh` 脚本，生成镜像。

```shell
sudo bash docker.sh
```

运行 `docker images` 命令，将看到 mall/mall-admin-web 镜像已经成功生成了。将镜像推送到阿里云镜像仓库。同理，请将下面命令中的 `cn-hangzhou` 和 `quanxi-hryang` 修改为您自己的镜像仓库地域和命名空间。

```shell
sudo docker tag mall/mall-admin-web:1.0-SNAPSHOT registry.cn-hangzhou.aliyuncs.com/quanxi-hryang/mall-admin-web:1.0-SNAPSHOT

sudo docker push registry.cn-hangzhou.aliyuncs.com/quanxi-hryang/mall-admin-web:1.0-SNAPSHOT
```

修改项目根目录下的 `s.yaml` ，和部署 mall-admin 类似，根据您的配置调整 `access`，`region`，将 `image` 改为上一步推送成功的镜像地址。

![](https://cdn.nlark.com/yuque/0/2021/png/995498/1635944297884-c8b02c31-29c7-4d5b-af65-7b179cc42439.png#crop=0&crop=0&crop=1&crop=1&id=NzBA1&originHeight=2266&originWidth=1866&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

执行 `s deploy`，当部署成功后，就能看到 mall-admin-web 服务的网址。通过浏览器访问，将看到登录页面。填入密码 `macro123`，就能看到完整的效果。

> 注意：第一次由于冷启动，登录页面可能会报超时错误。重新刷新页面即可。我们将在后面的性能调优文章中优化冷启动性能。


## 3. 总结

由于 Serverless 平台内置了网关，负责路由，实例拉起/运行/容错/自动扩缩容等功能，因此开发者上传应用代码包或者镜像后，就已经发布了一个弹性高可用的服务。总结起来，只要完成下面5步就在函数计算平台上完整部署了 mall 应用。后续对应用的更新，只需要重复步骤4和5。可见，Serverless 将环境配置和运维等重复性的工作免除了，开发运维效率大幅提升。

1. Clone 项目代码
1. 找到一台 VM，运行脚本一键式安装 MySQL，Redis 等依赖软件
1. 修改应用配置中 `host` 这一项，将值填写为步骤2中的 VM 公网 ip
1. 生成应用镜像，并推送到阿里云镜像仓库
1. 部署应用到函数计算平台
