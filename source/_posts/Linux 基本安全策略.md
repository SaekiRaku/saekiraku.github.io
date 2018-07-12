---
title: Linux 基本安全策略
category: 运维
tags:
 - Linux
 - 安全
banner: "https://saekiraku-1253597019.coscd.myqcloud.com/image/9zx2j.png"
abbrlink: 21343
date: 2018-07-11 16:07:40
---

在真实经历过“删库跑路”事件后，吸取教训、反思总结的一套服务器安全策略。

<!--more-->

---

> 本文章基于以下CC协议进行知识共享：署名（BY）-非商业性使用（NC）-禁止演绎（ND）

---

# 1. 用户与用户组

涉及命令：[useradd](http://man.linuxde.net/useradd), [groupadd](http://man.linuxde.net/groupadd), [userdel](http://man.linuxde.net/userdel), [groupdel](http://man.linuxde.net/groupdel), [passwd](http://man.linuxde.net/passwd) （点击命令可以查看帮助手册）

通过用户及用户组的管理和分配，可以很方便的针对不同目录、文件、程序进行读写及运行限制。** 因为让所有管理员直接登录 root 进行管理是非常危险的操作 **。通过用户（组）管理，我们可以做到：只允许 somebody 用户访问 /home，或者允许整个 admin 用户组的用户访问 /home。

## 用户组管理
```bash
# 新建 admin 用户组
groupadd admin

# 新建 test 用户组并指定 ID
groupadd -g 666 admin

# 删除 test 用户组
groupdel test
```

## 用户管理
```bash
# 新建 test 用户
useradd test

# 新建 somebody 用户并进行设置
user -c "某用户" -d /home -g admin -M -n
# -c : 备注
# -d : 登入时的初始目录
# -g : 所属用户组
# -M : 不要自动建立用户的登入目录
# -n : 取消建立以用户名称为名的群组

# 为 somebody 用户设置密码
passwd somebody
```

** 注意：手动设定用户ID值时要尽量要大于500，以免冲突。因为Linux安装后会建立一些特殊用户，一般0到499之间的值留给bin、mail这样的系统账号。**

## 查看系统中已有的用户和用户组
```
cat /etc/passwd
cat /etc/group
```

# 2. 禁止 root 账号直接登录

涉及命令：`vim`, [service](http://man.linuxde.net/service)

完成上述操作后，我们就可以配置 ssh 来禁止 root 直接登录了。因为 root 是所有 Linux 系统的默认超管账号，所以很容易进行穷举密码。更安全的方法是通过创建的账号登录到服务器后，使用 `su` 命令来认证到 root 账号。

在登录方面，也可以配置公钥或私钥认证来进行登录，一是可以免去每次登陆都要输密码的麻烦，二是避免在现实世界中输入密码被有心人看到并记下，三是加大穷举破解的难度。

```bash
# 编辑 sshd 配置文件
vim /etc/ssh/sshd_config

# 按下 ESC 键输入 /PermitRootLogin 进行搜索（vim技巧）
# 将 PermitRootLogin 的值改为 no 然后保存文件

# 重启 sshd 服务
service sshd restart
```

# 3. 目录权限设置

涉及命令：[chmod](http://man.linuxde.net/chmod), [chown](http://man.linuxde.net/chown)

chmod 命令是设置文件或应用的读写和可执行权限的，chown 是用来设置文件或应用的所有者的。具体关系为，不同用户针对不同的文件，拥有以下几种情况：1、所有者 `User` ：文件所属的用户。2、所有组 `Group` ：文件所属的用户组。3、`Other` 除了所有者和所有组之外。4、`All` 全部的用户。而针对不同的情况，可以设置不同的读写权限，例如：文件所有者可以读写和执行文件，其他用户或群组不可对该文件进行任何读写或执行。

一般情况下我们会将项目统一部署在一个根目录文件夹内，例如 `/home/project` 。因此，我们可以用 `chown` 命令对该文件夹设置所属用户组为管理用户组（admin），然后对 project 目录下的不同项目设置不同的所有者。最后，通过 `chmod` 对不同目录设置不同的读写执行权限来进行限制。同时，由于根目录下的所有文件夹默认是只有 root 用户可以进行写操作的，因此可以完美的限制运维管理账户的管理范围。

** 具体使用方法点击设计命令查看文档即可 **

# 4. Sudoer 配置

涉及命令：[sudo](http://man.linuxde.net/sudo), [visudo](http://man.linuxde.net/visudo)

`sudo` 命令是Unix/Linux平台上的一个非常有用的工具，允许为非 `root` 用户赋予一些合理的"权限"，让他们执行一些只有根用户或特许用户才能完成的任务，从而减少根用户的登陆次数和管理时间同时也提高了系统安全性。

`visudo` 命令则是用于编辑 `/etc/sudoer` 配置文件的，该文件主要描述了哪些用户或用户组可以使用 `sudo` 命令来执行什么命令。`/etc/sudoer` 配置文件本身是可以通过 `vim` 等工具进行编辑的，使用 `visudo` 的主要用途是：可以避免多个管理员同时进行修改，然后提供一些简单的语法检查。

# 5. Iptables 防火墙

涉及命令：[iptables](http://man.linuxde.net/iptables)

防火墙主要配置一些敏感服务的对外访问，例如 Docker Daemon Remote API、Redis 等等。类似这些不需要公开到外网的服务，应在 `iptables` 上做好访问限制。其次，由于 Linux 系统中 1024 以下的端口均为系统保留端口，所以当启用需要占用 80 端口的 Nginx、Apache 时会被提示没有权限。如果不想给运维管理人员 sudo 权限的话，可以考虑将 Nginx 的端口绑定到大于 1024 的端口号上，然后通过 `iptables` 的 NAT 表进行转发。

> 参考文章：
> * [https://blog.csdn.net/sethcss/article/details/73395617](https://blog.csdn.net/sethcss/article/details/73395617)
> * [http://blog.51cto.com/wt7315/2051136](http://blog.51cto.com/wt7315/2051136)

# 6. 应用服务安全

> 参考文章：
> * [常见的 PHP 安全防范](https://blog.csdn.net/zhezhebie/article/details/72645175)
> * [Nginx 基本安全优化](https://blog.csdn.net/wh2691259/article/details/72814419)