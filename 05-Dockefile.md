---
showonlyimage: true
title:      "Dockerfile"
subtitle:   ""
excerpt: "Dockerfile"
description: ""
date:       2019-02-02
author:     "李森"
image: "https://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/blog/post-bg-2.jpg"
published: true 
gitment: true
tags:
    - docker
    - Dockerfile
categories: [ PAAS ]
URL: "/2019/02/02/Dockerfile/"
---
### 前言
大家都知道docker确实是一个轻量快速，简单又好用的虚拟化技术，但如果打包镜像每次手工打包那肯定是痛苦不堪的，还好的是Dockerfile很好的解决镜像打包的诸多问题。

Dockerfile 是镜像的描述文件，定义了如何构建 Docker 镜像；Dockerfile将打包镜像所需的文件、环境依赖、步骤循序等等代码化，这样我们只要docker build -t xxx 就完成了一次打包镜像，有了Dockerfile,我们做自动化、标准化、
以及镜像共享分发就会容易很多了，我这里就简单的讲讲Dockerfile的基本用法。

### FROM
指定基础镜像，譬如以下指定nginx 为基础镜像后自定义index.html。
```bash
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```
#### scratch 镜像
这个镜像是虚拟的概念，它表示一个空白的镜像，意味着你不以任何镜像为基础。
这对于go 这种静态编译的程序，它所需的库都已经在可执行文件中了，可以不需要操作系统就能运行，所以GO是天然适合容器的语言的。

####base 镜像
的通常都是各种 Linux 发行版的 Docker 镜像，比如 Ubuntu, Debian, CentOS ，那为什么host主机能运行这些不同系统的容器呢？
而且我们拉的镜像文件都比传统的系统镜像要小很多，比如centos 我们安装完起码的好几个G ，镜像却只有几百兆，它是如何做到的呢？
我们来简单说说这两个问题：
1. 我们先来说为什么能运行容器镜像这么小

一般linux操作系统是分 内核空间跟 用户空间的，系统启动时由bootfs 然后完成rootfs的挂载。
bootfs(boot file system)主要包含 bootloader和kernel，bootloader主要是引导加载kernel，当kernel被加载到内存中后 bootfs就被umount了。 
rootfs (root file system) 包含的就是典型 Linux 系统中的/dev，/proc，/bin，/etc等标准目录和文件。

传统的Linux加载bootfs时会先将rootfs设为read-only，然后在系统自检之后将rootfs从read-only改为read-write，然后我们就可以在rootfs上进行写和读的操作了。

但Docker的镜像却不是这样，它在bootfs自检完毕之后并不会把rootfs的read-only改为read-write。
而是利用union mount（UnionFS的一种挂载机制）将一个或多个read-only的rootfs加载到之前的read-only的rootfs层之上。
在加载了这么多层的rootfs之后，仍然让它看起来只像是一个文件系统，
在Docker的体系里把union mount的这些read-only的rootfs叫做Docker的镜像。但是，此时的每一层rootfs都是read-only的，我们此时还不能对其进行操作。
当我们创建一个容器，也就是将Docker镜像进行实例化，系统会在一层或是多层read-only的rootfs之上分配一层空的read-write的rootfs。





### RUN
打包镜像时执行命令，譬如上例中：
```bash
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```



