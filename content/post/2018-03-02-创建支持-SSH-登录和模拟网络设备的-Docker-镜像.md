---
title: "创建支持 SSH 登录和模拟网络设备的 Docker 镜像"
author: "axiaoxin"
date: 2018-03-02T16:24:39+08:00
subtitle: ""
image: ""
tags: ["docker"]
slug: ""
---

## CentOS 主机安装 Docker

安装docker

```shell
yum install docker
docker version  # 1.12.6
```

移动docker的数据目录到较大磁盘下

```sh
mv /var/lib/docker/ /data/docker
ln -sb /data/docker  /var/lib/docker
```

设置docker OPTIONS：`/etc/sysconfig/docker`

```sh
# 添加如下配置 （存在就更新），docker.oa.com 是内网docker镜像地址
# 机器不支持NAT，把iptables设为false才能启动docker服务
OPTIONS='--log-driver=journald -b=none --iptables=true -H unix:///var/run/docker.sock --insecure-registry docker.oa.com'
```

启动docker

```sh
systemctl start docker
```


## 使用docker commit的方式制作镜像


拉取带有systemd的centos基础镜像（默认的centos7镜像不支持systemd不能使用systemctl，必须要支持systemd的镜像）

```sh
docker pull docker.oa.com/gaia/centos:systemd
```

创建容器 (使用host模式直接使用主机的网络)

```sh
docker run -d --privileged=true --net host --name building docker.oa.com/gaia/centos:systemd
```

复制要用到的程序包和配置等文件到容器中

```sh
docker cp CentOS-Base.repo building:/
docker cp setuptools-38.5.1.tar.gz building:/
docker cp pip-9.0.1.tar.gz building:/
docker cp snmpsimd.supervisor.conf building:/
docker cp sshd.supervisor.conf building:/
docker cp pip.conf building:/
docker cp supervisord.conf building:/
docker cp flowgen building:/
docker cp nginx-1.12.2.tar.gz building:/
docker cp nginx.conf building:/
docker cp nginx_status.conf building:/
docker cp nginx.supervisor.conf building:/
```

进入容器

```sh
docker exec -it building bash

# 更新yum源方便从内网安装包
cd /
mv -f CentOS-Base.repo /etc/yum.repos.d/
yum clean all && yum makecache

# 设置默认的root密码
echo "password" |passwd --stdin root

# 安装包
yum -y install gcc make pcre-devel zlib-devel openssh-server openssh-clients  net-snmp-utils net-tools lrzsz iproute
systemctl restart sshd  # 必须重启，第一次重启会自动生成密钥，否则需要手动生成
systemctl disable sshd  # 关闭开机启动，统一使用在创建容器时的CMD中使用supervisor启动各类开机进程
# 这样做的原因是在创建大量的容器时，比如创建1000个该容器，如果systemd的方式来进行开机启动进程的话总是会有很多容器的开机进程无法被正确启动，重启容器也不行，只能进入容器手动启动

# 安装pip
cd /
tar xzf setuptools-38.5.1.tar.gz ; cd setuptools-38.5.1; python setup.py install; cd -; rm -rf setuptools-38.5.1 setuptools-38.5.1.tar.gz
tar xzf pip-9.0.1.tar.gz ; cd pip-9.0.1; python setup.py install ; cd -; rm -rf pip-9.0.1.tar.gz pip-9.0.1

# 设置内网pip源
mkdir ~/.pip
mv -f /pip.conf ~/.pip

# 安装snmpsim用于模拟snmp网络设备，并设置一个自定义的jq-netmanager的community
pip install snmpsim
cd /usr/snmpsim/data/
cp public jq-netmanager -r
cp public.snmprec jq-netmanager.snmprec

# 安装supervisor用于管理容器启动进程
pip install supervisor
# 生成默认配置并制定配置include目录
mv -f /supervisord.conf /etc/supervisord.conf

# 创建配置目录并将snmpsimd的配置放到目录中
mkdir /etc/supervisor.conf.d
mv /snmpsimd.supervisor.conf /etc/supervisor.conf.d/
mv /sshd.supervisor.conf /etc/supervisor.conf.d/
mv /nginx.supervisor.conf /etc/supervisor.conf.d/

# netflow模拟器 https://github.com/mshindo/NetFlow-Generator
mv /flowgen /bin/flowgen
chmod +x /bin/flowgen

# nginx
cd /
tar xzf nginx-1.12.2.tar.gz && cd nginx-1.12.2 && ./configure --with-http_stub_status_module && make && make install && cd - && rm -rf nginx-1.12.2*
mv -f /nginx.conf /usr/local/nginx/conf
mkdir /usr/local/nginx/conf/conf.d
mv /nginx_status.conf /usr/local/nginx/conf/conf.d/

# 清理并退出
rm -rf /tmp/*
> ~/.bash_history
history -c
exit
```

创建镜像

```sh
docker commit building docker.oa.com/ashinchen/centos
```

push镜像

```sh
docker images  # 找到镜像id
docker tag IMGID docker.oa.com/ashinchen/centos
docker push docker.oa.com/ashinchen/centos
```

## 运行新的容器

创建模拟器容器

```sh
# 拉取镜像
# docker pull docker.oa.com/ashinchen/centos

# 创建自定义网络
docker network create --subnet=181.0.0.0/8 network181
# 创建指定ip的容器，必须加privileged才能ssh进去
docker run -d --privileged=true --name test --net network181 --ip=181.0.0.2 docker.oa.com/ashinchen/centos /usr/bin/supervisord -n
```

测试

```sh
snmpwalk -v 2c -c jq-netmanager 181.0.0.2
ssh root@181.0.0.2 -p 22  # 密码：password

## 镜像、容器持久化

# 导出当前构建完成的容器
docker export building > building_export.tar
# 压缩
gzip -r building_export.tar
```
