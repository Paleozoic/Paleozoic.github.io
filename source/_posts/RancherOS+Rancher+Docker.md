---
title: RancherOS+Rancher+Docker
date: 2018-01-23 15:51:55
categories: [Docker]
tags: [Docker,Rancher]
---
<Excerpt in index | 首页摘要>
RancherOS+Rancher+Docker实现分布式模拟环境，不过还是太耗费内存了，呵呵。<!-- more -->
<The rest of contents | 余下全文>

# RancherOS
本想使用CoreOS的，但是看了下RancherOS，只有60M，比CoreOS更加精简，所以重新玩一下RancherOS。
通过RancherOS搭建一个Docker集群来做开发环境。
- ISO镜像准备
  [RancherOS 最新版本](https://github.com/rancher/os/releases/)
  [RancherOS v1.1.3](https://github.com/rancher/os/releases/tag/v1.1.3)
  [rancheros.iso v1.1.3下载](https://github.com/rancher/os/releases/download/v1.1.3/rancheros.iso)
- 虚拟机准备
  此处以VMware为例，Hyper-V之类也是一样的，不在赘述。
- Boot from ISO
  [Boot from ISO](http://rancher.com/docs/os/v1.1/en/running-rancheros/workstation/boot-from-iso/)
在VMware新建虚拟机，安装镜像选择刚刚下载的rancheros.iso。此一步便是Boot from ISO。
- Installing RancherOS to Disk
  [Installing RancherOS to Disk](http://rancher.com/docs/os/v1.1/en/running-rancheros/server/install-to-disk/)
    * 初始化配置文件`cloud-config.yml`
      ```yml
      #cloud-config
      ssh_authorized_keys:
        - ssh-rsa AAA...ZZZ example1@rancher
        - ssh-rsa BBB...ZZZ example2@rancher
      ```
    * 下载`cloud-config.yml`
      参考[CoreOS初尝试](http://blog.1x1.space/2017/08/04/CoreOS%E5%88%9D%E5%B0%9D%E8%AF%95/)，通过[HTTP文件服务器HFS](http://rejetto.com/hfs/?f=dl)部署cloud-config.yml，
    然后通过`wget http://192.168.2.101/cloud-config.yml`获取文件。
    * 使用`ROS INSTALL`安装
      `cloud-config.yml` `ros install`后会被加载到RancherOS的`/var/lib/rancher/conf/`，每次重启都解析`cloud-config.yml`的配置信息。
    在`wget`到文件`cloud-config.yml`后，执行以下命令安装：
    （1）在线安装
        ```
        sudo ros install -c cloud-config.yml -d /dev/sda
        ```
    （2）离网安装
        ```
        sudo ros install -c cloud-config.yml -d /dev/sda -i <Image_Name_in_System_Docker>
        ```
    （3）合并yml文件:`sudo ros config merge -i <your yml file>`
    （4）其他`ros config`命令不一一列举。
    可以通过`ros config`或者编辑`/var/lib/rancher/conf/cloud-config.yml`修改配置文件，重启生效。
- SSH连接RancherOS
  `cloud-config.yml`里面的ssh-rsa是公钥，则通过ssh和私钥可以直接连接RancherOS，如下：
  ![keyboard](/resources/img/docker/ssh.png)
  ```
  ssh -i /path/to/private/key rancher@<ip-address>
  ```
  当然，如果没有通过`cloud-config.yml`安装公钥，则可以通过用户名/密码登录RancherOS，用户名/密码是：`rancher/rancher`。安装完ssh key后，只能通过key登录。如图（XShell登录）：
`ssh rancher@192.168.186.129`

私钥位置：`C:\Users\HP\Documents\NetSarang\SECSH\UserKeys`

- RancherOS集群
  克隆刚刚安装完毕的3个虚拟机，一起组成集群。

# Rancher
[Rancher Quick Start Guide](http://rancher.com/docs/rancher/latest/en/quick-start-guide/)
- Docker Hub配置
参考[CoreOS初尝试](http://blog.1x1.space/2017/08/04/CoreOS%E5%88%9D%E5%B0%9D%E8%AF%95/)里面的Docker Hub章节申请DaoCloud的镜像加速器。
执行脚本如下：
```
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://xxxx.m.daocloud.io
```
当然，也可以尝试一下[阿里云镜像加速器](https://cr.console.aliyun.com/#/accelerator)。
注意：此处curl命令无法识别，wget无法访问https，所以通过浏览器访问`https://get.daocloud.io/daotools/set_mirror.sh`下载set_mirror.sh并复制至rancher。
```
vi set_mirror.sh
# 复制脚本内容进去，然后授权
chmod +x set_mirror.sh
./set_mirror.sh  -s http://xxxx.m.daocloud.io
# 报错：Error: Unsupported OS, please set registry-mirror manually.  
```
解决(貌似不起作用，还是去了官方Docker Hub拉镜像)：
[Configuring Docker or System Docker](http://rancher.com/docs/os/v1.1/en/configuration/docker/)
```
#cloud-config
rancher:
  bootstrap_docker:
    registry_mirror: "http://xxxx.m.daocloud.io"
  docker:
    registry_mirror: "http://xxxx.m.daocloud.io"
  system_docker:
    registry_mirror: "http://xxxx.m.daocloud.io"
```
通过`ros config`慢慢set参数。
```
sudo ros config set rancher.bootstrap_docker.registry_mirror "http://xxxx.m.daocloud.io"
sudo ros config set rancher.docker.registry_mirror "http://xxxx.m.daocloud.io"
sudo ros config set rancher.system_docker.registry_mirror "http://xxxx.m.daocloud.io"
# 代理，根据需求设置
sudo ros config set rancher.network.http_proxy "http://172.20.0.2:8086"
sudo ros config set rancher.network.no_proxy  localhost,127.0.0.1
# ……
sudo system-docker restart docker
```
或者
```
# 进入root
sudo su - 
# 修改cloud-config.yml
vi /var/lib/rancher/conf/cloud-config.yml
# 修改registry_mirror
system-docker restart docker
# 视情况reboot
reboot
```

- 安装Rancher Server
在RancherOS集群里面选择1个节点作为主节点，执行以下命令：
```
sudo docker run -d --restart=unless-stopped -p 8080:8080 rancher/server:stable
# Tail the logs to show Rancher
sudo docker logs -f <CONTAINER_ID>
```
安装完毕后通过url：`http://<SERVER_IP>:8080` 访问Rancher UI。
之后登录UI，通过UI管理整个RancherOS集群，并进行Docker的管理维护。
安装rancher agent，页面有明显提示，在angent节点执行一下命令：
```
sudo docker run --rm --privileged -v /var/run/docker.sock:/var/run/docker.sock -v /var/lib/rancher:/var/lib/rancher rancher/agent:v1.2.9 http://192.168.0.114:8080/v1/scripts/FF30E1C18F73F11F401F:1514678400000:zuGJLcZ7klOHoi2DjKk2v5wwT4
```
- 添加镜像库
填写镜像加速器地址（阿里云或者DaoCloud）

- 初始化开发环境
 *  mysql：Rancher应用商店提供了mysql应用，并且自带主从实现
 *  redis：添加容器，新增redis镜像