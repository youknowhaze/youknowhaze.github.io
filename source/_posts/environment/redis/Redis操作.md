---
title: java & redis
date: 2020-07-05
categories:
 - command
tags:
 - redis
 - java
---

## Redis 常见操作

### 一、运维常见命令

**1、启动redis**
````
# 进入redis根目录
nohup src/redis-server redis.conf >start.log 2>&1 &
````

**2、启动redis-sentinel 哨兵**
````
# 进入redis根目录
nohup src/redis-sentinel sentinel.conf >start.log 2>&1 &
````

**3、停止redis  **
````
redis-cli shutdown
````

**4、选择数据库**
````
//默认连接的数据库所有是0,默认数据库数是16个。返回1表示成功，0失败
select db-index
````

**5、清空数据库**
````
flushdb    //删除当前选择数据库中的所有 key。生产上已经禁止。
flushall   //删除所有的数据库。生产上已经禁止。
````

**6、模拟宕机**
````
redis-cli debug segfault
````

**7、模拟hang**
````
redis-cli -p 6379 DEBUG sleep 30
````

**8、重命名命令**
````
rename-command
// 例如：rename-command FLUSHALL ""。必须重启。
````

**9、设置密码**
````
config set requirepass [passw0rd]
````

**10、验证密码**
````
auth passw0rd
````

**11、查看日志**
````
日志位置在/redis/log下，redis.log为redis主日志，sentinel.log为sentinel哨兵日志。
````

### 二、开发常见命令

自注：key值为队列名

**1、在名称为key的队列尾添加一个值为value的元素**
````
rpush(key, value)
````

**2、在名称为key的队列头添加一个值为value的 元素**
````
lpush(key, value)
````

**3、返回名称为key的队列的长度**
````
llen(key)
````

**4、返回名称为key的队列中start至end之间的元素**
````
lrange(key, start, end)
````

**5、截取名称为key的队列**
````
ltrim(key, start, end)
````

**6、返回名称为key的队列中index位置的元素**
````
lindex(key, index)
````

**7、给名称为key的队列中index位置的元素赋值**
````
lset(key, index, value)
````

**8、删除count个key的队列中值为value的元素**
````
lrem(key, count, value)
````

**9、返回并删除名称为key的队列中的首元素**
````
lpop(key)
````


**10、返回并删除名称为key的队列中的尾元素**
````
rpop(key)
````

**11、lpop命令的block版本**
````
blpop(key1, key2,… key N, timeout)
````

**12、rpop的block版本**
````
brpop(key1, key2,… key N, timeout)
````

**13、返回并删除名称为srckey的队列的尾元素，并将该元素添加到名称为dstkey的list的头部**
````
rpoplpush(srckey, dstkey)
````

**14、删除某个队列**
````
del key
````

**15、清空redis**
````
flushdb
````

### 三、java 连接redis

**1、引入redis的依赖：**
````
<dependency>
      <groupId>redis.clients</groupId>
      <artifactId>jedis</artifactId>
      <version>2.5.0</version>
</dependency>
````

**2、连接池**
````
// 用hash方式保存至redis
public boolean saveToRedis(String key,String event,String value){
        try {
            Jedis jedis = getJedisPool().getResource();
            jedis.hset(key,event,value);
        }catch (Exception ex){
            DpLogger.GETDATALOG.error("redis存储异常..",ex);
        }
        return true;
    }

// redis哨兵连接池，redis是类似的
private JedisSentinelPool getJedisPool(){
        if (jedisPool == null){
            lock1.lock();
            try {
                JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
                jedisPoolConfig.setMaxIdle(30);
                // 哨兵连接127.0.0.1:26379,127.0.0.1:26378,127.0.0.1:26377
                Set<String> sentinelSet = redisConfiguration.getSentinelHost().stream().collect(Collectors.toSet());
                // redisConfiguration.getSentinelService()为服务名
                JedisSentinelPool pool = new JedisSentinelPool(redisConfiguration.getSentinelService(),
                        sentinelSet, jedisPoolConfig, redisConfiguration.getSentinelPwd());

                return pool;
            }catch (Exception ex){
                DpLogger.GETDATALOG.error("创建redis连接失败..",ex);
            }finally {
                lock1.unlock();
            }
        }
        return jedisPool;
    }
````











