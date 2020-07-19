---
title: git 内部原理
category:
  - 系统
  - git
tags:
  - 系统
  - git
keywords: 'git'
abbrlink: 4fdd8a57
date: 2018-02-21 07:00:00
updated: 2018-02-21 07:00:00
---

本文整理自 [Pro Git](https://git-scm.com/book/zh/v2)。

从根本上来讲 git 是一个内容寻址（content-addressable）文件系统，并在此之上提供了一个版本控制系统的用户界面。

.git 目录包含：description 文件仅供 GitWeb 程序使用；config 文件包含项目特有的配置选项；info 目录包含一个全局性排除文件；hooks 目录包含客户端或服务端的钩子脚本；objects 目录存储所有数据内容；refs 目录存储指向数据（分支）的提交对象的指针；HEAD 文件指示目前被检出的分支；index 文件保存暂存区信息。

### git 对象

git 是一个内容寻址文件系统。这意味着，git 的核心部分是一个简单的键值对数据库（key-value data store）。 你可以向该数据库插入任意类型的内容，它会返回一个键值，通过该键值可以在任意时刻再次检索（retrieve）该内容。 通过底层命令 hash-object 可将任意数据保存于 .git 目录，并返回相应的键值。在 git 中，文件内容存储为数据对象/blob object，文件目录存储为树对象/tree object。一个树对象包含了一条或多条树对象记录（tree entry），每条记录含有一个指向数据对象或者子树对象的 SHA-1 指针，以及相应的模式、类型、文件名信息。

通常，Git 根据某一时刻暂存区所表示的状态创建并记录一个对应的树对象。所以创建树对象时，先须把文件写入暂存区，以此获得树对象的校验和。但是，树对象的校验和不便于记忆，即不便于获取该树对象。在 git 中，使用提交对象定位树对象，以及该树对象的创建时间，即通过提交对象的校验和获取或存储树对象。提交对象的格式为，先指定一个顶层树对象，代表当前项目快照；然后是作者/提交者信息（依据 user.name 和 user.email 配置来设定，外加一个时间戳）；留空一行，最后是提交注释。

运行 git add 和 git commit 命令时，git 所做的实质工作——将被改写的文件保存为数据对象，更新暂存区，记录树对象，最后创建一个指明了顶层树对象和父提交的提交对象。数据对象、树对象和提交对象最初均以单独文件的形式保存在 .git/objects 目录下（首先转换为带有如 “blob #{content.length}\0” 等头部信息的内存数据，其次计算校验和，最后再写入 .git/objects 目录下）。

在存储文件对象时，为避免同一文件的不同版本保存多份，git 使用包文件存储，即保存最新版本的完整数据，其他版本保存与最新版本的差异。使用 git gc 命令可执行打包过程；git verify-pack 命令查看打包的内容。自动打包过程发生在 push 命令触发或本地包含太多不同版本的文件，大约需要 7000 个以上的松散对象或超过 50 个的包文件才能让 git 启动一次真正的 gc 命令（可通过修改 gc.auto 与 gc.autopacklimit 的设置来改动这些数值）。打包后，.git/refs 目录将清空，相应创建 .git/packed-refs 目录；更新引用时，再相应创建 .git/refs 目录下文件；查找分支引用先从 .git/refs 目录找起，其次 .git/packed-refs 目录

```bash
echo 'test content' | git hash-object -w --stdin # 写入数据。-w 选项指示 hash-object 命令存储数据对象；若不指定此选项，则该命令仅返回对应的键值； --stdin 选项则指示该命令从标准输入读取内容；若不指定此选项，则须在命令尾部给出待存储文件的路径。返回40位校验和，以前两位作为目录名，后38位作为文件名，存储在 .git/objects 目录中
git hash-object -w test.txt # 写入数据
git cat-file -p <checksum> # 读取数据
find .git/objects -type f # 查看 .git/objects 目录下文件列表
git cat-file -p <checksum> > <filepath> # 将 .git/objects 目录下存储的某条数据导出到 filepath 文件中
git cat-file -t <checksum> # 查看 git 存储数据的对象类型，如返回 blob 或 tree

git cat-file -p master^{tree} # 查看树对象 master^{tree} 下的内容，可能包含 blob 或 tree 对象。master^{tree} 表示 master 分支上最新的提交所指向的树对象
git cat-file -p <checksum> # 查看树对象、提交对象下的内容，将获取最新版本的文件内容
git update-index --add --cacheinfo 100644 \
  83baae61804e65cc73a7201a7252750c76066a30 test.txt # 通过 update-index 命令为 test.txt 文件创建暂存区，选项 --add 将文件存储到暂存区中；选项 --cacheinfo 将文件添加到 git 数据库，而非当前目录中；文件模式 100644 表示为普通文件，100755 为可执行文件，120000 为符号链接
git update-index <filepath> # 更新暂存区中文件
git update-index --add <filepath> # 添加暂存区中文件
git write-tree # 将暂存区内容写入树对象，返回树对象的 SHA-1 值
git read-tree <checksum> # 将树对象读入暂存区
git read-tree --prefix=dirname <checksum> # 将树对象读入暂存区，--prefix 选项将其作为子树存储在 dirname 目录下

echo 'first commit' | git commit-tree <checksum> # 以树对象 checksum 创建提交对象，返回提交对象的校验和。创建的提交对象可通过 git log 查看
```

### git 引用

git 通过 .git/refs 目录下的文件存储提交对象的引用，以分支形式分类存储，不同分支创建不同的文件，同一文件下包含该分支的提交记录历史。

HEAD 引用通过 .git/HEAD 文件存储，文件内容如 ref: refs/heads/master，指向其他分支引用。

标签对象（tag object）非常类似于一个提交对象——它包含标签创建者信息、日期、注释信息，以及一个指针。 主要的区别在于，标签对象通常指向一个提交对象，而不是一个树对象。它像是一个永不移动的分支引用——永远指向同一个提交对象，分支引用指向的提交对象可以改写。所以标签对象可用于发布。另外要注意的是，标签对象并非必须指向某个提交对象；你可以对任意类型的 Git 对象打标签。标签对象存储在 .git/refs/tags 目录中，以标签名作为文件名。

.git/refs/remotes 目录下保存远程引用。远程引用和分支引用的主要区别在于，远程引用是只读的。虽然可以 git checkout 到某个远程引用，但是 git 并不会将 HEAD 引用指向该远程引用。因此，永远不能通过 commit 命令来更新远程引用，而只能通过 push 命令更新。

```bash
git update-ref refs/heads/master 1a410efbd13591db07496601ebc7a059dd55cfe9 # 在 master 分支中添加提交对象的校验和，比直接编辑文件更安全
git symbolic-ref HEAD # 查看 .git/HEAD 文件内容
git symbolic-ref HEAD refs/heads/test # 修改 HEAD 引用的指向
git update-ref refs/tags/v1.0 cac0cab538b970a37ea1e769cbbde608743bc96d # 创建轻量标签
git tag -a v1.1 1a410efbd13591db07496601ebc7a059dd55cfe9 -m 'test tag' # -a 选项用于创建附注标签
```

### 引用规格

引用规格/refspec 用于设定 fetch 命令的请求地址和拉取的分支，以小节形式存储在 .git/config 文件中。引用规格的格式由一个可选的 + 号和紧随其后的 : 组成，其中 是一个模式（pattern），代表远程版本库中的引用； 是那些远程引用在本地所对应的位置。 + 号告诉 Git 即使在不能快进的情况下也要（强制）更新引用。

执行 git remote add origin https://github.com/schacon/simplegit-progit 命令，.git/config 文件将添加引用规格如下：

```bash
[remote "origin"]
	url = https://github.com/schacon/simplegit-progit
	fetch = +refs/heads/*:refs/remotes/origin/* # 当修改为 fetch = +refs/heads/master:refs/remotes/origin/master 时，将只抓取远程 master 分支（fetch 可以设置多个，以拉取不同分支的引用）；或者单次执行 git fetch origin master:refs/remotes/origin/mymaster 命令，将远程 master 分支拉到本地的 origin/mymaster 分支
  # push = refs/heads/master:refs/heads/qa/master 用于设置推送，或执行 git push origin master:refs/heads/qa/master 命令
```

执行 git push origin :topic 命令可删除远程 topic 分支，因为 src 为空值，即代表删除。

### 数据恢复

引用日志/reflog 记录每一次你改变 HEAD 时它的值。每一次提交或改变分支时，引用日志都会被更新。引用日志也可以通过 git update-ref 命令更新。git reflog 命令用于查看引用日志。引用日志存放在 .git/logs/ 目录中。

引用日志可用于找回丢失的提交，如 reset, rebase 命令导致丢失。通过 git branch 命令创建新分支指向丢失的提交。

当引用日志也同样丢失时，可以使用 git fsck –full 查看没有被引用的提交对象，即丢失的提交对象。

```bash
git reflog # 查看引用日志
git log -g # 以标准日志格式查看引用日志
git count-objects -v # 查看内存占用，size-pack 指以 kb 为大小的包文件
```

### 疑问

1. git 的远程交互机制，参考传输协议
2. 移除git 数据库中的历史文件，参考维护与数据恢复
3. git 环境变量，参考环境变量