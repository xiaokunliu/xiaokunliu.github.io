---
title: SVN命令小结
category: tools
date: 2020-04-02 20:47:28
tags: tools
---

<!-- more -->


> 将文件checkout到本地目录

```
svn checkout path（path是服务器上的目录）
简写：svn co
```

> 往版本库中添加新的文件

```
svn add file
```

> 将改动的文件提交到版本库

```
svn commit -m “LogMessage” [-N] [--no-unlock] PATH(如果选择了保持锁，就使用–no-unlock开关)
简写：svn ci
```

> 加锁/解锁

```
svn lock -m “LockMessage” [--force] PATH
svn unlock PATH
```

> 更新到某个版本

```
svn update -r m path
简写：svn up
```

> 查看文件或者目录状态

```
1）svn status path（目录下的文件和子目录的状态，正常状态不显示）
2）svn status -v path(显示文件和子目录状态)
简写：svn st
```

> 删除文件

```
svn delete path -m “delete test fle”
简写：svn (del, remove, rm)
```

> 查看日志

```
svn log path
```

> 查看文件详细信息

```
svn info path
```

> 比较差异

```
svn diff path(将修改的文件与基础版本比较)
svn diff -r m:n path(对版本m和版本n比较差异)
简写：svn di
```

> 将两个版本之间的差异合并到当前文件

```
svn merge -r m:n path
```

> SVN 帮助

```
svn help
svn help ci
```

#### SVN不常用命令
> 版本库下的文件和目录列表

```
svn list path    显示path目录下的所有属于版本库的文件和目录
简写：svn ls
```

> 创建纳入版本控制下的新目录

```
svn mkdir: 创建纳入版本控制下的新目录

用法: 
1、mkdir PATH...
每一个以工作副本 PATH 指定的目录，都会创建在本地端，并且加入新增调度，以待下一次的提交。
2、mkdir URL... 创建版本控制的目录。 
每个以URL指定的目录，都会透过立即提交于仓库中创建。在这两个情况下，所有的中间目录都必须事先存在。
```

> 恢复本地修改

```
svn revert: 恢复原始未改变的工作副本文件 (恢复大部份的本地修改)。
用法: revert PATH... 注意: 本子命令不会存取网络，并且会解除冲突的状况。但是它不会恢复被删除的目录
```

> 代码库URL变更

```
svn switch (sw): 更新工作副本至不同的URL
用法: 
1、switch URL [PATH]        
更新你的工作副本，映射到一个新的URL，其行为跟“svn update”很像，也会将      服务器上文件与本地文件合并。这是将工作副本对应到同一仓库中某个分支或者标记的方法。 
2、switch --relocate FROM TO [PATH...]   
改写工作副本的URL元数据，以反映单纯的URL上的改变。当仓库的根URL变动     (比如方案名或是主机名称变动)，但是工作副本仍旧对映到同一仓库的同一目录时使用     这个命令更新工作副本与仓库的对应关系。
```

> 解决冲突

```
svn resolved: 移除工作副本的目录或文件的“冲突”状态。
用法: resolved PATH... 注意: 本子命令不会依语法来解决冲突或是移除冲突标记；它只是移除冲突的相关文件，然后让 PATH 可以再次提交。
```

> 输出指定文件或URL的内容。

```
svn cat 目标[@版本]...如果指定了版本，将从指定的版本开始查找。 
svn cat -r PREV filename > filename (PREV 是上一版本,也可以写具体版本号,这样输出结果是可以提交的）
```

#### SVN其它命令

> svn cleanup

当Subversion修改你的工作副本时（或者任何在.svn中的信息），它尝试尽可能做到安全。在改变一个工作副本前，Subversion把它的意 图写到一个日志文件中。接下来它执行日志文件中的命令来应用要求的修改。最后，Subversion删除日志文件。从架构上来说，这与一个日志文件系统 （journaled filesystem）类似。如果一个 Subversion操作被打断（例如，进程被杀掉了，或机器当掉了）了，日志文件仍在硬盘上。重新执行日志文件，Subversion可以完成先前开始 的操作，这样你的工作副本能回到一个可靠的状态。 
以下是svn cleanup所做的：它搜索你的工作副本并执行所有遗留的日志，在这过程中删除锁。如果Subversion曾告诉你你的工作副本的一部分被“锁定”了，那么你应该执行这个命令。另外， svn status会在锁定的项前显示L。 

```
$ svn status
L    somedir
M   somedir/foo.c 

$ svn cleanup
$ svn status
M      somedir/foo.c
```
> svn import

```
# 使用svn import是把未版本化的文件树复制到资料库的快速办法，它需要创建一个临时目录。 
$ svnadmin create /usr/local/svn/newrepos
$ svn import mytree file:///usr/local/svn/newrepos/some/project
Adding         mytree/foo.c
Adding         mytree/bar.c
Adding         mytree/subdir
Adding         mytree/subdir/quux.h

Committed revision 1.

# 上面的例子把在some/project目录下mytree目录的内容复制到资料库中。 

$ svn list file:///usr/local/svn/newrepos/some/project
bar.c
foo.c
subdir/
```

注意在导入完成后，原来的树没有被转化成一个工作副本。为了开始工作，你仍然需要svn checkout这个树的一个新的工作副本