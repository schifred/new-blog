---
title: 分布式 git
category:
  - 系统
  - git
tags:
  - 系统
  - git
keywords: 'git'
abbrlink: 79af9853
date: 2018-02-21 04:00:00
updated: 2018-02-21 04:00:00
---

本文整理自 [Pro Git](https://git-scm.com/book/zh/v2)。

### 分布式工作流

分布式工作流有三种，包含集中式工作流、集成管理者工作流、司令官和副官工作流。一般项目采用集中式工作流，当 A 用户提交了代码后，B 用户再次提交前，先须拉取代码并合并，然后才能提交；否则会报提交失败。集成管理者工作流通常在 github 开源代码中采用，即作为开源代码的贡献者，先须 fork 该项目，创建一个自己的仓库，更新提交后，发送消息给开源代码的维护者，等待维护者合并贡献者的代码并提交。司令官和副官工作流见于 linux 开发项目，普通开发者完成开发后，基于司令官的 master 执行变基操作；副官再将普通开发者的分支合并到自己的 master 分支上；司令官合并所有副官的 master 分支并提交。

#### 提交准则

1. 避免空白错误。
2. 尝试让每一个提交成为一个逻辑上的独立变更集，即针对每个问题，独立提交一次。
3. 完善提交信息，使用 vim 编辑提交信息，推荐使用 标题 + 空行 + 正文 形式。

### 贡献代码

#### 私有小型团队

用户 A 创建特性分支 feature1 并作修改后，再将特性分支合并到 master 分支上。用户 B 创建了特性分支 feature2 并作修改后，若在此时，用户 B 想把 feature2 内容合并到 master 分支并作提交，那他需要在 master 分支上执行 git fetch; git merge origin/master; git merge feature2 命令，即合并本地特性分支和远程协作者提交内容后，才可以正式提交代码。这样的工作流程最常见于实际项目中。

#### 私有大型团队

用户 A 和 B 在特性分支 feature1 上工作，用户 B 和 C 在特性分支 feature2 上工作。工作流程采用了整合-管理者工作流程，即独立小组的工作只能被特定的工程师整合，主仓库的 master 分支只能被那些工程师更新。当 feature1, feature2 开发并测试完成后，再由整合者将两个分支的内容合并到 master 分支上。对于 feature1, feature2 的开发工作流程，同小型私有团队，即提交前先要合并协作者上传到远端的代码。

#### 派生的公开项目

首先 clone 项目，然后在本地创建 feature1 分支(使 master 分支保持干净，避免维护者不采用你的代码时，需要回滚 master 分支到最初 clone 时的状态)，fork 仓库(即派生项目)后提交到远程自己的同名仓库中。通知开源项目的维护者拉取你的改动，这通常被称为拉取请求(pull request)，可通过执行 git request-pull origin/master feature 命令告知维护者改动是在你 fork 的仓库的 feature 分支上完成的。

#### 通过邮件的公开项目

有几个历史悠久的、大型的项目会通过一个开发者的邮件列表接受补丁。使用 git format-patch 来生成可以邮寄到列表的 mbox 格式的文件 - 它将每一个提交转换为一封电子邮件，提交信息的第一行作为主题，剩余信息与提交引入的补丁作为正文。-M 选项用于告诉 Git 查找重命名。通过 git config 设置 imap 区块的 folder = “[Gmail]/Drafts”, host = imaps://imap.gmail.com, user = user@gmail.com, pass = p4ssw0rd, port = 993, sslverify = false 属性(如果 IMAP 服务器不使用 SSL，无需设置 port, sslverify 属性，host 的值会是 imap:// 而不是 imaps://)；再执行 git imap-send 可以将补丁序列放在特定 IMAP 服务器的 Drafts 文件夹。或者通过 git config 设置 sendemail 区块的 smtpencryption = tls, smtpserver = smtp.gmail.com, smtpuser = user@gmail.com, smtpserverport = 587 属性；再执行 git send-email 通过 SMTP 服务器发送补丁。

```bash
git diff --check # 检查空白错误。空白错误是指行尾的空格、Tab 制表符，和行首空格后跟 Tab 制表符的行为
git format-patch -M origin/master # 生成 .patch 扩展名的补丁文件，可以编辑该文件，添加额外信息
cat *.patch |git imap-send # 通过 IMAP 发送正确格式化的补丁
git send-email *.patch # 通过 SMTP 服务器发送补丁
```

### 维护项目

#### 应用补丁

若补丁通过 git diff 生成，使用 git apply 命令可在当前工作分支中应用文件补丁。应用补丁前，使用 git apply –check 命令可检查补丁是否可以顺利应用。

若补丁通过 format-patch 生成，使用 git am 命令可以提取出 mbox 文件中的实际变更并应用，且自动创建新的提交。

#### 合并工作流

合并工作流，可以将特性分支逐次合并到 master 分支上；或者保持 master 分支的稳定性，再创建 develop 长期分支，将特性分支合并 develop 分支，等 develop 分支稳定后，再合并到 master 分支上；或者在 master 之外创建 next, pu(proposed updates，用于新工作), maint(maintenance backports，用于维护性向后移植工作) 分支，安全的特性分支先合并入 next 分支，再合并到 master 分支上。

#### 拣选

执行 git cherry-pick，见原理。

#### 重用冲突解决方案/reuse recorded resolution

执行 git config –global rerere.enabled true 缓存冲突解决方案。执行 git rerere 命令将从缓存中查找相似的冲突，并应用对应的解决方案。

```bash
git apply *.patch # 在当前分支中应用补丁
git apply --check *.patch # 检查补丁是否可被顺利应用
git am *.patch # 读取 mbox 文件实际变更并应用
git am -3 *.patch # -3 选项将使用三方合并应用补丁
git am -3 -i mbox # 使用交互模式应用多个补丁
git am --resolved # git am 报错后，手动解决冲突，执行下一个补丁的应用
git am --skip # 跳过当前补丁
git am --abort # 当前分支回退到应用补丁前状态，终止补丁应用
git log branch1 --not branch2 # 找出仅属于 branch1 但不属于 branch2 的提交记录
git diff branch # 当前工作分支与 branch 分支比较差异
git merge-base branch1 branch2 # 找出 branch1 和 branch2 的共同祖先
git diff branch1...branch2 # 比较 branch1 的最新提交和两个分支的共同祖先之间的差异
git cherry-pick <checksum> # 以某个提交对象为基底在当前工作分支上创建新的提交对象，将重新计算校验和，如 git cherry-pick e43a6fd3e94888d76779ad79fb568ed180e5fcdf
git config --global rerere.enabled true # 开启冲突解决方案缓存
git rerere # 查找相似的冲突解决方案并应用

# 发布
git tag -s <version> -m 'message' # 打标签
gpg --list-keys # 查找 GPG 公钥
gpg -a --export <gpgkey> | git hash-object -w --stdin # git hash-object 命令以 blob 对象形式在 git 中导入公钥，并返回该 blob 对象的 SHA-1 值。该 SHA-1 值可以用于发布标签，如 git tag -a <tagname> <sha-1>
git show <tagname> | gpg --import # 开发者获取并导入公钥，用于贡献代码
git describe <branch> # 生成构建号，包含最近的标签名、自该标签之后的提交数目和你所描述的提交的部分 SHA-1 值
git archive master --prefix='project/' | gzip > `git describe master`.tar.gz # 创建 .tar.gz 压缩包
git archive master --prefix='project/' --format=zip > `git describe master`.zip # 创建 .zip 压缩包
git shortlog --no-merges master --not v1.0.1 # 生成自 v1.0.1 版本发布以来所有提交的总结
```

疑问：

1. 公共项目的发布问题？