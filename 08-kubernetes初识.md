> 2019年 几乎所有 大厂和云厂商开始 all in kubernetes的一年，kubernetes 已然成为 下一代的 "操作系统"，基于此平台 推出的 istio以及knative更是未来云计算的明天。
所以 kubernetes的重要程度不言而喻，从此章开始我们共同来 了解学习下 kubernetes的基本概念、运行原理、体系架构、以及 周边生态等等，路漫漫其修远兮，吾将上下而求索~

# kubernetes简介
kubernetes是 谷歌基于 自家 Borg/Omega 推出的 开源 容器编排项目，站在巨人的肩膀上的 kubernetes 横空出世 便 一路 披荆斩棘，竞品 Swarm Mesos相继倒下，
在 2017年 kubernetes 就已经成了 容器编排的 事实标准了，具体的 容器编排 历史大家可以自行搜索，我这里就不再赘述了。

> kubernetes 本意是舵手，docker 中意是 码头工人，container是集装箱，取名就很好表达了 kubernetes的 设计理念：它旨在提供 “跨主机集群的自动部署、扩展以及运行应用程序容器的管理平台”。
>它支持一系列容器工具, 包括 Docker、rkt（已挂）、containerd（比docker更安全 也更轻量）等。

> 大家一般简称它为 k8s （kubernetes 英文单词 k到s之间有8个 字母）

# kubernetes 背景
我们通过之前的 docker 学习 清楚的了解到，docker 容器技术 为广大开发者们 提供一个 统一的 自动装箱的 沙盒技术，使得 应用程序的 集成发布以及 迁移部署
变得 轻而易举，docker 的确 给 后端技术 带来了集装箱式的 改革。 但 docker 只是 解决了打包发布 环境一致性的 难题，但并没有解决 我们 实际 希望 应用系统 能以
集群的方式 被合理 调度 、水平扩展、自我修复并提供 服务发现、负载均衡、路由网关等高级特性，显然 这一切的 需求 但靠 docker 是满足不了的，也就是在这
需求下，容器编排 就应运而生了；在此同时 docker公司发展如日中天，加紧了 商业化的步伐推出了 docker Swarm，这无疑是触动了云计算领域大佬们的蛋糕，Google 为了 对抗 docker公司 商业化步伐，牵手 Redhat 推了 容器编排平台 --kubernetes。

# Kubernetes的特点：
- 自动装箱。基于资源和依赖自动部署服务。
- 自我修复。当有一个容器挂了，能够自动启动一个新的同样服务替换故障的容器。
- 自动实现水平扩展。只要物理资源充足，设置触发阈值，可以实现自动实现水平伸缩服务。
- 自动服务发现和负载均衡。service能够提供稳定的访问端点和负载均衡。
- 自动发布和回滚。
- 秘钥和配置管理。
- 存储编排，存储卷动态供给。
- 批量处理执行。

# kubernetes 基本架构
Kubernetes项目 继承了谷歌内部 使用沉淀多年的Borg/Omega项目设计以及理论，在 与其他容器平台竞争中有着天然的优势，并且在社区的 推动下 不断的改进 和规避了 以前 Borg/Omega 很多缺陷和问题，那让我们来看下kubernetes 整体架构：
![整体架构](http://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker-kubernetes/k8s01.png)
![基本组成](http://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker-kubernetes/k8s02.png)

通过上面的 两张图，我们能了解到 kubernetes 集群 节点 分两个角色：
- master：即控制节点，是kubernetes的大脑，负责管理整个集群，提供 集群资源操作入口，并控制集群中的所有活动，例如容器的调度、容器状态的维护等，
核心组件有：kube-apiserver、kube-scheduler，kube-controller-manager，如果为 云厂商kubernetes还会有一个cloud-controller-manager。
- node：即负载节点，应用容器真正运行所在节点，核心组件为 kubelet、kube-proxy、container runtime。
- 集群数据存储：etcd，一般部署在master节点，集群数量必须为奇数。

下面就简单的说下这些组件的核心功能：
master角色：
- kube-apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- kube-controller-manager 负责维护集群的状态，实现 任务调度，比如deployment控制器、故障检测、自动扩展、滚动更新等；
- kube-scheduler 负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- cloud-controller-manager，为了使 kubernetes更好的跟 云厂商 生态做集成，独立出来的组件，目的是 让云服务商相关的代码和K8S核心解耦，相互独立演进，统一负责 跟云服务进行交互。 
- etcd 存储Kubernetes 的集群状态的，它除了具备状态存储的功能，还有事件监听和订阅、Leader选举的功能；
> 所谓事件监听和订阅，各个其他组件通信，都并不是互相调用 API 来完成的，而是把状态写入 ETCD（相当于写入一个消息），其他组件通过监听 ETCD 的状态的的变化（相当于订阅消息），然后做后续的处理，然后再一次把更新的数据写入 ETCD。
> 所谓 Leader 选举，其它一些组件比如 Scheduler，为了做实现高可用，通过 ETCD 从多个（通常是3个）实例里面选举出来一个做Master，其他都是Standby。

node角色：
- kubelet 可以看做一个agent，负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
- container runtime 负责镜像管理以及Pod和容器的真正运行（CRI），一般来说就是docker 或者 containerd；
- kube-proxy 负责为service提供cluster内部的服务发现和负载均衡；
- kubectl kubernetes原生的 命令行工具，通过 Kubectl 命令对 API Server 进行操作，实现对 kubernetes集群管理工作。

addons：
包括 coredns负载集群内的dns解析；flannel/calico网络插件，提供跨节点的网络访问，这两个addons是集群必需，其他的非必需addons 后面再进行介绍。


# kubernetes工作流程
现在我们了解了每个组件的核心功能，这里假设使用 kubectl 创建一个多实例的 nginx pod（容器组，后面进行解释） 运行在kubernetes上，大概的工作流程是怎么样的呢？

通过kubectl命令行 kubectl run nginx --image=nginx --replicas=2 ，创建一个包含nginx的deployment对象，kubectl会调用 apiserver 往etcd里面写入一个 deployment对象。

Deployment Controller 监听到有新的 Deployment对象被写入，就获取到对象信息，根据对象信息来做任务调度，创建对应的 Replica Set 对象。

Replica Set Controller监听到有新的对象被创建，也读取到对象信息来做任务调度，创建对应的Pod来。

Scheduler 监听到有新的 Pod 被创建，读取到Pod对象信息，根据集群状态将Pod调度到某一个节点上，然后更新Pod（内部操作是将Pod和节点绑定）。

Kubelet 监听到当前的节点被指定了新的Pod，就根据对象信息运行Pod。 

> 这里只是简单描述 大体流程，忽略了很多细节，让大家明白 各组件间 如何协作。

# kubernetes 基本概念
在 kubernetes 中 涉及了以前 你 可能从来 没有听闻的 概念，但 这些 涉及逻辑是 经过 Google 十几年甚至 几十年的经验沉淀，想做到 普适性，顶层涉及必须得先进。
kubernetes中 

## Pod
Pod 可以理解为 容器组，由一个或多个容器组成，是在Kubernetes集群中运行部署应用或服务的最小单元，Pod中的所有容器共享网络地址和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。
比如 你运行一个mysql 容器，你还想对mysql做监控，部署了一个mysql的监控服务，那么你就可以 让这两个不同的容器 跑在一个 Pod，这样他们 协作起来更快速、也更安全。


## RC/RS




## Deployment
## Namespace
## Service
## DaemonSet
## StatefulSet
## User Account、Service Account
## Job
## Volume
## PV、PVC
## Configmap、Secret






