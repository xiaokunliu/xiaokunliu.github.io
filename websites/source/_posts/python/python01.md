---
title: 搭建python环境
category: python
date: 2019-12-31 23:21:11
tags: Python基础
---

<!-- more -->
##### 安装Docker
```
## win10 && macOS直接下载安装
win10平台 ：https://www.docker.com/docker-windows
macOS平台：https://www.docker.com/docker-mac
## 需要使用Toolbox工具箱来安装Docker machine，并在docker machine下启动docker虚拟机
win7平台：https://www.docker.com/products/docker-toolbox
参考链接：http://www.docker.org.cn/book/install/c24.html
```

##### 构建Docker镜像
> 基于Dockerfile构建镜像

```
## 参考链接：https://docs.docker.com/engine/examples/running_ssh_service/#build-an-eg_sshd-image
FROM ubuntu:16.04
RUN apt-get update && apt-get install -y openssh-server
RUN mkdir /var/run/sshd
RUN echo 'root:root123' | chpasswd   ## 自己修改账户名和密码
RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
# SSH login fix. Otherwise user is kicked off after login
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd
ENV NOTVISIBLE "in users profile"
RUN echo "export VISIBLE=now" >> /etc/profile
EXPOSE 22 80 3306 ## 这里暴露80、22、3306
CMD ["/usr/sbin/sshd", "-D"]
```

> 构建镜像

```
## 进入本地自己的dockerfile所在目录，当前目录是在 ~/mywork/docker/docker
docker build -t 自定义镜像名称:tag标签名 .
docker build -t docker-ssh:v1 .
```

> 查看docker镜像

```
docker images
```

> 构建docker-ssh容器实例

```
docker run -d -P --name pydev -v $PWD/data:/opt/data docker-ssh:v1
```

> 进入linux环境

```
ssh root@127.0.0.1 -p 32770
```

##### 安装ZSH
```
apt-get install -y zsh && apt-get install -y wget
apt-get install -y git
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | sh
```

##### 安装python必备的apt软件包
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install -y build-essential
sudo apt-get install -y libsqlite3-dev
sudo apt-get install -y libreadline6-dev
sudo apt-get install -y libgdbm-dev
sudo apt-get install -y zliblg-dev
sudo apt-get install -y libbz2-dev
sudo apt-get install -y sqllite3
sudo apt-get install -y tk-dev
sudo apt-get install -y zip
```

##### 安装python相关包
```
### 安装python-dev包
sudo apt-get install -y python-dev
### 安装distribute包
sudo chmod -R 0755 /usr/local   ### 修改本地/usr/local权限
sudo chgrp -R keithl /usr/lcoal  ### 更改文件所属用户组
wget http://python-distribute.org/distribute_setup.py
sudo python distribute_setup.py
```

##### pip安装

```
#### 参考url:https://pip.pypa.io/en/stable/installing/
wget https://bootstrap.pypa.io/get-pip.py
sudo python get-pip.py
```

##### python虚拟环境搭建

```
pip install virtualenvwrapper
### 配置.bashrc or .bash_profile or .zshrc文件
if [ -f /usr/local/bin/virtualenvwrapper.sh ]; then
   export WORKON_HOME=~/workdir/python/pyenv
   export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python
   export VIRTUALENVWRAPPER_VIRTUALENV=/usr/local/bin/virtualenv
   export PROJECT_HOME=/Users/wind/projects/python/
   source /usr/local/bin/virtualenvwrapper.sh
fi
### 配置生效
source ~/.bash_profile(.bashrc/.zshrc)

### help帮助命令
mkvirtualenv --help

### 创建python开发目录并指定python版本
mkvirtualenv --python=/usr/bin/python2.7 pyen2.7 
OR
mkvirtualenv --python=/usr/bin/python3.5 pyen3.5

### 官网参考
https://virtualenvwrapper.readthedocs.io/en/latest/
```

##### python便捷工具
> 检查代码风格工具

```
pip install pep8
```

> 语法检查工具

```
pip install pyflakes
```

> 命令自动补全

```
### 1 way
pip completion --zsh >> .zprofile
source ~/.zprofile
### 2 way,在~/.zshrc里面一行
eval "pip completion --zsh"
### 3.使用bash
pip completion --bash >> ~/.profile
```
