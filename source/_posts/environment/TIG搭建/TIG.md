---
title: TIG搭建
date: 2020-06-26
categories:
 - encironment
tags:
 - TIG
 - Linux
---


## TIG监控

TIG监控是一种硬件、业务数据监控组件，主要用于监控服务器状态及业务数据量、时间等信息。

-    Influxdb:一种时序性数据库，主要用于存储数据

-    Telegraf：收集数据组件，本实验中用于收集硬件、业务指标

-    Grafana：用于实时展示收集到的指标数据

#### 这里选择在目录/opt/app/TIG下进行



### 一、influxdb

#### 1.解压安装包(将安装包放到目录下)：
````
cd /opt/app/TIG(或者其他服务器上传目录下)
tar -xvzf influxdb-1.2.0_linux_amd64.tar.gz
cd influxdb-1.2.0-1
````

#### 2.查看InfluxDB默认端口8083,8086端口是否被占用，若未占用，则无信息返回
````
 lsof -i:8083
 lsof -i:8086
````

#### 3.如果未占用,则如下修改配置文件.否则修改为其他端口号
````
vi etc/influxdb/influxdb.conf
````

#### 4.将admin下的enabled与bind-address修改为如下.：
````
[admin]
   # Determines whether the admin service is enabled.
   enabled = true

   # The default bind address used by the admin service.
   bind-address = ":8083"

   # Whether the admin service should use HTTPS.
   # https-enabled = false

   # The SSL certificate used when HTTPS is enabled.
   # https-certificate = "/etc/ssl/influxdb.pem"
````

#### 5.启动influxdb
````
nohup usr/bin/influxd -config etc/influxdb/influxdb.conf &
````

#### 6.查看influxdb进程是否正常启动
````
ps aux|grep influxdb
````

#### 7.在本地访问：http://ip:8083/

   ip为linux服务器对外映射的ip


####问题及注意：
- 添加虚拟机ip访问的代理忽略（即本地访问不使用代理）

- 若未安装lsof命令，则运行sudo yum install lsof进行安装


### 二、telegraf

#### 1.解压安装包
````
cd /opt/app/TIG(或者自定义上传目录下)
tar -xzvf telegraf-1.6.1_linux_amd64.tar.gz
cd telegraf
````

#### 2.修改配置文件，将收集的信息放入刚才安装的influxd数据库
````
vi etc/telegraf/telegraf.conf
# 修改的节点为outputs.influxdb下的urls和database配置如下(P.S:IP改为InfluxDB部署服务器的IP地址)
[[outputs.influxdb]]
   ## The full HTTP or UDP URL for your InfluxDB instance.
   ##
   ## Multiple URLs can be specified for a single cluster, only ONE of the
   ## urls will be written to each interval.
   # urls = ["unix:///var/run/influxdb.sock"]
   # urls = ["udp://127.0.0.1:8089"]
   urls = ["http://10.221.155.69:8086"]

   ## The target database for metrics; will be created as needed.
   database = "telegraf"
````

#### 3.启动telegraf，并查看进程是否正常运行
````
usr/bin/telegraf -config etc/telegraf/telegraf.conf &
ps aux|grep telegraf
````
#### 4.验证telegraf是否将硬件指标插入influxdb数据库
   访问inflxudb admin页面http://influxdb的ip地址:8083/

#### 5.查看插入的表信息（页面query栏输入）
````
   SHOW MEASUREMENTS
````

#### 6.查看CPU信息（页面query栏输入）
````
select * from cpu order by time desc limit 10
````

#### 收集业务指标
 业务场景：将日志文件：/opt/applog/influxdb.log的内容收集到influxdb数据库中。
#### 7.新增telegraf业务指标配置文件
		vi etc/telegraf/telegraf_tail.conf

#### 8.配置文件内容如下(P.S:IP改为InfluxDB部署服务器的IP地址):
````
	   [[outputs.influxdb]]
          urls = ["http://ip:8086"]
          database = "tail"
          precision = "s"
          timeout = "5s"
          username = "root"
          password = "root"
        [[inputs.tail]]
          ## files to tail.
          ## These accept standard unix glob matching rules, but with the addition of
          ## ** as a "super asterisk". ie:
          ##   "/var/log/**.log"  -> recursively find all .log files in /var/log
          ##   "/var/log/*/*.log" -> find all .log files with a parent dir in /var/log
          ##   "/var/log/apache.log" -> just tail the apache log file
          ##
          ## See https://github.com/gobwas/glob for more examples
          ##
          files = ["/opt/applog/influxdb.log"]
          ## Read file from beginning.
          from_beginning = false
          ## Whether file is a named pipe
          pipe = false

          ## Method used to watch for file updates.  Can be either "inotify" or "poll".
          watch_method = "poll"

          ## Data format to consume.
          ## Each data format has its own unique set of configuration options, read
          ## more about them here:
          ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
          data_format = "influx"
````

#### 9.新增一条业务日志
````
echo 'testtable,ip=127.0.0.1,user=taobao,service=airbook servicetime=31'  >> /opt/applog/influxdb.log
````
#### 10.启动telegraf,并查看进程是否正常启动
````
usr/bin/telegraf -config etc/telegraf/telegraf_tail.conf &
ps aux|grep telegraf
````

#### 11.验证业务日志是否插入到influxdb中
   访问inflxudb admin页面http://influxDB的ip地址:8083/

#### 12.查看插入的表信息
````
SHOW MEASUREMENTS
````

#### 13.查看表信息
````
select * from testtable order by time desc limit 10
````

### 三、安装grafana

#### 1.解压安装包：
````
 cd /opt/app/TIG(或者本机上传目录下)
 tar -zxvf grafana-4.2.0.linux-x64.tar.gz
 cd grafana-4.2.0
````

#### 2.启动grafana,并检查进程是否正常启动
````
bin/grafana-server &
ps aux|grep grafana
````

#### 3.验证grafana是否能正常访问
   http://grafana的ip地址:3000/login

#### 4.登录grafana
默认账户：admin   密码：admin
备注：账户密码可配置，配置文件路径：conf/defaults.ini

#### 5.新增硬件DataSource（配置数据库）访问地址：

http://192.168.56.1:3000/datasources/new

![grafana 1](/TIG/grafana1.png)<br>

![grafana 2](/TIG/grafana2.png)<br>

#### 6.新增CPU看板（dashboard）访问地址：

   http://192.168.56.1:3000/dashboard/new?orgId=1

![grafana 3](/TIG/grafana3.png)<br>

![grafana 4](/TIG/grafana4.png)<br>



