---
title: git 分支
category:
  - 系统
  - git
tags:
  - 系统
  - git
keywords: 'git'
abbrlink: b3382855
date: 2018-02-21 02:00:00
updated: 2018-02-21 02:00:00
---

本文整理自 [Pro Git](https://git-scm.com/book/zh/v2)。

### 基本命令

```bash
git log --oneline --decorate # --decorate 选项用于查询各个分支及 HEAD 指针指向哪个提交对象
git log --oneline --decorate --graph --all # 查看提交历史的分叉情况，各个分支及 HEAD 指针的指向
git branch # 查看所有分支，带 '*' 的为当前分支
git branch -v # 查看所有分支最后一次提交记录
git branch --merged # 查看已合并到当前分支的所有分支
git branch --no-merged # 查看未合并到当前分支的所有分支
git branch --merged <branchname> # 查看已合并到 branchname 分支的所有分支
git branch --no-merged <branchname> # 查看未合并到 branchname 分支的所有分支
git branch <branchname> # 创建分支，但不切换分支
git branch -d <branchname> # 删除分支，branchname 未合并到当前分支，将不予删除
git branch -D <branchname> # 强制删除分支
git checkout <branchname> # 切换分支，即改变 HEAD 指针指向。修改文件但未提交到版本库的，git 将阻止切换
git checkout -b <branchname> # 创建并切换分支
git merge <branchname> # 将 branchname 分支合并到当前分支
git status # 查看冲突文件
git mergetool # 使用图形化工具解决冲突，默认使用 opendiff 工具，可选用工具包含 opendiff, kdiff3, tkdiff, xxdiff, meld, tortoisemerge, gvimdiff, diffuse, diffmerge, ecmerge, p4merge, araxis, bc3, codecompare, vimdiff, emerge
git mergetool --tool-help # 合并工具帮助信息
```

### 工作流

推荐使用的本地开发工作流/workflow 有两种，长期分支模式(Long-Running Branches)，特性分支模式(Topic Branches)。当然，这只是一种参考。

长期分支模式根据稳定性创建分支，需要较高稳定性的分支创建在前，较低稳定性的创建在后，如 master 分支先于 develop 分支，develop 分支先于 topic 分支，合并时依序将 topic 分支合并到 develop 分支，develop 分支合并到 master 分支。适用于大型项目，作为分支创建的基础结构，在每个分支可以再次使用特性分支模式。

特性分支模式适用于开发者想法多变的场景，比如在 master 分支基础创建 hotfix 分支，接着创建 hotfix_v2，同时开发 hotfix, hotfix_v2 分支，最终又启用 hotfix 分支；与此同时，开发过程中 master 分支不只产生了新的提交，后续又创建了实现新想法的 idea 分支，最后将 hotfix_v2, idea 分支合并到 master 分支。

### 远程分支

```bash
git ls-remote <shortname> # 查看远程引用清单
git remote show <shortname> # 查看远程引用详细信息
git push <shortname> <branch> # 将当前分支的提交内容推送到远程 branch 分支，git 自动将分支名扩展为 refs/heads/branch:refs/heads/branch，意为将本地 branch 分支推送并更新远程 branch 分支
git push shortname localebranch:remotebranch # 推送本地 localebranch 分支，将其作为远程 remotebranch 分支
git merge shortname/branchname # 将远程跟踪分支 shortname/branchname 合并到当前分支，合并前须 git fetch
git checkout -b localebranch shortname/remotebranch # 通过远程跟踪分支创建本地分支，自动跟踪远程分支，可使用 git pull|push 简化拉取、提交操作；且可以在命令行中使用 @{upstream} 或 @{u} 代替 shortname/remotebranch
git checkout --track shortname/remotebranch # 上一条命令的简写形式，创建 remotebranch 同名分支
git checkout remotebranch # 上一条命令的简写形式，条件是本地未存在 remotebranch 同名分支，远程存在
git branch -u|--set-upstream-to shortname/remotebranch # 当前分支切换跟踪远程 remotebranch 分支。-u 选项可用于 pull, push 命令
git branch -vv # 查看本地分支在跟踪哪个远程分支，以及ahead, behind, up to date等提交状态。这条命令基于最后一次 fetch 的数据，命令本身没有连接服务器
git fetch <shortname|url> # 抓取远程仓库资源，包含所有分支，并创建本地远程跟踪分支，却不会创建可修改、供开发的本地同名分支。只下载远程仓库资源，不会更改工作区内容，需手动 merge
git pull # git fetch, git merge 命令的结合，拉取数据并合并
git push origin --delete remotebranch # 删除远程分支
```

### 变基

```bash
git rebase <basebranch> # 将当前分支以补丁形式在 basebranch 分支上创建新的提交对象。完成开叉分支的整合操作还需要在 basebranch 分支上 merge 当前分支
git rebase <basebranch> <topicbranch> # 同上，该命令不需要切换到 topicbranch 分支后执行
git rebase --onto <basebranch> <topicbranch1> <topicbranch2> # --onto 选项用于获取在 topicbranch2 分支内但不在 topicbranch1 内的文件修改补丁，并在 basebranch 分支上创建新的提交对象，即变基到 basebranch 
git rebase remote/branch # 将本地修改变基到远程引用 remote/branch 分支上
```