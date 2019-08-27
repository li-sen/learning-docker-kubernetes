> 这是本人 根据 以往的记录 整理出来的，后续 会逐步把之前的记录 添加到此，需要特别说明的是 此部署文档适用于 开发测试环境，如果要 使用在生产环境，还需要进行 不少优化填坑，望悉知。
```shell script
### redis
docker run --name redis-master -d --restart=always -e TZ="Asia/Shanghai" --network=host -P  -v /opt/redis/data:/data  redis:latest redis-server --appendonly yes --requirepass "123456"
docker run --name redis-slave -d --restart=always -e TZ="Asia/Shanghai" --network=host -P  -v /opt/redis/data:/data  redis:latest redis-server --slaveof redis-master 6379 --requirepass "123456" --masterauth "123456"

# https://blog.csdn.net/qq_39211866/article/details/88044546


### mysql
mkdir /opt/mysql -pv
cat >> /opt/mysql/my.cnf << EOF
[client]
default-character-set = utf8mb4

[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
init_connect = 'SET NAMES utf8mb4'
init_connect='SET collation_connection = utf8mb4_unicode_ci' 
skip-name-resolve
query_cache_size = 0
log-error = /data/db/error.log
slow-query-log = 1
long_query_time = 2
slow-query-log-file = /data/db/slow.log
expire-logs-days = 1
default-storage-engine = innodb
innodb-buffer-pool-size = 1G
lower_case_table_names = 1
performance-schema-instrument='memory/%=COUNTED'
interactive_timeout = 172800
wait_timeout = 172800
max_allowed_packet = 1024M
max_connections = 500

[mysql]
default-character-set = utf8mb4
EOF

docker run --name mysql -d --restart=always -e TZ="Asia/Shanghai" --network=host -P -v /opt/mysql/my.cnf:/etc/mysql/my.cnf -v /opt/mysql/logs:/logs -v /opt/mysql/data:/mysql_data -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7

# 主从复制
# 这里就贴下 主从相关的部分配置 其他配置 参考 单实例的 配置文件
master:
[root@localhost hsdp]# cat config/my.cnf 
[mysqld]
server_id = 1

log-bin= mysql-bin
read-only=1
innodb_flush_log_at_trx_commit=1
sync_binlog=1

replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
[root@localhost hsdp]# 

slave:
[root@localhost hsdp]# cat config/my.cnf 
[mysqld]
server_id = 2

log-bin= mysql-bin
read-only=0
innodb_flush_log_at_trx_commit=1
sync_binlog=1

replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
[root@localhost hsdp]#

# master:
GRANT REPLICATION SLAVE ON *.* to 'slave'@'%' identified by '123456';
mysql> show master status\G;
*************************** 1. row ***************************
             File: mysql-bin.000003
         Position: 590
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)

# slave：
change master to master_host='10.10.2.7',master_user='slave',master_password='123456',master_log_file='mysql-bin.000003',master_log_pos=590,master_port=3306;

//启动从库同步
start slave;
show slave status\G;
# 如果 show slave status\G命令结果中出现： Slave_IO_Running: Yes Slave_SQL_Running: Yes 以上两项都为Yes，那说明没问题了。

# 如在配置主从同步前master中已有数据，则需提前进行数据同步操作
登录master，执行锁表操作
mysql -u root -p1234 -h 127.0.0.1 -P 3306
FLUSH TABLES WITH READ LOCK;

将master中需要同步的db的数据dump出来，也可以直接 scp 主节点的 数据目录 到 从节点 进行数据同步
mysqldump -u root -p -h 127.0.0.1 -P 3306  mytest > mytest.dump 

将数据导入slave
mysql -u root -p -h 127.0.0.1 -P 3306 mytest < mytest.dump

按前文所述步骤开启主从同步，然后解锁master
UNLOCK TABLES;
# 参考连接：https://testerhome.com/topics/15056


### consul  
# server
docker run -d --name consul-server1 --restart=always -e TZ="Asia/Shanghai" --network=host -P -e'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' consul agent -server -bind=172.17.0.97 -bootstrap-expect=1 -client 0.0.0.0 -ui
172.17.0.97:8500/ui


### php
 docker run --name nginx -p 8080:80 -e TZ="Asia/Shanghai" -v /etc/localtime:/etc/localtime -v /opt/pacs/pacs:/data/www -v /opt/pacs/nginx/pacs.conf:/usr/local/nginx/conf/vhost/pacs.conf  -d skiychan/nginx-php7
 
php的编译安装完成了，接下来我们做phpredis的扩展：
下载phpredis的扩展包
https://github.com/nicolasff/phpredis/archive/2.2.4.tar.gz  下载redis的扩展包。
# 去github下新版
wget https://github.com/nicolasff/phpredis/archive/3.1.0.tar.gz

然后放到 /usr/local 下边，
解压 tar xzvf  2.2.4.tar.gz
cd 2.2.4 
/usr/local/php/bin/phpize   生成 configure的配置文件
如果不能生成configure的话： yum install -y autoconf 
./configure --with-php-config=/usr/local/php/bin/php-config 
make
make install

vim php.ini   最后一行添加
extension="redis.so"


### nacos
https://www.jianshu.com/p/c410845f0dca

> 部署 Nacos
1、部署 MySQL 5.7 集群 master & slave
Docker 部署方式请参考：使用 Docker 部署 MySQL 5.7 & 8.0 主从集群
2、创建数据库 nacos
docker run -it --rm -e TZ="Asia/Shanghai" --network common-network mysql mysql -hmysql-master -uroot -pPassw0rd \
 -e "create database nacos;"

3、在 mysql-master 上执行 SQL
SQL 文件：https://github.com/alibaba/nacos/blob/master/config/src/main/resources/META-INF/nacos-db.sql

# 进入容器
docker exec -it mysql-master bash

# 连接 mysql
mysql -pPassw0rd

# 执行 SQL
# 略。。。。

3、运行 Nacos (单机模式)
docker run -d \
--name nacos-server \
--network common-network \
-e PREFER_HOST_MODE=hostname \
-e MODE=standalone \
-e TZ="Asia/Shanghai" \
-e SPRING_DATASOURCE_PLATFORM=mysql \
-e MYSQL_MASTER_SERVICE_HOST=mysql-master \
-e MYSQL_MASTER_SERVICE_PORT=3306 \
-e MYSQL_MASTER_SERVICE_USER=root \
-e MYSQL_MASTER_SERVICE_PASSWORD=Passw0rd \
-e MYSQL_MASTER_SERVICE_DB_NAME=nacos \
-e MYSQL_SLAVE_SERVICE_HOST=mysql-slave \
-e MYSQL_SLAVE_SERVICE_PORT=3306 \
-p 8848:8848 \
nacos/nacos-server

> 访问 Nacos
基本信息
访问地址：http://localhost:8848/nacos
账号密码：nacos / nacos

相关链接
Nacos 官网：https://nacos.io/zh-cn/index.html
Nacos Github: https://github.com/alibaba/nacos
Nacos Docker Hub: https://hub.docker.com/r/nacos/nacos-server


### jenkins 
docker run -itd --name jenkins --memory 1G -p 18080:8080 -p 50000:50000 -v /opt/jenkins:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock -v /usr/bin/docker:/usr/bin/docker  -v /etc/localtime:/etc/localtime -v /etc/timezone:/etc/timezone -e TZ="Asia/Shanghai" --env JAVA_OPTS="-Xms1024m -Xmx1024m  -XX:MaxNewSize=512m"  -u root  -v /usr/lib64/libltdl.so.7:/usr/lib/x86_64-linux-gnu/libltdl.so.7 jenkins/jenkins:lts


### java容器时区问题
每一个docker中的容器或服务使用的默认时区都不是一致的，大概分为:
时区功能	    时区说明
Java	    JVM启动时候的时区。如果不指定，则默认取系统时区
MySQL	    数据库有时区，直接用：select now()可以查看确认。但是，Java的JDBC直接新增的数据是不会参考Mysql本身的时区。举个例子：INSERT什么时间就是什么时间！
Docker宿主	实际上是操作系统OS的时区。如果安装系统的时候选择中国上海时区，那一般是CST（中国时区）。
Docker容器	容器本身也是一个操作系统(虚拟系统)。当容器时区和宿主时区不一致，可以修改。如果有疑问可以直接exec到容器里面查看容器时区。
Tomcat服务	Tomcat是运行Java的Web应用服务器，所以，它的时区实际上就是JVM的时区。

方式一 修改 catalina.sh文件，加入一行jvm参数，使其默认时区为上海时区
JAVA_OPTS="$JAVA_OPTS -Djava.protocol.handler.pkgs=org.apache.catalina.webresources -Duser.timezone=GMT+08"
方式二 启动tomcat容器时就制定默认时区为上海时区
docker run -it -d 
--name tomcat1 
-p 8080:8080 
-v $PWD/logs:/usr/local/tomcat/logs 
-v $PWD/webapps:/usr/local/tomcat/webapps 
-v $PWD/conf:/usr/local/tomcat/conf 
-v /etc/localtime:/etc/localtime 
-e TZ="Asia/Shanghai" 
tomcat:7.0.62
或者
echo "Asia/Shanghai" > /etc/timezone
docker run -it -v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime:ro -v /etc/timezone:/etc/timezone:ro java /bin/sh

```