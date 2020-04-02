---
title: linux工作常用命令小结1
category: Linux技术小结
date: 2020-04-02 20:47:28
tags: Linux技术小结
---

<!-- more -->


#### linux之用户管理命令小结
##### 1. 添加用户

> 为用户创建相应的帐号和用户目录/home/username
```bash
# 添加用户操作也会相应的增加一个同名的组，用户属于同名组
useradd -m username
```

> 设置密码

```bash
passwd username
```

> 完全的删除用户信息，使用-r选项

```bash
userdel -r username
```

> 帐号切换为userB

```bash
su userB
```

##### 2. 添加用户组

> 查看当前用户所属的组

```bash
groups
```

> 将用户加入到组

```bash
usermod -G groupNmame username
```

> 变更用户所属的根组

```bash
# 将用户加入到新的组，并从原有的组中除去
usermod -g groupName username
```

> 查看系统的所有组

```bash
more /etc/passwd
```

> 查看所有的用户组及权限

```bash
more /etc/group
```

> 查看文件所属用户、用户所在组以及其他用户的读写权限

```bash
# 文件属性字段总共有10个字母组成
# 第一个字母表示文件类型，如果这个字母是一个减号”-”,则说明该文件是一个普通文件。字母”d”表示该文件是一个目录，字母”d”,是dirtectory(目录)的缩写
# 后面的9个字母为该文件的权限标识，3个为一组，分别表示文件所属用户、用户所在组、其它用户的读写和执行权限
ls -l /etc/group
-rw-r--r-- 1 root root 3042 Apr 10 10:51 /etc/group
```

> 更改读写权限(字母方式)

```bash
# 语法：chmod userMark(+|-)PermissionsMark file_name
# userMark：u：用户 | g：组 | o：其它用户 | a：所有用户
# PermissionsMark: r:读 | w：写 | x：执行

chmod a+x file_name     # 所有用户可执行
chmod g+x file_name2    # 用户所在组可执行
chmod o-x file_name3    # 其他用户不可执行
```

> 更改读写权限(数字方式)

```bash
# 4(读)、2(写)、1(执行)三种数值的和来确定权限
# 第一位指定属主的权限，第二位指定组权限，第三位指定其他用户的权限
chmod 640 file_name    # 设置用户为读写，组为可读不可写，其他的用户不可读写权限
```

> 更改文件或目录所属拥有者

```bash
chown username dirOrfile
```

> 使用-R选项递归更改该目下所有文件的拥有者

```bash
chown -R username dir/
```

##### 3.环境变量

* bashrc与profile都用于保存用户的环境信息，bashrc用于交互式non-loginshell，而profile用于交互式login shell

```text
/etc/profile，/etc/bashrc 是系统全局环境变量设定
~/.profile，~/.bashrc用户目录下的私有环境变量设定
```

> 登入系统获得一个shell进程时，其读取环境设置脚本分为三步

* 首先读入的是全局环境变量设置文件/etc/profile，然后根据其内容读取额外的文档，如/etc/profile.d和/etc/inputrc
* 读取当前登录用户Home目录下的文件~/.bash_profile，其次读取~/.bash_login，最后读取~/.profile，这三个文档设定基本上是一样的，读取有优先关系
* 读取~/.bashrc

> ~/.profile 与 ~/.bashrc的区别

* 都具有个性化定制功能
* ~/.profile可以设定本用户专有的路径，环境变量等，它只能登入的时候执行一次
* ~/.bashrc也是某用户专有设定文档，可以设定路径，命令别名，每次shell script的执行都会使用它一次

> 上述的应用

```bash
# .bashrc
alias m='more'
alias cp='cp -i'
alias mv='mv -i'
alias ll='ls -l'
alias lsl='ls -lrt'
alias lm='ls -al|more'

log=/opt/applog/common_dir
unit=/opt/app/unittest/common

.bash_profile
. /opt/app/tuxapp/openav/config/setenv.prod.sh.linux
export PS1='$PWD#'

# 通过上述设置，我们进入log目录就只需要输入cd $log即可
```
