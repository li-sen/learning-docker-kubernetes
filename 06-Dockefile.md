> 大家都知道docker是一个轻量快速，简单又好用的虚拟化技术，但如果打包镜像每次手工打包那肯定是痛苦不堪的，还好的是有Dockerfile帮我们很好的解决镜像打包的诸多问题。

Dockerfile 是镜像的描述文件，定义了如何构建 Docker 镜像；Dockerfile将打包镜像所需的文件、环境依赖、步骤循序等等代码化，这样我们只要docker build -t xxx 就完成了一次打包镜像，有了Dockerfile,我们做自动化、标准化、
以及镜像共享分发就会容易很多了，我这里就简单的讲讲Dockerfile的基本用法。
 
 我们以mysql官方的 Dockerfile 自己修改下作为示例：
 ```dockerfile
FROM debian:stretch-slim
# 指定基础镜像，必须为第一个命令
# Dockerfile文件必需以FROM命令开始，然后按照文件中的命令顺序逐条进行执行
# 以#开始的内容会被看做是对相关命令的注释。

MAINTAINER Li Sen <lisen2023@gmail.com>
# 维护者信息， 为了说明Dockerfile 本人自己添加
# 不过新版 推荐使用 LABEL

LABEL LiSen="lisen@gmail.com"
LABEL version="1.0" description='Dockerfile example'
# 为镜像指定标签 , LABEL会继承基础镜像种的LABEL，如有相同KEY，就被新值覆盖。

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r mysql && useradd -r -g mysql mysql
# 构建镜像时运行的命令

RUN apt-get update && apt-get install -y --no-install-recommends gnupg dirmngr && rm -rf /var/lib/apt/lists/*

USER mysql
# 指定 运行用户，对 RUN、CMD、ENTRYPOINT等指令生效
# 要临时获取管理员权限可以使用 gosu，而不推荐 sudo

# add gosu for easy step-down from root
ENV GOSU_VERSION 1.7
# 设置镜像中的 环境变量

RUN set -x \
	&& apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/* \
	&& wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
	&& wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
	&& gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& gpgconf --kill all \
	&& rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu nobody true \
	&& apt-get purge -y --auto-remove ca-certificates wget

RUN mkdir /docker-entrypoint-initdb.d

RUN apt-get update && apt-get install -y --no-install-recommends \
# for MYSQL_RANDOM_ROOT_PASSWORD
		pwgen \
# for mysql_ssl_rsa_setup
		openssl \
# FATAL ERROR: please install the following Perl modules before executing /usr/local/mysql/scripts/mysql_install_db:
# File::Basename
# File::Copy
# Sys::Hostname
# Data::Dumper
		perl \
	&& rm -rf /var/lib/apt/lists/*

RUN set -ex; \
# gpg: key 5072E1F5: public key "MySQL Release Engineering <mysql-build@oss.oracle.com>" imported
	key='A4A9406876FCBD3C456770C88C718D3B5072E1F5'; \
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$key"; \
	gpg --batch --export "$key" > /etc/apt/trusted.gpg.d/mysql.gpg; \
	gpgconf --kill all; \
	rm -rf "$GNUPGHOME"; \
	apt-key list > /dev/null

ENV MYSQL_MAJOR 8.0
ENV MYSQL_VERSION 8.0.17-1debian9

RUN echo "deb http://repo.mysql.com/apt/debian/ stretch mysql-${MYSQL_MAJOR}" > /etc/apt/sources.list.d/mysql.list

# the "/var/lib/mysql" stuff here is because the mysql-server postinst doesn't have an explicit way to disable the mysql_install_db codepath besides having a database already "configured" (ie, stuff in /var/lib/mysql/mysql)
# also, we set debconf keys to make APT a little quieter
RUN { \
		echo mysql-community-server mysql-community-server/data-dir select ''; \
		echo mysql-community-server mysql-community-server/root-pass password ''; \
		echo mysql-community-server mysql-community-server/re-root-pass password ''; \
		echo mysql-community-server mysql-community-server/remove-test-db select false; \
	} | debconf-set-selections \
	&& apt-get update && apt-get install -y mysql-community-client="${MYSQL_VERSION}" mysql-community-server-core="${MYSQL_VERSION}" && rm -rf /var/lib/apt/lists/* \
	&& rm -rf /var/lib/mysql && mkdir -p /var/lib/mysql /var/run/mysqld \
	&& chown -R mysql:mysql /var/lib/mysql /var/run/mysqld \
# ensure that /var/run/mysqld (used for socket and lock files) is writable regardless of the UID our mysqld instance ends up having at runtime
	&& chmod 777 /var/run/mysqld

VOLUME /var/lib/mysql
# 容器运行时应该尽量保持容器存储层不发生写操作，对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷(volume)中。
# 为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在 Dockerfile 中，可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据

# Config files
COPY config/ /etc/mysql/
# 拷贝 本地目录config 到 镜像 /etc/mysql/ 目录下，与此类似的 还有一个 ADD，不过 ADD 有一些其他的特性，比如 拷贝的对象是 tar包 使用 ADD会自动解压等
# 一般优先 使用 COPY， 它比 ADD 更透明，只有 拷贝作用。
# src 是目录得注意 它拷贝的事 dir/* 目录本身是不会拷贝的
# dest 如果是目录必须得是以/ 结尾，不然会被识别成文件

WORKDIR /
# 指定 工作目录 相当于 linux 命令中的 cd
# 对 Dockerfile 中的 RUN、CMD、ENTRYPOINT、COPY、ADD指定工作目录
# 如果不存在则会创建，也可以设置多次。


COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s usr/local/bin/docker-entrypoint.sh /entrypoint.sh # backwards compat
ENTRYPOINT ["docker-entrypoint.sh"]
# 设置镜像启动时运行的 主命令，可以通过 docker run 的参数 --entrypoint 来替代镜像中默认的ENTRYPOINT，通过 --entrypoint 传的必须是可执行的二进制程序， 即不会以sh -c 形式执行。
# 如果 Dockerfile 有多个 ENTRYPOINT指令，则以最后一个为准。

EXPOSE 3306 33060
# 暴露容器运行时监听的端口，在启动容器时需要通过 -P，Docker 主机会自动分配一个端口转发到指定的端口。

CMD ["mysqld"]
# 设置容器启动后默认执行的命令及其参数，但 CMD 能够被 docker run 后面跟的命令行参数替换，
# 如果 Dockerfile 有多个 CMD 指令，则以最后一个为准。
# 如果 Dockerfile 同时有 ENTRYPOINT 和 CMD，CMD会被加到 ENTRYPOINT 后面当参数运行，但是如果 ENTRYPOINT 的命令格式是 Shell 格式，会忽略任何 CMD 或 docker run 提供的参数
```

# 其他指令
```bash
ONBUILD
语法格式：
ONBUILD <其它指令>

说明：指定以当前镜像为基础镜像构建的下一级镜像运行的命令
例如：ONBUILD COPY ./package.json /app或者ONBUILD RUN [ "npm", "install" ]
注：在当前镜像构建时不会执行，只有以当前镜像为基础镜像，去构建下一级镜像的时候才会被执行。

ARG
语法：
ARG <name>[=<default value>]
设置变量命令，ARG命令定义了一个变量，在docker build创建镜像的时候，使用 --build-arg <varname>=<value>来指定参数
如果用户在build镜像时指定了一个参数没有定义在Dockerfile种，那么将有一个Warning
提示如下：
[Warning] One or more build-args [foo] were not consumed.

我们可以定义一个或多个参数，如下：
FROM busybox
ARG user1
ARG buildno
...
也可以给参数一个默认值：

FROM busybox
ARG user1=someuser
ARG buildno=1
...
如果我们给了ARG定义的参数默认值，那么当build镜像时没有指定参数值，将会使用这个默认值

对于这个 参数 适用的场景 比如 我想根据 build 传过来的参数 来决定 打什么样的 镜像，或者 打镜像包时 传 版本号。 
ARG JAVA_OPTS="-Xmx2688M -Xms2688M -Xmn960M -XX:MaxMetaspaceSize=256M -XX:MetaspaceSize=256M"
ENV JAVA_OPTS=${JAVA_OPTS}
比如这种方式 引用。

STOPSIGNAL
语法：
STOPSIGNAL signal
STOPSIGNAL命令是的作用是当容器推出时给系统发送什么样的指令，例如9，或SIGNAME格式的信号名，例如SIGKILL。

HEALTHCHECK
容器健康状况检查命令
语法有两种：
1. HEALTHCHECK [OPTIONS] CMD command
2. HEALTHCHECK NONE
第一个的功能是在容器内部运行一个命令来检查容器的健康状况

第二个的功能是在基础镜像中取消健康检查命令

[OPTIONS]的选项支持以下三中选项：
    --interval=DURATION 两次检查默认的时间间隔为30秒
    --timeout=DURATION 健康检查命令运行超时时长，默认30秒
    --retries=N 当连续失败指定次数后，则容器被认为是不健康的，状态为unhealthy，默认次数是3

注意：
HEALTHCHECK命令只能出现一次，如果出现了多次，只有最后一个生效。
CMD后边的命令的返回值决定了本次健康检查是否成功，具体的返回值如下：
0: success - 表示容器是健康的
1: unhealthy - 表示容器已经不能工作了
2: reserved - 保留值

例子：
HEALTHCHECK --interval=5m --timeout=3s \
CMD curl -f http://localhost/ || exit 1
  
健康检查命令是：curl -f http://localhost/ || exit 1
两次检查的间隔时间是5秒
命令超时时间为3秒
```

# 特别说的 几点
## 基础镜像
制作自己私有镜像时，尽量使用 官方 制作好的 镜像为 基础镜像，比如JDK tomcat nginx 等，自己在此基础上在做自定义，尽量避免从头制作，这样只会吃力不讨好，当然官方镜像不满足需求另说。

在这推荐两个 系统镜像：
- alpine镜像
保持最小化精简系统的同时还有完善的包管理工具，现在很多官方镜像 都推出 alpine 版本。
- scratch 镜像
这个镜像是虚拟的概念，它表示一个空白的镜像，意味着你不以任何镜像为基础。
这对于go 这种静态编译的程序，它所需的库都已经在可执行文件中了，可以不需要操作系统就能运行，所以GO是天然适合容器的语言的。

## CMD 和 ENTRYPOINT
```bash
CMD
支持三种格式
CMD ["executable","param1","param2"]  exec模式 ，推荐方式；
CMD command param1 param2             shell模式，提供给需要交互的应用；
CMD ["param1","param2"]               提供给 ENTRYPOINT 的默认参数；
指定启动容器时 默认 执行命令，每个 Dockerfile 只能有一条 CMD 命令。如果指定了多条命令，只有最后一条会被执行。
如果用户启动容器时候指定了运行的命令，则会覆盖掉 CMD 指定的命令。

ENTRYPOINT
两种格式：
ENTRYPOINT ["executable", "param1", "param2"] exec模式 ，推荐方式
ENTRYPOINT command param1 param2              shell模式
设置镜像启动时运行的 主命令，可以通过 docker run 的参数 --entrypoint 来替代镜像中默认的ENTRYPOINT，通过 --entrypoint 传的必须是可执行的二进制程序， 即不会以sh -c 形式执行。
每个 Dockerfile 中只能有一个 ENTRYPOINT，当指定多个时，只有最后一个起效。

# 如果 Dockerfile 同时有 ENTRYPOINT 和 CMD，CMD会被加到 ENTRYPOINT 后面当参数运行，但是如果 ENTRYPOINT 的命令格式是 Shell 格式，会忽略任何 CMD 或 docker run 提供的参数

```

## 多阶段构建
docker在之前的老版本中，只支持一个FROM指令，这就意味着 每次 编译环境 和 程序的运行环境 没法解耦，造成 最后的镜像包含了 很多无用 的组件，使得 镜像 变得很大；
譬如 这是 之前不支持 多阶段构建的 打包运行 go：
```dockerfile
# Go语言环境基础镜像
FROM golang:1.10.3

# 将源码拷贝到镜像中
COPY server.go /build/

# 指定工作目录
WORKDIR /build

# 编译镜像时，运行 go build 编译生成 server 程序
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GOARM=6 go build -ldflags '-w -s' -o server

# 指定容器运行时入口程序 server
ENTRYPOINT ["/build/server"]
```
使用 多阶段构建后：
```dockerfile
# 编译阶段
FROM golang:1.10.3

COPY server.go /build/

WORKDIR /build

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GOARM=6 go build -ldflags '-w -s' -o server

# 运行阶段
 FROM scratch

# 从编译阶段的中拷贝编译结果到当前镜像中
COPY --from=0 /build/server /

ENTRYPOINT ["/server"]
```
这里的 关键词 就是 COPY --from=0  ，在多个FROM 语句中， 0 代表第一个 阶段，也就是将 第一个阶段中的 指定的 文件 拷贝都 当前阶段中引用，
可以 对阶段 设置 别名  例如：
```dockerfile
# 编译阶段 命名为 builder
FROM golang:1.10.3 as builder

# ... 省略

# 运行阶段
FROM scratch

# 从编译阶段的中拷贝编译结果到当前镜像中
COPY --from=builder /build/server /
```
也可以从 已有的镜像中 拷贝 我们所需的 文件 进行 打包
```dockerfile
FROM ubuntu:16.04

COPY --from=quay.io/coreos/etcd:v3.3.9 /usr/local/bin/etcd /usr/local/bin/
```
有了多阶段构建 这个特性，极大的 方便我们 打包镜像，同时进一步 减少 镜像体积，一举多得的 好方法。

## 最小化镜像层数
当我们使用 Dockerfile 构建镜像时，应该尽量 减少 RUN、COPY 和 ADD 指令， 因为他们 运行一次就会 增加 一层镜像层，造成 镜像变大，尽量使用多阶段构建。

## docker-entrypoint.sh
我们在 很多 官方的镜像 的 Dockerfile 都发现 ENTRYPOINT ["docker-entrypoint.sh"] 这样的 写法，这里简单说下 docker-entrypoint.sh的用处。

一般来说 我们很多应用在容器里面 运行之前 需要进行一些初始化配置，譬如 我们的 应用 为了实现：一次 ci ，然后根据 不同环境（开发、测试、生产）进行cd 部署，这就 涉及到 配置文件变更的问题。
最好的做法 肯定是 用 配置中心的 方式 每个镜像起来的时候传 环境参数即可，不过 你也可以更 轻量 精简的方式 解决 那就是 使用  docker-entrypoint.sh 做 容器启动前的 预处理：

譬如 我这里前端容器 在启动前 根据 环境 将配置进行 替换：
```bash
cat docker-entrypoint.sh
#!/bin/bash
set -e
find /dist -name '*.js' | xargs sed -i "s localhost:8080 $PRO_API_HOST g"
find /dist -name '*.html' | xargs sed -i "s localhost:8080 $PRO_API_HOST g"
exec "$@"

docker run -d --restart=always \
  -e PRO_API_HOST=api.xxx.com\
  xxx
```
## .dockerignore 文件
.dockerignore， 它的功能类似 .gitignore，它需要存放在构建环境根目录下才会起作用，.dockerignore 可以避免和排除不必要的大型或敏感文件和目录进行 ADD 或者 COPY . 拷贝这些文件和目录。

简单的 .dockerignore 文件如下：
```shell script

# comment
*/temp*
*/*/temp*
temp?
规则	解释
# comment	注释，忽略
*/temp*	排除根目录一级子目录下所有以 temp 开头的文件和目录。如 /somedir/temp、/somedir/temporary.txt 都将会被排除
*/*/temp*	排除根目录下二级子目录下所有以 temp 开头的文件和目录，如 /somedir/subdir/temporary.txt 会被排除
temp?	? 号表示占用一个字符串，如 /tempa、/tempb 文件目录都会被排除
.dockerignore 的匹配规则遵循 Go 的 filepath.Match 规则。除了该规则外，Docker 还支持了一些特殊的通配符，** 匹配任意层级的目录。例如，**/*.go 将排除构建环境根目录下所有以 .go 为后缀的文件。! 表示忽略排除，如下：
*.md
!README.md
表示排除根目录当前层级除了README.md 外所有以 .md 为后缀的文件。

匹配是有顺序的，如果前后的规则有重叠或者冲突，则后面的规则生效。如果 !README.md 在 *.md 之前，则以 *.md 为规则，README.md 依然会被排除。
可以通过 .dockerignore 来排除 Dockerfile 和 .dockerignore 文件。但是这些文件依然会发送到 Docker daemon。不过，ADD 和 COPY 指令将不会拷贝它们。
```

# 参考链接
[docker官方文档](https://docs.docker.com/engine/reference/builder/)