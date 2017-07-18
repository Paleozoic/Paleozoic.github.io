---
title: Docker基础：概念与原理
date: 2016-10-11 16:45:23
categories: [Docker]
tags: [Docker]
typora-root-url: ..
---
<Excerpt in index | 首页摘要>
Docker基础：概念与原理，该blog记录于2016-10。（资料来源于互联网，但是由于时间久远，部分资料引用源已经不知。）<!-- more -->
<The rest of contents | 余下全文>

# Docker与VM

Containers and virtual machines have similar resource isolation and allocation benefits -- but a different architectural approach allows containers to be more portable and efficient.

![docker_vm](/resources/img/docker/vm.png)![docker_vm](/resources/img/docker/docker.png)

- VIRTUAL MACHINES

> Virtual machines include the application, the necessary binaries and libraries, and an entire guest operating system -- all of which can amount to tens of GBs.

- CONTAINERS

> Containers include the application and all of its dependencies --but share the kernel with other containers, running as isolated processes in user space on the host operating system. Docker containers are not tied to any specific infrastructure: they run on any computer, on any infrastructure, and in any cloud.

# docker的隔离（命名空间）

Docker Engine uses namespaces such as the following on Linux:

 

- The pid namespace: Process isolation (PID: Process ID).

- The net namespace: Managing network interfaces (NET: Networking).

- The ipc namespace: Managing access to IPC resources (IPC: InterProcess Communication).

- The mnt namespace: Managing filesystem mount points (MNT: Mount).

- The uts namespace: Isolating kernel and version identifiers. (UTS: Unix Timesharing System).

# Docker组件

- Docker Client 是用户界面，它支持用户与Docker Daemon之间通信。

- Docker Daemon运行于主机上，处理服务请求。

- Docker Index是中央registry，支持拥有公有与私有访问权限的Docker容器镜像的备份。

# Docker元素

- Docker Containers负责应用程序的运行，包括操作系统、用户添加的文件以及元数据。

- Docker Images是一个只读模板，用来运行Docker容器。

- DockerFile是文件指令集，用来说明如何自动创建Docker镜像。

> Docker使用以下操作系统的功能来提高容器技术效率：

- Namespaces 充当隔离的第一级。确保一个容器中运行一个进程而且不能看到或影响容器外的其它进程。

- Control Groups是LXC的重要组成部分，具有资源核算与限制的关键功能。

- UnionFS（文件系统）作为容器的构建块。为了支持Docker的轻量级以及速度快的特性，它创建了用户层。

![docker_flow](/resources/img/docker/docker_flow.png)

# 联系

## image与container

[Docker的容器与镜像](http://blog.csdn.net/x931100537/article/details/49633107)

- image：镜像，存储态，只读。

- container：容器，运行时状态，读写。

container由image创建而来，可以理解为container是image的一个运行实例。

![container](/resources/img/docker/container.png)

### 注意：根据官网，这个图应该是：

https://docs.docker.com/engine/userguide/storagedriver/images/container-layers.jpg

![container-layers](/resources/img/docker/container-layers.png)

Docker官方：An image is software you load into a container。

所以理解应如下：

```

Read Layer+Read Layer... = Image layers = Image

Image layers + RW Layer(Container Layer) = Container

```

> PS：docker的设计思路类似于版本控制，

image 对应 repository

container 对应 working copy

## Image Definition

镜像（Image）就是一堆只读层（read-only layer）的统一视角

![image](/resources/img/docker/image.png)

> 图左是多个重叠的只读层。除了最下面一层，其它层都会有一个指针指向下一层。这些层是Docker内部的实现细节，并且能够在主机（译者注：运行Docker的机器）的文件系统上访问到。统一文件系统（union file system）技术能够将不同的层整合成一个文件系统，为这些层提供了一个统一的视角，这样就隐藏了多层的存在，在用户的角度（俯视图）看来，只存在一个文件系统。我们可以在图右看到这个视角的形式。

## Container Definition

 

容器（container）的定义和镜像（image）几乎一模一样，也是一堆层的统一视角，唯一区别在于容器的最上面那一层是可读可写的。

![container1](/resources/img/docker/container1.png)

要点：容器 = 镜像 + 可读层。并且容器的定义并没有提及是否要运行容器。

## Running Container Definition

 一个运行态容器（running container）被定义为一个可读写的统一文件系统加上隔离的进程空间和包含其中的进程。

![rw_file_system](/resources/img/docker/rw_file_system.png)

正是文件系统隔离技术使得Docker成为了一个前途无量的技术。一个容器中的进程可能会对文件进行修改、删除、创建，这些改变都将作用于可读写层（read-write layer）

![rw_layer](/resources/img/docker/rw_layer.png)

## Image Layer Definition

为了将零星的数据整合起来，我们提出了镜像层（image layer）这个概念。下面的这张图描述了一个镜像层，通过图片我们能够发现一个层并不仅仅包含文件系统的改变，它还能包含了其他重要信息。

![image-layer](/resources/img/docker/image-layer.png)

元数据（metadata）就是关于这个层的额外信息，它不仅能够让Docker获取运行和构建时的信息，还包括父层的层次信息。需要注意，只读层和读写层都包含元数据。

![metaspace](/resources/img/docker/metaspace.png)

除此之外，每一层都包括了一个指向父层的指针。如果一个层没有这个指针，说明它处于最底层。

![read-layer](/resources/img/docker/read-layer.png)

## 全局理解（Tying It All Together）

结合docker的命令来理解：

**`docker create <image-id>`**

![docker_create](/resources/img/docker/docker_create.png)

docker create 命令为指定的镜像（image）添加了一个可读层，构成了一个新的容器。注意，这个容器并没有运行。

![docker_create1](/resources/img/docker/docker_create1.png)

**`docker start <container-id>`**

![docker_start](/resources/img/docker/docker_start.png)

Docker start命令为容器文件系统创建了一个进程隔离空间。注意，每一个容器只能够有一个进程隔离空间。

**`docker run <image-id>`**

![docker_run](/resources/img/docker/docker_run.png)

docker start 和 docker run命令有什么区别? 

![docker_start_vs_run](/resources/img/docker/docker_start_vs_run.png)

显然，docker run = docker create + docker start

> PS：与Git对比，我认为docker run命令类似于git pull命令。git pull命令就是git fetch 和 git merge两个命令的组合，同样的，docker run就是docker create和docker start两个命令的组合。

运行程序步骤：

- 1. 构建一个镜像

- 2. 运行容器

用户通过Docker Client输入命令，发送信号至Docker Daemon，之后：

（1）构建镜像：Docker Image是一个构建容器的只读模板，它包含了容器启动所需的所有信息，包括运行程序和配置数据。构建的镜像可以发送到Docker Index供人使用。（类似maven）

（2）运行容器：