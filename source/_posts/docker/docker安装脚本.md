---
title: docker一键式安装脚本
date: 2020-06-18
categories:
 - Docker
tags:
 - Linux
 - Docker
---


## docker一键式安装脚本

将以下脚本拷贝至shell脚本中，附于执行权限，使用管理员身份执行即可安装docker

````
#!/bin/bash

# 环境初始化：指定用户需要有sudo权限

# step 1: 移除旧的docker
sudo yum remove -y docker \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-selinux \
                docker-engine-selinux \
                docker-engine \
                docker-ce
sudo rm -rf /var/lib/docker

# step 2: 安装相关组件和配置yum源
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
sudo yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# step 3: 配置缓存
sudo yum makecache fast
# step 4: 执行安装
sudo yum install -y docker-ce-18.06.3.ce-3.el7
****


# step 5: 启动docker并配置开机启动并软连接配置镜像存储路径
sudo systemctl start docker
sudo systemctl enable docker
# 注意路径的变更
sudo mv /var/lib/docker /data/docker
sudo ln -s /data/docker /var/lib/docker

# step 6: 配置当前用户对docker命令的执行权限
sudo groupadd docker
sudo gpasswd -a ${USER} docker
sudo systemctl restart docker

# vim /etc/yum.repos.d/docker-ce.repo
#   将[docker-ce-test]下方的enabled=0修改为enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.0.ce.1-1.el7.centos)
# sudo yum  install -y docker-ce-[VERSION]
# sudo yum  install -y docker-ce-18.06.3.ce-3.el7

echo 'docker安装成功...'
docker -v

````
