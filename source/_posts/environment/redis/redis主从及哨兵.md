---
title: Linix & redis
date: 2020-07-03
categories:
 - encironment
tags:
 - redis
 - Linux
---

## redis主从及哨兵

比较简陋的搭建一个主从及哨兵

### 1、创建目录/opt/app/redis,如下：
````
mkdir /opt/app/redis
````

### 2、下载redis安装包至目录下，复制3份，各自重命名，并解压：
````
tar xvfz m.tar.gz
````

### 3、自定义配置redis.conf。
````
# 使得Redis服务器可以跨网络访问，也可以注释掉，否则可能无法访问端口
bind 0.0.0.0
# 设置密码
requirepass "123456"
# 指定主服务器，注意：有关slaveof的配置只是配置从服务器，主服务器不需要配置
slaveof 192.168.11.128 6379
# 主服务器密码，注意：有关slaveof的配置只是配置从服务器，主服务器不需要配置
masterauth 123456
````

**注意：若在同一台虚拟机上搭建，需要将端口号修改为不同的**

### 4、启动redis

````
nohup /src/redis-server redis.conf >/dev/null 2>&1 &
# 查看启动情况
ps aux|grep redis
````

这里可能找不到redis-server,原因是redis解压后没有构建

````
# 进入redis的src目录进行构建
make install
````

这里由于前置环境的原因，可能会报缺少gcc，安装即可
````
 yum -y install gcc-c++
````

依次启动redis后，往主redis放数据，从redis会同步数据。

### 5、修改哨兵配置
````
# 禁止保护模式
protected-mode no
# 配置监听的主服务器，这里sentinel monitor代表监控，mymaster代表服务器的名称，可以自定义，192.168.11.128代表监控的主服务器，6379代表端口，2代表只有两个或两个以上的哨兵认为主服务器不可用的时候，才会进行failover操作。
sentinel monitor mymaster 192.168.11.128 6379 2
# sentinel author-pass定义服务的密码，mymaster是服务名称，123456是Redis服务器密码
# sentinel auth-pass <master-name> <password>
sentinel auth-pass mymaster 123456
````

若不用默认的mymaster服务名，修改服务名时，需将配置中的所有mymaster替换为新服务名（如：sentinel-server）

### 6、启动哨兵

````
nohup /src/sentinel-server sentinel.conf >/dev/null 2>&1 &
````

代码中连接哨兵即可使用redis主从，且当主redis宕机，哨兵会自动切换至从redis
