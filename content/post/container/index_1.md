---
title: "containerd 简介"
description: 
date: 2023-04-01T10:47:41+08:00
slug: rb2G925BJdW
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:
  - containerd
  - Kubernetes
  - 容器
categories:
  - 容器
---

官网： https://containerd.io/

用途： 搭配 `Kubernetes` 为集群提供容器运行时服务

### containerd 的前世今生

很久以前，Docker 强势崛起，以“镜像”这个大招席卷全球，对其他容器技术进行致命的降维打击，使其毫无招架之力，就连 Google 也不例外。Google 为了不被拍死在沙滩上，被迫拉下脸面（当然，跪舔是不可能的），希望 Docker 公司和自己联合推进一个开源的容器运行时作为 Docker 的核心依赖，不然就走着瞧。Docker 公司觉得自己的智商被侮辱了，走着瞧就走着瞧，谁怕谁啊！
很明显，Docker 公司的这个决策断送了自己的大好前程，造成了今天的悲剧。
紧接着，Google 联合 Red Hat、IBM 等几位巨佬连哄带骗忽悠 Docker 公司将 libcontainer 捐给中立的社区（OCI，Open Container Intiative），并改名为 runc，不留一点 Docker 公司的痕迹~~
这还不够，为了彻底扭转 Docker 一家独大的局面，几位大佬又合伙成立了一个基金会叫 CNCF（Cloud Native Computing Fundation），这个名字想必大家都很熟了，我就不详细介绍了。CNCF 的目标很明确，既然在当前的维度上干不过 Docker，干脆往上爬，升级到大规模容器编排的维度，以此来击败 Docker。
Docker 公司当然不甘示弱，搬出了 Swarm 和 Kubernetes 进行 PK，最后的结局大家都知道了，Swarm 战败。然后 Docker 公司耍了个小聪明，将自己的核心依赖 containerd 捐给了 CNCF，以此来标榜 Docker 是一个 PaaS 平台。
很明显，这个小聪明又大大加速了自己的灭亡。

![](/img/img.png)

巨佬们心想，想当初想和你合作搞个中立的核心运行时，你死要面子活受罪，就是不同意，好家伙，现在自己搞了一个，还捐出来了，这是什么操作？也罢，这倒省事了，我就直接拿 containerd 来做文章吧。
首先呢，为了表示 Kubernetes 的中立性，当然要搞个标准化的容器运行时接口，只要适配了这个接口的容器运行时，都可以和我一起玩耍哦，第一个支持这个接口的当然就是 containerd 啦。至于这个接口的名字，大家应该都知道了，它叫 CRI（Container Runntime Interface）。
这样还不行，为了蛊惑 Docker 公司，Kubernetes 暂时先委屈自己，专门在自己的组件中集成了一个 shim（你可以理解为垫片），用来将 CRI 的调用翻译成 Docker 的 API，让 Docker 也能和自己愉快地玩耍，温水煮青蛙，养肥了再杀。。。
就这样，Kubernetes 一边假装和 Docker 愉快玩耍，一边背地里不断优化 containerd 的健壮性以及和 CRI 对接的丝滑性。现在 containerd 的翅膀已经完全硬了，是时候卸下我的伪装，和 Docker say bye bye 了。后面的事情大家也都知道了~~
Docker 这门技术成功了，Docker 这个公司却失败了。

### containerd 架构

时至今日，containerd 已经变成一个工业级的容器运行时了，连口号都有了：超简单！超健壮！可移植性超强！
当然，为了让 Docker 以为自己不会抢饭碗，containerd 声称自己的设计目的主要是为了嵌入到一个更大的系统中（暗指 Kubernetes），而不是直接由开发人员或终端用户使用。
事实上呢，containerd 现在基本上啥都能干了，开发人员或者终端用户可以在宿主机中管理完整的容器生命周期，包括容器镜像的传输和存储、容器的执行和管理、存储和网络等。大家可以考虑学起来了。
先来看看 containerd 的架构：

![](/img/img_1.png)

可以看到 containerd 仍然采用标准的 C/S 架构，服务端通过 GRPC 协议提供稳定的 API，客户端通过调用服务端的 API 进行高级的操作。
为了解耦，containerd 将不同的职责划分给不同的组件，每个组件就相当于一个子系统（subsystem）。连接不同子系统的组件被称为模块。
总体上 containerd 被划分为两个子系统：
Bundle : 在 containerd 中，Bundle 包含了配置、元数据和根文件系统数据，你可以理解为容器的文件系统。而 Bundle 子系统允许用户从镜像中提取和打包 Bundles。
Runtime : Runtime 子系统用来执行 Bundles，比如创建容器。
其中，每一个子系统的行为都由一个或多个模块协作完成（架构图中的 Core 部分）。每一种类型的模块都以插件的形式集成到 containerd 中，而且插件之间是相互依赖的。
例如，上图中的每一个长虚线的方框都表示一种类型的插件，包括 Service Plugin、Metadata Plugin、GC Plugin、Runtime Plugin 等，其中 Service Plugin 又会依赖 Metadata Plugin、GC Plugin 和 Runtime Plugin。每一个小方框都表示一个细分的插件，例如 Metadata Plugin 依赖 Containers Plugin、Content Plugin 等。 
总之，万物皆插件，插件就是模块，模块就是插件。

![](/img/img_2.png)

这里介绍几个常用的插件：

Content Plugin : 提供对镜像中可寻址内容的访问，所有不可变的内容都被存储在这里。
Snapshot Plugin : 用来管理容器镜像的文件系统快照。镜像中的每一个 layer 都会被解压成文件系统快照，类似于 Docker 中的 graphdriver。
Metrics : 暴露各个组件的监控指标。
从总体来看，containerd 被分为三个大块：Storage、Metadata 和 Runtime，可以将上面的架构图提炼一下：

![](/img/img_3.png)

这是使用 bucketbench 对 Docker、crio 和 containerd 的性能测试结果，包括启动、停止和删除容器，以比较它们所耗的时间：

![](/img/img_4.png)

可以看到 containerd 在各个方面都表现良好，总体性能还是优越于 Docker 和 crio 的。