---
title: git 基础
category:
  - 系统
  - git
tags:
  - 系统
  - git
keywords: 'git'
abbrlink: d327b106
date: 2018-02-21 01:00:00
updated: 2018-02-21 01:00:00
---

本文整理自 [Pro Git](https://git-scm.com/book/zh/v2)。

### 创建本地仓库

创建 git 本地仓库，指定 git init 或 git clone [projectName] 命令（projectName 用于指定本地目录名）。git clone 将拷贝远程仓库的所有版本及所有文件（当服务器磁盘损坏时，方便使用本地仓库的命令将远程仓库的资源回滚到拷贝前），创建 .git 目录，并检出最新分支。git clone 时，可使用 https 协议 或 git:// 起始的 SSH 协议。

```bash
git init # 创建 git 本地仓库
git clone <url> [projectName] # 克隆远程仓库，指定远程仓库的简称默认为 origin，且本地 master 分支将跟踪/track 远程 master，可使用 git pull|push 命令
```

### 文件状态

工作区的文件状态有两种，已追踪/tracked, 未追踪/untracked。暂存区的最新快照中包含某文件，该文件即被追踪；若不包含，该文件即未追踪。已追踪文件又分为三种状态，未变更/unmodified, 已变更/modified, 已提交到暂存区/staged。在切换分支时，git 会校验工作区的文件改动是否提交到版本库中，未提交，则阻止切换，用以防止丢失工作区的改动。

```bash
git status # 获取文件状态
git status -s|--short # 文件状态扼要信息。?? 未追踪文件，A 新增文件提交到到暂存区，M 修改文件提交到暂存区，MM 暂存区和工作区文件状态
git diff # 工作区和暂存区文件差异，显示文件更新细节，包括添加的行、删除的行，合称为文件更新补丁/patch
git diff --staged|--cached # 暂存区和版本库文件差异
git difftool # 使用 Araxis, emerge, vimdiff 等软件以图形化或其他格式显示文件差异
git difftool -tool-help # 查看系统支持的 git diff 插件
git add <files> # 工作区文件提交到暂存区，将产生暂存区的历史快照/historical snapshort。参数为文件或目录路径
git commit # 暂存区快照提交版本去，需由命令行编辑器设置 message，# 起始内容将被忽略。命令行编辑器中默认以 # 添加 git status 输出内容。-v 选项可用于注入 git diff 完整信息
git commit -m 'message' # 设置 message 并提交
git commit -a # 直接将工作区中已追踪的文件提交到版本库，跳过暂存区
git rm <pattern> # 暂存区删除文件，工作区也作相应删除，需要提交到版本库。直接删除工作区中文件，不会影响暂存区
git rm -f|--force <pattern> # 修改后文件提交到暂存区，与版本库中文件有差异，需使用 -f 选项强制删除
git rm --cached <pattern> # 暂存区删除文件，但保留工作区文件，适用于未配置 .gitignore 的文件
git mv file_from file_to # 移动文件，可实现文件重命名。重命名时，等同于 mv file_from file_to; git rm file_from; git add file_to。git 没有显式追踪文件移动操作，通过执行的命令获知用户重命名行为
```

### .gitignore

.gitignore 配置文件，即忽略文件列表。可设置多个 .gitignore 文件，当前文件所在目录自底向上获取 .gitignore 文件，作为优先级顺序。编写 .gitignore 文件可使用标准 glob 模式(shell 中简易正则，* 零或多个任意字符，** 任意中间目录，其余同正则)；空行或 ‘#’ 开头将被忽略；’/‘ 开头相对于工程目录；’/‘ 结尾匹配目录；’!’ 开头置否值。

### commit 历史记录

无论本地仓库，还是克隆下来的远程仓库，都可以使用 git log 命令查看提交记录。

```bash
git log # 显示提交记录，反序排列，包含 SHA-1 哈希算法校验和(作为索引), 作者, 日期, message
git log -p|--patch # 显示内容差异
git log --stat # 显示每次更新的文件修改统计信息
git log --shortstat # 只显示 --stat 中最后的行数修改添加移除统计
git log --name-only # 仅在提交信息后显示已修改的文件清单
git log --name-status # 显示新增、修改、删除的文件清单
git log --abbrev-commit # 仅显示 SHA-1 的前几个字符，而非所有的 40 个字符
git log --relative-date # 使用较短的相对时间显示，如 '2 weeks ago'
git log --graph # 显示 ASCII 图形表示的分支合并历史
git log --pretty=oneline # 指定显示格式，--pretty 选项的可选值为 oneline, short, full, fuller, format。其中，oneline 以一行展示提交信息
git log --pretty=format:"%h - %an, %ar : %s" # 指定输出信息的模板字符串，用于提取分析报告。占位符包含 %H 提交对象/commit的完整哈希字串, %h 提交对象的简短哈希字串, %T 树对象/tree的完整哈希字串, %t, %P 父对象/parent的完整哈希字串, %p, %an 作者, %ae 作者的邮箱, %ad 作者修订日期, %ar 作者的相对修订日期, %cn 提交者, %ce, %cd, %cr, %s 说明
git log --oneline # --pretty=oneline --abbrev-commit 简写形式
git log -<n> # -n 指定只显示最后两次提交记录，如 -2。通常不需要使用。因为 git log 显示采用分页形式，用户只能看到第一页
git log --since|--after=2.weeks # --since 选项指定起始时间，可用相对时间或绝对时间，如 '2 years 1 day 3 minutes ago'
git log --until|--before="2008-01-15" # 同上，指定结束时间
git log --author=authorname # 过滤作者
git log --committer=committername # 过滤提交者
git log --grep=messagekey # 按 message 中关键字检索，与 --author 合用时为或匹配，添加 --all-match 选项强制匹配 grep 指定关键字
git log -S function_name # 指定字符串检索更改相应字符串的提交记录
git log --no-merges # 不显示 merge 记录
git log --filename # 作为最后一个选项，检索更改相应文件或目录的提交记录
```

### 撤销

撤销操作不能回滚，容易引起数据丢失。

```bash
git commit --amend # 合并上一次提交记录，适用于文件漏提交等轻微改动场景。若暂存区文件未作更新，commit 快照将保持不变，只更改 message 或者连 message 也未作更改。只保留当前的提交记录；撤销上一次提交记录，即不会存在于提交记录中
git reset HEAD <filename> # 暂存区文件重置为提交前状态。执行 git status 命令，工作位已更改文件的状态为 unstaged。git reset 命令不加选项，只更改暂存区。git reset --hard 命令将可能导致工作区的当前进度全部丢失，相当危险
git checkout -- <file> #  使用暂存区的文件替换工作区的文件，即工作区文件回滚
```

### 远程协作

```bash
git remote # 列出远程服务器的简称，默认为 origin
git remote -v # 列出远程服务器的简称和 url 列表，同一个项目可能指定了多个远程仓库。fetch, push 操作的服务器资源也可能不同，由远程仓库决定
remote add <shortname> <url> # 添加远程仓库
git fetch <shortname|url> # 抓取远程仓库资源，包含所有分支，并创建本地远程跟踪分支，却不会创建可修改、供开发的本地同名分支。只下载远程仓库资源，需手动 merge
git pull # 若本地分支跟踪远程分支，git pull 命令自动拉取远程仓库中指定的分支，并尝试 merge
git pull <remotename> <branchname> # 拉取远程分支资源，并尝试 merge。选项 remotename 指定远程仓库，branchname 指定分支
git push # 若本地分支跟踪远程分支，git push 将资源推送到远程仓库下的指定分支
git push <remotename> <branchname> # 推送资源到指定远程下的指定分支
git remote show <remotename> # 查看远程仓库 fetch|push 操作的 url，包含的分支，以及跟踪分支信息
git remote rename shortname_before shortname_after # 改写远程仓库的简称，同时改变分支名，如更新为 shortname_after/master
git remote remove|rm <remotename> # 本地移除某远程仓库，同时影响其下分支和配置，对远程仓库无影响
```

### 打标签

git 标签分为两种，轻量标签lightweight, 附注标签 annotated。轻量标签通过 git tag 不带选项创建，附注标签通过 git tag -a 创建。附注标签以对象形式存储在 git 数据库中，包含标签作者，电邮，日期，标签信息/message，并且可以使用 GPG 签名和验证。git tag 命令通常用来发布不再修改的版本(通过 git push 命令)，且发布以后，不能再作修改，需要在本地创建新分支拉取标签资源，作适当修改后再发布。

```bash
git tag # 列出所有标签，以字母顺序罗列
git tag -l|-list <pattern> # 只罗列匹配的标签，如 git tag -l "v1.8.5*"。pattern 中若含有通配符，-l 选项不可缺失
git tag <tagname> # 创建轻量标签，git show <tagname> 时只显示 commit 信息
git tag -a <tagname> -m "message" # 创建附注标签，git show <tagname> 时除显示 commit 信息外，还显示标签作者，电邮，日期，标签信息等
git show <tagname> # 查看标签信息及对应的提交信息
git tag -a <tagname> <checksum> # 提交后再打标签，选项 checksum 为 hash算法校验和，即快照的索引，可以只包含部分 hash 值
git push origin <tagname> # 将标签推送到远程共享服务器上
git push --tags # 将本地所有标签推送到服务器上
git checkout <tagname> # 检出标签，但不能真正检出标签，本地仓库会进入 "detached HEAD" 状态，提交将不从属于任何分支，只有 commit checksum 校验和能被 git 感知到
git checkout -b <branchnam> <tagname> # 在本地创建新分支，并拉取指定标签资源。修改后，重新创建标签并 push
```

### 疑问

1. 暂存区存在的意义？
2. 强制删除的意义？
3. commit 提交记录以何种形式存储，才能实现快速检索？关于 git database？
4. 本地版本库如何回滚？
5. git pull|push 只拉取或推送快照，还是所有历史记录？
6. git 远程服务器对版本管理的实现是否和本地相同，可否指定局域网中某台机器作为拉取资源的源头？
7. 打标签只针对 commit 操作？且标签只针对不会再修改的版本？git push origin [tagname] 推送到共享服务器和 git push origin [branchname] 的差异？
8. 切换分支时，为什么需要提交到版本库，而不是暂存库，这样就可以避免工作区的变更丢失了？