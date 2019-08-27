# 前言
docker 拥有多方面很好的特性如镜像技术、轻量快速、移植性好等等，目前来说最大的短板应该是 网络以及 隔离方面，今天来说说docker 网络。

# docker 网络实现
docker现有的网络模型主要是通过使用network namespace、linux Bridge、iptables、veth pair等技术实现的。
- network namespace：主要提供了关于网络资源的隔离，包括网络设备、IPv4和IPv6协议栈、IP路由表、防火墙、/proc/net目录、/sys/class/net目录、端口（socket）等。
- linux Bridge：功能相当于物理交换机，为连在其上的设备（容器）转发数据帧。如docker0网桥。
- iptables：主要为容器提供NAT以及容器网络安全。
- veth pair：两个虚拟网卡组成的数据通道。在Docker中，用于连接Docker容器和Linux Bridge。一端在容器中作为eth0网卡，另一端在Linux Bridge中作为网桥的一个端口。

# docker网络模式
我们创建docker容器时，可以用 \-\-network string 参数 来指定四种网络模式：host、container、bridge（默认）、none。
```bash
[root@app001 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
26760ef96e10        bridge              bridge              local
62ace721e9c6        host                host                local
44f0735d2a7d        none                null                local
```
> container 此处没有展现，那接下来我们就简单了解下这四种模式。 

## host
之前介绍过，docker 是用过 namespace 对容器进行 资源隔离，每个容器都有独立的 PID、Mount 等等，\-\-network host 表示 容器与宿主机共用一个network namespace，
也是说跟宿主机共用网卡 路由等。
```bash
# --network host
[root@app001 ~]# docker run -itd --name my-web --network host  nginx
6205f8c5b91f122be6ce4a9b017dcfe80fc673830cd47de39bc8061a5af4470d
[root@app001 ~]# netstat -lnpt|grep 80
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      13067/nginx: master
# 也可以进容器看下ip地址 容器ip 是跟 宿主机ip 一致。
[root@app001 ~]# docker exec -it my-web bash
root@app001:/# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.234  netmask 255.255.255.0  broadcast 192.168.0.255
        ether 00:16:3e:14:69:89  txqueuelen 1000  (Ethernet)
        RX packets 8769906  bytes 5816538203 (5.4 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5732760  bytes 12237794673 (11.3 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@app001:/# exit
exit
[root@app001 ~]# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.234  netmask 255.255.255.0  broadcast 192.168.0.255
        ether 00:16:3e:14:69:89  txqueuelen 1000  (Ethernet)
        RX packets 8769961  bytes 5816542722 (5.4 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5732792  bytes 12237806084 (11.3 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

## container
容器模式，跟host模式类似，只不过其网络命名空间共享的对象为一个容器而非宿主机。
```bash
# --network "container:name/ID"
[root@app001 ~]# docker run  -itd --name test01 busybox
[root@app001 ~]# docker run  -itd --name test02  --network "container:test01" busybox
[root@app001 ~]# docker exec -it test02 sh
/ # ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # exit
[root@app001 ~]# docker exec -it test01 sh
/ # ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ #
```

## none
none 表示容器默认有自己的network namespace，但不为Docker容器进行任何网络配置，
这将所有网络创建操作完全自定义，实现更加灵活复杂的网络。
```bash
# --network none
[root@app001 ~]# docker run  -itd --name test01  --network none busybox
6ca8ad797625a46d3a5af4af4bc13605b6677507e551f81a19ef3d0f15e0496d
# 只会看到lo 网络信息
[root@app001 ~]# docker exec -it test01 sh
/ # ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
/ # ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ #
```

## bridge
桥接模式，docker run 默认模式，此模式会为容器分配network namespace、设置IP等，
并将容器桥接到一个虚拟网桥docker0上，可以和同一宿主机上桥接模式的其他容器进行通信。
```bash
# --network bridge (默认方式，可以不用指明)
[root@app001 ~]# docker run -itd --name test01 busybox
482e701f3f3dc07b4602e7261ac4917433d2fc491bc501a766e9e4636d6d6591
[root@app001 ~]# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.234  netmask 255.255.255.0  broadcast 192.168.0.255
        ether 00:16:3e:14:69:89  txqueuelen 1000  (Ethernet)
        RX packets 8784552  bytes 5826618359 (5.4 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5739538  bytes 12240873302 (11.4 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# ping 宿主机是通的，ping 同一bridge上的其他容器也是通的
[root@app001 ~]# docker exec -it test01 sh
/ # ping 192.168.0.234
PING 192.168.0.234 (192.168.0.234): 56 data bytes
64 bytes from 192.168.0.234: seq=0 ttl=64 time=0.103 ms
64 bytes from 192.168.0.234: seq=1 ttl=64 time=0.076 ms
64 bytes from 192.168.0.234: seq=2 ttl=64 time=0.075 ms
64 bytes from 192.168.0.234: seq=3 ttl=64 time=0.081 ms
^C
--- 192.168.0.234 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.075/0.083/0.103 ms
/ #
```

# bridge模式网络实现
> - 虚拟网桥：可以理解成物理交换机
> - veth pair：可以理解成一根虚拟网线的两端，成对出现的数据通道。它有一个特性就是:数据从这边进，肯定会从另一端出，所以veth设备常用来连接两个网络设备。

## 创建docker0虚拟网桥
```bash
docker server 启动--> 创建docker0虚拟网桥-->分配私网地址例如172.17.0.1/16(选择与宿主机网络不同的网段)

[root@app001 ~]# ifconfig docker0
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:bf:72:76:96  txqueuelen 0  (Ethernet)
        RX packets 32772  bytes 1857581 (1.7 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 76910  bytes 251083015 (239.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
[root@app001 ~]# docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "26760ef96e1038a0671b7678185987965332a73741a9642ee0223db01b17bff4",
        "Created": "2018-12-13T10:26:48.508623339+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```
> 这里可以看出docker bridge的默认ip为172.17.0.1/16，上面已连接了0个容器（"Containers": {}）

## 创建容器网络
当我们docker run 一个容器时，会第一时间创建一对veth pair设备：一端放入创建的容器中，重命名为eth0，然后在172.17.0.0/16网段中分配一个ip地址
并设置docker0的ip地址为容器网关，另一端放入docker0 的虚拟网桥上，从而实现容器与docker0网桥通信。
```bash
# 起初docker0 bridge 上没有接口
[root@app001 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242bf727696	no

# 创建一个容器
[root@app001 ~]# docker run -itd --name test01 busybox
9f0dfa02b23794adde393dd32bf268230f61610554f498ec7082abd0079186e3

# 查看docker0 bridge上 多了一个test01的容器
[root@app001 ~]# docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "26760ef96e1038a0671b7678185987965332a73741a9642ee0223db01b17bff4",
        "Created": "2018-12-13T10:26:48.508623339+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "9f0dfa02b23794adde393dd32bf268230f61610554f498ec7082abd0079186e3": {
                "Name": "test01",
                "EndpointID": "4528c1aa442c0e038c073352b236a58b1c891854fd9f61081e0602daa5d387d5",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]

# docker0 bridge上多了一个veth pair设备
[root@app001 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.0242bf727696	no		vethae526de
[root@app001 ~]# ip addr show vethae526de
84: vethae526de@if83: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP
    link/ether 8a:e3:07:f5:39:21 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    
# 测试连通性
[root@app001 ~]# docker exec -it test01 sh
/ # ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
83: eth0@if84: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
/ # ping 172.17.0.1
PING 172.17.0.1 (172.17.0.1): 56 data bytes
64 bytes from 172.17.0.1: seq=0 ttl=64 time=0.103 ms
64 bytes from 172.17.0.1: seq=1 ttl=64 time=0.085 ms
64 bytes from 172.17.0.1: seq=2 ttl=64 time=0.083 ms
64 bytes from 172.17.0.1: seq=3 ttl=64 time=0.093 ms
64 bytes from 172.17.0.1: seq=4 ttl=64 time=0.077 ms
64 bytes from 172.17.0.1: seq=5 ttl=64 time=0.077 ms
64 bytes from 172.17.0.1: seq=6 ttl=64 time=0.090 ms
^C
--- 172.17.0.1 ping statistics ---
7 packets transmitted, 7 packets received, 0% packet loss
round-trip min/avg/max = 0.077/0.086/0.103 ms
/ #
```
> 如果找不到brctl，yum install bridge-utils

## bridge网络拓扑图
![网络拓扑](https://lisen-imgs.oss-cn-hangzhou.aliyuncs.com/learning-docker/docker_network.png)

由于docker0 bridge上的所有容器都在同一网段中，所以此bridge上的容器可以成功通信，
如果有安全要求，可以在DOCKER_OPTS变量中设置–icc=false，禁止它们之间通信，这样只有使用--link才能使两个容器通信。

## 链接另一个容器
我们可以使用--link使用另一个容器的服务。
```bash
# 创建mysql容器
docker run --name my_db -e MYSQL_ROOT_PASSWORD=123456 -d mysql

#创建一个应用，并使用刚才创建的mysql容器服务（网络是连通的）
docker run --name myweb --link my_db:mysql -p 8001:80 -d nginx
```
> 实际情况很少使用link，后续新版本提供了docker network来替代 link方式，大家了解就好。

## bridge网络连通性
- 不通的bridge之间默认不通的，要相通，采取以下措施：
   1. host开启路由转发: net.ipv4.ip_forward = 1
   2. 为 需要跟其他网桥通信的容器添加一块 指定该网桥 的网卡，这个可以通过docker network connect 命令实现。
   或者修改iptables规则: iptables DROP 掉了网桥 docker0 与 其他网桥 之间双向的流量（不推荐修改）
    > 不过 一般已经分成了多个网桥了，目的就是为了隔离通信，基本很少做这种多余操作。

- 默认可以访问外网  
  主机上有有iptables NAT规则：
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE

- 外部访问容器  
  DNAT端口映射: docker run -p 80:80 nginx
  host 都会启动一个 docker-proxy 进程来处理访问DNAT的流量

## 自定义 容器网络
- 改docker0 网段  
  vim /etc/docker/daemon.json: {"bip": "192.16.5.1/24"}
- 改docker0 dns
  - docker run --dns DNS_SERVER: docker run -itd --dns 8.8.8.8 busybox:latest 
  - vim /etc/docker/daemon.json: {"dns": ["8.8.8.8", "114.114.114.114"]}
> 注意是json格式 引号为 双引号

> bip: bridge ip，只要改它就行，后面的网段，默认网关会自动计算得出

# docker跨主机通信
docker容器 跨主机通信可以通过 host 这种方式，也可以通过 bridge NAT方式，
不过这种方式很容易端口冲突，也难管理，并且有比较大的性能损耗，更是无法做到网络隔离，基本不建议使用。

目前docker跨主机通信主流方式是Overlay网络，具体的项目有：Flannel、Calico 等等，不过在这里我就不细讲，
后续会在k8s部分详解这两个主流方案。