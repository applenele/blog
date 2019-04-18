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

1. 主要是保证redis 集群的高可用

2. 心跳检测主备的主节点。

3. 如果主节点挂了，就从从节点里面选一个做为主节点

<https://www.cnblogs.com/jaycekon/p/6237562.html>

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

即使使用哨兵，redis每个实例也是全量存储，每个redis存储的内容都是完整的数据，浪费内存且有木桶效应。为了最大化利用内存，可以采用cluster群集，就是分布式存储。即每台redis存储不同的内容。

采用redis-cluster架构正是满足这种分布式存储要求的集群的一种体现。redis-cluster架构中，被设计成共有16384个hash slot。每个master分得一部分slot，其算法为：hash_slot = crc16(key) mod 16384 ，这就找到对应slot。采用hash slot的算法，实际上是解决了redis-cluster架构下，有多个master节点的时候，数据如何分布到这些节点上去。key是可用key，如果有{}则取{}内的作为可用key，否则整个可以是可用key。**群集至少需要3主3从**，且每个实例使用不同的配置文件。

![redis-cluster](./images/redis-cluster.png)

- [
  首页](http://blog.jobbole.com/)
- [最新文章](http://blog.jobbole.com/all-posts/)
- [IT 职场](http://blog.jobbole.com/category/career/)
- [前端](http://web.jobbole.com/)
- [后端](http://blog.jobbole.com/114270/#)
- [移动端](http://blog.jobbole.com/114270/#)
- [数据库](http://blog.jobbole.com/tag/database/)
- [运维](http://blog.jobbole.com/114270/#)
- [其他技术](http://blog.jobbole.com/114270/#)





[伯乐在线](http://www.jobbole.com/) > [首页](http://blog.jobbole.com/) > [所有文章](http://blog.jobbole.com/all-posts/) > [IT技术](http://blog.jobbole.com/category/it-tech/) > Redis 架构演变与 Redis-cluster 群集读写方案



# Redis 架构演变与 Redis-cluster 群集读写方案

2018/08/14 · [IT技术](http://blog.jobbole.com/category/it-tech/) · [Redis](http://blog.jobbole.com/tag/redis/), [数据库](http://blog.jobbole.com/tag/database/)



原文出处： [PingLee](https://my.oschina.net/u/2600078/blog/1923696)   

## 导言

Redis-cluster 是近年来 Redis 架构不断改进中的相对较好的 Redis 高可用方案。本文涉及到近年来 Redis 多实例架构的演变过程，包括普通主从架构（Master、slave 可进行写读分离）、哨兵模式下的主从架构、Redis-cluster 高可用架构（Redis 官方默认 cluster 下不进行读写分离）的简介。同时还介绍使用Java的两大redis客户端：Jedis与Lettuce用于读写redis-cluster的数据的一般方法。再通过官方文档以及互联网的相关技术文档，给出redis-cluster架构下的读写能力的优化方案，包括官方的推荐的扩展redis-cluster下的Master数量以及非官方默认的redis-cluster的读写分离方案，案例中使用Lettuce的特定方法进行redis-cluster架构下的数据读写分离。

 

## 近年来redis多实例用架构的演变过程

redis是基于内存的高性能key-value数据库，若要让redis的数据更稳定安全，需要引入多实例以及相关的高可用架构。而近年来redis的高可用架构亦不断改进，先后出现了本地持久化、主从备份、哨兵模式、redis-cluster群集高可用架构等等方案。

 

### 1、redis普通主从模式

通过持久化功能，Redis保证了即使在服务器重启的情况下也不会损失（或少量损失）数据，因为持久化会把内存中数据保存到硬盘上，重启会从硬盘上加载数据。 。但是由于数据是存储在一台服务器上的，如果这台服务器出现硬盘故障等问题，也会导致数据丢失。为了避免单点故障，通常的做法是将数据库复制多个副本以部署在不同的服务器上，这样即使有一台服务器出现故障，其他服务器依然可以继续提供服务。为此， Redis 提供了复制（replication）功能，可以实现当一台数据库中的数据更新后，自动将更新的数据同步到其他数据库上。

在复制的概念中，数据库分为两类，一类是主数据库（master），另一类是从数据库（slave）。主数据库可以进行读写操作，当写操作导致数据变化时会自动将数据同步给从数据库。而从数据库一般是只读的，并接受主数据库同步过来的数据。一个主数据库可以拥有多个从数据库，而一个从数据库只能拥有一个主数据库。
![img](http://jbcdn2.b0.upaiyun.com/2018/08/207ba1f265e3f8284aaaba6bcf90b430.png)

主从模式的配置，一般只需要再作为slave的redis节点的conf文件上加入“slaveof master*ip master*port”， 或者作为slave的redis节点启动时使用如下参考命令：















| 1    | redis-server --port 6380 --slaveof masterIp masterPort |
| ---- | ------------------------------------------------------ |
|      |                                                        |

redis的普通主从模式，能较好地避免单独故障问题，以及提出了读写分离，降低了Master节点的压力。互联网上大多数的对redis读写分离的教程，都是基于这一模式或架构下进行的。但实际上这一架构并非是目前最好的redis高可用架构。

 

### 2、redis哨兵模式高可用架构

当主数据库遇到异常中断服务后，开发者可以通过手动的方式选择一个从数据库来升格为主数据库，以使得系统能够继续提供服务。然而整个过程相对麻烦且需要人工介入，难以实现自动化。 为此，Redis 2.8开始提供了哨兵工具来实现自动化的系统监控和故障恢复功能。 哨兵的作用就是监控redis主、从数据库是否正常运行，主出现故障自动将从数据库转换为主数据库。

顾名思义，哨兵的作用就是监控Redis系统的运行状况。它的功能包括以下两个。

（1）监控主数据库和从数据库是否正常运行。
（2）主数据库出现故障时自动将从数据库转换为主数据库。

![img](http://jbcdn2.b0.upaiyun.com/2018/08/17ab8892f84f16670be20a9f8c68a290.png)

可以用info replication查看主从情况 例子： 1主2从 1哨兵,可以用命令起也可以用配置文件里 可以使用双哨兵，更安全，参考命令如下：















| 1234 | redis-server --port 6379 redis-server --port 6380 --slaveof 192.168.0.167 6379 redis-server --port 6381 --slaveof 192.168.0.167 6379redis-sentinel sentinel.conf |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

其中，哨兵配置文件sentinel.conf参考如下：















| 1    | sentinel monitor mymaster 192.168.0.167 6379 1 |
| ---- | ---------------------------------------------- |
|      |                                                |

其中mymaster表示要监控的主数据库的名字。配置哨兵监控一个系统时，只需要配置其监控主数据库即可，哨兵会自动发现所有复制该主数据库的从数据库。
Master与slave的切换过程：
（1）slave leader升级为master
（2）其他slave修改为新master的slave
（3）客户端修改连接
（4）老的master如果重启成功，变为新master的slave

 

### 3、redis-cluster群集高可用架构

> 参考< http://blog.jobbole.com/114270/>

即使使用哨兵，redis每个实例也是全量存储，每个redis存储的内容都是完整的数据，浪费内存且有木桶效应。为了最大化利用内存，可以采用cluster群集，就是分布式存储。即每台redis存储不同的内容。
采用redis-cluster架构正是满足这种分布式存储要求的集群的一种体现。redis-cluster架构中，被设计成共有16384个hash slot。每个master分得一部分slot，其算法为：hash_slot = crc16(key) mod 16384 ，这就找到对应slot。采用hash slot的算法，实际上是解决了redis-cluster架构下，有多个master节点的时候，数据如何分布到这些节点上去。key是可用key，如果有{}则取{}内的作为可用key，否则整个可以是可用key。群集至少需要3主3从，且每个实例使用不同的配置文件。

在redis-cluster架构中，redis-master节点一般用于接收读写，而redis-slave节点则一般只用于备份，其与对应的master拥有相同的slot集合，若某个redis-master意外失效，则再将其对应的slave进行升级为临时redis-master。
**在redis的官方文档中，对redis-cluster架构上，有这样的说明：在cluster架构下，默认的，一般redis-master用于接收读写，而redis-slave则用于备份，当有请求是在向slave发起时，会直接重定向到对应key所在的master来处理。但如果不介意读取的是redis-cluster中有可能过期的数据并且对写请求不感兴趣时，则亦可通过readonly命令，将slave设置成可读，然后通过slave获取相关的key，达到读写分离**。具体可以参阅redis官方文档（https://redis.io/commands/readonly）等相关内容.





#### spring boot 连接主备

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