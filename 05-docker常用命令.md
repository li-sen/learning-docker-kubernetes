> 在这里记录下docker常用的基础命令，一些不常用的参数我这就不一一写了。

# dokcer 版本信息
```bash
1. docker version
# 查看docker 版本信息
docker version
 
2. docker info
# 查看宿主机docker 详细信息
docker info
```

# docker 容器操作
```bash
1. docker run
# 运行容器
docker run --name mydocker -itd centos
# --name 容器别名，默认随机命名
# -i 交互式界面，默认是false
# -t 伪终端，默认false
# -d 让容器在后台运行
# --rm 运行后删除

# docker网络
# 新建docker网络
docker network create redis-net
docker run --name redis -d --restart=always -e TZ="Asia/Shanghai" --network=redis-net -P  -v /opt/redis/data:/data  redis:latest redis-server --appendonly yes --requirepass "xxxxx"
# 查看新建网络
docker network inspect  redis-net

# 端口映射
docker run -itd -p 80 nginx:v1 /bin/bash  # 端口随机
docker run -itd -p 8080:80 nginx:v1 /bin/bash # 端口指定
docker run -itd -p 8080:80:udp nginx:v1 /bin/bash # 指定协议
docker run -itd -p 0.0.0.0:80 nginx:v1 /bin/bash # 所有ip地址随机端口
docker run -itd -p 0.0.0.0:8080:80 nginx:v1 /bin/bash # 所有ip指定端口
docker run -itd -P nginx:v1 /bin/bash # 将容器中暴露的全部端口映射到宿主机随机端口

# 数据卷
docker run  -itd --name test1 -v /data1  ubuntu
# 挂载本地已有目录到容器中
docker run  -itd  --name test2 -v /tmp/data2:/data2  ubuntu
# 挂载本地已有目录到容器中，指定只读
docker run  -itd  --name test3  -v /tmp/data3:/data3:ro ubuntu
# 默认读写

# 数据卷容器
docker run -itd --name test04 -v /opt:/opt centos
docker run -itd --name test05 --volumes-from test04 centos
# 只会同步/opt下的内容，其他目录不同步，就算test04状态为exit，一样可以提供数据服务


2. docker ps 
# 查看宿主机运行或停止docker容器

# 查看运行中的docker容器
docker ps

# 显示所有容器（包括停止 退出）
docker ps -a

3. docker exec
# 进入容器
docker exec -it mydocker /bin/bash

4. docker logs
# 查看容器标准输出
docker logs -f mydocker

5. docker stats
# 查看宿主机容器运行资源情况

6. docker top
# 查看容器内部运行的进程
docker top mydocker

7. docker inspect
# 查看容器或镜像 详细信息 ip地址、挂载情况、执行的命令等
docker inspect mydocker

8. docker start/stop/kill
# 重启/停止/杀死容器
docker stop mydocker
docker start mydocker
docker kill mydocker

9. docker rm 
# 删除容器
docker rm mydocker
docker rm -f mydocker
# 默认如果容器启动时无法删除，-f 强制删除

10. docker port
# 查看容器端口

11. docker cp
# 从容器里面拷贝文件/目录到本地一个路径  
docker cp Name:/container_path to_path  
docker cp ID:/container_path to_path

12. docker 查看容器ip
docker inspect --format='{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -aq)

13. docker 内存 排序
docker stats --no-stream --format "table {{.Name}}\t{{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}" | sort -k 4 -h
14. cpu排序
docker stats --no-stream --format "table {{.Name}}\t{{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}" | sort -k 3 -h
```

# docker 镜像操作
```bash
1. docker login
# 登录镜像仓库
docker login harbor.xxx.com -u admin -p xxxxxx

2. docker logout
# 登出镜像仓库
docker logout harbor.xxx.com

3. docker search 
# 查找镜像，--filter=stars= stars数大于100的镜像
docker search mysql --filter=stars=100

4. docker pull
# 拉取镜像
docker pull mysql:5.7

# 同一个镜像，做验证，防止篡改            
# docker pull mysql@sha256:c23e9bfe66eeffc990cf6bce4bb0e9c5c85eb908170f3b3dde3e9a12c5a91689

5. docker inspect 
# 查看镜像/容器的元数据
docker inspect mysql:5.7

6. docker images
# 查看本地镜像
docker images

7. docker rmi
# 删除本地镜像
docker rmi nginx

# 按IMAGE ID 删除
docker rmi ea4c82dcd15a

# 强制删除，一般删除需要先stop才能删除
docker rmi -f nginx

8. docker commit
# 定制镜像，利用centos基础镜像，安装mariadb，做一个mariadb镜像
docker run --name mydb -itd centos:7
docker exec -it mydb /bin/bash  # 进入容器后 yum install -y mariadb
# 查看改动
docker diff mydb 
docker commit -m 'mariadb' -a 'lisen' mydb mariadb:v1
docker images |grep mariadb
# 一般不推荐使用commit制作镜像，在特殊场合如被入侵中毒了，为了保持现场可以使用；制作镜像推荐用Dockerfile。
 
9. docker save
# 导出镜像
docker save  ubuntu:latest |gzip > ubuntu.gz

10. docker load
# 导入镜像
docker load < ubuntu.gz

11. docker tag
# 镜像打标签
docker tag  ubuntu:latest  harbor.xxx.com/pub/ubuntu:latest

12. docker push
# 推送镜像
docker push harbor.xxx.com/pub/ubuntu:latest

13. dockerfile构建
cat Dockerfile
FROM java:8-jre
MAINTAINER Li Sen <lisen2023@gmail.com>
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo 'Asia/Shanghai' >/etc/timezone && \
    echo "export LC_ALL=en_US.UTF-8" >> /etc/profile && \
    . /etc/profile

docker build -t 172.16.100.86/pub/springboot:v1 .
docker push 172.16.100.86/pub/springboot:v1
docker images
```

# volume管理
```bash
# 查看宿主机上所有volume
docker volume ls
# 查看某一容器具体的volume信息
docker inspect <containerid>
# 删除某一个volume
docker volume rm <volumeid>
如果想批量删除孤儿 volume，可以执行：
docker volume rm $(docker volume ls -q)
```

# 其他常用命令
```bash

# 新建docker网络
docker network create redis-net

# docker清理
# 列出数据卷。
docker volume ls
# 列出 network
docker network ls
# 查看Docker的磁盘使用情况
docker system df

#删除已停止的容器、dangling 镜像、未被容器引用的 network 和构建过程中的 cache
docker system prune 
# --volumns 删除无用数据卷，默认不删除
# --force 强制删除，无提示
# --all 参数后会删除所有未被引用的镜像而不仅仅是 dangling 镜像。
# dangling images：未被任何镜像引用的镜像，比如在你重新构建了镜像后，那些之前构建的且不再被引用的镜像层就变成了 dangling images：
表现为 tag 为<none>；还有一种 <none> 镜像，它们的 repository 和 tag 列都表现为 <none>，此种镜像被称为 intermediate 镜像(就是其它镜像依赖的层)


# 我们还可在不同在子命令下执行 prune，这样删除的就是某类资源：
# 删除所有退出状态的容器
docker container prune
# 删除未被使用的数据卷
docker volume prune
# 删除 dangling 或所有未被使用的镜像
docker image prune 

# 清理所有docker资源（慎用）
docker container stop $(docker container ls -a -q) && docker system prune --all --force --volumns

# 和前面的 prune 命令类似，也可以完全删除某一类资源：
# 删除容器
docker container rm $(docker container ls -a -q)
# 删除镜像
docker image rm $(docker image ls -a -q)
# 删除数据卷
docker volume rm $(docker volume ls -q)
# 删除 network
docker network rm $(docker network ls -q)

# .dockerignore使用

类似.gitignore一样，运行Dockerfile里的COPY指令的时候会根据.dockerignore进行部分目录或者文件忽略。


# docker 升级脚本
cat update_docker.sh
#!/bin/bash
node=$1
role=$2
kubectl drain $node --ignore-daemonsets --delete-local-data --force \
&& sleep 30 \
&& kubectl delete node  $node \
&& echo "$node $role delete  successful!" \
&& sleep 3 \
&& ansible $node -m copy -a 'src=/opt/bin/docker/xxxx dest=/opt/bin/docker/xxxx mode=0744' \
&& ansible $node -m shell -a "systemctl restart docker" \
&& sleep 3 \
&& ansible $node -m shell -a "systemctl restart kubelet" \
&& sleep 30 \
&& kubectl uncordon $node \
&& kubectl label node $node kubernetes.io/role=$role --overwrite \
&& echo "$node $role update docker successful!"

ansible $node -m shell -a "docker ps -a && docker info"

# Debian GNU/Linux 9 :  容器安装软件
sed -i "s@http://deb.debian.org@http://mirrors.163.com@g" /etc/apt/sources.list
cat /etc/apt/sources.list
apt-get update
apt-get install net-tools
apt-get install dnsutils

```