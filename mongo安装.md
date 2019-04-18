#### 安装

```shell
wget http://downloads.mongodb.org/linux/mongodb-linux-x86_64-rhel62-v4.0-latest.tgz

tar -zxvf mongodb-linux-x86_64-rhel62-v4.0-latest.tgz

```

重命名

在mongo目录下创建 data 和 log目录用于存放数据和日志。

mongo的配置文件

```properties
dbpath=/usr/local/soft/mongoserver/data
logpath=/usr/local/soft/mongoserver/log/mongodb.log
port=27017
fork=false
journal=false
bind_ip = 0.0.0.0
storageEngine=wiredTiger  #存储引擎有mmapv1、wiretiger、mongorocks
```



配置文件说明

```properties
port=27017 #端口  
dbpath= /usr/mongodb/db #数据库存文件存放目录  
logpath= /usr/mongodb/mongodb.log #日志文件存放路径  
logappend=true #使用追加的方式写日志  
fork=false #不以守护程序的方式启用，即不在后台运行  
maxConns=100 #最大同时连接数  
noauth=true #不启用验证  
journal=true #每次写入会记录一条操作日志（通过journal可以重新构造出写入的数据）。
#即使宕机，启动时wiredtiger会先将数据恢复到最近一次的checkpoint点，然后重放后续的journal日志来恢复。
storageEngine=wiredTiger  #存储引擎有mmapv1、wiretiger、mongorocks
bind_ip = 0.0.0.0  #这样就可外部访问了，例如从win10中去连虚拟机中的MongoDB
```

启动mongo

```shell
/mongod -f /usr/local/soft/mongoserver/etc/mongodb.conf
```

