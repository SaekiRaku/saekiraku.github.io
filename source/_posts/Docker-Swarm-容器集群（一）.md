---
title: Docker Swarm 容器集群（一）
abbrlink: 53786
date: 2018-10-29 09:34:11
tags:
banner: https://saekiraku-1253597019.coscd.myqcloud.com/image/m4cz7.png
---

入门篇，简单介绍了基于 Docker Machine 搭建 Swarm 集群的方法。使用 Docker warm 可以让我们更容易的对应用、服务进行编排、管理，并解决很多 Runtime 问题。

<!-- more -->

---

> 本文章基于以下CC协议进行知识共享：署名（BY）-非商业性使用（NC）-禁止演绎（ND）

---

阅读本文需要有 Linux、Docker、Docker Compose 的基础。

# 简介
Docker Swarm 和 Docker Compose 一样，都是 Docker 官方容器编排项目，但不同的是，Docker Compose 是一个在单个服务器或主机上创建多个容器的工具，而 Docker Swarm 则是将若干台主机抽象为一个整体，并且通过一个入口统一管理这些主机上的各种 Docker 资源。Swarm 和 Kubernetes 比较类似，但是更加轻，具有的功能也较 Kubernetes 更少一些。

# Docker Machine

Docker Machine 是运行在桌面系统（Windows、MacOS）的 Docker Toolbox 中的一个工具，主要的功能是用来快速创建包含 Docker 环境的 Linux 虚拟机。实际上不使用 Docker Machine 也是可以使用 Swarm 集群的，使用 Docker Machine 只是为了模拟真实的服务器环境。

## 基本环境搭建

Docker Machine 本身并不包含虚拟机环境，它只负责调用相关的虚拟机接口（Microsoft Hyper-V、Virtual Box），自动下载并部署包含 Docker 全家桶环境的系统镜像。因此，在开始前我们需要安装以下虚拟机软件。

### MacOS：Virtual Box

Virtual Box 是开源的虚拟机软件。在 MacOS 系统下，Docker Machine 会默认使用该虚拟机软件。

[Virtualbox.org 官网下载](https://www.virtualbox.org)

### Windows：Microsoft Hyper-V 

在 Windows 下会默认使用 Hyper-V 但是通过 `--driver` 参数可以强制指定使用其他虚拟机软件。如果你使用的是 Windows 10 专业版，那么这个虚拟机软件已经集成在该版本的系统中了。

## 常用命令表

```bash
docker-machine create <虚拟机名称>
# 创建虚拟机

docker-machine ls
# 显示虚拟主机列表

docker-machine ssh <虚拟机名称>
# 使用 SSH 连接到虚拟机

docker-machine ip <虚拟机名称>
# 获取虚拟机IP地址

docker-machine start <虚拟机名称>
# 启动虚拟机

docker-machine stop <虚拟机名称>
# 停止虚拟机

docker-machine kill <虚拟机名称>
# 强制停止虚拟机

docker-machine rm <虚拟机名称>
# 删除虚拟机
```

## 创建虚拟主机

为了模拟真实环境，我们创建 3 个节点，其中 1 个是 `管理节点` 还有 2 个 `工作节点` 。

```bash
docker-machine create manager
docker-machine create worker1
docker-machine create worker2
# manager、worker1、worker2 为虚拟机名称
```

## 挂载目录

为了在 Docker Machine 的虚拟机中部署应用，我们可以通过 `docker-machine scp` 命令将项目文件传入。或者为了方便我们随时修改，我们可以通过挂载目录的方式，将本地文件的任意修改，及时同步到虚拟机中。

\* 由于 mount 命令挂载目录是基于 `sshfs` 的，因此，需要先安装该工具及其前置依赖，以 MacOS 为例：
```
brew install osxfuse
brew install gettext
brew install sshfs
```

```bash
mkdir manager
mkdir worker1b
mkdir worker2
# 在本地为 3 个虚拟机分别创建目录

docker-machine mount manager:/home/docker ./manager
# 挂载目录 mount 命令的第一个参数指明了虚拟机名称和虚拟机目录，第二个参数指明了本地目录

touch manager/README.md
# 在本地目录新建一个文件
docker-machine ssh manager ls
# 使用 ssh 命令查看工作节点中的文件是否存在
```

OK，到这里 Docker Machine 的准备工作就已经完成了。

# Docker Swarm

Swarm 是一组基于 Docker 的物理主机集群，虽然集群模式相对以往的运行模式不同，但是你仍然可以使用你所熟悉的 Docker API 进行管理。并且多数情况下，你只需要在 `管理节点` 中进行部署操作，而无需登录到指定的机器中。

## 创建集群

首先，我们使用 `docker-machine ls` 命令，查看我们已经创建好的虚拟机，并记下 `管理节点` manager 的 IP 地址。

| NAME       | ACTIVE   | DRIVER       | STATE     | URL                         | ... |
|:----------:|:--------:|:------------:|:---------:|:---------------------------:|:---:|
| manager    | -        | virtualbox   | Running   | tcp://192.168.99.100:2376   | ... |
| worker1    | -        | virtualbox   | Running   | tcp://192.168.99.101:2376   | ... |
| worker2    | -        | virtualbox   | Running   | tcp://192.168.99.102:2376   | ... |

```bash
docker-machine ssh manager
# 进入管理节点

docker swarm init --advertise-addr 192.168.99.100
# 初始化 swarm 通过 --advertise-addr 参数确定集群管理节点所绑定的网卡

docker swarm join-token worker
# 使用 join-token 命令生成供工作节点使用的加入集群的 Token 串，并复制下来。

####### Ctrl + D 离开管理节点 #######

docker-machine ssh worker1
# 进入 worker1 工作节点

docker swarm join --token SWMTKN-1-1zcuycf2uqevg5zfg91sorwnyi2itcm99wj1evr8h8mvtt0vtg-blnoyanjibhkmlw6cdubn2dnl 192.168.99.100:2377
# 粘贴刚才在主节点获得的 Token 串

####### Ctrl + D 离开 worker1 节点，并对 worker2 重复上述操作 #######

```

最后我们回到 `管理节点` 使用 `docker node ls` 来查看我们创建的集群。

| ID                            | HOSTNAME            | STATUS              | AVAILABILITY        | MANAGER STATUS      | ENGINE VERSION    |
|:-----------------------------:|:-------------------:|:-------------------:|:-------------------:|:-------------------:|:-----------------:|
| mmbh5fpw0mljoiddyzldwekwh *   | manager             | Ready               | Active              | Leader              | 18.06.1-ce        |   
| w68ntsp3l8vcqns6doma5hqd4     | worker1             | Ready               | Active              |                     | 18.06.1-ce        |   
| ilq3qvycdp4739ghtvty02lsf     | worker2             | Ready               | Active              |                     | 18.06.1-ce        |

如果上述步骤中有任何的操作问题，都可以使用 `docker swarm leave` 命令来离开集群并重建。

## 部署应用

确定完成以上步骤后（创建虚拟机、创建 Swarm 集群、挂载目录），我们开始部署应用。为了降低学习成本，我们编写一个简单的 `Docker Compose` 文件。

```yml
# 该文件名为：docker-compose.yml
version: "3"
services:
  service-manager:
    image: alpine:3.8
    command:
      - top
```

然后将该文件放入我们挂载的目录中，使用 `docker-machine ssh manager` 命令进入管理节点。（**请注意：** 针对 Swarm 集群进行的操作，都必须在**管理节点中进行**）

```bash
docker swarm deploy -c docker-compose.yml swarm-manager
# 使用 docker swarm deploy 命令部署应用
# -c 参数指明了 Docker Compose 的配置文件
# 结尾的参数指明了该 stack 的堆名
```

最后使用 `docker ps -a` 命令，就可以看到我们部署的服务容器了。

| CONTAINER ID        | IMAGE               | COMMAND             | CREATED             | STATUS              | ... |
|:-------------------:|:-------------------:|:-------------------:|:-------------------:|:-------------------:|:---:|
| 3aa35fe94403        | alpine:3.8          | "top"               | 49 seconds ago      | Up 48 seconds       | ... |

使用 `docker stack ls` 命令可以查看当前集群中部署的所有应用堆

| NAME                | SERVICES            | ORCHESTRATOR  |
|:-------------------:|:-------------------:|:-------------:|
| swarm-manager       | 1                   | Swarm         |

如果你的容器没有运行起来，可以使用 `docker stack ps swarm-manager --no-trunc` 命令查看具体是因为什么原因而出错的。

## 将应用部署至特定的主机节点

部署特定主机节点，需要通过 Docker Compose 文件中的 `deploy.placement.constraints` 配置进行约束，例如我们可以将服务部署至所有 `Ubuntu` 系统上，或者只在 `工作节点` 上部署。当然，为了完全确定部署至哪个节点，我们需要给主机节点增加 `Labels` 标签，并通过标签进行约束。

```bash
docker-machine ssh manager
docker node update --label-add role=manager
# --label-add 参数传入的值是 KV 类型的，= 符号之前的 role 为键名，= 符号之后的 manager 为键值

docker-machine ssh worker1
docker node update --label-add role=worker1

docker-machine ssh worker2
docker node update --label-add role=worker2
```

然后，我们再为 `工作节点` 编写 2 个 Docker Compose 配置文件：

```yml
# worker1.yml
version: "3"
services:
  service-worker1:
    image: alpine:3.8
    deploy:
      placement:
        constraints:
          - node.labels.role == worker1
    command:
      - top

  service-worker2:
    image: alpine:3.8
    deploy:
      placement:
        constraints:
          - node.labels.role == worker1
    command:
      - top
```

最后，在管理节点执行部署命令。

```bash
docker swarm deploy -c worker1.yml swarm-worker1
```

此时，如果你在 `管理节点` 使用 `docker ps -a` 命令会发现 Docker 并没有创建新的容器。因为我们约束了应用的部署位置，之后我们进入 `worker1` 节点，查看容器列表就可以看到我们部署的应用了。

## 优雅的编写 Nginx 负载均衡

完成应用部署后，现在我们进入 `管理节点`，使用 `docker ps -a` 获得容器 ID，然后使用 `docker exec -it 容器ID /bin/sh` 进入容器内部。然后执行 `ping service-worker1` 会发现是可以 ping 通的，实际上当没有特殊配置时，容器会部署在 Swarm 自带的 Ingress 网络环境中（Overlay类型），在该网络环境中的所有容器服务，是可以 `跨主机` 进行通信的，并且会自动将服务名映射为 Hostname。现在，假设我们有 5 个工作节点，运行了一批服务，需要使用 Nginx 进行负载均衡，要怎么写？

```conf
upstream my-load-balance{
    server service-worker1:80;
    server service-worker2:80;
    server service-worker3:80;
    server service-worker4:80;
    server service-worker5:80;
    ...
}
```

简直爽爆了有没有，对于负载均衡器来说，我们完全不需要去管服务节点到底部署到了哪个主机上或者环境是怎样的。我们只需要知道服务名即可。

## 集群内置的负载均衡

类似上方简单策略的负载均衡，Swarm 集群本身也是可以做到的，你可以亲自试一试，将相同服务名的服务，部署在不同的主机上，然后在管理节点使用 ping 命令，其请求会随机转发到 2 台主机节点上。

# THE END

关于服务节点的管理、应用堆的管理、集群内容器的管理等文章，请参考我的下一篇文章（暂无）。