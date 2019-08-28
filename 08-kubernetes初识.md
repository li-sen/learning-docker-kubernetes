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