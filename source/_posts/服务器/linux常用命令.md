---
title: Linux 常用命令
date: 2020-06-29
categories:
 - Linux
tags:
 - Linux
---


### 一、权限

	chown -R 用户名 文件名(文件夹名)     #更改文件所属用户
	chmod 777 路径+文件名（文件夹）      #文件（文件夹）赋权
	chmod +x test.txt                  #赋予执行权限（当前用户？）
	chmod -x test.txt                  #没收执行权限
	chmod username+x test.txt          #某用户增加执行权限
	chmod o+x test.txt                 #所有用户增加执行权限

### 二、编辑

	vi 文件名                           #编辑文件，已存在则修改，未存在则自动创建
	vi /etc/profile                    #编辑linux环境变量
	export KAFKA_HOME=/DATA/software/kafka_2.11-2.0.1    #引入环境变量
	source /etc/profile                #让修改后的环境变量立即生效
	tar xvfz m.tar.gz                  #linux解压安装包
	ls > cmd.txt                       #重定向  把命令的结果，重新写到文件里，会覆盖已有的内容
	ls >> cmd.txt                      #向文件中追加内容
	ps -ef|grep redis                  #查找redis进程
	man bash | col -b > bash.txt       #过滤控制字符，解决乱码
	grep ok test.txt                  #查找test.txt文件中的ok字段

### 三、查找/查看/开启

	ps -ef|grep 应用名称                 #得到(进程)进程号
	ls -l /proc/进程号/cwd               #得到 服务/进程 的路径
	history |grep 命令（应用）           #查找命令中含有“命令”的历史记录
	date                                #显示系统日期
	uname -r                            #显示正在使用的内核版本
	more 文件名                          #查看文件内容
	tail -300f 文件名                    #动态查看最近300行日志
	chkconfig --list                    #显示所有自动应用
	chkconfig vsftpd on                 #开启自动启动
	ls -l | grep "^d"                  #只列出目录
	rpm -qa                            #列出系统所有安装包
	df /home                           #查看挂载点
	which java                         #查找命令在哪个目录
	env                                #查看环境变量

### 四、磁盘

	df -h  /  df -lh                     #查看系统各目录磁盘空间及使用情况
	du -h --max-depth=1                  #查看当前目录下各文件所占磁盘空间情况
	top                                  #top命令是Linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况，类似于Windows的任务管理器。退出 top 的命令为 q （在 top 运行中敲 q 键一次）。
	du -s -h ./*                         #查看当前目录下各文件所占磁盘空间情况
	du -sh *| sort -hr                   #当前目录下占用磁盘空间大小排序

### 五、内存

	top -d 1                             #监控内存，然后shift+m内存排列
	free -g                              #查看内存使用，单位为g
	free -m                              #查看内存使用，单位M
	grep MemTotal /proc/meminfo | cut -f2 -d:     #查看服务器总内存，单位kb
	grep MemTotal /proc/meminfo          #查看服务器总内存，单位kb

### 六、系统命令

	reboot                               #重启系统
	shutdown -r now                      #重启系统
	logout                               #注销登陆
	shutdown -h now                      #关闭系统
	date 041217002007.00                 #设置日期和时间 - 月日时分年.秒

### 七、防火墙

	service iptables stop                #关闭防火墙
	systemctl stop                       #关闭防火墙
	firewalld.services                   #关闭防火墙
	service iptables starts              #打开防火墙
	systemctl start                      #打开防火墙
	iptables.services                    #打开防火墙
	service iptables restart             #重启防火墙
	systemctl start                      #重启防火墙
	ip6tables.service                    #重启防火墙
	systemctl status firewalld          #查看防火墙状态
	systemctl disable firewalld         #开机禁用
	systemctl enable firewalld          #开机自启

### 八、TCP

	netstat -an | grep ESTABLISHED | wc -l       #查看apache当前并发访问数
	netstat -anp|more                            #显示整个系统目前的网络情况。例如目前的连接、数据包传递数据、或是路由表内容。
	netstat -tunlp|grep 5601                     #查找5601端口占用进程id

### 九、CPU
#### 总核数 = 物理CPU个数 X 每颗物理CPU的核数
#### 总逻辑CPU数 = 物理CPU个数 X 每颗物理CPU的核数 X 超线程数
	cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l        #查看物理CPU个数
	cat /proc/cpuinfo| grep "cpu cores"| uniq                       #查看每个物理CPU中core的个数(即核数)
	cat /proc/cpuinfo| grep "processor"| wc -l                      #查看逻辑CPU的个数
	cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c           #查看CPU信息（型号）
	vmstat 1                                                        #监控CPU.
	top                                                             #监控进程
	more /proc/cpuinfo | grep "model name"                          #查看cpu
	grep "model name" /proc/cpuinfo                                 #查看cpu
	grep "model name" /proc/cpuinfo | cut -f2 -d:                   #查看cpu
	getconf LONG_BIT                                               #查看CPU位数(32 or 64)

### 十、用户相关
	useradd username              #创建用户（新创建的用户会在/home下创建一个用户目录incredible-test）
	passwd username               #给已创建的用户设置密码
	usermod --help                #修改用户这个命令的相关参数
	userdel username              #删除用户
	groupadd groupname            #添加组
	groupdel groupname            #删除组
	su username                   #切换登陆用户,exit退出到root账户？
	rm -rf /home/usernmae         #删除用户后，删除该用户目录
	passwd                        #登陆root账户情况下，修改root密码
	passwd username              #修改用户密码
	useradd username -g groupname     #指定用户到某个组
	usermod -g groupname username     #修改用户所在组
	whoami                            #查看当前登陆的用户


