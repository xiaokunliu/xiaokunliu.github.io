---
title: python使用bump管理版本号和打包
category: Python
date: 2017-12-31 23:21:11
tags: Python
---

<!-- more -->


##### 使用bump管理版本号和打包
```text
bump依赖于bumversion工具[https://github.com/peritus/bumpversion]
```

> 项目添加bumpversion配置文件
```text
# 在当前项目根目录添加.bumpversion.cfg，内容如下（注意配置里面的缩进空白字符为Tab）：
[bumpversion]
commit = True
tag = True
current_version = 1.0.0

[bumpversion:file:setup.py]
search = version='{current_version}'
replace = version='{new_version}'

[bumpversion:file:CHANGELOG.md]
search = Unreleased
----------
replace = Unreleased
----------
v{new_version} ({now:%Y-%m-%d})
-------------------

# 说明
上面的配置意思是使用bump打包时，会执行以下操作：
1.根据传入的参数（majar|minor|patch）计算新版本号{new_version}
2.替换./setup.py中的version='{current_version}' 为 version='{new_version}'
3. 替换./CHANGELOG.md中的Unreleased\n---------- 为 Unreleased\n----------\nv{new_version}(YYYY-mm-dd)\n-------------------
4. 将以上修改进行本地提交：git commit
5. 使用新版本打一个tag：git tag v{new_version}
```

> 添加CHANGELOG文件
```text
# 在当前项目根目录添加CHANGELOG.md，用于记录发版历史，初始内容如下（注意格式必须匹配）：
# pyzipkin的CHANGELOG，其中Unreleased块表示还没有发布的新功能（下次发版发布），v1.1.17块为上一次发版的内容
Unreleased
----------

- #146732 修复Redis Sentinel Server Address错误问题
- imapp支持patch TaskBase.done

v1.1.17 (2017-08-24)
-------------------

- 区分http.uri和http.qs
- 记录nameko HTTP接口uri和host
- 改用util.get\_ipv4
- 当HTTP服务的不支持zipkin时记录http.host
- 将get\_ipv4移到util
...

注意可以手动编写该文件，但是要注意格式：
1.每条CHANGELOG占一行，以横杠加空格开头
2.如果CHANGELOG有Markdown格式字符，需要进行转义（如下划线前要用反斜杠转义）
3.版本号格式为vX.X.X (YYY-mm-dd)，前面不要有空白
```

> 安装bump打包工具
```text
# 内部开发的bump工具（注意不是pypi官网的bump），支持以下功能
1.自动从Git提交历史抽取日志到CHANGELOG
2. 使用bumpversion进行版本管理和打包 
3. 使用rsysnc将pip包上传到xxx.com

$ pip install -U bump -i https://xxx.com

查看帮助：

$ bump -h
Usage: bump [-a|--use-all-logs] [-u|--upload] [-T|--no-tag] [-y|--auto-yes]
[-v] [-h|--help] [majar|minor|patch|vX.X.X]

Example:
bump -a patch # use all logs as CHANGELOG, and release patch version
bump -uy # upload latest version to pypi without yes/no confirm
bump -u v1.0.0 # upload given version to pypi

Environment variables:
PYPI_SECRET Encryped secret for uploading to xxx.com
```

> 打包代码
```text
# 使用上传发版以来的所有提交日志更新到CHANGELOG，并提升patch位的版本号
$ bump -a patch # -a 表示使用上次发版以来的所有（all）提交日志更新CHANGELOG，patch可以不填

# 使用特殊的提交日志更新到CHANGELOG，并提升patch位的版本号
特殊的提交日志包括：以`close #xxx`, `finish #xxx`, `ref #xxx`, `fix #xxx`, 或`_VER_`开头的日志。
$ bump

# 手工编辑CHANGELOG，并提升patch版本号，并将新的pip包上传到xxx.com
$ vi CHANGELOG.md && git commit # 按上面的规则编辑CHANGELOG，并提交
$ bump -u
```

> 推送更新到git服务器
```text
# 本地打包完成后，所有修改都还在本地，可以使用git log -p查看打包进行了哪些修改，使用git tag -l查看tag。
# 所以需要手工执行以下命令推送代码和tag到gitlab服务器：
$ git push && git push --tags
```

> 上传pip包（可选操作）
```text
如果当前项目不包含setup.py文件，则不用执行这个步骤；
否则，打包操作时会执行python setup.py sdist，在当前目录的./dist下创建新版本的pip包。
可以使用pip install dist/xxx.tar.gz来安装新包进行测试，测试OK后使用bump -u将最新创建的pip包上传到pypi.cc.163.com：

$ bump -uy # -u 表示打包成功后，将pip包上传（upload），-y 表示提示确认时自动输入yes
# 可以浏览https://xxx.com页面查看新上传的包，
# 然后可以使用 pip install xxx -i https://xxx.com来安装
```

#### 本文转载于公司内部文章,若有转载请标明当前博客地址,谢谢!