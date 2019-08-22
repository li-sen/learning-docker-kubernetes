# docker 存储
docker存储目前分两类：

- storage driver ： 镜像和容器文件系统
- data volume： 有状态数据保持

下面我们来分开简单介绍下这两类存储

# storage driver 
> docker 之所以成功，很关键的因素就是镜像技术，而镜像分层这一特性 就靠 docker独特的联合挂载文件系统实现的，这就不得不提AUFS了。

## AUFS
AUFS Advance UnionFS 的简称，是 UnionFS 后期版本，UnionFS 是一个联合目录挂载文件系统，目的就是把不同物理位置的目录合并mount到同一个目录中进行读写操作。
AUFS作者也是挺有意思的一个日本人，不过由于项目代码质量被Linus鄙视，至今AUFS也没进入linux主干中，不过还好比较激进的ubantu接纳了他，现在这么多人能使用它这种思想的文件系统也算是一种成功了。

那我们先简单来说下AUFS。

我们一起来做个AUFS示例：
```bash
# 先创建两个目录，然后进行联合挂载，我这里使用的ubantu系统 centos要使用AUFS需要升级带aufs模块的内核。
root@test01:~/test# tree
.
├── a
│   ├── 1
│   └── 2
└── b
    ├── 2
    └── 3

2 directories, 4 files
root@test01:~/test# mkdir mnt
root@test01:~/test# mount -t aufs -o dirs=./a:./b none ./mnt

# 往 mnt 2中写入内容
root@test01:~/test# echo 'test' > ./mnt/2
root@test01:~/test# cat a/2
test
root@test01:~/test# cat b/2
root@test01:~/test#
# 可以看到 a/2 内容变了，而 b/2 却没变
# 这是因为 aufs 默认上来说，命令行上第一个（最左边）的目录是可读可写的，后面的全都是只读的。

# 当然我们也可以指定权限 进行挂载
root@test01:~/test# > ./mnt/2
root@test01:~/test# cat a/2
root@test01:~/test# cat b/2
root@test01:~/test# umount ./mnt
root@test01:~/test# mount -t aufs -o dirs=./a=rw:./b=rw none ./mnt
root@test01:~/test# echo 'test' > ./mnt/2
root@test01:~/test# cat a/2
test
root@test01:~/test# cat b/2
root@test01:~/test#
# 可见，如果有重复的文件名，在mount命令行上，越往前的就优先级越高，只对优先级高的进行操作。

```
通过上面的示例我们简单了解了下 AUFS 的大体 操作流程，docker镜像就是 由一层层的文件系统 联合挂载实现的，从而比较完美实现了 镜像的封装 分发：

比如我们想做一个 java应用镜像 其Dockerfile如下：
> dockerfile 是制作自定义镜像一种 标准通用镜像打包 文件格式，后续会详细讲解

```dockerfile
# 使用官方提供的java环境作为基础镜像
FROM java:8-jre
MAINTAINER Li Sen <lisen2023@gmail.com>

# 对基础镜像进行本地化 改造
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo 'Asia/Shanghai' >/etc/timezone && \
    echo "export LC_ALL=en_US.UTF-8" >> /etc/profile && \
    . /etc/profile

# 将工作目录切换为/opt， 相当于linux中的cd
WORKDIR /opt

# 拷贝 当前目录的 jar包 到 镜像中的/opt下
COPY app.jar /opt/app.jar

# 暴露容器的 8080端口
EXPOSE 8080

# 容器的启动命令
CMD java $JAVA_OPTS $SKYWALKING_OPTS -jar app.jar

```
通过以上 文件的内容可以清晰看到一个 自定义的镜像的操作流程，那docker 是怎么 通过镜像 运行的呢？那我们来了解下docker容器的文件系统：

docker 容器运行 ，容器的最上层是一个可读写的临时层，下面是只读的镜像层，他们以从下往上以栈的方式联合挂载。

> Init层 在只读层和读写层之间。Init层是Docker项目单独生成的一个内部层，专门用来存放/etc/hosts、/etc/resolv.conf等信息。
这些文件本来属于只读镜像层的一部分，但是用户往往需要在启动容器时写入一些指定的值比如hostname，所以就需要在可读写层对它们进行修改。
可是，这些修改往往只对当前的容器有效，我们并不希望执行docker commit时，把这些信息连同可读写层一起提交掉。
所以，Docker做法是，在修改了这些文件之后，以一个单独的层挂载了出来。而用户执行docker commit只会提交可读写层，所以是不包含这些内容的。

当我们在运行的容器中 修改一个文件时，该文件会 从
镜像只读层中 复制一份到 容器运行 的读写层 进行修改，而只读版本仍然存在于 镜像层中，只是 在读写层中进行了 隐藏屏蔽， 这样就避免了 对镜像层的 修改，保证了镜像的 一致性。

![容器文件系统](https://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker/docker_fs.png)

此种联合文件系统，利用Copy-on-Write 机制如下特性（以AUFS 为例）：

- AUFS 是一种联合文件系统，它把若干目录按照顺序和权限 mount 为一个目录并呈现出来
- 默认情况下，只有第一层（第一个目录）是可写的，其余层是只读的。
- 增加文件：默认情况下，新增的文件都会被放在最上面的可写层中。
- 删除文件：因为底下各层都是只读的，当需要删除这些层中的文件时，AUFS 使用 whiteout 机制，它的实现是通过在上层的可写的目录下建立对应的whiteout隐藏文件来实现的。
- 修改文件：AUFS 利用其 CoW （copy-on-write）特性来修改只读层中的文件：AUFS 工作在文件层面，对只读层中的文件做修改，在第一次修改时，文件都会被拷贝到可写层然后再被修改。
- 节省空间：AUFS 的 CoW 特性能够允许在多个容器之间共享分层，从而减少物理空间占用。
- 查找文件：AUFS 的查找性能在层数非常多时会出现下降，层数越多，查找性能越低，因此，在制作 Docker 镜像时要注意层数不要太多。
- 性能：AUFS 的 CoW 特性在写入大型文件时第一次会出现延迟

> AUFS实现原理: [AUFS](http://www.youruncloud.com/blog/120.html)  
> 使用OverlayFS 从3.18开始，才进入了Linux内核主线，建议进行内核升级，最好是4.xx 较新的ml或lt版本。

通过 镜像的分层设计， 大家可以在 各种官方镜像的基础上，进行增量的操作封装 ；在满足各自的需求同时，用户频繁拉取不同镜像会有很大一定的比例很多镜像层是重复的，
比如拉取 或者 运行 一个java镜像 跟一个 mysql镜像 其底层镜像 可能都是 一个 alpine 基础镜像，用户就可以复用之前的基础镜像 只要拉取 对应改动 镜像层即可，极大的减小了 镜像 分发部署时的 启动速度 网络流量 镜像大小 下载速度。

不过 容器 有一个特性就是 当 运行容器 被删后，其读写层中的数据 也会被删掉，如要保留 需要 使用 volume 进行挂载到本地进行存储。

那么下面来简单讲下docker的存储


## OverlayFS
docker 基本概念和原理中说明了 docker image 是通过 分层构建 联合挂载来实现的，早期  storage driver 为AUFS，目前docker 18.x 默认为
OverlayFS（overlay2），它可以看做AUFS的升级加强版，两者基本原理类似：

OverlayFS关联的底层目录称为lowerdir， 
对应的高层目录称为upperdir。合并过后统一视图称为merged。下图是一个docker镜像和docke容器的分层图，docker镜像是lowdir，docker容器是upperdir。而统一的视图层是merged层 

![overlayfs](http://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker/docker_overlayfs.jpg)

读文件的时候，如果文件在upperdir则直接读取，文件不在upperdir则从lowerdir读，如果写的文件不在uppderdir在lowerdir，则从lowerdir里面copy到upperdir，不管文件多大，copy完再写，删除或者重命名镜像层的文件都只是在容器层生成whiteout文件标志。

overlay2支持多个容器访问相同文件时公用page cache，，也就是说如果多个容器访问同一个文件，可以共享同一个页缓存，这使得overlay/overlay2驱动高效地利用了内存。 
由于overlay2层数更少，copy_up 只发生在一次写入文件，一旦文件被复制所有的读写都在upperdir，避免了 更多的 复制操作。

[docker官网相关介绍](https://docs.docker.com/storage/storagedriver/overlayfs-driver/)

与AUFS相比，OverlayFS有以下特性：
1. 设计更简单
2. 合并Linux内核
3. 性能更好
4. docker默认存储驱动，支持度兼容性更好。

我们来简单了解下 overlay2 目录结构：
```bash
root@test01:~# docker pull ubuntu
Using default tag: latest
latest: Pulling from library/ubuntu
35c102085707: Pull complete
251f5509d51d: Pull complete
8e829fe70a46: Pull complete
6001e1789921: Pull complete
Digest: sha256:d1d454df0f579c6be4d8161d227462d69e163a8ff9d20a847533989cf0c94d90
Status: Downloaded newer image for ubuntu:latest
root@test01:~#

root@test01:~# ll /var/lib/docker/overlay2/
total 28
drwx------  7 root root 4096 Aug 22 16:47 ./
drwx--x--x 14 root root 4096 Jul 18 20:24 ../
drwx------  4 root root 4096 Aug 22 16:47 489e667e749d93c1d7657bcee399d0b5b98390bb1af737e4129e7d0a0ca1b7cb/
drwx------  3 root root 4096 Aug 22 16:47 5609fe148f9ea3a1f2e18187e54f24918a067869c7ec19483e15c7c4d018a2d9/
drwx------  4 root root 4096 Aug 22 16:47 5d1ed4a6bbe38a244785a6d800ce71075fff29aa504f6d5c0fa9a78ad87d3d24/
drwx------  4 root root 4096 Aug 22 16:47 fede1e6229f0f69d1042be107e6bff46e417e172192b508104e796a3448fc4e6/
drwx------  2 root root 4096 Aug 22 16:47 l/
root@test01:~#
# 可见 Ubuntu 镜像包含了4层镜像

root@test01:~# ll /var/lib/docker/overlay2/l/
total 24
drwx------ 2 root root 4096 Aug 22 16:47 ./
drwx------ 7 root root 4096 Aug 22 16:47 ../
lrwxrwxrwx 1 root root   72 Aug 22 16:47 2CXWTWYE2YOJB3AUF33ZIT5JDU -> ../5609fe148f9ea3a1f2e18187e54f24918a067869c7ec19483e15c7c4d018a2d9/diff/
lrwxrwxrwx 1 root root   72 Aug 22 16:47 ODLL5GTXUTVPVVO22OM4PINQX7 -> ../5d1ed4a6bbe38a244785a6d800ce71075fff29aa504f6d5c0fa9a78ad87d3d24/diff/
lrwxrwxrwx 1 root root   72 Aug 22 16:47 PBHZH5FEGROA4GW4WNQMXCVUR6 -> ../489e667e749d93c1d7657bcee399d0b5b98390bb1af737e4129e7d0a0ca1b7cb/diff/
lrwxrwxrwx 1 root root   72 Aug 22 16:47 S6ZTYITDR2QGUA6MNPIFQMJOKS -> ../fede1e6229f0f69d1042be107e6bff46e417e172192b508104e796a3448fc4e6/diff/
root@test01:~#
# l目录包含了很多软连接，使用短名指向了 5层镜像层目录下的diff目录，短名用于避免挂载命令不会超出长度

root@test01:~# ll /var/lib/docker/overlay2/5609fe148f9ea3a1f2e18187e54f24918a067869c7ec19483e15c7c4d018a2d9/diff/
total 84
drwxr-xr-x 21 root root 4096 Aug 22 16:47 ./
drwx------  3 root root 4096 Aug 22 16:47 ../
drwxr-xr-x  2 root root 4096 Aug  7 21:03 bin/
drwxr-xr-x  2 root root 4096 Apr 24  2018 boot/
drwxr-xr-x  2 root root 4096 Aug  7 21:03 dev/
drwxr-xr-x 29 root root 4096 Aug  7 21:03 etc/
drwxr-xr-x  2 root root 4096 Apr 24  2018 home/
drwxr-xr-x  8 root root 4096 May 23  2017 lib/
drwxr-xr-x  2 root root 4096 Aug  7 21:03 lib64/
drwxr-xr-x  2 root root 4096 Aug  7 21:02 media/
drwxr-xr-x  2 root root 4096 Aug  7 21:02 mnt/
drwxr-xr-x  2 root root 4096 Aug  7 21:02 opt/
drwxr-xr-x  2 root root 4096 Apr 24  2018 proc/
drwx------  2 root root 4096 Aug  7 21:03 root/
drwxr-xr-x  4 root root 4096 Aug  7 21:02 run/
drwxr-xr-x  2 root root 4096 Aug  7 21:03 sbin/
drwxr-xr-x  2 root root 4096 Aug  7 21:02 srv/
drwxr-xr-x  2 root root 4096 Apr 24  2018 sys/
drwxrwxrwt  2 root root 4096 Aug  7 21:03 tmp/
drwxr-xr-x 10 root root 4096 Aug  7 21:02 usr/
drwxr-xr-x 11 root root 4096 Aug  7 21:03 var/
root@test01:~# cat /var/lib/docker/overlay2/5609fe148f9ea3a1f2e18187e54f24918a067869c7ec19483e15c7c4d018a2d9/link
2CXWTWYE2YOJB3AUF33ZIT5JDU
# diff 存放的是这一层的实际文件内容， 同级的 link文件 是存放 之前l 目录的软连接 短名称，实际对应的目录就是 diff


root@test01:~# docker run -d --name test ubuntu sleep 3600
37e6d5c3449e1564b4faedc6d40f8419426d6cb277340fd20fba833c7f3dea3e
# 启动一个测试镜像

root@test01:~# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
37e6d5c3449e        ubuntu              "sleep 3600"        4 minutes ago       Up 4 minutes                            test
# 我们可以看到CONTAINER ID 37e6d5c3449e， 那怎么确认 容器包含了哪些镜像层呢？我们可以直接在 /var/lib/docker/image/overlay2/imagedb/content/sha256/ 找到答案

root@test01:~# cat /var/lib/docker/image/overlay2/imagedb/content/sha256/a2a15febcdf362f6115e801d37b5e60d6faaeedcb9896155e5fe9d754025be12|grep rootfs
..."os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:6cebf3abed5fac58d2e792ce8461454e92c245d5312c42118f02e231a73b317f","sha256:f7eae43028b334123c3a1d778f7bdf9783bbe651c8b15371df0120fd13ec35c5","sha256:7beb13bce073c21c9ee608acb13c7e851845245dc76ce81b418fdf580c45076b","sha256:122be11ab4a29e554786b4a1ec4764dd55656b59d6228a0a3de78eaf5c1f226c"]}}
# 可以看到rootfs的diff_ids是一个包含了4个元素的数组，其实这3个元素正是组成镜像的4个layerID
# 从左到右 第一个为 最底层 依次堆叠到 最高层， 我们可以拿着这些 layerID 去找对应的layer

root@test01:~# ll /var/lib/docker/image/overlay2/layerdb/sha256/
total 24
drwxr-xr-x 6 root root 4096 Aug 22 16:47 ./
drwx------ 5 root root 4096 Aug 22 17:06 ../
drwx------ 2 root root 4096 Aug 22 16:47 2b4ed599a73ad82b15d4e3488d95af4f387037b90401d63ad6cb433374b3e3d3/
drwx------ 2 root root 4096 Aug 22 16:47 6cebf3abed5fac58d2e792ce8461454e92c245d5312c42118f02e231a73b317f/
drwx------ 2 root root 4096 Aug 22 16:47 8f06e3a624319de717230f0d766dd8922d048441adb9e0387860a8047d000409/
drwx------ 2 root root 4096 Aug 22 16:47 fdc47e80ad3dbe1767a5c2141442c0c7aa93a14a817357a0d4353cd8ae48ee58/
root@test01:~#
# 我们在这个目录下只找到了 最底层的6cebf3abed5，其他的层呢？原来是 docker使用了chainID的方式去保存这些layer
# chainID=sha256sum(H(chainID) diffid)

root@test01:~# echo -n  'sha256:6cebf3abed5fac58d2e792ce8461454e92c245d5312c42118f02e231a73b317f sha256:f7eae43028b334123c3a1d778f7bdf9783bbe651c8b15371df0120fd13ec35c5'|sha256sum -
8f06e3a624319de717230f0d766dd8922d048441adb9e0387860a8047d000409  -
root@test01:~#
# 最底层layerID+ 上一层layerID sha256sum计算得出 上一层的 chainID， 这样就可以一次得到所有 layer的 chainID
# /var/lib/docker/image/overlay2/layerdb存的只是元数据，那么真实的rootfs到底存在哪里呢？其中cache-id就是我们关键所在了
root@test01:~# cat /var/lib/docker/image/overlay2/layerdb/sha256/8f06e3a624319de717230f0d766dd8922d048441adb9e0387860a8047d000409/cache-id
5d1ed4a6bbe38a244785a6d800ce71075fff29aa504f6d5c0fa9a78ad87d3d24
# 这个id 对应的就是/var/lib/docker/overlay2/5d1ed4a6bbe38a244785a6d800ce71075fff29aa504f6d5c0fa9a78ad87d3d24
root@test01:~# ll /var/lib/docker/overlay2/5d1ed4a6bbe38a244785a6d800ce71075fff29aa504f6d5c0fa9a78ad87d3d24/
total 24
drwx------ 4 root root 4096 Aug 22 16:47 ./
drwx------ 9 root root 4096 Aug 22 17:07 ../
drwxr-xr-x 3 root root 4096 Aug 22 16:47 diff/
-rw-r--r-- 1 root root   26 Aug 22 16:47 link
-rw-r--r-- 1 root root   28 Aug 22 16:47 lower
drwx------ 2 root root 4096 Aug 22 16:47 work/
 # 这样就可以 依次找到 容器 涉及的所有 镜像层了

root@test01:~# mount|grep overlay
overlay on /var/lib/docker/overlay2/ab465cf928ac6081ef29148d396e2a11162f00a43348c9fec3d8aefe96f0ffd5/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/W53MZD3AGBDLCKSKCYYHAC7677:/var/lib/docker/overlay2/l/PBHZH5FEGROA4GW4WNQMXCVUR6:/var/lib/docker/overlay2/l/S6ZTYITDR2QGUA6MNPIFQMJOKS:/var/lib/docker/overlay2/l/ODLL5GTXUTVPVVO22OM4PINQX7:/var/lib/docker/overlay2/l/2CXWTWYE2YOJB3AUF33ZIT5JDU,upperdir=/var/lib/docker/overlay2/ab465cf928ac6081ef29148d396e2a11162f00a43348c9fec3d8aefe96f0ffd5/diff,workdir=/var/lib/docker/overlay2/ab465cf928ac6081ef29148d396e2a11162f00a43348c9fec3d8aefe96f0ffd5/work)
root@test01:~#
# mount也能 看出每一层的挂载顺序
```
从docker的文件系统特性所知，无状态的容器中的数据，让storage driver 自行维护是比较推荐的做法，但对于一些有状态的容器，比如 数据库容器、中间件容器等，这些
数据是要持久性保存的，也就是说容器挂了，容器数据必须还在，数据可能在宿主机本地或者是网络存储，只要重启一个容器挂载之前的存储就可以恢复了，这就
得介绍下docker data volume 也就是数据卷。

# data volume
docker 的数据卷，简单来说就是将 宿主机或网络存储 以文件或目录方式，可读写的挂载到容器中，这样即使容器挂了，但它数据还在,
从而实现了容器中的应用跟数据进行了解耦。

## volume type
docker中有两种类型的数据卷：
- bind-mount
- docker-managed 

### bind-mount
bind-mount 就是挂载指定宿主机目录或者是网络存储 到运行的容器中，通常是在容器启动时 -v 参数指定：
```bash
docker run -itd -v HOSTDIR:VOLUMEDIR:ro nginx

HOSTDIR: 一般是指宿主机本地路径
VOLUMEDIR: 指容器内部路径
ro: 只读挂载，可省略，默认为读写挂载。
```
使用 bind-mount 注意以下几点：
- 挂载的对象可以是目录，也可以是文件。
- -v 可以多个使用，进行多个文件或目录挂载，默认的权限是读写的，可以设置ro  
- bind-mount 如果挂载的是宿主机的目录或文件的时候，这样容器就跟宿主机进行了绑定，也就意味着限制了容器的可移植性。
- 挂载文件时：host 中的源文件必须要存在，不然会当作一个新目录 bind mount 给容器。

### docker-managed 
docker-managed 就是数据卷有docker自动管理，而非bind-mount 这样指定挂载
```bash
docker run -itd -v VOLUMEDIR nginx
只要指定 VOLUMEDIR 即可
```
docker-managed 有以下特点：
- docker-managed 默认会在宿主机 /var/lib/docker/volumes下 其生成一个随机目录，进行相应挂载。
- docker-managed 支持目录，不支持单个文件

### 两者对比
对比项|bind-mount|docker-managed 
---|---|---
volume 位置|可任意指定|一般为/var/lib/docker/volumes/...可以自定义
对已有挂载点 影响|隐藏并替换为 volume|原有数据复制到 volume
是否支持单个文件|支持|不支持，只能是目录
权限控制|可以设置只读或读写|无法控制，只能读写
移植性|移植性弱，与host path绑定|移植性强，无需指定host目录

# 容器数据共享
## 容器与host共享
1. bind-mount 方式：直接将目录mount到容器中
2. docker-managed 方式：docker cp 将内容拷贝到目标容器中
```bash
docker cp ./index.html containerid:/opt/html/
```
## 容器间共享
- 挂载宿主机同一个目录到容器中
- volume container
volume container 是专门为其他容器提供 volume 的容器。它提供的卷可以是 bind mount，也可以是 docker managed volume，
共享volume container 时 用 --volumes-from 参数指定。
```bash
[root@app001 html]# echo 'my nginx!!' > index.html
# volume container 一般只要create 就行了，不行run 起来
[root@app001 test]# docker create --name my-nginx -v /root/test/html:/usr/share/nginx/html/ nginx:latest
4f29f1d8f76d7c8338d46d47b920acd49fbbaac81b5a1b31babc9271692421f7
[root@app001 test]# docker run -itd --name test1 -p 1180:80 --volumes-from my-nginx nginx:latest
57d83a09b216801adfbc0c27df2924da0d484d10bbe9a5b700de40187c8752dc
[root@app001 test]# curl 127.0.0.1:1180
my nginx!!
[root@app001 test]# docker run -itd --name test2 -p 1181:80 --volumes-from my-nginx nginx:latest
f55396966e6bec9c4ac94512dee677d2bcfdf5d9ec0f3ee4180db961115c080d
[root@app001 test]# curl 127.0.0.1:1181
my nginx!!
  ```
> 使用volume container 后，我们不必要一个个去指定运行容器的 host path，所有path都在 volume container 中就定义好了，基于volume container run起来就完成了，
这样实现了容器跟 宿主机的解耦，有利于我们以后做大规模、标准化部署。
- data-packed volume container  
volume container 的数据归根到底还是在 host 上，要完全解耦host，只能通过将数据打包进镜像的方式
，然后 将此容器 当作 volume container 。
```bash
[root@app001 test]# cat Dockerfile
FROM nginx:latest
COPY html/index.html /usr/share/nginx/html/index.html
VOLUME /usr/share/nginx/html/
[root@app001 test]# docker build -t test.com/nginx:v1 .
Sending build context to Docker daemon  3.584kB
Step 1/3 : FROM nginx:latest
 ---> 62f816a209e6
Step 2/3 : COPY html/index.html /usr/share/nginx/html/index.html
 ---> 80b8656d3349
Step 3/3 : VOLUME /usr/share/nginx/html/
 ---> Running in 33659f1dd261
Removing intermediate container 33659f1dd261
 ---> a5000c3bc611
Successfully built a5000c3bc611
Successfully tagged test.com/nginx:v1
[root@app001 test]# docker create --name test03 test.com/nginx:v1
60bae84035d6a01576bcf2b28d84fe9b5fa892ff19cc430172ba4ac3d93257d2
[root@app001 test]# docker run -itd --name test04 -p 1182:80 --volumes-from test03 nginx:latest
550a75b39b283334fe31de4f593d9cadc7a7846b9889ff3fca063fe226b66966
[root@app001 test]# curl 127.0.0.1:1182
my nginx!!
```
> 使用 data-packed volume container 可以完全解耦 宿主机，具有很强的一致性，不过这种容器间数据共享的场景不多，基本很少使用。
