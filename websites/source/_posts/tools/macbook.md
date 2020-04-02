---
title: linux(macbook)实际工作笔录
category: tools
date: 2020-04-02 20:47:28
tags: tools
---

<!-- more -->


>  批量拷贝某一个目录下的所有指定的文件到另一个指定的目录
```bash
## 比如拷贝视频文件,针对目录下存在空格命名情况,会导致无法处理成功,需要临时使用系统变量的IFS
#!/bin/bash
old_IFS=$IFS
IFS=$(echo -en "\n\b")
find . -name "*.mp4" | while read filepath 
do 
	echo "move file to 100036 dir: $filepath "
	mv -f $filepath ./100036
done
# 恢复系统原理的设置
IFS=$old_IFS
```

> 基于云端的API服务进行批量处理
```bash
### 比如github上批量删除无效仓库项目
# 在本地新建一个文件repos,逐行贴上对应的一个仓库名称: githubName/repoName,如:
xiaokunliu/delRepo1
xiaokunliu/delRepo2
...
# github上申请对应开发者账户的删除权限token
### 编写脚本完成
#!/bin/bash
cat repos | while read repo
do
    echo "delete github for repo: https://api.github.com/repos/$repo"
    curl -XDELETE  -H 'Authorization: token token' "https://api.github.com/repos/$repo"
done
```
