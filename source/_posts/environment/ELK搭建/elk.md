---
title: ELK搭建
date: 2020-06-22
categories:
 - encironment
tags:
 - ELK
 - Linux
---

## ELK

ELK由elasticsearch（存储层）、logstash（收集）、kibana（展示）三部分组成，三个组件的版本建议保持一致。此文档是基于ELK7.6.2版本，其余版本在配置上可能会有差异。

| ip地址 | 应用                    |
| ------ | ----------------------- |
| netip1 | es                      |
| netip2 | es                      |
| netip3 | kibana                  |
| netip4 | logstash 业务日志收集   |
| netip5 | logstash 防火墙日志收集 |



### elasticsearch

主要介绍es的搭建，没有采用docker进行搭建，后续再加上docker搭建的文档

#### 一、环境准备
##### 1、下载es的安装包，上传至服务器，或者在服务器直接使用命令下载。
这里给出当前最新版本安装包</br>   [点击下载es](https://www.elastic.co/cn/downloads/elasticsearch)

##### 2、es是基于java环境的，所以安装es前需要先配置jdk，这里配置的是jdk8。7.6的es要求是jdk11以上，但他是向下兼容的，所以jdk8即可。java环境的配置是基本操作这里就不赘述了。

##### 3、搭建2台es作为集群吧，先准备两台服务器/虚拟机（net1，net2【这里用netip1、netip2代替ip地址】）

#### 二、编辑es配置
1、创建文件夹
````
--创建elk的文件夹
mkdir /opt/elk
-- 存放es的数据
mkdir /opt/elk/data/es/data
-- 存放es的日志
mkdir /opt/elk/data/es/logs
-- 将目录权限赋值给非root用户，非root用户创建请忽略
chown -R prouser /opt/elk
````

##### 2、解压文件并重命名
````
tar zvxf elasticsearch-7.6.2-linux-x86_64.tar.gz
mv elasticsearch-7.6.2  elasticsearch
````

##### 3、编辑配置文件
````
-- 先进入根目录
cd elasticsearch
vi config/elasticsearch.yml
````

##### 4、配置文件需要配置的内容如下,两个服务器配置只有nodename和ip不同
````
-- 集群设置，es会默认将相同网段的es作为集群，当有多个集群时，通过该字段进行区分
cluster.name: dp-es
-- 节点名称，集群必须定义，单台可不用定义
node.name: dp-122
path.data: /opt/elk/data/es/data
path.logs: /opt/elk/data/es/logs
network.host: netip1
http.port: 9200
-- 可以作为master的节点，集群必不可少
cluster.initial_master_nodes: ["netip1","netip2"]
-- 可以初始化的节点，集群必不可少
discovery.seed_hosts: ["netip1","netip2"]
-- 检测到几个节点时开始恢复数据
gateway.recover_after_nodes: 1
-- 跨域设置，这个选择性设置，非必要
http.cors.enabled: true
http.cors.allow-origin: "*"
````

#### 三、启动es前的准备
es对环境有比较高的要求，现对服务器做如下配置：</br>
````
-- 1、切换至root用户
su root

-- 2、编辑打开文件数限制
vi /etc/security/limits.conf
-- 编辑如下内容，注意*号不要复制掉了
* soft nofile 65536
* hard nofile 65536

-- 3、更改vm
echo 'vm.max_map_count=262144'>> /etc/sysctl.conf
-- 查看更改
sysctl -p

-- 4、编辑限制
vi /etc/security/limits.d/90-nproc.conf
-- 内容如下
* soft nproc 4096
````

#### 四、启动es
##### 1、依次启动es，注意观察启动日志。
````
-- 进入es根目录，先控制台启动
bin/elasticsearch

-- 无问题再切换为后台启动
nohup bin/elasticsearch >/dev/null 2>&1 &
````

##### 2、确认es是否启动
(1)、命令确认：
````
curl -X GET http://netip1:9200
````
(2)、访问页面 http://netip1:9200 </br>

返回如下情况则表示安装成功：

![build succes](/ELK搭建/es-return.png)<br>

#### 五、可能会出现严重错误导致无法启动
可能会出现如下错误，导致无法启动：

![build error](ELK搭建/es-error01.png)<br>

````
-- 跳过filter
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
````


### kibana
这段将介绍简单的搭建kibana

#### 一、环境准备
##### 1、下载kibana的安装包，上传至服务器，或者在服务器直接使用命令下载。
这里给出当前最新版本安装包</br>
[点击下载kibana](https://www.elastic.co/cn/downloads/kibana)

##### 2、需要一台服务器/虚拟机，这里就放在netip3

#### 二、编辑kibana配置
##### 1、解压文件并重命名
````
cd /opt/elk
tar zvxf kibana-7.6.2-linux-x86_64.tar.gz
mv kibana-7.6.2 kibana
````

##### 2、 配置kibana文件
````
-- 配置kibana
cd kibana
vi config/kibana.yml
````

##### 3、 配置内容如下
````
server.port: 5601
server.host: "netip3"
-- es集群
elasticsearch.hosts: ["http://netip1:9200"，"http://netip2:9200"]
````

#### 三、启动kibana
````
nohup bin/kibana >/dev/null 2>&1 &
````
kinaba不依赖于java环境，所以查询只能通过端口
````
netstat -tunlp|grep 5601
````

确认启动成功访问kiban：http://netip3:5601



### Logstash
本节简单介绍logstash的配置启动

#### 一、环境准备
##### 1、logstash和es一样需要java环境，也是向下兼容的，jkd8即可.
##### 2、准备服务器，至少1台，这里选取119
##### 3、下载logstash</br>
[点击下载logstash](https://www.elastic.co/cn/downloads/logstash)
#### 二、配置编辑
##### 1、解压并重命名
````
tar zvxf logstash-7.6.2.tar.gz
mv logstash-7.6.2 logstash
````

##### 2、logstash的本身配置文件无需任何更改，新建一个自定义配置:dp-logstash.conf
````
cd logstash
vi /config/dp-logstash.conf
````

这里放入119的测试环境配置，暂时只有推送和拉取模块的，将两个模块的配置放在一起进行收集，如下：

````
input {
  file {
        path => "/DATA/applog/dslog/datahanderinfo.log"
        type => "obtain"
        codec => multiline {
            pattern => "^%{YEAR}"
            negate => true
            what => "previous"
        }
    }
  file {
        path => "/DATA/applog/dslog/pushloginfo.log"
        type => "pushhttp"
        codec => multiline {
            pattern => "^%{YEAR}"
            negate => true
            what => "previous"
        }
    }
}

filter {
          if ([message] =~ "collectd|SourceAddr|Priority|MsgType=|<Subevent|result.isSuccess|没有获取到") {
           drop {}
          }

          if ([type] == "obtain"){
             mutate {
                   split => { "message" => "MsgContent=" }
                   add_field =>   {"msgcontent" => "%{[message][1]}"}
                   remove_field => ["message"]
                   add_field =>   {"datatype" => "origindata"}
                   replace => { "host" => "netip4" }
                 }
          }
          if ([type] == "pushhttp"){
              mutate {
                   split => { "message" => "-run," }
                   add_field =>   {"msgcontent" => "%{[message][1]}"}
                   remove_field => ["message"]
                   replace => { "host" => "netip4" }
                 }
          }
}

output {
    elasticsearch { hosts => ["netip1:9200","netip2:9200"]
                    index => "logstash-%{type}-%{+YYYY.MM.dd}"
}
    stdout { codec=> rubydebug }
}
````

##### 再附上一个防火墙系统日志的收集
````
# 输入
input{
  udp{
    host => "netip5"
    port => 514
    codec => line
  }
}

# 过滤器
filter {

  # 拆分字段
  kv {
	source => "message"
	include_keys => ["src","dst","proto","sport","dport","inpkt","outpkt","sent","rcvd","duration","connid","parent","op" ]
  }

  # 上下行流量转为Integer类型
  mutate{
    convert => ["rcvd","integer"]
    convert => ["sent","integer"]
  }

  #删除无用字段,host固定为服务所在ip
  mutate {
	replace => { "host" => "netip5" }
  }
  #从源ip中获取楼层信息
  grok {
    match => {
      "src" => "101.26.(?<floor>\d)(0*).\d"
    }
  }
}

# 输出
output{
  elasticsearch{
    hosts => "netip5:9200"
    index => "netfire-%{+YYYY.MM.dd}"
    manage_template => false
    template_name => "netfire_template"
  }
}
````


#### 三、启动logstash

##### 1、 通过指定配置文件的方式启动logstash
````
nohup bin/logstash -f config/dp-logstash.conf >start.log 2>&1 &
````

#### 详细的配置请参考官方文档

