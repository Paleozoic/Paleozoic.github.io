---
title: CoreOS初尝试
date: 2017-08-04 01:28:32
categories: [Docker]
tags: [CoreOS]
typora-root-url: ..
---
<Excerpt in index | 首页摘要>
CoreOS初尝试<!-- more -->
<The rest of contents | 余下全文>

# CoreOS安装

[CoreOS官网](https://coreos.com/)

[booting-with-iso官方文档](https://coreos.com/os/docs/latest/booting-with-iso.html)

- 准备材料：
  - [HTTP文件服务器HFS](http://rejetto.com/hfs/?f=dl)
  - [CoreOS ISO镜像](http://stable.release.core-os.net/amd64-usr/current/coreos_production_iso_image.iso)


- 安装HFS

- Win10自带Hyper-V创建虚拟机（分配至少2G内存），选择第一代虚拟机。

- 在虚拟机安装CoreOS ISO镜像

- 制作cloud-config.yml。注意：ssh_authorized_keys很重要，如果丢了，不知道如何才能登陆上CoreOS（PS：CoreOS默认是没有用户名/密码登录的）。

  ```yml
  #cloud-config
  coreos:
    units:
      - name: settimezone.service
        command: start
        content: |
          [Unit]
          Description=Set the time zone

          [Service]
          ExecStart=/usr/bin/timedatectl set-timezone Asia/Shanghai
          RemainAfterExit=yes
          Type=oneshot

  ssh_authorized_keys:
    - "ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA0f3eocEpfb47NoD3YpTjSa8EMVVscUM+YoGXU1Y6ejCSujsVh+17rr75JCx7WP71a7OJWe0xaBYgEPTSUKNk4ltKoYbFl4UL9PTWuIH74MqdGEQS84oLE33IRWNdmJ5dvpcM18d0ySgpIpozKlBcE/4EugLxtwXyfi+C249bdxOlkk60Cj28iSALh9HiM9S8Ta7vQ9DkHlas4npvLDeEoAEgAlnaxFMiEFq3xQhIS+KKRyY6dX4F8rXklmqTPJkBnKMEuMrTtxHmn01jJNSOkvzU8Jbq7n4E94FaqIGJFO08h3w5SliPSjlRMwDXeW1ydYnK8+/KcPiOHApuM8q1/w=="
  ```

-  将cloud-config.yml上传至HFS

  ![hfs](/resources/img/docker/hfs.png)

- 通过`wget http://192.168.2.101/cloud-config.yml`获取文件

- 通过`sudo su - root` 进入root用户。

- `coreos-install -d /dev/sda -C stable -c /home/core/cloud-config.yml`

  执行此命令，会下载以下文件（速度较慢）。

  镜像下载：http://stable.release.core-os.net/amd64-usr/1409.7.0/coreos_production_image.bin.bz2

  签名下载：http://stable.release.core-os.net/amd64-usr/1409.7.0/coreos_production_image.bin.bz2.sig

  PS：当然，我们也可以先离线下载以上文件，然后将以上文件放在HFS上面，再通过修改[coreos-install](https://raw.githubusercontent.com/coreos/init/master/bin/coreos-install)脚本，指定本地的URL而不是CoreOS的URL来实现离线安装。具体可参看[平台云基石-CoreOS之离线安装篇（无需互联网）](https://yq.aliyun.com/articles/42288) 。不过由于咱们是Win10环境，所以文件服务器用的是HFS。

- 之后弹出ISO，执行命令`reboot`

# 手工更新ssh key

- Xshell菜单：工具-》新建用户秘钥生成向导：默认密钥类型RSA，密钥长度2048位。

- 一直next，用户名密码留空。生成并保存密钥：id_rsa_2048.pub（默认文件）,复制文件内容，在CoreOS上执行如下命令：

  ```bash
  echo 'ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA0f3eocEpfb47NoD3YpTjSa8EMVVscUM+YoGXU1Y6ejCSujsVh+17rr75JCx7WP71a7OJWe0xaBYgEPTSUKNk4ltKoYbFl4UL9PTWuIH74MqdGEQS84oLE33IRWNdmJ5dvpcM18d0ySgpIpozKlBcE/4EugLxtwXyfi+C249bdxOlkk60Cj28iSALh9HiM9S8Ta7vQ9DkHlas4npvLDeEoAEgAlnaxFMiEFq3xQhIS+KKRyY6dX4F8rXklmqTPJkBnKMEuMrTtxHmn01jJNSOkvzU8Jbq7n4E94FaqIGJFO08h3w5SliPSjlRMwDXeW1ydYnK8+/KcPiOHApuM8q1/w==' | update-ssh-keys -a core
  ```


> 注意：以上命令在Hyper-V虚拟机并不能直接复制粘贴，然而，手敲的话估计敲断手指![keyboard](E:\github\hexo_blog\source\resources\img\docker\keyboard.gif)
>
> 所以咱们将以上代码封装成一个脚本文件：coreos_ssh.sh，然后借用一个HTTP文件服务器，将coreos_ssh.sh放在HFS上。如图：
>
> ![hfs1](/resources/img/docker/hfs1.png)
>
> 在CoreOS上输入：`wget http://192.168.2.101/coreos_ssh.sh`将脚本下载至当前目录，在执行：`sh coreos_ssh.sh`，之后便可以通过XShell直连CoreOS了。

# Docker Hub

[Docker Hub](https://hub.docker.com/)服务器在国内比较慢，所以我们需要使用[DaoCloud加速器](https://www.daocloud.io)。

- 注册DaoCloud

- 登录后点击DaoCloud右上角火箭图标（即加速器），获取加速器地址。得到如下格式地址：`http://xxx.m.daocloud.io`

- 然后参考[coreos中更改docker镜像地址](https://segmentfault.com/a/1190000002965664)进行registry-mirror的更改

  * 通过`sudo su - root` 进入root用户。


  * `cp /usr/lib/systemd/system/docker.service /etc/systemd/system`

  * 然后 `vim /run/flannel_docker_opts.env`，

    并添加如下文本：`DOCKER_OPTS="--registry-mirror=http://xxx.m.daocloud.io"`

  * 然后执行：

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```

# 复制虚拟机

经过以上配置，得到一个XShell可SSH访问、并有DaoCloud加速器的CoreOS系统，直接在Win10上复制Virtual Hard Disks的文件进行备份。

并创建3个一模一样的CoreOS（复制导入即可）。

# Rancher

容器编排调度，类似于k8s。





