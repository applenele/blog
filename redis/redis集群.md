## redis 集群



#### redis安装

```shell
wget http://download.redis.io/releases/redis-4.0.6.tar.gz

 tar -zxvf redis-4.0.6.tar.gz

yum install gcc

 cd redis-4.0.6
 make MALLOC=libc
 cd src && make install
 
```

#### redis 启动

修改配置文件

```properties
daemonize no
改为
daemonize yes #(开启后台运行)

开启远程
第一步： 
修改bind 127.0.0.1改为 #bind 127.0.0.1 
第二步：修改 protected-mode no 
第三步：设置密码(远程访问需要设置密码才能生效)： 
找到redis.cli的安装目录，运行redis.cli客户端;执行命令： 
config set requirepass yourpassword 

```

启动

```shell

cd src

./redis-server /usr/local/redis-4.0.6/redis.conf 
```





#### 主从复制

 修改配置文件 redis.conf

SAVEOF <master_ip> <master_port>

**设置主从复制，从服务器只能读。主服务器可读写**

查看集群详情：输入 info命令



#### redis 哨兵机制

> https://www.cnblogs.com/jaycekon/p/6237562.html

1. 主要是保证redis 集群的高可用

2. 心跳检测主备的主节点。

3. 如果主节点挂了，就从从节点里面选一个做为主节点

##### 配置

修改sentinel.conf 文件

1、配置端口

   在sentinel.conf 配置文件中， 我们可以找到port 属性，这里是用来设置sentinel 的端口，一般情况下，至少会需要三个哨兵对redis 进行监控，我们可以通过修改端口启动多个sentinel 服务。

```properties
# port <sentinel-port>
# The port that this sentinel instance will run on
port 26379
```

2、配置主服务器的ip 和端口

我们把监听的端口修改成6380，并且加上权值为2，这里的权值，是用来计算我们需要将哪一台服务器升级升主服务器

```properties
sentinel monitor mymaster 127.0.0.1 6380 2
```

3、启动Sentinel

```properties
/sentinel$ redis-sentinel sentinel.conf
```



#### redis 集群

> <http://blog.jobbole.com/114270/>

即使使用哨兵，redis每个实例也是全量存储，每个redis存储的内容都是完整的数据，浪费内存且有木桶效应。为了最大化利用内存，可以采用cluster群集，就是分布式存储。即每台redis存储不同的内容。

采用redis-cluster架构正是满足这种分布式存储要求的集群的一种体现。redis-cluster架构中，被设计成共有16384个hash slot。每个master分得一部分slot，其算法为：hash_slot = crc16(key) mod 16384 ，这就找到对应slot。采用hash slot的算法，实际上是解决了redis-cluster架构下，有多个master节点的时候，数据如何分布到这些节点上去。key是可用key，如果有{}则取{}内的作为可用key，否则整个可以是可用key。**群集至少需要3主3从**，且每个实例使用不同的配置文件。

![redis-cluster](./images/redis-cluster.png)







#### spring boot 连接主备redis

跟连接单机的redis没有区别，直接指定redis的地址是master即可



#### spring boot 连 redis cluster

引用pom文件与application.yml配置文件

```properties
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>
```

```properties
spring:
  redis:
    jedis:
      pool:
        max-wait:5000
        max-Idle:50
        min-Idle:5
    cluster:
      nodes:127.0.0.1:7001,127.0.0.1:7002,127.0.0.1:7003,127.0.0.1:7004,127.0.0.1:7005,127.0.0.1:7006
    timeout:500
```



#### spring boot 连接到哨兵

引用pom

```properties
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>
```

配置文件

```properties
########################################################
###REDIS (RedisProperties) redis基本配置；
########################################################
# database name
spring.redis.database=0
# server host1 单机使用，对应服务器ip
#spring.redis.host=127.0.0.1  
# server password 密码，如果没有设置可不配
#spring.redis.password=
#connection port  单机使用，对应端口号
#spring.redis.port=6379
# pool settings ...池配置
spring.redis.pool.max-idle=8
spring.redis.pool.min-idle=0
spring.redis.pool.max-active=8
spring.redis.pool.max-wait=-1
# name of Redis server  哨兵监听的Redis server的名称
spring.redis.sentinel.master=mymaster
# comma-separated list of host:port pairs  哨兵的配置列表
spring.redis.sentinel.nodes=127.0.0.1:26379,127.0.0.1:26479,127.0.0.1:26579
```



#### redis java 客户端

1. jedis
2. lettuce (spring boot 2.0 默认)