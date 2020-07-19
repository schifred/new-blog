---
title: git 工具
category:
  - 系统
  - git
tags:
  - 系统
  - git
keywords: 'git'
abbrlink: d9d1d52e
date: 2018-02-21 05:00:00
updated: 2018-02-21 05:00:00
---

本文整理自 [Pro Git](https://git-scm.com/book/zh/v2)。

### 选取提交记录

根据提交对象的 SHA-1 值查看提交记录。引用日志只在本地生成，不能通过远程交互被协作者拷贝。

```bash
git log --abbrev-commit --pretty=oneline # 获取简短且唯一的 SHA-1 值
git show <sha-1>|<branch> # 显示提交记录
git rev-parse <branch> # 获取最近一次提交的 SHA-1 值
git reflog # 查看引用日志，引用日志记录了最近几个月的 HEAD 和分支引用所指向的历史，包含提交对象的简短 SHA-1 值、HEAD 记录和提交信息
git show HEAD@{5} # 查看第五次提交记录
git show <branch>@{yesterday} # 查看 branch 分支昨天指向了哪个提交
git show HEAD^ # ^ 引用指上一次提交，HEAD^ 即上一次提交，HEAD^^ 即之前第二次提交
git show <sha-1>^ # 特定提交的上一次提交
git show HEAD~ # ~ 指父引用，~2 指之前第二次提交
git show <sha-1>~num # 特定提交前第 num 次提交

git log branch1..branch2 # branch1..branch2 指在 branch2 分支中而不在 branch1 分支中的提交，branch2 留空，自动以 HEAD 填充，如 git log origin/master..HEAD 或 git log origin/master..。branch1..branch2可用于其他命令
git log refA refB ^refC # 指定被 refA, refB 包含，但不被 refC 包含的提交记录
git log refA refB --not refC # 同上
git log branch1...branch2 # 选出不被两个分支同时包含的提交记录
git log --left-right branch1...branch2 # 选出不被两个分支同时包含的提交记录，并显示提交对象属于 branch1 还是 branch2
```

### 交互式暂存

```bash
git add -i|--interactive # --interactive 选项用于开启交互式暂存命令行，执行后左侧显示已暂存的，右侧显示未暂存的；根据提示命令面板执行后续操作，如 1/status 显示状态, 2/update 提交到暂存区, 3/revert 从暂存区撤回, 4/add untracked 提交未追踪文件到暂存区, 5/patch 暂存补丁，可将部分修改提交到暂存区, 6/diff 查看已暂存内容的区别
git add -p|--patch # 暂存部分文件
git reset --patch # 部分重置文件
git checkout --patch # 部分检出文件
git stash save --patch # 部分暂存文件
```

### 储藏

通过 git stash 命令可以将工作区的改动储藏起来，却不是提交到暂存区；再使用 git stash apply 应用之前的储藏(可以在另一个分支上应用储藏；且当前分支做过修改后，仍可以应用储藏，必要时需解决冲突)。

```bash
git stash # 储藏工作区的改动，可用于切换分支前避免与远程代码的合并
git stash save # 同上
git stash --keep-index # 同上，--keep-index 选项将不储藏已使用 git add 暂存的内容
git stash -u|--include-untracked # 默认 git stash 不包含未追踪的文件，--include-untracked 选项将包含未追踪的文件
git stash --patch # 交互式地指引用户需要将那些改动储藏，那些改动保存在工作区
git stash --all # 移除修改，并储藏起来
git stash list # 查看所有储藏内容
git stash apply # 将储藏重新应用在工作分支上，默认为最近的储藏
git stash apply stash@{num} # 将第 num 次储藏应用在工作分支上，0 为最近的储藏
git stash apply --index # 应用储藏，同时应用暂存
git stash branch <branchname> # 创建一个新分支，检出储藏工作时所在的提交，重新在那应用工作，并扔掉储藏
git stash drop stash@{num} # 移除储藏
git stash pop stash@{num} # 应用并移除储藏
```

### 清理

```bash
git clean # 从工作目录中移除未被追踪的文件
git clean -f -d # 强制移除工作目录中所有未追踪的文件以及空的子目录
git clean -d -n # 移除项预览
git clean -n -d -x # git clean 默认不移除 .gitignore 中匹配的文件，-x 选项将移除该类文件
git clean -x -i|--interactive # 交互式执行 clean 命令
```

### 签署标签

参考：https://git-scm.com/book/zh/v2/Git-工具-签署工作

### 搜索

```bash
git grep string/regexp # 从提交历史或工作目录中查找匹配正则或字符串的内容，--and 选项用于设置更复杂字符串或正则匹配规则
git grep string/regexp <filepath> <version> # 指定文件或文件夹、版本号搜索
git grep -n string/regexp # 输出行号
git grep -p string/regexp # 查看匹配的行属于哪一个方法或函数
git grep -count string/regexp # 输出概要信息
git grep --break --heading string/regexp # 使输出更易读

# 日志搜索
git log -string --oneline # 从提交历史或 diff 内容中检索匹配字符串或正则的提交记录，-S 选项只查看新增和删除匹配内容的提交，-G 选项使用正则表达式检索
git log -L :string:filepath # 行日志搜索，搜索特定文件中 string 变更的提交记录
```

### 重写历史

git commit –amend 用于修改前一次提交。
git rebase -i 交互式修改多次提交记录，可拆分，可弃用，可合并，可重置等。须在本地环境中使用，提交到线上再调用变基，会使协作者变得不方便。
git filter-branch 使用脚本的方式改写大量提交记录。

```bash
git commit --amend # 修改前一次提交记录，修改暂存区再执行该命令时可改变提交内容，否则只是改变提交信息
git rebase -i HEAD~2^|HEAD~3 # -i 选项开启交互式变基命令，将唤起编辑器，反序修改前三次提交，最后一次修改在前。因为之前的提交已提交，显示 p, pick 默认提交，修改为 r, reword 重写提交信息；e, edit 使用  git commit --amend 重置提交，可用于拆分提交；s, squash 将提交信息合并到之前执行 pick 命令的提交对象中，且将多次提交信息合并；f, fixup 同 squash，但丢弃提交信息，x, exec，唤起 shell 提交。弃用某次提交记录，可直接从 git rebase -i HEAD~2^|HEAD~3 显示的列表中删除
git filter-branch --tree-filter 'rm -f passwords.txt' HEAD # --tree-filter 选项在检出项目的每一个提交后运行指定的命令然后重新提交结果。--all 选项用于改写所有分支
git filter-branch --subdirectory-filter trunk HEAD # 让 trunk 子目录作为每一个提交的新的项目根目录
git filter-branch --commit-filter '
  if [ "$GIT_AUTHOR_EMAIL" = "schacon@localhost" ];
  then
    GIT_AUTHOR_NAME="Scott Chacon";
    GIT_AUTHOR_EMAIL="schacon@example.com";
    git commit-tree "$@";
  else
    git commit-tree "$@";
  fi' HEAD # 修改多个项目的提交邮箱，且重写每个提交的校验和，不只匹配邮箱地址的提交
```

### 重置揭密

```bash
git cat-file -p HEAD # 查看版本库当前提交信息
git ls-tree -r HEAD # 查看版本库中提交文件的 checksum 及其文件名
git ls-files -s # 查看暂存区中文件的 checksum 及其文件名

git reset <checksum> # 将版本库和暂存区内容替换为 checksum 指向的提交对象
git reset --soft <checksum> # 将版本库内容替换为 checksum 指向的提交对象
git reset --mixed <checksum> # 将版本库和暂存区内容替换为 checksum 指向的提交对象
git reset --hard <checksum> # 将版本库、暂存区和工作区内容替换为 checksum 指向的提交对象
git reset <filepath> # 从 HEAD 中获取 filepath 文件或目录，复制到暂存区
git reset <checksum> <filepath> # 从 checksum 中获取 filepath 文件或目录，复制到暂存区
```

### 高级合并

合并前可以通过保存到临时分支或通过执行 git stash 命令储藏起来，避免合并对工作区的影响，使工作区内容丢失。

子树合并是项目某目录下包含一个子工程，同时 git 分支也包含一个从属于子工程的分支，从而引起子分支和子工程目录的合并操作。

```bash
git merge --abort # 退出合并，工作区内容恢复到合并前
git reset --hard HEAD # 退出合并，将版本库、暂存区和工作区内容重置到合并前
git merge -Xignore-space-change # 忽略任意数量的已有空白的修改
git merge -Xignore-space-change # 忽略所有空白修改
git show :1:<conflictFileName> > commonFileName # 拷贝并导出冲突的共同祖先文件
git show :2:<conflictFileName> > oursFileName # 拷贝并导出冲突的工作目录文件
git show :3:<conflictFileName> > theirsFileName # 拷贝并导出冲突的待合并文件
git ls-files -u # 查看冲突文件的完整 SHA-1 值
git merge-file -p \
  oursFileName commonFileName theirsFileName > fileName # oursFileName, commonFileName, theirsFileName 拷贝的冲突文件手工解决冲突后，合并冲突文件
git diff --ours # 查看合并结果和当前工作分支的差别，-b 选项用于移除空格比较，--base 选项查看比较文件两边是如何改动的
git diff --theirs # 查看合并结果和待合并分支的差别
git clean -f # 清理为合并拷贝的文件，如 commonFileName 等
git checkout --conflict=diff3 # 查看工作分支、待合并分支、共同祖先分支的冲突文件
git checkout --conflict=merge # 默认，查看工作分支、待合并分支的冲突文件
git checkout --ours # 合并时使用当前工作分支文件内容
git checkout --theirs # 合并时使用待合并分支文件内容
git log --oneline --left-right branch1...branch2 # 查看合并文件的提交记录来源
git log --oneline --left-right --merge # 查看合并过程中，有冲突文件的提交记录来源
git diff # 查看冲突文件或冲突结果
git log --cc -p -1 # 查看冲突结果
git reset --hard HEAD~ # 撤销合并，将版本库撤回到合并前状态
git revert -m 1 basebranch # 撤销合并，将提交对象撤回到 basebranch 分支内容。-m 1 标记指出 “mainline” 需要被保留下来的父结点。此时将无法合并待合并分支的内容
git revert <checksum> # 撤销 git revert -m 1 basebranch 命令
git merge -Xours <branch> # 选用当前工作分支内容进行合并
git merge -Theirs <branch> # 选用待合并分支内容进行合并
git merge-file --ours <filepath> # 选用当前工作分支内容进行合并
git merge-file --theirs <filepath> # 选用待合并分支内容进行合并
git merge -s ours <branch> # 假合并，直接将当前工作分支内容作为合并结果

# 子树合并
git read-tree --prefix=dirname/ -u sub_project_branch # 将子工程分支的内容拷贝到 dirname 目录中
git merge --squash -s recursive -Xsubtree=dirname sub_project_branch # 将 sub_project_branch 分支内容合并到 dirname 目录中
git diff-tree -p sub_project_branch # 查看差异
git diff-tree -p <remote>/<branch> # 和远程引用相比较
```

### rerere 命令

通过 git config –global rerere.enabled true 命令开启 rerere 功能。rerere 记录冲突解决方案，自动解决冲突。

```bash
git config --global rerere.enabled true # 开启 rerere 功能
git rerere status # 查看合并前的状态
git rerere diff # 查看解决前后的文件内容差异。冲突解决后，可查看 git rerere 将要记录的内容
git ls-files -u # 查看各冲突文件 checksum
```

### 使用 git 调试

使用 git blame 命令可以查看文件内容的历次更改，适用在确知某方法会引起 bug 的场景，可查看该方法的改动细节。

使用 git bisect 命令可以定位哪次提交引起了异常。

```bash
git blame -L startLine,endLine filepath # 查看 filepath 文件自 startLine 起始、到 endLine 行结束的变更记录。-L 选项用于限制行号。-C 选项会分析拷贝之后再重命名文件的原始出处

git bisect start # 启动二分查找
git bisect bad # 告知 git 当前提交有问题
git bisect good <checksum> # 告知 git 将 checksum 指向的提交对象标记为好的。由 git 告知用户在正常提交和错误提交之间有多少次提交，二次查找中间那次提交，通过 git bisect bad 或 git bisect good 标记好坏。git 会自动定位到提交中点，再由用户判断提交结果的好坏
git bisect reset # 重置 HEAD 指针

# 通过执行脚本定位 bug 来源。checksum1 是坏的提交，checksum2 是好的提交
git bisect start <checksum1>|HEAD <checksum2>
git bisect run filepath
```

### 子模块

子模块虽然在工程目录中创建了子目录，但是若不在该子目录中，git 并不会跟踪子模块的文件内容，而是将它看作该仓库中的一个特殊提交。子模块提交时标记为 160000 模式，意味着将一次提交记作一项目录记录，而非将它记录成一个子目录或者一个文件。

```bash
git submodule add <remoteurl> <dirname> # 克隆远程仓库，以远程仓库名创建文件夹，或者自定义文件夹。同时会添加 .gitmodules 文件，用于保存本地目录和远程仓库地址
git config submodule.<dirname>.url <url> # 重设远程仓库地址
git diff --cached --submodule # 查看子模块的文件差异
git submodule init # 子模块初始化
git submodule update # 获取远程仓库数据，并检出父项目中属于子模块的提交
git clone --recursive mainurl # mainurl 为父项目的 url，同时更新子模块
git submodule update --remote subproject # 在父项目工作目录中进行更新，避免手动进入子模块抓取和合并
git submodule update --remote --merge # 合并
git submodule update --remote --rebase # 变基
git submodule update --init # 初始化拉取远程内容
git log -p --submodule # 查看子模块的提交日志
git push --recurse-submodules=check # 推送时校验子项目有否提交
git push --recurse-submodules=on-demand # 推送子模块，或者子项目中执行 git push 命令
git submodule foreach 'git stash' # 在所有子模块中运行 git stash 命令
git rm -r <dirname> # 从暂存区移除子项目目录
rm -Rf <dirname> # 移除子项目目录

# 子模块命令别名
git config alias.sdiff '!'"git diff && git submodule foreach 'git diff'"
git config alias.spush 'push --recurse-submodules=on-demand'
git config alias.supdate 'submodule update --remote --merge'
```

### 打包

```bash
git bundle create bundlefilename HEAD <branch> # 将 branch 分支所有提交数据打包到 bundlefilename 文件中
git bundle create bundlefilename <branch> ^<checksum> # 设定提交区间，并打包文件。提交区间可以用 ... 等符号操作
git clone bundlefilename dirname # 将打包文件解压到 dirname 目录中
git bundle verify bundlefilename # 校验打包文件是不是合法的 Git 包
git bundle list-heads bundlefilename # 查看打包文件包含哪些提交对象和分支
git fetch bundlefilename branch:local-branch # 将打包文件中的分支导入本地工程中
```

### 替换

大型项目可分成一个短历史提交记录和一个长历史提交记录，短历史供新的开发者使用，后者给喜欢数据挖掘的人使用。制作短历史提交记录通过 git commit-tree 命令合并提交记录，再通过 git rebase 命令将必要的提交变基到刚创建的合并提交记录上，就可以制作短历史提交记录。执行 git replace 命令，可以查看长历史提交记录。

```bash
git branch <branch> <checksum> # 以提交对象 checksum 创建分支 branch
git push remote localbranch:remotebranch # 将本地 localbranch 分支推送到远程仓库 remote 下的 remotebranch 分支
git commit-tree <checksum>^{tree} # 将 checksum 提交对象及其前的提交合并为新的提交对象，返回新提交对象的 SHA-1 
git rebase --onto checksum1 checksum2 # 将 checksum2 后的提交变基到 checksum1 上，checksum1 后产生新的提交记录
git replace checksum1 checksum2 # 将 checksum1 及其前的提交记录替换为 checksum2 及其前的提交记录
git cat-file -p <checksum> # 查看 checksum 提交的提交树 SHA-1 值及其父提交
```

### 凭证存储

使用 SSH 连接远端，且设置了一个没有口令的密钥，这样就可以在不用输入用户名和密码的情况下安全连接远端。使用 HTTP 存储，每一个连接都需要输入用户名和密码。git 凭证机制默认不存储用户名和密码，’cache’ 模式内存中暂时存储用户名和密码，’store’ 模式将用户名和密码存储在磁盘中，mac 下 ‘osxkeychain’ 以加密方式将凭证缓存到系统用户的钥匙串，windows 下可借助 ‘winstore’ 工具以加密方式将凭证存储在磁盘中。

```bash
git config --global credential.helper cache|store # 设置凭证存储模式
git config --global credential.helper store --file <path> # 设置存储目录
git config --global credential.helper cache --timeout <seconds> # 设置过期时间
git credential fill # 交互式设置凭证，通过 protocol/协议, host/主机名 设置
```

### 疑问

1. 获取某个特定的提交对象后，可否对该提交对象进行操作？
2. 关于签署标签？
3. 关于 git filter-branch？
4. 关于子模块？