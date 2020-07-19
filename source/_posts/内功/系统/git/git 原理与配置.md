---
title: git 原理与配置
category:
  - 系统
  - git
tags:
  - 系统
  - git
keywords: 'git'
abbrlink: 81df1cc2
date: 2018-02-21 00:00:00
updated: 2018-02-21 00:00:00
---

本文整理自 [Pro Git](https://git-scm.com/book/zh/v2)。

### 原理

git 版本管理通过 SHA-1 哈希算法为工作区的文件生成快照实现，存放于本地暂存区或版本库中（.git 目录下保存所有快照信息）。hash 算法基于文件内容和目录结构，git 保存的信息都通过 hash 值(SHA-1 哈希算法校验和/SHA-1 checksum)进行索引。修改工作区/working tree 的文件后(modified)，通过 git add 命令提交到暂存区/staging range(staged)，再通过 git commit 命令提交到版本库/git directory(commited)；若文件未变更，将沿用原有的快照。因此本地多版本管理不需要远程交互。

当文件提交到暂存区时(stage)，git 会为每个文件计算校验和/checksum，在 git 仓库中保存 blob 对象以引用文件快照，校验和将保存到暂存区中。当提交到版本库时(commit)，git 会计算目录的校验和并构建树对象(blod 文件对象将以索引形式存储在树对象中)，随后创建提交对象，该提交对象包含作者名、电邮、message、指向上述树对象的指针、指向之前提交版本的指针。git 仓库可理解为存储系列提交对象的单链表结构(以哈希算法的校验和值作为索引)。在多分支开发的特殊情况下，提交对象会转变为树形结构(单链表可视为只有主干的树)，即多个提交对象的 parent 指针可以指向同一个提交对象父节点，提交历史产生分叉。在提交对象模型的基础上，git 分支就是指向某个提交对象的移动指针(通过校验和指向提交对象，文件形式存储)，其提交历史来自于提交对象模型，形成单链表结构。多分支管理、及单分支的回滚和提交动作均衍生于树结构的提交对象模型，分支仅承担着指向改变的任务；创建新分支即是创建一个指向当前提交对象的指针；提交时，创建新的提交对象，并改变当前分支的指向。在 git 中，HEAD 指针用于指向当前工作的分支，通过改变 HEAD 指针的指向即为切换分支操作。需要说明的是，当切换回较旧的分支时，不只改变了 HEAD 指针的指向，同时也使工作目录变更为该分支指向的提交对象，即资源快照。多分支开发时，新的提交对象将作为提交对象模型的叶子节点，这一过程也可以通过 HEAD 指针感知当前的提交对象属于哪个分支。因此在 git 中，分支不是多次提交记录的集合，而是在抽象所有提交记录为单一的模型后、衍伸而来的理念。遵照这样的设计，master 分支和其他分支拥有相同的特征，只是 master 分支会在 git 仓库初始化时被创建。

在 git 中，HEAD 指针指向版本库，INDEX 指针指向暂存区，版本库和暂存区都存储在 .git 文件夹中。因此，git add 命令将工作区内容复制到暂存区/INDEX。git commit 将暂存区内容复制到版本库/HEAD。git status 比较工作区、暂存区、版本库内容是否相同，若工作区和暂存区内容不相同，提示 not staged，需要通过 git add 暂存；若暂存区和版本库内容不同，提示 not committed，需要通过 git commit 提交。执行 git checkout, git clone 命令时，先将 HEAD 指针指向切换的分支，再将版本库内容复制到暂存区，再将暂存区内容复制到工作区。

git 提供了 git reset 命令，执行该命令，不仅改变 HEAD 指针的指向，同时将工作分支指向 checksum 提交对象上。如果 HEAD 指向 master 分支，运行 git reset 9e5e64a 将会使 master 指向 9e5e64a，将撤销 9e5e64a 之后的提交，也可以用于重置 9e5e64a 之后的提交；git reset 命令默认将暂存区内容也替换为 9e5e64a 指向的提交对象，这和显式执行 git reset –mixed 命令相同。git reset –soft 命令只将版本库中内容替换为指定的提交对象。git reset –hard 命令可同时将版本库、暂存区和工作区内容替换为指定的提交对象。使用 git reset 或 git reset 命令将目标从提交对象转向 filepath 路径指定的文件或目录，操作是从指定的提交对象中获取内容，复制到暂存区。

在 git checkout 命令执行过程中，会比较工作区和暂存区的差别，并尝试合并，这是 git reset 命令所没有的操作。当执行 git checkout 命令时，该命令不仅会影响暂存区，同时会尝试合并工作区，这也是 git reset 命令所没有的操作。

合并/merge 分支时，会向上遍历文档对象模型，寻找共同的祖先节点作为合并基础。若待合并分支的提交/commit 动作在当前工作分支(work-in-progress branch)之后产生，当前分支将采用快进/fast-forward 方式修改，即将当前分支指向待合并分支的提交对象，指针右移。合并分叉分支时，即待合并的两个分支为同时开发，将获取这两个分支的提交对象，以及他们共同祖先节点的提交对象作为合并基础，作三方合并(three-way merge)，最终将构建出新的提交对象(该提交对象有两个父节点，使文档对象模型由树结构转变为合流结构)。若两个分叉分支同时对同一块区域做更改，git 不会自动合并，将产生一个合并冲突(merge conflict)，并阻止 git 的后续执行流程，如构建新的提交对象等，git 还会为冲突文件注入标准的冲突解决标记(standard conflict-resolution markers)。通过 git status 命令可查看冲突文件。冲突解决标记中，以 <<<<<<< HEAD 起始部分为当前工作分支内容，以 >>>>>>> branchname 结束部分为待合并分支内容，中间以 =======。解决冲突后，需要依序提交到暂存区和版本库中，将产生一条合并记录，可能包含冲突解决记录。合并分支可使用 git mergetool 命令，启用图形化合并工具，参看 Git Branching。

git merge 合并发生冲突时，暂存区/stages 下缓存着共同祖先版本/stage-0，当前工作版本/stage-1，待合并的版本/stage-2。通过 git show :1|2|3: > fileName 可以以拷贝形式导出冲突的共同祖先文件等。其中，:1|2|3: 为各冲突文件 bolb 对象相关 SHA-1 值的简写形式，:1: 为祖先冲突文件，:2: 为工作分支文件，:3: 为待合并文件。导出三份拷贝文件后，手工修复冲突，可使用 git merge-file 命令合并冲突文件，使用 git diff 命令可比较冲突结果与待比较文件的差异，git clean 命令用于清理为合并拷贝出来的文件。

另外，在合并发生冲突时，使用 git checkout –conflict=diff3 命令可以查看不止于当前工作分支的冲突文件和待合并的冲突文件，还包含共同祖先的冲突文件。默认只查看当前工作分支的冲突文件和待合并的冲突文件，即 git checkout –conflict=merge 命令。使用 git config –global merge.conflictstyle diff3 命令，将全局的合并方案改成 diff3。git checkout –ours 合并时使用当前工作分支文件内容；git checkout –theirs 合并时使用待合并分支文件内容。git 在合并过程中，会将合并成功的文件提交到暂存区，因此 git diff 命令可以查看冲突文件；解决冲突后，仍可使用 git diff 或 git log –cc -p -1 命令查看解决冲突后的文件。

若想撤销文件合并，可使用 git reset –hard HEAD~ 命令将合并取消，提交记录返回到合并前；或者使用 git revert -m 1 basebranch 命令撤销合并，将提交对象撤回到 basebranch 分支内容。需要注意的是，git revert -m 1 basebranch 命令将无法合并待合并分支起始的提交内容，针对这一问题，可使用 git revert 撤销还原。
参看 Git Tools。

远程引用(remote references)是对远程仓库中分支、标签等的引用或指针。更恰当的说，引用内容即为远程分支中的提交对象模型，单链表形式。远程跟踪分支(remote-tracking branches)将远程分支的引用保存在本地，不受用户影响，git fetch|pull|push 操作时更新引用，使本地 shortname/branchname 指向远程服务器的同名分支。远程引用采用 shortname/branchname 形式命名分支。当 git clone 命令执行时，将在本地创建 origin/master 分支和 master 分支，两者均指向相同的提交对象模型。当用户在 master 分支开发并提交时，另一用户将自己的代码 push 到远程 master 分支上，本地 origin/master 的指向将不作改变；假使在这时候执行 git fetch 命令，将抓取远程仓库新添加的提交对象，本地 origin/master 也将右移、指向另一用户创建的提交对象上，提交对象模型转化为树形结构。

变基/rebase 是 git 中除了 merge 以外整合两个分支的另一种方式。当提交历史呈分叉状态，可以将其中一个分叉的提交记录基于合并基础、提取为更新补丁，再将其合并到另一个分叉中，这个过程就是变基。变基命令在提取为补丁的分支上执行行，因此需要切换到该分支如 patch 分支，然后执行 git rebase basebaranch 命令，提取补丁并以该补丁产生新的提交对象(patch 分支指向该提交对象)，此时分叉的提交历史将转变为单链表形式，而 basebaranch 分支的指向仍保持不变。切换到 basebaranch 分支，再执行 git merge patch 命令，可以将 basebaranch 分支的指针右移，指向 rebase 命令新创建的提交对象。经过上述步骤后，整个开叉分支的整合操作也就完成了，patch 分支的提交对象将被抹去，但 patch 分支仍然存在，需要手动删除。变基同合并比较，最大的优点即是使提交历史呈线性状态，而不是分叉、合流结构。在开源项目中贡献代码通常采用变基命令。使用 git rebase basebranch patch 命令执行变基操作时，可以不用将工作分支切换到 patch 上，即提取 patch 分支的修改补丁，基于修改补丁在 basebranch 分支上创建新的提交对象。执行 git rebase –onto basebranch patch1 patch2 命令，获取 patch2 分支不同于 patch1 分支的修改补丁，基于修改补丁再在 basebranch 分支上创建新的提交对象。

执行变基操作有一条原则，即不能在你的本地仓库执行变基操作。变基操作能改变本地仓库的提交历史，进而影响远程仓库的提交记录，但是对于协作者而言，在分支开叉时拉取编程变更，又在变基后拉取远程变更，他本地远程引用的提交记录中即会保留变基前的开叉，又会有变基后新增的提交对象。git 变基的实现原理是，在每次提交时，git 不只计算本次提交的校验和，还会计算本次修改内容的校验和 patch-id。通过 patch-id，git 能分辨出新增的本地修改。在本地分支上，使用 git rebase remote/branch 命令将本地修改变基到远程引用，同时本地的提交历史将转变为单链表结构，这能解决前述变基前后的操作同时提交到远程仓库的问题。

除了合并和变基以外，git 支持拣选/cherry-pick 操作，其意义以校验和获取某次提交补丁，再应用到当前工作分支上。因为操作时间的不同，通过拣选在当前分支创建的提交会重新计算校验和。具体命令如 git cherry-pick e43a6fd3e94888d76779ad79fb568ed180e5fcdf。其中，e43a6fd3e94888d76779ad79fb568ed180e5fcdf 为之前提交对象的校验和/checksum，作为提交对象的索引。

参考：
[分支简介](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%AE%80%E4%BB%8B)
[分支的新建与合并](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%9A%84%E6%96%B0%E5%BB%BA%E4%B8%8E%E5%90%88%E5%B9%B6)
[变基](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%8F%98%E5%9F%BA)
[重置揭密](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E9%87%8D%E7%BD%AE%E6%8F%AD%E5%AF%86)

### 配置

git 配置分为三类，系统级，用户级，和项目级；优先级从右到左。git config 命令中，–system 选项指定系统级，–global 选项指定用户级，两者均没有为项目级。

```bash
git config --list # 查看配置
git config user.name # 查看用户配置
git config --global user.name <name> # 指定用户
git config --global user.email <email> # 指定邮箱
git config --global core.editor emacs # 指定编辑器，默认为系统自带编辑器，可选 vim, emacs, notepad++
git config --global commit.template <filepath> # 以 filepath 文件作为提交信息模板，打开编辑器时作为前缀
git config --global core.pager '' # 设置 git log|diff 命令的分页器，可选值 more, less。'' 空字符串为完整显示
git config --global user.signingkey <gpg-key-id> # 设置 GPG 签署密钥，影响 git tag -s <tag-name> 命令
git config --global core.excludesfile <filepath> # 设置忽略的文件
git config --global help.autocorrect 1 # 输入命令有误时，0.1s 后自动执行模糊匹配的命令
git config --global credential.helper cache # 将 https 推送需要的用户密码缓存在内存中，时效为几分钟
git config --global color.ui false # 禁用有彩色输出
git config --global color.[diff|branch|interactive|status].meta "blue black bold" # 设置命令的颜色
git config --global core.autocrlf true # Windows 使用回车（CR）和换行（LF）两个字符来结束一行，而 Mac 和 Linux 只使用换行（LF）一个字符。core.autocrlf 设置为 true，提交时自动将回车和换行转换成换行，检出时将换行转换成回车和换行。core.autocrlf 设置为 input，提交时自动把回车和换行转换成换行，检出时不转换
git config --global core.whitespace \
  trailing-space,space-before-tab,indent-with-non-tab # 空格检测。默认开启项 blank-at-eol，查找行尾的空格；blank-at-eof，盯住文件底部的空行；space-before-tab，警惕行头 tab 前面的空格。默认关闭项 indent-with-non-tab，揪出以空格而非 tab 开头的行（你可以用 tabwidth 选项控制它）；tab-in-indent，监视在行头表示缩进的 tab；cr-at-eol，告诉 Git 忽略行尾的回车。git diff 时将应用空格检测；git apply --whitespace=warn <patch> 应用补丁时检测空格；git apply --whitespace=fix <patch> 自动修复；git rebase --whitespace=fix 变基时自动修复

# 别名
git config --global alias.co checkout # git co 将等价于 git checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.unstage 'reset HEAD --' # git unstage 将等价于 git reset Head -- 
git config --global alias.last 'log -1 HEAD' # git last 查看最后一次提交信息
git config --global alias.visual '!npm' # git visual 将调用外部命令 npm

git config --global pull.rebase true # 更改 pull.rebase 的默认配置
git config --global merge.conflictstyle diff3 # 以 diff3 方式查看冲突文件，包含当前工作分支、待合并分支、共同祖先分支的冲突文件内容

git config --system receive.fsckObjects true # 推送生效前检验每个对象的有效性以及 SHA-1 检验和是否保持一致
git config --system receive.denyNonFastForwards true # 禁用强制更新 git push -f
git config --system receive.denyDeletes true # 避免删除远程分支后，再推送本地分支
```

### 帮助

```bash
git help config # 打印 git config 命令的完整手册
man git-confgi # 同上
git-config -h|--help # 简要帮助信息
```

### 疑问

1. 保存快照不吃磁盘空间吗？git 命令在运行时比较文件差异，如 add, commit？那执行效率如何提升？
2. 数据库怎样存储历史版本数据？
3. 游戏在用户端以补丁形式下载并更新，若代码使用 git 管理，怎样做到根据文件差异只做局部封信，而不是全量下载并更新？同样的，git pull 等命令怎样做到效率地只修改局部资源？
4. 在多分支开发，又相继合并到 master 分支的情形下，提交对象链表会形成怎样的构造？根据提交时间形成单链表形式？
5. git reset 命令执行后，如何改变提交历史，新的提交对之前提交历史的影响？如重置第三次提交，新的提交会在第一次提交之后创建提交对象，还是在之前创建提交对象？
6. git checkout 命令可以针对某次提交对象，即可以执行 git checkout 命令？