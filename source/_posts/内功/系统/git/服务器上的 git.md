---
title: 服务器上的 git
category:
  - 系统
  - git
tags:
  - 系统
  - git
keywords: 'git'
abbrlink: 5fbc34b4
date: 2018-02-21 03:00:00
updated: 2018-02-21 03:00:00
---

本文整理自 [Pro Git](https://git-scm.com/book/zh/v2)。

### 传输协议

git 可以使用四种传输资料的协议：Local 本地协议，Http 协议，SSH 协议和 Git 协议。

Local 协议常见于协作者对同一个共享的文件系统(如一个挂载的 NFS)都有访问权限。git clone /opt/git/project.git 命令将采用硬链接或直接拷贝文件；git clone file:///opt/git/project.git 将触发用于网路传输资料的进程，传输效率较低；git remote add local_proj /opt/git/project.git 命令用于增加本地版本库。

Http 协议既支持像 git:// 协议一样设置匿名服务，也可以像 SSH 协议一样提供传输时的授权和加密。Git 协议，要么谁都可以克隆这个版本库，要么谁也不能。

### 搭建 git 服务器

首先需要把现有 git 仓库导出为一个裸仓库，即不包含工作目录的仓库。这一过程通过 –bare 选项完成，具体命令为 git clone –bare my_project my_project.git。其次将裸仓库放在服务器上，若已在 git.example.com 搭好服务器，且需要在 /opt/git 目录下放置所有 git 仓库，通过执行 scp -r my_project.git user@git.example.com:/opt/git 复制裸仓库到 /opt/git 目录下。此时，对服务器 /opt/git 目录拥有可读权限的用户就可以通过 git clone user@git.example.com:/opt/git/my_project.git 克隆仓库。若在仓库中执行 git init –bare –shared 命令，可将仓库的权限设为可写(基于服务器文件系统权限)。连接服务器的权限通过服务器自有的 SSH 服务实现，创建访问账户也针对服务器，而不是 git 仓库。访问权限也可以在服务器创建 git 账户，然后将需要写权限的用户的 SSH 公钥加入 git 账户的 ~/.ssh/authorized_keys 文件中；或者让 SSH 服务器通过某个 LDAP 服务，或者其他已经设定好的集中授权机制，来进行授权。

生成 SSH 公钥通过以下步骤完成：通过 ssh-keygen 命令创建公钥(存放在 id_dsa.pub 文件中)和密钥(存放在 id_dsa 文件中)；将公钥复制到 git 账户的 ~/.ssh/authorized_keys 文件中(通过 git 服务器管理员完成，或自动化脚本实现)。

在服务器创建 git 账户时，将使所有获得授权的用户都能以系统用户 git 的身份登录服务器从而获得一个普通 shell，即能对服务器进行一定操作。若想对此加以限制，则需要修改 passwd 文件中(git 用户所对应)的 shell 值；或者通过 git 软件包自带的 git-shell 工具代替 bash 或 csh 作为用户的登录 shell，用户即不能通过登录 shell 执行非 git 命令(上述过程，通过 sudo chsh git 命令改变系统用户的登录 shell 实现)。

```bash
git clone --bare my_project my_project.git # 通过 --bare 选项创建裸仓库
cp -Rf my_project/.git my_project.git # 同上
git init --bare # 新疆一个空仓库
git init --bare --shared # 新疆一个空仓库，并将仓库的权限设为可写
```

### 疑问

1. 协议相关知识再梳理？
2. 搭建 git 服务器过程再梳理，包含守护进程、git-http-backend 脚本在Apache 服务器上的使用、GitWeb 网页查看、GitLab 服务器等？