---
title: git常用命令
category: tools
date: 2020-04-02 20:47:28
tags: tools
---

<!-- more -->

#### 版本表示法

> 显示git分支

```bash
git rev-parse --symbolic --branches
```

> git显示里程碑

```bash
git rev-parse --symbolic --tags
```

> 显示定义的所有引用

```bash
git rev-parse --symbolic --glob=refs/*
```

> 显示HEAD对应的SHA的哈希值

```bash
git rev-parse HEAD
```

#### git使用资料

> git使用分支的场景

```text
http://stormzhang.com/git/2014/01/29/git-flow/
http://nvie.com/posts/a-successful-git-branching-model/
```

> git使用参考

```bash
https://github.com/geeeeeeeeek/git-recipes/wiki
https://www.atlassian.com/git/tutorials/setting-up-a-repository/git-init
```

#### 常用命令

> git删除远程分支

```bash
# 一般先查看远程分支是否存在
git branch -a | grep remote_branch_name
git push origin :remote_branch_name
git branch -a | grep remote_branch_name
```

> 删除本地分支

```bash
git branch -D branchname
```

> git创建本地分支并推送到远程新建一个分支

```bash
git branch local_branch_name
git checkout local_branch_name
git add files
git commit -m "working context for local branch name"
git push origin local_branch_name
```

> git切换到远程分支

```bash
git branch remotes/origin/xxx
git checkout -b xxx
```

> git设置不自动转换项目工程所在平台的换行符

```bash
git config --global core.autocrlf false
git config --global core.safecrlf true  
```

> git设置项目自动记住账户密码
```bash
git config --global credential.helper store
```

> git将更新的文件回退(本地)

```bash
git log # 获取当前最新分支的hash值
git checkout commit_id -- file      # 回退本地file文件的修改
```

> git-merge合并操作

```bash
# 背景：当前的工作需要引入线上最新代码,则合并线上分支代码到当前分支下
git checkout feature
git merge master

or

git merge master feature
```

> git-rebase合并操作

```bash
# 背景:会把整个 feature 分支移动到 master 分支的后面，有效地把所有 master 分支上新的提交并入过来。但是，rebase 为原分支上每一个提交创建一个新的提交，重写了项目历史，并且不会带来合并提交
# 将feature并入master分支
git checkout feature
git rebase master
```

> 交互式rebase操作

```bash
# 背景：被用于将 feature 分支并入 master 分支之前，清理混乱的历史
git checkout feature
git rebase -i master
```

> Rebase 的黄金法则

```bash
# git rebase 的黄金法则便是，绝不要在公共的分支上使用它(不要在develop分支上使用rebase)
# 你运行 git rebase 之前，一定要问问你自己[有没有别人正在这个分支上工作]
# 将master分支移动到feature分支后面
git checkout master
git rebase feature  # 确保master分支上没有人在进行工作

git push --force  ## 如果没有使用--force，git会停止push
```

> git-reset操作

```bash
# reset 将一个分支的末端指向另一个提交。这可以用来移除当前分支的一些提交
# 比如在一个热修复的分支提交了两次不必要的操作,想要撤回（仅限本地未提交到远程）,下次 Git 执行垃圾回收的时候，这两个提交会被删除
git checkout hotfix
git reset HEAD~2

--soft – 缓存区和工作目录都不会被改变
--mixed – 默认选项。缓存区和你指定的提交同步，但工作目录不受影响
--hard – 缓存区和工作目录都同步到你指定的提交
```

> git-checkout操作

```bash
# 传入分支名时，可以切换到那个分支
git checkout branchname

# 还可以传入提交的引用来 checkout 到任意的提交
git checkout HEAD~2
```

> git-revert操作(Revert 撤销一个提交的同时会创建一个新的提交)

```bash
# 找出倒数第二个提交，然后创建一个新的提交来撤销这些更改，然后把这个提交加入项目中
git checkout hotfix
git revert HEAD~2

# git revert 可以用在公共分支上，git reset 应该用在私有分支上
# git revert 当作撤销已经提交的更改，而 git reset HEAD 用来撤销没有提交的更改
```


#### git使用日志查看提交历史

> 显示limit个提交

```bash
git log -n <limit>
```

> 将每个提交压缩到一行。当你需要查看项目历史的上层情况时这会很有用

```bash
git log --oneline
```

> 哪些文件被更改了，以及每个文件相对的增删行数

```bash
git log --stat
```

> 显示每个提交全部的差异

```bash
git log -p
```

> 搜索特定作者的提交,<pattern> 可以是字符串或正则表达式

```bash
git log --author="<pattern>"
```

> 搜索提交信息匹配特定 <pattern> 的提交。<pattern> 可以是字符串或正则表达式

```bash
git log --grep="<pattern>"
```

> 显示在 some-feature 分支而不在 master 分支的所有提交的概览

```bash
git log --oneline master..some-feature
```

> git切换为远程分支

```text
git fetch origin
git checkout -t origin/remote_branch_name
```

> git 本地回滚

```text
git checkout filename
```

> git冲突

```text
# 内容冲突
1. 相同文件相同区块下的内容存在差异则会出现冲突说明
2. 相同文件相同的区块无差异，相同文件不同区块存在增删代码，则会合并

# 树冲突
1. git提交的文件信息重命名或删除

# 逻辑冲突
git项目代码引入的文件名称被修改过
```

