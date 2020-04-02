---
title: git发版使用
category: tools
date: 2020-04-02 20:47:28
tags: tools
---

<!-- more -->

##### 使用git打包发布版本

> 拉取git项目的代码(仅执行一次)

```bash
# 拉取项目
git clone projects.git  # url
cd projects
git checkout master && git pull
# 创建基于master的git flow的实例化操作
git flow init -d
```

> 基于develop分支进行发版

```bash
# 拉取开发分支
git checkout develop && git pull

# 查看当前项目的版本号(自定义于所属项目中),如
projects/
    projects/
        __init__.py
            __version__ = "1.2.0"
            __release_date__ = "2018-02-02"
# 版本号规定为三位数字x.y.z
    # x表示为项目大的改动,比如重构,换新的技术等,在原有的基础递增
    # y表示在一个周期时间段内的版本发布,如每隔2周发布新的功能,在原有的基础递增
    # z可以表示紧急修复抑或是bug修复抑或功能上的改进等等,在原有的基础递增
    
# 基于当前的一个版本号,根据项目实际情况进行发版,比如现在是新功能发布,则版本号为1.3.0

# 基于当前的release版本进行打包
git flow release start 1.3.0  # 此版本仅用于bug或者是功能上的修复,不做新功能的发布

# 修改版本号以及changelog
# 将上述修正为
__version__ = "1.3.0"
__release_date__ = "2018-02-06"   # 以当前时间为准
 
# 添加修改的日志记录在changelog,添加的内容视团队以及项目规范而定
...
```

> git log查看上次打包以来的修改日志，对比CHANGELOG.txt中的是否有遗漏

```bash
git log <tag>..HEAD                  # 获取某Tag以来的日志，如：git log 1.0.8..HEAD
git log --since=<last-release-date>  # 获取某个时间以来的日志，如：git log --since=2015-01-14
```

> 将新建的release分支publish到服务器

```bash
git add files   # 本次发版修改的文件内容
git commit -m "commit message"
git flow release publish 1.3.0
```

> 通知团队其他可以签出该分支

```bash
git flow release track 1.3.0
```

> 进行bug修复和提交

```bash
git pull --rebase              # 定期拉取别人提交的代码
                                       # 如果pull出现冲突，则解决冲突后git add xxx
                                       # 然后git rebase --continue继续操作。

git status                     # 查看有哪些文件修改
git diff                       # 查看具体修改内容
git add xxx                    # 添加准备提交的文件或文件夹
git commit -m"fix #xxx xxx"    # 提交代码

git push                       # 定期将自己的代码推到服务器
```

> 将release分支合并到本地master和develop

```bash
git checkout master && git pull
git checkout develop && git pull
git checkout release/1.3.0
git pull && git flow release finish 1.3.0
```

> 合并分支的时候可能遇到冲突，注意解决冲突方法(无则跳过)

```bash
# 查看冲突的文件
git status

# 先处理二进制文件
git checkout --thires -- a.swf a.jpg   # 使用release的二进制文件
git checkout --ours   -- b.swf b.jpg   # 使用本地的二进制文件
git add c.swf c.jpg                    # 同上，使用本地的二进制文件

# 再处理文本文件
git mergetool -t vimdiff               # 使用mergetool，也可手工编辑文件，然后git add

# 然后再commit提交，注释参考：`Merge branch 'release/x.x.x'`
# 在该操作中会提示输入Tag描述，比如`发布WebCC x.x.x版本`然后保存
# git flow会自动创建一个以版本号为名的Tag

NOTE: 如果合并分支错误，且已Push到master，可以重新合并
        git reflog                     # 找回被删的release分支的版本号[sha1]
        git checkout [sha1]            # 将被删release checkout到本地
        git checkout -b release/x.x.x  # 将分支名改为release/x.x.x
        git checkout master            # 切回master分支
        git rebase --hard [sha2]       # 将master分支设置回错误合并前（如上一版本HEAD^）
        git merge release/x.x.x        # 重新合并release/x.x.x分支，合并成功后再提交
```

> 删除远程分支

```bash
git push origin --delete release/x.x.x
```

> 将合并后的本地master,develop,tag推送到服务器上

```bash
git push origin master && git push origin develop && git push --tags
```

#### 紧急修复bug

* 当线上遇到必须要紧急修复的BUG，不能推到下个版本修复时，可以在master基础上创建hotfix分支来修复BUG。BUG修复后，将修改内容合并回master和develop分支。

* 新发布版本号规则为：
  - 如果上一个版本没有第4位，则设置第4位为1
  - 如果上一个版本含有第4位，则递增第4位

* clone代码，初始化git flow，该操作只做一次

```bash
git clone git@projects.git
cd projects
git checkout master  # 由于默认分支改成了develop，需要先导入master分支
```

* 更新本地master分支到最新版本

```bash
git checkout master && git pull
```

* 创建基于本地master的hotfix分支

```bash
# 创建分支（分支名尽量能提示修复什么bug）
git flow hotfix start fix-xxx

# 在projects查看当前版本号，按规则确定新版本号并修改(视项目以及团队定义的版本文件为准)

# 在CHANGELOG.txt添加该版本的BUG修复说明
```

* 进行bug修复和提交（hostfix分支不能做新功能开发，只修bug）

```bash
git status                     # 查看有哪些文件修改
git diff                       # 查看具体修改内容
git add xxx                    # 添加准备提交的文件或文件夹
git commit -m"fix #xxx xxx"    # 提交代码
```

* Bug修复完成后，将hotfix分支合并到本地master和develop

```bash
git checkout master && git pull
git checkout develop && git pull
git checkout hotfix/fix-xxx
git flow hotfix finish fix-xxx
```

* 合并分支的时候可能遇到冲突，注意解决冲突方法

```bash
# 查看冲突的文件
git status

# 先处理二进制文件
git checkout --thires -- a.swf a.jpg   # 使用release的二进制文件
git checkout --ours   -- b.swf b.jpg   # 使用本地的二进制文件
git add c.swf c.jpg                    # 同上，使用本地的二进制文件

# 再处理文本文件
git mergetool -t vimdiff               # 使用mergetool，也可手工编辑文件，然后git add

# 然后再commit提交，注释参考：`Merge branch 'hotfix/fix-xxx'`

# 在该操作中会提示输入Tag描述，比如`修复xxx问题`，然后保存退出（比如VI编辑器按Esc，再按:wq）。

# git flow会自动创建一个hotfix同名的Tag
```

* 最后将合并后的本地master,develop,tag推送到服务器上

```bash
git push origin master && git push origin develop && git push --tags
```

#### git使用分支进行开发

* 当要做的功能不是一两行代码就可以完成的时候，需要在本地创建基于develop的feature分支
* 如果是多人合作，可以将分支publish到服务器上
* 分支完成并自己测试通过后才合并到develop，再由QA在develop上进行测试和修复bug

> clone代码，初始化git flow，该操作只做一次

```bash
git clone git@projects.git
cd projects
git checkout master  # webcc默认分支为develop，首次clone需先导入master分支
git flow init -d
```

> 更新本地develop分支到最新版本

```bash
git checkout develop && git pull
```

> 创建基于本地develop的feature分支

```bash
# 创建分支
git flow feature start feature-xxx

# 如果是多人合作开发某个功能
# 可以将该分支推送到服务器上
git flow feature publish feature-xxx

# 然后通知其他人可以使用track签出该分支
git flow feature track feature-xxx
```

> 进行代码修改和提交

```bash
git pull --rebase               # 如果合作开发，定期拉取别人提交的代码
                                # 如果pull出现冲突，则解决冲突后git add xxx
                                # 然后git rebase --continue继续操作。

git status                      # 查看有哪些文件修改
git diff                        # 查看具体修改内容
git add xxx                     # 添加准备提交的文件或文件夹
git commit -m"close #xxx xxx"   # 提交代码

git push                        # 如果合作开发，定期将自己的代码推到服务器
```

> 功能开发完成且自己测试OK后

```bash
# 基于该feature分支测试，所有发现的问题列出一个列表并收集起来
# 测试完成且确定放该功能后将feature分支合并到develop
git checkout develop && git pull              # 拉取最新的develop
git checkout feature/feature-xxx && git pull  # 如果合作开发，则先拉取服务器上的最新代码
git flow feature finish feature-xxx           # 完成分支开发，合并代码到本地develop
```

> 合并分支的时候可能遇到冲突，注意解决冲突方法

```bash
# 查看冲突的文件
git status

# 先处理二进制文件
git checkout --thires -- a.swf a.jpg   # 使用release的二进制文件
git checkout --ours   -- b.swf b.jpg   # 使用本地的二进制文件
git add c.swf c.jpg                    # 同上，使用本地的二进制文件

# 再处理文本文件
git mergetool -t vimdiff               # 使用mergetool，也可手工编辑文件，然后git add

# 然后再commit提交，注释参考：`Merge branch 'feature/x.x.x'`

# 在该操作中会提示输入Tag描述，比如`发布WebCC x.x.x版本`然后保存（比如VI编辑器按Esc，再按:wq）。

# git flow会自动创建一个以版本号为名的Tag。
# 然后在CHANGELOG.txt顶部in development段添加功能说明，并提交修改：
```

> 最后将合并后的本地develop推送到服务器上

```bash
git push origin develop
```

> 如果合作开发，在新功能发布到线上后再删除远程分支

```bash
git push origin --delete feature/x.x.x
```

> git打标签

```text
git tag version     # git打标签版本
git push --tags     # git打标签推到服务器上
git tag             # 查看标签
```
