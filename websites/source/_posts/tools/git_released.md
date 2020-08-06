---
title: git需求管理规范
category: tools
date: 2020-08-05 22:07:07
tags:
---



<!--more -->

#### 需求管理与git流程控制

##### 迭代周期与git版本管理

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/tools/git/git_timeline.jpg)

##### 开发流程

> 需求开发流程

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/tools/git/git_flow.jpg)

> 线上紧急修复bug流程

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/tools/git/git_flow_hotfix.jpg)



##### 版本号管理

打包发版是基于release分支,因此进行版本管理是在release分支进行的,对此关于项目版本号的简要说明如下:

当代码在feature开发测试完成并merge回develop后，在发版本前一天打一个基于develop的 release分支。在该分支上进行测试和修复bug，不开发新功能。

新发布版本号规则为：

- 如果是重大功能变更，则递增第1位，第2、3位归0。（如整体换UI、改用新技术）
- 如果是添加大功能（工作量按周计算），则递增第2位，第1位不变，第3位归0。（如新增贵族功能）
- 如果是添加小功能（工作量不满一周），则递增第3位，第1、2位不变。（如添加某某接口）

当线上遇到必须要紧急修复的BUG，不能推到下个版本修复时，可以在master基础 上创建hotfix分支来修复BUG。BUG修复后，将修改内容合并回master和develop分支。

新发布版本号规则为：

- 如果上一个版本没有第4位，则设置第4位为1
- 如果上一个版本含有第4位，则递增第4位

#### gitflow简介

##### gitflow进行需求开发

上述的流程规范可以使用gitflow工具来提升基于git开发流程,具体实践如下:

- 首先是项目的前期准备工作

```bash
## 这里我直接在gitee上创建一个新项目
git clone https://gitee.com/xiaokunliu/tpp.git && cd tpp

## 基于master新建一个develop分支
git pull origin master && git branch develop && git brnach -a

## 切换到develop分支
git checkout develop

## 推送develop到远程服务器
git push origin develop

## 查看分支
git branch -a | grep develop
```

- 其次,是需要开发一个新的需求功能,基于develop分支新建一个feature分支

``` bash
## 将远程仓库拉取下来
## 如果当前分支未提交或者是存在冲突先进行提交或者是冲突的问题解决
git pull origin master && git pull origin develop

## 第一次使用gitflow
git flow init -d

## 更新develop分支
git checkout develop && git pull

## 基于最新的develop分支新建一个feature分支,比如开发支付模块
git flow feature start feature-payment

## 如果是多人合作的时候,需要将本地的feature分支推送到远程服务器进行功能测试
## 发布feature分支并推送到远程服务器
git flow feature publish feature-payment

## 通知处理相同需求的同事将feature分支检出
git flow feature track feature-payment

## 如果中间feature分支存在bug,那么需要将feature拉取下来进行bug修复
git pull origin master && git pull origin develop && git pull origin feature-payment && git checkout feature-payment

## 进行bug修复并提交
git add ..
git commit -m ""
git push origin feature-payment

## 开发并且QA测试完成之后需要将feature分支合并到develop分支进行回归测试
git flow feature finish feature-payment
```

- 然后,feature分支已经完成开发,需要进行发版

```bash
## 首先登录测试服务器,将代码拉取下来
git pull origin master && git pull origin develop && git checkout develop

## 其次基于develop分支进行打包发版,需要打一个release分支
## 比如现在的版本号为1.1.0,在项目中增加一个配置文件CHANGLOG.txt用于记录版本号以及对应的版本号修改的时间
version='1.1.0'
version_date="2020-08-01"

## 此时的版本为1.2.0,假设现在是周期版本的需求迭代
git flow release start 1.2.0

## 然后更新上述配置文件的版本号以及时间
version='1.2.0'
version_date="2020-08-06"

## 然后提交CHANGLOG.txt
git add CHANGLOG.txt
git commit -m "change version and version date"

## 将release版本进行发布到测试环境进行回归测试
git flow release publish 1.2.0

## 如果回归测试有问题,需要通知对应需求的功能的同事进行修复
## 将项目检出
git flow release track 1.2.0

## 进行bug修复然后提交,release分支只用于bug修复,不做新的需求功能开发,如果有新的需求功能
## 不紧急的话走周期版本,紧急需求功能将feature分支合并到release分支,注意此时所处的时间点是回归测试阶段

## 完成bug修复之后,准备打包到稳定的master版本
git checkout master && git pull && git checkout develop && git checkout release/1.2.0 && git pull

## 打包并合并到当前测试服务器的develop以及master
git flow release finish 1.2.0

## 如果有冲突再解决冲突

## 最后将develop以及master推送到远程
git push origin master && git push origin develop && git push --tags
```

- 最后,将master版本推送到预发布线上preview环境,线上preveiw环境ok之后再走生产环境,实现逐个单节点发布.

##### gitflow进行紧急修复bug

```bash
## 紧急bug修复需要基于master分支
git checkout master && git pull

## 开启一个hotfix的分支
git flow hotfix start fix-payment-notoreders

## 本地进行bug修复之后,需要进行bug测试回归,需要发布hotfix分支到测试环境上
git flow hotfix publish fix-payment-notoreders
## 登录测试服
git checkout master && git pull && git checkout develop && git pull && git checkout fix-payment-notoreders
## 进行测试如果没有问题,那么就将hotfix的分支进行打包
git flow hotfix finish fix-payment-notoreders

## 更新到master以及develop分支上
git push origin master && git push origin develop && git push --tags

## 最后将代码发布到线上即可
```

##### 上线之后删除功能分支以及release分支

```bash
## 删除本地分支
git branch -D feature/feature-payment

## 删除远程分支
git push origin :feature/feature-payment
```

#### git与gitflow学习资料参考

```reStructuredText
## github上学习资源: 
https://github.com/geeeeeeeeek/git-recipes/wiki

## gitflow参考资料:
https://datasift.github.io/gitflow/IntroducingGitFlow.html
https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow
```









