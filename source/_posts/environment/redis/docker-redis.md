---
title: docker & redis
date: 2020-07-04
categories:
 - encironment
tags:
 - Docker
 - redis
---

## docker搭建redis主从及哨兵

### 1、前提准备

docker环境（这里我只使用了一台docker服务器，通过端口区别实现集群）

### 2、创建外部挂载目录

创建目录如下，每个目录下再创建data和conf目录用于外部挂载：
![挂载目录](/docker-redis/docker-redis01.png)<br>

### 3、下载redis配置和哨兵配置

````
wget http://download.redis.io/redis-stable/redis.conf

wget http://download.redis.io/redis-stable/sentinel.conf
````
文件下载后拷贝改名，分别放至对应目录下

redis:（命名分别为：redis_6379.conf/redis_6371.conf/redis_6372.conf）
sentinel:（命名分别为：sentinel_1.conf/sentinel_2.conf/sentinel_3.conf）

### 4、修改redis配置
````
# 注释掉使得Redis服务器可以跨网络访问
# bind 0.0.0.0
# 设置密码
requirepass "123456"
# 指定主服务器，注意：有关slaveof的配置只是配置从服务器，主服务器不需要配置
slaveof 192.168.11.128 6379
# 主服务器密码，注意：有关slaveof的配置只是配置从服务器，主服务器不需要配置
masterauth 123456
# 选择性配置，若配置需确保文件目录存在
logfile ""
````
**配置文件需更改对应端口**

### 5、启动一台redis
````
docker run -p 6372:6372
--name redis-6372
-v /opt/redis/redis03/conf/redis_6372.conf:/etc/redis/redis_6372.conf
-v /opt/redis/redis03/data:/data
-d redis redis-server /etc/redis/redis_6372.conf
--appendonly yes    # 开启持久化
````
启动之后使用命令查看
````
docker ps
````
未查找到运行中的redis容器，使用命令查看redis容器启动日志
````
docker logs ID|less
````
发现是logfile日志目录不存在，程序没有自动创建，这里我直接删除这个配置再启动
````
# 得到容器ID
docker ps -a
# 移除容器
docker rm ID

#启动redis
````
使用redis-manager工具连接试试，发现可以连接。再启动其余几台redis，同样进行验证

### 6、更改redis哨兵配置
````
# 禁止保护模式
protected-mode no
# 配置监听的主服务器，这里sentinel monitor代表监控，mymaster代表服务器的名称，可以自定义，192.168.11.128代表监控的主服务器，6379代表端口，2代表只有两个或两个以上的哨兵认为主服务器不可用的时候，才会进行failover操作。
sentinel monitor mymaster 192.168.11.128 6379 2
# sentinel author-pass定义服务的密码，mymaster是服务名称，123456是Redis服务器密码
# sentinel auth-pass <master-name> <password>
sentinel auth-pass mymaster 123456
````

**这里需要注意的是，如果不使用默认的服务名mymaster，如更改为了sentinel-server，则需要将配置中所有涉及服务名的地方进行更改，如超时时间，且端口应配置为不一样**

### 7、启动哨兵
````
docker run -p 26379:26379
--name sentinel-26379
-v /opt/redis/sentinel01/conf/sentinel_1.conf:/etc/redis/sentinel.conf
-v /opt/redis/sentinel01/data:/data
-d redis redis-sentinel /etc/redis/sentinel.conf
````

启动后执行命令查看是否启动成功
````
docker ps
````

最后情况应该如下：
![运行情况](/docker-redis/docker-redis02.png)<br>
