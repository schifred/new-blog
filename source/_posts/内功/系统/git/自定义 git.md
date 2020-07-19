---
title: 自定义 git
category:
  - 系统
  - git
tags:
  - 系统
  - git
keywords: 'git'
abbrlink: '5450410'
date: 2018-02-21 06:00:00
updated: 2018-02-21 06:00:00
---

本文整理自 [Pro Git](https://git-scm.com/book/zh/v2)。

### 自定义合并、比较工具
借助 Perforce 图形化合并工具（P4Merge）来合并文件和解决冲突。

1. 下载 P4Merge。
2. 创建一个名为 extMerge 的脚本包装 merge 命令，让它把参数转发给 p4merge 二进制文件。

```bash
# cat /usr/local/bin/extMerge
#!/bin/sh
/Applications/p4merge.app/Contents/MacOS/p4merge $*
```

3. 创建一个名为 extDiff 的脚本包装 diff 命令，让它把参数转发给 p4merge 二进制文件。git 传递以下参数给 diff：path old-file old-hex old-mode new-file new-hex new-mode。实际仅需要 old-file 和 new-file 参数。

```bash
# cat /usr/local/bin/extDiff
#!/bin/sh
[ $# -eq 7 ] && /usr/local/bin/extMerge "$2" "$5"
```

4. 使 extMerge, extDiff 脚本拥有可执行权限。

```bash
sudo chmod +x /usr/local/bin/extMerge
sudo chmod +x /usr/local/bin/extDiff
```

5. 修改配置文件，自定义合并和比较文件。

```bash
git config --global merge.tool extMerge # 通知 Git 该使用哪个合并工具
git config --global mergetool.extMerge.cmd \
  'extMerge \"$BASE\" \"$LOCAL\" \"$REMOTE\" \"$MERGED\"' # 规定命令运行的方式
git config --global mergetool.extMerge.trustExitCode false # 通知 Git 程序的返回值是否表示合并操作成功
git config --global diff.external extDiff # 通知 Git 该用什么命令做比较
```

6. 使用 git diff 命令，将自动唤起 P4Merge 图形化工具。
7. 使用 git mergetool 命令，将自动唤起 P4Merge 图形化工具。

### git 属性

通过 .gitattributes 文件或 .git/info/attributes 文件设置属性，可用于识别二进制文件、比较二进制文件。

### 识别二进制文件
只需配置 .gitattributes 文件。

```bash
*.pbxproj binary # 将 pbxproj 识别为二进制文件，git 将不会对其修改，如 CRLF 问题
```

### 比较 word 文件

比较二进制 Microsoft Word 文件步骤：

1. 下载 docx2txt。
2. 编写可执行脚本，即 docx2txt 文件。

```bash
#!/bin/bash
docx2txt.pl $1 -
```

3. 编辑配置。

```bash
git config diff.word.textconv docx2txt
```

4. 配置 .gitattributes 文件。

```bash
*.docx diff=word # 使用 “word” 过滤器，即借助 docx2txt 程序将 Word 文档转为可读文本文件
```

### 比较图像文件

在比较时对图像文件运用一个过滤器(如 exiftool)，提炼出 EXIF 信息——这是在大部分图像格式中都有记录的一种元数据。

```bash
echo '*.png diff=exif' >> .gitattributes
git config diff.exif.textconv exiftool
```

### 其他

```bash
test/ export-ignore # .gitattributes 文件中设置不必导出的文件或文件夹
LAST_COMMIT export-subst # 设置 LAST_COMMIT 文件用于接受提交记录，git commit 提交后，执行 git archive 命令，将提交记录导出到 LAST_COMMIT 文件中。LAST_COMMIT 文件内容如 'Last commit date: $Format:%cd by %aN$'
database.xml merge=ours # 设置合并策略，采用工作目录中的文件
```

### git 钩子

git 钩子放置在 .git/hooks 子目录中。git init 初始化项目时有示例，可以用 Ruby 或 Python 编程。

#### 客户端钩子

克隆远程仓库时不被复制。

* pre-commit 钩子在键入提交信息前运行。使用 git commit –no-verify 来绕过这个环节。
* prepare-commit-msg 钩子在启动提交信息编辑器之前，默认信息被创建之后运行。
* commit-msg 钩子在编辑器退出后运行。
* post-commit 钩子在整个提交过程完成后运行。
* applypatch-msg 钩子在应用补丁之前运行，git am 命令执行。
* pre-applypatch 钩子在应用补丁之后、产生提交之前运行，git am 命令执行。
* post-applypatch 钩子在提交产生之后运行，git am 命令执行。
* pre-rebase 钩子在变基之前运行。
* post-rewrite 钩子被那些会替换提交记录的命令调用，比如 git commit –amend 和 git rebase 命令。
* post-checkout 钩子在检出之后运行。
* post-merge 钩子在合并之后运行。
* pre-push 钩子在推送之前运行。
* re-auto-gc 钩子会在垃圾回收开始之前被调用。git 的一些日常操作在运行时，偶尔会调用 git gc –auto 进行垃圾回收。

#### 服务器端钩子

* pre-receive 钩子在推送过程运行，同时向多个分支推送的情形下也只触发一次。
* update 钩子在推送过程运行，同时向多个分支推送的情形下触发多次。
* post-receive 钩子在整个过程完结以后运行，可以用来更新其他系统服务或者通知用户。它的用途包括给某个邮件列表发信，通知持续集成（continous integration）的服务器，或者更新问题追踪系统（ticket-tracking system） —— 甚至可以通过分析提交信息来决定某个问题（ticket）是否应该被开启，修改或者关闭。

### 疑问
1. 文件检出和暂存时设置过滤器，参考 git 属性？
2. 使用钩子，参考使用强制策略的一个例子