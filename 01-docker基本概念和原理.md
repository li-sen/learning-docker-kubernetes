# 什么是docker

> 先说一个使用docker的场景：要部署运行一个tomcat应用

**传统方式**：
```text
1. 准备基础系统，针对tomcat优化系统环境
2. 下载jdk的，安装配置jdk
3. 下载tomcat 安装配置tomcat 。。。
4. 拷贝war包 运行应用
```
> 不同的人有不同的习惯，如果没有很好的规范化操作手册，每个人的部署tomcat环境
可能jdk版本、端口配置、基本参数、甚至存放路径都不一样，无法标准化，尤其是A安装的应用，
如果让B来管理，他肯定无从下手。

**容器方式**：
```text
1. 准备基础系统，安装Docker
2. docker run -v /xxx/xx.war:/usr/local/tomcat/webapps -p 8080:8080 tomcat:8 
```
> 有了docker，在应用部署时，无需关系底层系统到底的 centos 还是 ubantu，应用环境的版本是不是一致，只要有docker，一条命令就完成了部署！！而快速部署这仅仅是docker魅力之一，下面就来简单的说说docker。

docker 中文意思为 码头工人，把应用当集装箱的概念, 它其实也是虚拟化的一种实现，只不过是更轻量级的容器虚拟化；主要利用linux内核中的Namespaces、Cgroup以及chroot等技术实现的 沙盒环境，进行内核级别的资源隔离以及限制。

简单的来说 docker容器技术就是将应用以及对应的环境环境依赖（基本的操作系统，以及运行环境）打成一个包，这个包就叫做镜像，把镜像
扔进docker引擎中就可以直接运行了，比如将mysql跟centos打成一个镜像，在有docker引擎的机器上，直接docker run就完成mysql的部署了

---
# docker 实现原理
docker 其实是一种轻量级的虚拟化技术，尽管这样说不够严谨，但 可以让大家更好理解；docker 利用了 linux 容器技术 技术实现 的沙盒 环境， 譬如 我们 一台centos的宿主机上 docker run -it busybox /bin/sh 
这条命令的含义就是 启动一个 busybox 的容器， 在容器里执行 /bin/sh， 并提供一个终端给我跟容器进行交互。

当我们在容器里 运行 ps 命令时， 会发现 容器运行的 /bin/sh 竟然是 第一号进程， 而这 容器里面只有 两个进程运行：
```bash
/ # ps
PID  USER   TIME COMMAND
  1 root   0:00 /bin/sh
  10 root   0:00 ps

```
这就说明了 容器 已经跟 宿主机 隔离在不同的 "世界"中了， 这就是靠 PID Namespace 这种障眼法 来实现的，而操作系统 提供了 不同的namespace 进行 资源隔离。

## namespace、cgroup
关于namespace以及cgroup 做下简单说明，如果需要深入 请自行谷歌

- **namespace** ：简单来讲就是可以实现每个容器只能看到和使用它自己的资源(资源隔离)。

namespace|内核版本|隔离内容|隔离效果
---|---|---|---
Mount|2.4.19|文件系统挂接点|每个容器能看到不同的文件系统层次结构
UTS|2.6.19|主机名和域名|每个容器可以有自己的 hostname 和 domainame
IPC|2.6.19|信号量、消息队列和共享内存|只有在同一个 IPC namespace 的进程之间才能互相通信
PID|2.6.24|进程ID|每个容器有独立的PID
Network|2.6.29|网络相关的系统资源|每个容器用有其独立的网络设备，IP地址，IP路由表，端口号等等
User|3.8|用户和用户组|每个容器有独立的用户和用户组 
> centos 6 基本就不能用了，其默认内核为2.6 比较老， USER 3.8才支持，即使内核升级也会不稳定；默认docker也适合不支持32位系统，所以使用docker推荐使用centos 7.4_x64
最好将内核升级到4.x 。

- **cgroup** : 通过 cgroup 来控制容器使用的资源配额，包括 CPU、内存、磁盘三大方面，基本覆盖了常见的资源配额和使用量控制。

子系统|说明
---|---
blkio|限制每个块设备的输入输出。
cpu|为每个进程组设置一个使用CPU的权重值，以此来管理进程对cpu的访问。
cpuacct|CPU资源使用报告。
cpuset|多处理器平台上的CPU集合。
devices|设备访问，可允许或者拒绝 cgroup 中的任务访问设备。
freezer|挂起或恢复任务。
memory|内存用量限制以及生成内存资源报告。
net_cls|提供对网络带宽的访问限制，比如对发送带宽和接收带宽进程限制。

还有其他子系统，这里就不一一列举了。

docker 或者容器 在 操作系统层 来看就是 一种 特殊 进程， 只是 这个进程 通过 linux容器相关技术 进行资源隔离、限制， 虚拟的一种沙盒或沙箱 环境，使 不同的应用程序 互不干扰，和谐相处。

---
# docker vs vm
docker 也是虚拟化技术的一种，不过它比传统的vm更加轻量、快速，
更关键的是他能提供一个统一的、可控的软件环境，有点类似jvm 跨平台一样，一次编译到处运行；这使得软件的开发以及交付 对于vm是一种质飞跃。

# 两者对比图：
## 架构对比
![架构对比](https://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker/container_vm.jpeg)
    
> 很明显docker 与 vm 最大的区别是: 
vm 需要先去虚拟一堆硬件，然后在此之上新建一个os，还得去部署运行环境才能运行应用，
docker 直接通过 LXC 与 宿主机 共享内核 在操作系统级别进行虚拟，隔离 ，下镜像后直接运行应用，而且性能接近原生。

## 容器优势
特性|docker|vm
---|---|---
启动速度|秒级|分钟级别
硬盘使用|MB|GB
性能|接近原生|有一定的性能损失
系统支持量|单机上千|单机几十
隔离性|内核|OS
> 从很多方面来说docker 对比 vm 几乎是碾压的优势，只是在安全隔离方面弱于vm，不过k8s/docker生态越来越完善的趋势下，
这一问题肯定会得到很好的解决，瑕不掩瑜。

## 一个容器一个程序（进程）
容器有个默认规范或约定，就是一个容器 只运行一个程序或进程，这是一个很nice的设计哲学，这体现在：

1. 安全隔离：一个程序用一个容器，挂了、被入侵了可以很好的隔离，对其他程序无影响

2. 最小粒度控制资源:
    - 最少化使用资源：哪些程序需要资源多，提高对应容器的资源即可，不用一锅粥的提升 
    - 更好的管理： 比如扩缩容nginx 副本数，只要对nginx容器进行副本数增加即可；还可以对应用进行版本管理，更好做金丝雀部署等等。
    
这些都是一个容器跑多个程序做不到，不过这仅仅是我个人的一些见解。

---
# docker基本概念
其实容器虚拟化技术很早就有了，那为什么现在才突出重围，docker做了哪些改变才是容器爆发出来？

docker 最大的突破是引入镜像技术，可以让开发者轻松打包他们的应用以及相关的环境依赖生成docker镜像，然后利用镜像仓库可以快速发布到任何支持docker的
主机上，开发者不用再关注部署环境主机硬件、系统版本、运行环境等等环境配置问题，只要有docker服务，docker run 就解决了，很好解决了环境一致性问题，从而极大的提高了软件开发交付效率，就像java语言一样，一次编译，到处运行，从根本上解决了 运维开发人员 困扰多年的打包部署难题！

镜像：用于创建 Docker 容器的模板，可以看作是面向对象中的类 静态的

容器：以进程模式独立运行的一个或一组应用，可以看作是编程语言中 类对应的实例 动态的 有生命周期的

仓库：就是存放镜像的地方，可以理解为代码控制中的代码仓库


---
# docker运行架构
![docker 运行流程](https://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker/docker03.png)
 
docker 也是c/s架构 ，基本运行流程是这样的：用户通过docker client(docker) 与docker daemon(dockerd) 通信，docker daemon 接收请求后，查询本地是否存有对应镜像，如果有就使用本地镜像，启动
对应容器，如果没有就去指定 Registry 仓库拉取，然后存在本地，再执行启动步骤。

---
# docker镜像
## image
比如你要制作一个jdk 镜像  你只需要 引入一个基础系统镜像 在加一个jdk环境即可  然后 把两层 一起挂载起来 就完成了 一个jdk 环境 保存成一个专有文件系统的文件就是镜像文件了。

专有文件系统发展历程中有 aufs devicemapper overlay overlay2(目前默认)
> 镜像文件每一层是只读的，要修改 只能在顶层 封装一层自定义的一层  （删除就是在最新一层标记不可见，修改就是复制一份再修改）Copy-on-Write 跟 lvm 快照原理差不多

## 镜像仓库registry
镜像仓库跟 包仓库一样，最大的目的就是镜像分发，很好的进行版本管理等
- registry 用于保存docker的镜像的镜像仓库，用户可以自建registry，也可以用官方的docker hub。

我们拉镜像如果没指明registry地址默认就是去 docker hub上拉取进行
```bash
譬如： docker pull mysql:8.0 就是默认去docker hub上去拉取mysql 8.0 镜像
      docker pull registry.xxx.com/mysql:8.0 就是去指定的私有或第三方registry拉取镜像
```
> 这里说下VMware中国 开发并开源的私有镜像仓库harbor，已入CNCF进行孵化了，目前官方推出的registry功能太单一了基本没啥人用，
而作为registry的二次开发版harbor大受欢迎，这里贴下官网[官网地址](https://goharbor.io/)

- registry 默认为https 用http 得在配置文件中明确指明：
```bash
echo '{ "insecure-registries":["registry.xxx.com:5000","xxx.xxx.xxx.xxx:5000"] }' > /etc/docker/daemon.json
```
- repository、tag
```bash
譬如： docker pull mysql:8.0 
中的 mysql 就是repository， 8.0是tag 也就是版本号，如果没有tag 
就会默认拉取latest
```
> docker 镜像技术 实现了容器 分层构建 联合挂载，使用镜像仓库使得容器能快速部署、标准化移植交付。

---
# docker安装
## 移除老版本
```bash
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```
## 配置yum源并安装
```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker
docker info
```
> 建议安装docker前升级 系统内核 ，避免老版本内核 各种bug ，这里推荐4.19.x lt版本。
> [官方安装文档连接](https://docs.docker.com/install/linux/docker-ce/centos/#set-up-the-repository)

## 配置加速器和存放路径
```bash
cat /etc/docker/daemon.json
{
  "registry-mirrors": ["https://dockerhub.azk8s.cn", "https://docker.mirrors.ustc.edu.cn"],
  "insecure-registries": ["127.0.0.1/8"],
  "max-concurrent-downloads": 10,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
    },
  "data-root": "/opt/docker"
}
```

# 推荐链接：
- 官方文档： [Docker Documentation](https://docs.docker.com)
- dockerinfo文档： [dockerinfo](http://www.dockerinfo.net/document)
- runoob文档： [runoob](https://www.runoob.com/docker/docker-tutorial.html)
- 云栖社区文档： [云栖社区文档](https://yq.aliyun.com/articles/713839?spm=a2c4e.11155435.0.0.47b366bbpsiI7S)
