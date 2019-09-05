> 2019年 几乎所有 大厂和云厂商开始 all in kubernetes的一年，kubernetes 已然成为 下一代的 "操作系统"，基于此平台 推出的 istio以及knative更是未来云计算的明天。
所以 kubernetes的重要程度不言而喻，从此章开始我们共同来 了解学习下 kubernetes的基本概念、运行原理、体系架构、以及 周边生态等等，由于kubernetes体系是一个非常庞大的生态体系，真是 路漫漫其修远兮，吾将上下而求索~

# kubernetes简介
kubernetes是 谷歌基于 自家 Borg/Omega 推出的 开源 容器编排项目，站在巨人的肩膀上的 kubernetes 横空出世 便 一路 披荆斩棘，而 竞品 Swarm Mesos 相继倒下；
在 2017年 kubernetes 就已经成了 容器编排的 事实标准了，具体的 容器编排 历史大家可以自行搜索，我这里就不再赘述了。

> kubernetes 本意是舵手，docker 中意是 码头工人，container是集装箱，取名就很好表达了 kubernetes的 设计理念：
>
> 旨在提供 “跨主机集群的自动部署、扩展以及运行应用程序容器的管理平台”；它支持一系列容器工具, 包括 Docker、rkt（已挂）、containerd（比docker更安全 也更轻量）等。

> 大家一般简称它为 k8s （kubernetes 英文单词 k到s之间有8个 字母）

# kubernetes 背景
我们通过之前的 docker 学习 清楚的了解到，docker 容器技术 为广大开发者们 提供一个 统一的 自动装箱的 沙盒技术，使得 应用程序的 集成发布以及 迁移部署变得 轻而易举，docker 的确 给 后端技术 带来了集装箱式的 改革。

docker 只是 解决了打包发布 环境一致性的 难题，并没有解决 我们 实际 希望 应用系统 能以
集群的方式 被合理 调度 、水平扩展、自我修复并提供 服务发现、负载均衡、路由网关等高级特性，显然 这一切的 需求 单靠 docker 是满足不了的，也就是在这需求下，容器编排 就应运而生了；

彼时的docker公司发展如日中天，加紧了 商业化的步伐推出了 docker Swarm，这无疑是触动了云计算领域大佬们的蛋糕，Google 为了 对抗 docker公司 商业化步伐，牵手 Redhat 推了 容器编排平台 -- kubernetes。

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

![大概流程](http://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker-kubernetes/k8s03.png)

> 这里只是简单描述 大体流程，忽略了很多细节，让大家明白 各组件间 如何协作。

# kubernetes 基本概念
在 kubernetes 中 涉及了以前 你 可能从来 没有听闻的 概念，但 这些 涉及逻辑是 经过 Google 十几年甚至 几十年的经验沉淀，想做到 普适性，顶层涉及必须得先进，让我们一起来了解下
k8s中的基本概念，这里只需要大家 了解下即可，后续会有文章专门讲解一些核心 概念。

## Pod
Pod 可以理解为 容器组，由一个或多个容器组成，是在Kubernetes集群中运行部署应用或服务的最小单元，Pod中的所有容器共享网络地址和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。
比如 你运行一个mysql 容器，你还想对mysql做监控，部署了一个mysql的监控服务，那么你就可以 让这两个不同的容器 跑在一个 Pod，这样他们 协作起来更快速、也更安全。

> 你可以简单理解pod 就是一个轻量级 虚拟机。

![Pod](http://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker-kubernetes/k8s04.png)

## RC、RS、Deployment
RC副本控制器、RS副本集，都是控制 pod 的个数的控制器。单个部署pod，如果pod因为各种原因退出了，pod 自己是不会自动恢复的，一般 无状态（比如nginx、java业务应用等不需要持久保存数据、可以任意扩展的我们称为无状态）的 pod ，需要维护 pod状态和副本数，如果pod 发生异常，此控制器会kill异常pod并重新启动一个新的pod 以维持 所期望的pod数量。

Deployment 简单来说就是 基于RS 做版本控制，比如我发布了一个应用 pod，不单单需要维持pod 副本数，还得对进行版本控制，以方便 pod 进行更新或回滚，对于无状态应用 一般都用此控制器。可以把 Deployment 看做 RS 版本控制器，如果一个应用发布多次，会有多个版本的RS。

> RC 在新版 kubernetes中已经被 RS 替代

> 一般 我们不会直接用 RS 而是通过 Deployment 控制器 来控制 RS。

![RC-RS-Deployment](http://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker-kubernetes/k8s05.png)

## Label、Label Selector、Annotation

Label标签，在kubernetes中Label是一个极其重要的概念，一个Label是一个`key=value`的键值对，几乎所有的资源都能打上标签，用来做资源标识或分类。

Label Selector 标签选择器： 用来查询或筛选 指定Label的资源对象，类似RDS中的SQL一样。

RS 识别并控制其所属pod， 就是通过 Label 和Label Selector完成：在创建RS 时 会在 其创建的pod上打上 例如：release: nginx 的标签，后续 RS 通过  Label Selector机制定义 spec.selector.matchLabels: release: nginx，就可以完成对 所属pod进行数量控制了。

Annotation 注解，和label一样都是key/value键值对映射结构，可以给一些对象内部信息说明，一般都用来 做 配置信息。所谓的内部信息，指的是对这些信息感兴趣的，是Kubernetes组件本身，而不是用户。

比如我们 配置ingress时可以指定是用traefik还是nginx：metadata. annotations: kubernetes.io/ingress.class: "nginx"

## Service、Ingress

Service简单来讲就是 为pod通过服务入口并提供负载均衡。比如 一个java应用 以deployment方式 部署了多个 Pod，那 其他的应用或外部用户怎么来访问？这就需要一个统一的入口进行代理，并且还能做流量负载，在kubernetes中 Service就是起到这样的作用。

![Service](http://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker-kubernetes/k8s06.jpg)

Services 一般用于内部集群应用间调用，Ingress 用于集群内部服务以http协议方式暴露到外部。

单纯创建 Ingress 还不够，需要结合 Ingress controller 才能真正实现服务的外部暴露。当前 Ingress controller 有 [ingress-nginx](https://github.com/kubernetes/ingress-nginx)、[Traefik](https://github.com/containous/traefik) 等。

你可以简单理解 Ingress 就是一系列的规则，然后通过此规则解析成 nginx或traefik的配置文件从而定义访问路径。

```yam
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kube-dashboard
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - hosts:
    - dashboard.xx.cn
    secretName: dashboard-tls
  rules:
  - host: dashboard.xx.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```



## DaemonSet、StatefulSet

DaemonSet就是在kubernetes集群中每一个 node节点 有且只有一个 同类的 pod副本，一般是用来部署一些 日志、监控应用的 agent。

比如 elk中的 filebeat，我只需要在每一个node节点启一个就可以了；或者是 监控系统  Prometheus Node Exporter等

StatefulSet 是为了解决 有状态应用部署 而设计的控制器；所谓有状态应用就是pod数据需要持久化、需要有稳定唯一的网络标识以及有序的部署和扩缩容。常见的有状态应用比如 数据库 中间件等，他们启动前需要做一系列的初始化工作、关闭也需要提前做清理工作，尤其是以集群环境方式部署时关闭启动时有一定顺序的，更关键时 当pod 异常退出后 重新调度后 还需要访问到相同的持久化数据。而这一切的需求，只能通过StatefulSet 方式来进行部署。

## Volume、PV、PVC、StorageClass

Volume 跟容器的 Volume类似 都是用来持久保存数据，只不过kubernetes中Volume使用对象为 pod，当pod中的容器异常退出后，kubelet会启动一个新的容器用来代替，此时pod中的容器中之前的文件就会丢失，而且 pod内的多个容器有需要文件共享；Volume则可以很好的解决pod 内容器数据持久化和共享这两个问题。

PersistentVolume (PV) 和 PersistentVolumeClaim (PVC) ，简单来讲就是 PV 是kubernetes集群中提供实际存储资源，PV可以是NFS、rbd块存储等等，而 PVC 则是集群中 存储资源请求或申请。

比如一个 NFS 的 PV 可以定义为

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv01
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /tmp
    server: 172.17.0.2
```

就像 Pod 消耗Node节点资源，PVC消耗PV资源，Pod可以申请计算资源（CPU和内存），PVC也可以申请不同的存储资源（存储大小和访问模式）。

PV 的访问模式（accessModes）有三种：

- ReadWriteOnce（RWO）：是最基本的方式，可读可写，但只支持被单个节点挂载。
- ReadOnlyMany（ROX）：可以以只读的方式被多个节点挂载。
- ReadWriteMany（RWX）：这种存储可以以读写的方式被多个节点共享。不是每一种存储都支持这三种方式，像共享方式，目前支持的还比较少，比较常用的是 NFS。在 PVC 绑定 PV 时通常根据两个条件来绑定，一个是存储的大小，另一个就是访问模式。

PV 的回收策略（persistentVolumeReclaimPolicy，即 PVC 释放卷的时候 PV 该如何操作）也有三种

- Retain，不清理, 保留 Volume（需要手动清理）
- Recycle，删除数据，即 `rm -rf /thevolume/*`（只有 NFS 和 HostPath 支持）
- Delete，删除存储资源，比如删除 AWS EBS 卷（只有 AWS EBS, GCE PD, Azure Disk 和 Cinder 支持）

StorageClass 一般用来动态创建PV，当kubernetes 使用者需要一个块存储，如果集群管理员 用手工方式一个个去创建 PV，那将浪费大量时间，所以有了 StorageClass ，用来封装不同存储供 PVC 申请使用。

## ConfigMap、Secret

ConfigMap 用于保存pod 需要的 配置文件，一般来说 镜像 部署到不同环境，只需要改下配置文件即可，但不会说 在 kubernetes中的所有Node节点都放一份配置文件然后 挂载到 pod中，这是极其丑陋呆笨的方式，kubernetes提供了 统一配置文件存储 这就是 ConfigMap。它可以用来保存单个属性，也可以用来保存配置文件。

ConfigMap 一般是用来存储一些非敏感的配置信息，如果涉及到一些安全相关的敏感数据的话用`ConfigMap`就非常不妥了，因为 ConfigMap 明文存储的，这时候就需要 Secret了。

Secret 用来保存敏感信息，将这些信息放在Secret中比放在`Pod`的定义中或者`docker`镜像中来说更加安全和灵活。

Secret有三种类型：

- Opaque：base64 编码格式的 Secret，用来存储密码、密钥等；但数据也可以通过base64 –decode解码得到原始数据，所以其加密性很弱。
- kubernetes.io/dockerconfigjson：用来存储私有docker registry的认证信息。
- kubernetes.io/service-account-token：用于被`serviceaccount`引用，serviceaccout 创建时Kubernetes会默认创建对应的secret。Pod如果使用了serviceaccount，对应的secret会自动挂载到Pod目录`/run/secrets/kubernetes.io/serviceaccount`中。



## User Account、Service Account

都是用于账号认证，User Account用于 用户，Service Account用于Pod内进程。

## Job、CronJob
Job 见名知意，负责批量处理短暂的一次性任务 (short lived one-off tasks)，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束。

CronJob 即定时任务，就类似于 Linux 系统的 crontab，在指定的时间周期运行指定的任务。

## Namespace

kubernetes中的 资源隔离 就是用 Namespace 来完成，可以将 Namespace 来分割 不同的项目组、用户组或者 运行环境；比如 Kubernetes 自带的服务一般运行在 `kube-system` namespace 中，例如生产、测试、开发划分不同的namespace。

kubernetes安装后，有三个默认Namespace：
- default 没有其他命名空间的对象的默认命名空间。
- kube-system Kubernetes系统创建的对象的命名空间。
- kube-public此命名空间是自动创建的，并且所有用户（包括未经过身份验证的用户）都可以读取，该命名空间一般用来存放公共的信息。

常见的 pod, service, replication controller 和 deployment 等都是属于某一个 namespace 的（默认是 default），而 node, persistent volume，namespace 等资源则不属于任何 namespace。

要查看哪些Kubernetes资源在命名空间中，哪些不在：

```bash
# In a namespace
kubectl api-resources --namespaced=true

# Not in a namespace
kubectl api-resources --namespaced=fals
```

注意点：
1. 删除一个 namespace 会自动删除所有属于该 namespace 的资源。
2. `default` 和 `kube-system` 命名空间不可删除。
3. PersistentVolume 是不属于任何 namespace 的，但 PersistentVolumeClaim 是属于某个特定 namespace 的。
4. Event 是否属于 namespace 取决于产生 event 的对象。





