---
title: 基于 Ansible 的分布式爬虫自动化部署
abbrlink: 23722
date: 2018-06-14 15:24:10
category: 
 - 运维
banner: 'https://saekiraku-1253597019.coscd.myqcloud.com/image/jsope.png'
---

使用 NodeJS + Redis + Ansible 实现分布式爬虫并且自动化批量部署上线。

<!-- more -->
---

# 分布式爬虫

之前在爬取一个网站的数据时，为了保证可以稳定的访问站点，不被网站屏蔽。尝试了很多方法，除了伪造请求头的UA、Cookie，还要对请求频率进行限制，或者通过代理服务器访问来改变请求来源。但是限制频率和请求代理会导致爬虫的效率变得非常低。因此，决定改成分布式爬虫，由多台服务器同时进行爬取。

该爬虫主要使用了 `NodeJS` 开发，并使用了 `Cheerio` 和 `ioredis` 两个框架。使用 Redis 同步各个爬虫的工作进度，并存储主要数据。

使用分布式爬虫，就需要为每个服务器安装环境，初始化配置等等。2、3个服务器还好说，但是如果有100台服务器，我们肯定不能手动的为每台服务器都配置一遍，1是人工容易出错，2是效率太低。

# Ansible 自动化运维工具

> 基于Python开发，集合了众多运维工具（puppet、cfengine、chef、func、fabric）的优点，实现了批量系统配置、批量程序部署、批量运行命令等功能。ansible是基于模块工作的，本身没有批量部署的能力。真正具有批量部署的是ansible所运行的模块，ansible只是提供一种框架。

除了 Ansible 以外，还有一款运维工具也是做服务器批量管理的 —— SaltStack。但是它的文档不够友好，并且由于通信机制的原因，需要在所有服务器上初始化一次 Salt Stack Minion，故没有选择此方案。（虽然 SaltStack 在后来也加入了 ssh 连接的支持）

## 安装 Ansible

在官方文档中有很多安装方法，但是在这里我选择了使用 pip 进行安装（懒，图省事）。
```bash
pip install ansible
```

**注意：使用 Ansible 时，进行管理的宿主机，必须是 Linux/MacOS/Unix 系统。**

## 服务器与分组

在很多教程中，对 Ansible 的服务器配置都是要修改 `/etc/ansible/hosts` 文件来实现的，但是通过 pip 安装是不会产生这个文件的。

当然我们可以手动去创建，但是尝试在终端中输入 `ansible --help` 就可以在帮助手册中看到能通过 -i 参数来指定 `inventory` 文件。

`inventory` 文件的格式简单的说就是用 `[]` 表示分组，用于批量管理，然后在分组下面写上服务器IP就可以了。IP后面写连接配置，例如一个服务器的登录用户名是root，密码是123456的配置写法是：

```ini
[master] # 分组名称
127.0.0.1 ansible_ssh_user=root ansible_ssh_pass=123456
```
**注意：这种在配置中记录密码的方式并不安全，所以不推荐。最好是使用SSH密钥。**

这个时候，我们就可以执行一些简单的命令，来测试配置是否正确了。在终端中输入 `ansible master -i ./inventory -m ping` 当你得到如下图所示的输出时，就算配置正确了。

![使用 ping 命令测试与服务器的连接](https://saekiraku-1253597019.coscd.myqcloud.com/image/zd7ol.png)

最后，我们来分析一下真实的情况来写一个实际可用的配置。在这套爬虫系统中，除了每个服务器运行1个爬虫以外，还要有个服务器运行 redis 服务作为信息中枢。所以：所有服务器环境配置相同，单独拿出其中一个服务器再安装一个 redis 服务就可以了。同时，服务器连接使用 SSH 密钥的方式，最终配置如下。

```ini
[master]
0.0.0.0 ansible_ssh_private_key_file=./private.pem

[default]
0.0.0.0 ansible_ssh_private_key_file=./private.pem
0.0.0.0 ansible_ssh_private_key_file=./private.pem
0.0.0.0 ansible_ssh_private_key_file=./private.pem
0.0.0.0 ansible_ssh_private_key_file=./private.pem
......
# 配置中省略了服务器真实IP和数量
```

## 编写 playbook.yml

`playbook.yml` 是 Ansible 的主要配置文件，通过它我们可以安装环境、执行脚本、编排任务等等。我们先以一个最简单的例子上手 —— 安装 Python 运行环境（假定你的服务器是 CentOS 系统，使用 yum 软件包管理器）。

```yml
---
- hosts: default
  # 服务器组
  remote_user: root
  # 登录服务器用的用户名
  sudo: yes
  # 是否使用 sudo 执行命令
  tasks:
  # 任务列表
  - name: 安装 python 环境
    # 任务名称
    yum: 
    # 任务模块
      name: python
      # yum 模块中的 name 参数（安装的软件包）
      state: installed
      # yum 模块中的 state 参数（要达到的状态）
```

然后我们在终端中输入 `ansible-playbook -i ./inventory playbook.yml -f 10`（-f 的数值是启动的进程数），就可以完美的跑起来这个脚本了。最后我们手动连入服务器，运行 `python --version` 来检查是否安装成功。（PS：在部分系统镜像中，python可能已经内置了）

在之前的简介中我们也说过 `Ansible是基于模块工作的，本身没有批量部署的能力。真正具有批量部署的是ansible所运行的模块，ansible只是提供一种框架。` 所以，如果我们想通过 yum 安装软件包，那么就用 yum 模块；如果我们想管理 docker 容器，那么就用 docker_container 模块。当然，服务软件那么多，Ansible 的模块肯定不能做到完全覆盖。因此，我们还可以通过 command 模块写自定义命令来实现上述功能。

```yml
---
- hosts: default
  remote_user: root
  sudo: yes
  tasks:
  - name: 安装 python 环境
    command: yum install -y python
```

那么这两种模式有什么区别？实际上我们观察 yum 模块的 `state` 的参数就会发现，他的值写的是 `install` 的过去式 `installed` 。这就表示 Ansible 追求的是任务达到了某个状态，而不是去做某个动作。这样做实际上是为了实现`幂等性`，也就是不管我们重复多少次操作，该操作的最终结果应该是一致的。

举个例子，假设 yum 包管理器不会为我们自动跳过已经安装的软件包，那么我们每执行一次这个配置，都会重复安装一遍 python，最后就会导致硬盘容量越来越小，浪费了很多空间。所以，在多数情况下，我们应该尽可能的使用 Ansible 的模块来保证`幂等性`。

最后，根据我的项目的实际情况，写出如下的 `playbook` 配置：

```yml
---
- hosts: default
  remote_user: root
  sudo: yes
  tasks:
  - name: 安装 python 环境
    yum: 
      name: python
      state: installed
  - name: 安装 git 版本管理工具
    yum: 
      name: git
      state: installed
  - name: 安装 Docker 环境（yum-utils）
    yum: 
      name: yum-utils 
      state: installed
  - name: 安装 Docker 环境（device-mapper-persistent-data）
    yum: 
      name: device-mapper-persistent-data 
      state: installed
  - name: 安装 Docker 环境（lvm2）
    yum: 
      name: lvm2 
      state: installed
  - name: 安装 Docker 环境（add repo）
    yum_repository: 
      name: docker-ce
      description: Docker CE Stable
      baseurl: https://download.docker.com/linux/centos/7/$basearch/stable
      gpgkey: https://download.docker.com/linux/centos/gpg
      gpgcheck: yes
  - name: 安装 Docker 环境（docker-ce）
    yum: 
      name: docker-ce
      state: installed
      enablerepo: docker-ce
  - name: 启动 Docker
    systemd: 
      name: docker 
      state: started
      enabled: 
  - name: 安装 docker-py
    pip:
      name: docker-py
  - name: 下拉 nodejs 镜像
    docker_image: 
      name: "node:9.11-alpine"


- hosts: master
  remote_user: root
  tasks:
  - name: 下拉 redis 镜像
    docker_image: 
      name: "redis:3.2-alpine"
  - name: 启动 redis
    docker_container:
      name: redis
      image: "redis:3.2-alpine"
      restart_policy: always
      ports:
        - "6379:6379"
      volumes:
        - /app/data/redis:/data
      env: 
        appendonly: yes
      state: started

# 配置中省略了 NodeJS 容器的安装、运行等操作，因为大体的配置基本一样。更多操作直接参考官方文档即可。
```

最后，我们在终端中输入 `ansible-playbook -i ./inventory playbook.yml -f 10`，就可以自动化的初始化系统环境，并运行相应服务了。

![运行效果](https://saekiraku-1253597019.coscd.myqcloud.com/image/l5mia.png)
![服务器列表](https://saekiraku-1253597019.coscd.myqcloud.com/image/dckaf.png)