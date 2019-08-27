> 通过之前的讲解我们知道 docker是 通过 namespace 进行资源隔离、cgroup进行资源限制的，那今天我们来简单介绍下 docker的资源限制。
# 资源类型
这里先简单说下 linux系统下的 资源类型：
- 可压缩资源：
CPU资源是一种 可压缩资源，意思就是超额了 会挂起等待资源释放，不会对 应用进行任何处理；
- 不可压缩资源：
内存是一种不可压缩资源，一旦超额就会 OOM killer了，应用程序挂了。

# docker的 资源限制
docker 启动容器后 基本上不进行资源硬限制的，cpu 方面还好 CPU share 默认值为 1024（参数后面在解释），但 对内存是不进行限制的，
显而易见不做资源限制是很危险的，不符合实际业务需求; 尤其java 的后端应用经常性的oom，造成服务器的雪崩，所以做好 容器的资源限制 是保证
业务 稳定运行的一个重要且必须的手段。

那说到资源限制主要就是 三块内容： cpu、内存、io，docker 也不例外，接下来我们先来看下cpu。

# cpu限制
docker 启动 的容器 在宿主机角度上 看就是一个进程，进程使用 多线程cpu 默认为 CFS 调度算法，cpu 采用时间分片的 机制进行 资源分配，cpu 分配给
一个进程运行时间段，这个时间段允许 该进程使用cpu进行运算，时间结束后，cpu将会被其他进程 使用，cpu 就是按照这样的 规则依次的轮流运行不同的 进程，完成不同程序的计算任务。

## CPU Share
设置docker容器 cpu的权重，使用 --cpu-shares 参数设置 ，默认所有的容器对于 CPU 的利用占比都是一样的，-c 或者 --cpu-shares 可以设置 CPU 利用率权重，默认为 1024，
如果设置选项为 0，则系统会忽略该选项并且使用默认值 1024。

不过这个数值是相对的，比如你 宿主机上运行 2个容器 默认都为 1024 则 他们的 cpu权重都为50%，如果在把其中的一个容器 cpu-shares调整为512， 则 他的权重 为 67% 和 33%，
这样的分配方法好处就是 相对公平，充分利用了 cpu资源，但坏处就是 无法 给容器做 确定的硬限制。

我们这里来做一个测试实例：

我这里有台四线程的 虚机：
```shell script
[root@test-01 ~]# docker run   -ti   --rm  polinux/stress stress --cpu 4 --verbose

# top 情况：
top - 17:27:15 up 54 days,  7:22,  3 users,  load average: 7.69, 16.20, 10.31
Tasks: 183 total,   5 running, 113 sleeping,   0 stopped,   0 zombie
%Cpu(s): 98.3 us,  1.4 sy,  0.0 ni,  0.1 id,  0.0 wa,  0.0 hi,  0.1 si,  0.1 st
KiB Mem :  8167532 total,  3031192 free,  1492472 used,  3643868 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  6405688 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 1457 root      20   0     744     36      0 R  99.0  0.0   0:13.68 stress
 1458 root      20   0     744     36      0 R  98.0  0.0   0:13.63 stress
 1459 root      20   0     744     36      0 R  96.7  0.0   0:13.59 stress
 1460 root      20   0     744     36      0 R  95.7  0.0   0:13.43 stress
 5280 root      20   0   10.1g  99740  19328 S   3.3  1.2   3812:33 etcd
 6005 root      20   0  547500 380616  68304 S   3.0  4.7   3083:04 kube-apiserver


我这里在另外开个窗口 运行 下面的 命令：
[root@test-01 ~]# docker run   -ti   --rm  -c 512 polinux/stress  stress --cpu 4  --verbose

# 此时的 top情况：
top - 17:28:30 up 54 days,  7:24,  3 users,  load average: 7.99, 14.25, 10.07
Tasks: 189 total,   9 running, 116 sleeping,   0 stopped,   0 zombie
%Cpu(s): 96.7 us,  2.7 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.2 si,  0.3 st
KiB Mem :  8167532 total,  3004336 free,  1518308 used,  3644888 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  6379304 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 1460 root      20   0     744     36      0 R  63.5  0.0   1:16.35 stress
 1458 root      20   0     744     36      0 R  55.5  0.0   1:16.78 stress
 1457 root      20   0     744     36      0 R  55.1  0.0   1:17.08 stress
 1459 root      20   0     744     36      0 R  53.5  0.0   1:17.12 stress
 2333 root      20   0     744     32      0 R  31.6  0.0   0:08.04 stress
 2331 root      20   0     744     32      0 R  31.2  0.0   0:07.54 stress
 2332 root      20   0     744     32      0 R  31.2  0.0   0:08.35 stress
 2334 root      20   0     744     32      0 R  27.2  0.0   0:07.36 stress
 7343 root      20   0  284304 237632  27828 S  17.9  2.9   1180:18 calico-node
``` 
因为默认的 cpu-shares 为 1024，我第二个容器 指定为 512  从cpu使用率来看 两者之间的权重差不多为2：1 符合我们设置的预期。
如果想了解stress更多参数，使用 stress --help查看

## cpu 核数
docker 提供了 --cpus 参数来 限制容器的 cpu 核数，这个参数是一个 确定的硬性限制，而且 可以使用 小数来进行限制 譬如 2.5 表示现在 2.5 核，支持
精确到2位小数比如1.05，现在我们 同样的环境 来测试下：

```shell script
[root@test-01 ~]# docker run   -ti   --rm  --cpus 2 polinux/stress  stress --cpu 4  --verbose
# top情况：
top - 13:58:40 up 54 days,  3:54,  2 users,  load average: 0.41, 0.26, 0.25
Tasks: 183 total,   6 running, 115 sleeping,   0 stopped,   0 zombie
%Cpu0  : 55.0 us,  4.3 sy,  0.0 ni, 39.1 id,  0.3 wa,  0.0 hi,  0.0 si,  1.3 st
%Cpu1  : 53.5 us,  4.3 sy,  0.0 ni, 40.9 id,  0.0 wa,  0.0 hi,  0.0 si,  1.3 st
%Cpu2  : 54.0 us,  4.0 sy,  0.0 ni, 40.7 id,  0.3 wa,  0.0 hi,  0.0 si,  1.0 st
%Cpu3  : 52.2 us,  5.0 sy,  0.0 ni, 41.2 id,  0.0 wa,  0.0 hi,  0.7 si,  1.0 st
KiB Mem :  8167532 total,  2410060 free,  1500040 used,  4257432 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  6378660 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
32395 root      20   0     744     36      0 R  51.5  0.0   0:18.89 stress
32394 root      20   0     744     36      0 R  51.2  0.0   0:19.49 stress
32396 root      20   0     744     36      0 R  49.8  0.0   0:19.49 stress
32397 root      20   0     744     36      0 R  47.2  0.0   0:18.96 stress

# 看top情况符合预期设置，特别说明一点 如果设置 的核数 大于 大于宿主机实际的核数 会直接报错。
# 还有一点就是 如果 多个 容器 设置的 cpu 总 核数 超过 超过宿主机实际的核数，容器会根据 cpu-shares值 竞争 使用，所以--cpus参数设置的资源 只在 宿主机资源充足的情况下有保证。
 
```

## 指定运行核
docker 运行 指定 容器 只运行在哪个 cpu 核上， 通过 --cpuset 参数 设定，比如：
```shell script
[root@test-01 ~]# docker run   -ti   --rm  --cpuset-cpus=0,3 polinux/stress  stress --cpu 2  --verbose
# top情况
top - 18:04:47 up 54 days,  8:00,  2 users,  load average: 2.29, 2.61, 3.82
Tasks: 178 total,   3 running, 111 sleeping,   0 stopped,   0 zombie
%Cpu0  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  3.1 us,  1.0 sy,  0.0 ni, 94.9 id,  0.7 wa,  0.0 hi,  0.3 si,  0.0 st
%Cpu2  :  4.1 us,  2.0 sy,  0.0 ni, 93.6 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  : 99.7 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
KiB Mem :  8167532 total,  3051672 free,  1466128 used,  3649732 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  6432272 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
28738 root      20   0     744     36      0 R 100.0  0.0   2:09.77 stress --cpu 2 --verbose
28739 root      20   0     744     36      0 R 100.0  0.0   2:09.79 stress --cpu 2 --verbose

# 符合预期 只运行在 0 和 3 核上，不过这种用法 基本很少使用 了解即可。
```

## cpu cgroup文件
```shell script
[root@test-01 ~]# cat /sys/fs/cgroup/cpu/docker/6e77c4f070368725f919d5115a71d5c0bc4461a67865c552692248c1b2df83aa/cpu.shares
1024
[root@test-01 ~]# cat /sys/fs/cgroup/cpuset/docker/6e77c4f070368725f919d5115a71d5c0bc4461a67865c552692248c1b2df83aa/cpuset.cpus
0-3

# --cpus 限制 CPU 核数并不像上面两个参数一样有对应的文件对应，它是由 cpu.cfs_period_us 和 cpu.cfs_quota_us 两个文件控制的。如果容器的 --cpus 设置为 2，这两个文件值为：
[root@test-01 ~]# cat /sys/fs/cgroup/cpu/docker/8ff438e6e8adb83f22dad6d9c22aad18c61874d1afd414eca10148c26933ee72/cpu.cfs_period_us
100000
[root@test-01 ~]# cat /sys/fs/cgroup/cpu/docker/8ff438e6e8adb83f22dad6d9c22aad18c61874d1afd414eca10148c26933ee72/cpu.cfs_quota_us
200000
# 1.12 以及之前的版本，都是通过 --cpu-period 和 --cpu-quota 这两个参数控制容器能使用的 CPU 核数的。
# 前者表示 CPU 的周期数，默认是 100000，单位是微秒，也就是 1s，一般不需要修改；后者表示容器的在上述 CPU 周期里能使用的 quota，真正能使用的 CPU 核数就是 cpu-quota / cpu-period，因此对于 2 核的容器，对应的 cpu-quota 值为 200000。
```

# 内存限制
docker 默认是 对内存是 不做限制的，为了隔离应用相互影响，预防服务器雪崩，对容器 进行 内存的限制必不可少，现在来了解下 内存 相关的参数:

与操作系统类似，容器可使用的内存包括两部分：物理内存和 swap，如果物理内存充足的情况下，基本要避免使用swap，swap为硬盘区，性能极其低下；不过这里还是简单讲解下 Docker 下面的参数来控制这两种内存的使用量。

## 参数说明
- -m 或 --memory：物理内存大小，最小值为 4m(单位有b、k、m、g，分别对应 bytes、KB、MB、和 GB）
- --memory-swap：swap大小
- --memory-swappiness：默认情况下，主机可以把容器使用的匿名页（anonymous page）swap 出来，你可以设置一个 0-100 之间的值，代表允许 swap 出来的比例
- --memory-reservation：设置一个内存使用的 soft limit，如果 docker 发现主机内存不足，会执行 OOM 操作。这个值必须小于 --memory 设置的值
- --kernel-memory：容器能够使用的 kernel memory 大小，最小值为 4m。
- --oom-kill-disable：是否运行 OOM 的时候杀死容器。只有设置了 -m，才可以把这个选项设置为 false，否则容器会耗尽主机内存，而且导致主机应用被杀死

> 特别说明：
- --memory-swap是一个修饰符标志，只有在--memory设置时才有意义，
- 如果--memory-swap设置为正整数，那么 --memory和 --memory-swap必须设置。
  --memory-swap表示可以使用的物理内存和swap总量，--memory控制物理内存使用的大小。如果--memory="300m"和--memory-swap="1g"，容器可以使用300m的物理内存和700m（1g - 300m）swap。
- 如果--memory-swap设置为0，则忽略该设置，并将该值视为未设置。 
- 如果--memory-swap设置为与值相同的值--memory，并且--memory设置为正整数，则容器无权访问swap。  
- 如果--memory-swap未设置，--memory设置指定值，则容器可以使用两倍于--memory 值的 swap空间。例如，如果--memory="300m"和--memory-swap未设置，容器可以使用300m的物理内存和600m的swap。
- 如果--memory-swap明确设置为-1，则允许容器无限制使用swap。
  
  
 > 在容器内部，使用free命令查看的是主机的swap，而不是容器的swap。
 > 我们应当尽量防止容器使用swap：如果--memory和--memory-swap设置为相同的值，则可以防止容器使用任何swap。

让我们来看下示例：
```shell script
[root@test-01 ~]# docker run   -ti   --rm  -m 64m polinux/stress  stress  --vm 1 --vm-bytes 64M --vm-hang 0  --verbose
stress: info: [1] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
stress: dbug: [1] using backoff sleep of 3000us
stress: dbug: [1] --> hogvm worker 1 [6] forked
stress: dbug: [6] allocating 67108864 bytes ...
stress: dbug: [6] touching bytes in strides of 4096 bytes ...
stress: FAIL: [1] (415) <-- worker 6 got signal 9
stress: WARN: [1] (417) now reaping child worker processes
stress: FAIL: [1] (421) kill error: No such process
stress: FAIL: [1] (451) failed run completed in 0s
# 资源紧张 直接运行报错了
[root@test-01 ~]# docker run   -ti   --rm  -m 64m polinux/stress  stress  --vm 1 --vm-bytes 32 --vm-hang 0  --verbose
stress: info: [1] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
stress: dbug: [1] using backoff sleep of 3000us
stress: dbug: [1] --> hogvm worker 1 [6] forked
stress: dbug: [6] allocating 32 bytes ...
stress: dbug: [6] touching bytes in strides of 4096 bytes ...
stress: dbug: [6] sleeping forever with allocated memory
```
swap 和 kernel memory 基本用不到，推荐大家去官网了解下即可。

## cgroup 文件
```shell script
[root@test-01 ~]# cat /sys/fs/cgroup/memory/docker/635d83e973719c34664227a75bc0bcb4045f80e8f27f529c1cf071254848cb68/memory.limit_in_bytes
67108864
```

# IO限制
docker 在io 方面目前做到了 磁盘io限制，对于网络io限制基本为0，因为 现在的服务器网络千奇八怪、每个应用诉求也是不尽相同，尤其实际场景是 跨主机多节点的 网络模型，实现容器网络io限制依靠docker 本身实现 是很困难的，
不过有人 使用tc 可以 实现，有兴趣的 大家可以自行 Google。

docker 目前支持对磁盘的读写速度进行限制，但是并没有方法能限制容器能使用的磁盘容量（一旦磁盘 mount 到容器里，容器就能够使用磁盘的所有容量）。
而磁盘 读写限制 可以通过以下方面进行设置：

## 基于权重
默认情况下，所有容器能平等地读写磁盘，可以通过设置 blkio-weight 参数来改变容器 block IO 的优先级。
blkio-weight 与 cpu-shares 类似，设置的是相对权重值，默认为 500。在下面的例子中，container_A 读写磁盘的带宽是 container_B 的两倍。
```shell script
docker run -it --name container_A --blkio-weight 600 ubuntu   
docker run -it --name container_B --blkio-weight 300 ubuntu
```

## bps 和 iops
bps 是 byte per second，每秒读写的数据量。
iops 是 io per second，每秒 IO 的次数。

可通过以下参数控制容器的 bps 和 iops：
--device-read-bps，限制读某个设备的 bps。
--device-write-bps，限制写某个设备的 bps。
--device-read-iops，限制读某个设备的 iops。
--device-write-iops，限制写某个设备的 iops。

```shell script
[root@test-01 ~]# docker run -it --rm --device /dev/vda:/dev/sda --device-read-bps /dev/vda:1mb ubuntu:16.04 bash
root@826a2cbd8176:/# cat /sys/fs/cgroup/blkio/blkio.throttle.read_bps_device
253:0 1048576
root@826a2cbd8176:/# dd iflag=direct,nonblock if=/dev/sda of=/dev/null bs=5M count=10
10+0 records in
10+0 records out
52428800 bytes (52 MB, 50 MiB) copied, 50.0223 s, 1.0 MB/s
root@826a2cbd8176:/#
# bps也符合预期

[root@test-01 ~]# docker run -it --rm --device /dev/vda:/dev/sda --device-read-iops /dev/vda:100 ubuntu:16.04 bash
root@bea9d4d2abde:/# dd iflag=direct,nonblock if=/dev/sda of=/dev/null bs=1k count=1000
1000+0 records in
1000+0 records out
1024000 bytes (1.0 MB, 1000 KiB) copied, 9.91174 s, 103 kB/s
root@bea9d4d2abde:/#
# 从测试中可以看出，容器设置了读操作的 iops 为 100，在容器内部从 block 中读取 1m 数据（每次 1k，一共要读 1000 次），共计耗时约 10s，换算起来就是 100 iops/s，符合预期结果。
# 其他的参数，大家自行参考测试，这里就不一一列出了。
```

## cgroup 文件
```shell script
[root@test-01 ~]# cat /sys/fs/cgroup/blkio/docker/bea9d4d2abde8bb1f6a741db017aacb2a270e98af36fb91626bf5a5dfd18df13/blkio.throttle.read_iops_device
253:0 100
[root@test-01 ~]# cat /sys/fs/cgroup/blkio/docker/8f5f2a948c4e6d72b235c33f467d29c8a88c3f1d6a8a07c9115c1a7d8ed557dd/blkio.throttle.read_bps_device
253:0 1048576
[root@test-01 ~]#
```

# LXCFS
在容器中使用top、free等命令仍然是一个较为普遍存在的需求，但是容器中的/proc、/sys目录等还是挂载的宿主机目录，有一个开源项目：LXCFS。LXCFS是基于FUSE实现的一套用户态文件系统，使用LXCFS，让你在容器里面继续使用free等命令变成了可能。注意，LXCFS目前只支持为容器生成下面文件：
```shell script
/proc/cpuinfo
/proc/diskstats
/proc/meminfo
/proc/stat
/proc/swaps
/proc/uptime
```

如果命令是通过解析这些文件实现，那么在容器里面可以继续使用，否则只能通过读取CGroups获取资源情况。

## LXCFS 测试
```shell script
wget https://copr-be.cloud.fedoraproject.org/results/ganto/lxd/epel-7-x86_64/00486278-lxcfs/lxcfs-2.0.5-3.el7.centos.x86_64.rpm
yum install lxcfs-2.0.5-3.el7.centos.x86_64.rpm -y
systemctl start lxcfs

[root@test-01 ~]# docker run -it --rm -m 256m  --cpus 2 --cpuset-cpus "0,1" \
>       -v /var/lib/lxcfs/proc/cpuinfo:/proc/cpuinfo:rw \
>       -v /var/lib/lxcfs/proc/diskstats:/proc/diskstats:rw \
>       -v /var/lib/lxcfs/proc/meminfo:/proc/meminfo:rw \
>       -v /var/lib/lxcfs/proc/stat:/proc/stat:rw \
>       -v /var/lib/lxcfs/proc/swaps:/proc/swaps:rw \
>       -v /var/lib/lxcfs/proc/uptime:/proc/uptime:rw \
>       ubuntu:latest /bin/sh
# free -h
              total        used        free      shared  buff/cache   available
Mem:           256M        1.0M        254M         49M        168K        254M
Swap:          256M          0B        256M

# top
top - 03:32:42 up 0 min,  0 users,  load average: 0.38, 0.86, 0.86
Tasks:   2 total,   1 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu0  :  3.4 us,  1.7 sy,  0.0 ni, 95.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  3.0 us,  2.0 sy,  0.0 ni, 94.6 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :   262144 total,   260600 free,      848 used,      696 buff/cache
KiB Swap:   262144 total,   262144 free,        0 used.   260600 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0    4628    852    788 S   0.0  0.3   0:00.04 sh
    8 root      20   0   36632   3212   2748 R   0.0  1.2   0:00.00 top
```

# jvm run in docker
这里简单说下 容器中跑 java应用 关于jvm的设置问题，首先我们要明白的是
- docker中的jvm检测到的是宿主机的内存信息，它无法感知容器的资源上限，这样可能会导致意外的情况。
- -m参数用于限制容器使用内存的大小，超过大小时会被OOMKilled。
- -Xmx:  默认为物理内存的1/4。

基于以上的问题，在容器中的 java应用必须的做相关措施，才能保证 应用稳定运行。
1. 必须设置 容器的 内存限制，避免内存泄露 造成 服务器雪崩。
2. 应用启动必须设置Xmx的值，可设置为容器上限减去256m，根据具体业务考虑，因为栈内存等是不包含在堆内存中的。
4g 内存的 容器 jvm推荐设置： -Xmx2688M -Xms2688M -Xmn960M -XX:MaxMetaspaceSize=256M -XX:MetaspaceSize=256M
3. 最好开启 LXCFS，并在容器启动时进行相关挂载。 

# 参考链接
[docker官方文档](https://docs.docker.com/config/containers/resource_constraints/)
[参考blog](https://cizixs.com/2017/08/04/docker-resources-limit/)
