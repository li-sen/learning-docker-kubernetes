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
- kube-apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- kube-controller-manager 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- kube-scheduler 负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- cloud-controller-manager，为了使 kubernetes更好的跟 云厂商 生态做集成，独立出来的组件，目的是 让云服务商相关的代码和K8S核心解耦，相互独立演进，统一负责 跟云服务进行交互。 
- etcd 一个基于raft协议实现高可用的 Key/Value 存储系统，它通过kube-apiserver保存了整个集群的状态；

- kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
- container runtime 负责镜像管理以及Pod和容器的真正运行（CRI），一般来说就是docker 或者 containerd；
- kube-proxy 负责为Service提供cluster内部的服务发现和负载均衡；

- kubectl kubernetes原生的 客户端工具，通过 Kubectl 命令对 API Server 进行操作，实现对 kubernetes集群管理工作。
- addons： 包括 coredns负载集群内的dns解析；flannel/calico网络插件，提供跨节点的网络访问，这两个addons是集群必需，其他的非必需addons 后面再进行介绍。


这里详细讲下etcd，以及kubernetes为什么使用它存储集群状态：

首先etcd 是一个基于raft协议实现高可用的 Key/Value 存储系统，并且有一个特写就是 消息发布与订阅，可以调用它的api监听其中的数据，一旦数据发生变化了，就会收到通知。有了这个特性之后，kubernetes中的每个组件只需要监听etcd中数据，就可以知道自己应该做什么。

在kubernetes中，集群的数据是随时发生变化的，比如说用户提交了新任务、增加了新的Node、Node宕机了、容器死掉了等等，都会触发状态数据的变更，状态数据变更之后呢，都需要通知 kubernetes上的 一系列组件 进行相应的动作来保证集群状态健康，
这些特性 etcd 完美适配，

试想一下，如果没有etcd，那么要怎样做？
一种是消息的方式，比如说NodeA有了新的任务，Master直接给NodeA发一个消息；一种是轮询的方式，大家都把数据写到同一个地方，每个人自觉地盯着看，及时发现变化。
前者演化出rabbitmq这样的消息队列系统，后者演化出一些有订阅功能的分布式系统。

第一种方式的问题是，所有要通信的组件之间都要建立长连接，并且要处理各种异常情况，比例如连接断开、数据发送失败等。不过有了消息队列(message queue)这样的中间件之后，问题就简单多了，组件都和mq建立连接即可，将各种异常情况都在mq中处理。

那么为什么kubernetes没有选用mq而是选用etcd呢？mq和etcd是本质上完全不同的系统，mq的作用消息传递，不储存数据（消息积压不算储存，因为没有查询的功能），etcd是个分布式存储（它的设计目标是分布式锁，顺带有了存储功能），是一个带有订阅功能的key-value存储。如果使用mq，那么还需要引入数据库，在数据库中存放状态数据。

选择etcd还有一个好处，etcd使用raft协议实现一致性，它是一个分布式锁，可以用来做选举。如果在kubernetes中部署了多个kube-schdeuler，那么同一时刻只能有一个kube-scheduler在工作，否则各自安排各自的工作，就乱套了。怎样保证只有一个kube-schduler在工作呢？通过etcd选举出一个leader。

需要注意，虽然etcd可以存储数据，但不要滥用，它运行raft协议保证整个系统中数据一致，因此写入、读取速度都是很慢的。kubernetes早期支持的Node数量有限，就是因为etcd是瓶颈。etcd主要在一些分布式系统中提供选举功能。


现在我们了解了每个组件的核心功能，这里假设使用 kubectl 创建一个 nginx pod（容器组，后面进行解释） 运行在kubernetes上，大概的工作流程是怎么样的呢？

流程图是这样：

![k8s工作流程](http://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker-kubernetes/k8s03.png)

用户使用  kubectl run nginx --image=nginx命令  通过 kube-apiserver REST API 创建一个 Pod

kube-apiserver 将其写入 etcd

kube-scheduluer 监听etcd 收到任务，根据pod 要求选取 node 节点，开始调度并更新 Pod 的 Node 绑定

kubelet 检测到有新的 Pod 调度过来，通过 container runtime 运行该 Pod

kubelet 通过 container runtime 取到 Pod 状态，并更新到 apiserver，写入etcd中 






