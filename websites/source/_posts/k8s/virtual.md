---
title: vagrant安装linux-ubuntu
category: k8s
date: 2021-06-16 23:44:43
tags: k8s
---

<!-- more -->

#### 官网下载vagrant

```text
https://www.vagrantup.com/
```

#### vagrant 文档

```text
https://learn.hashicorp.com/tutorials/vagrant/getting-started-index?in=vagrant/getting-started
```

#### vagrant 镜像搜索

```text
https://app.vagrantup.com/boxes/search
https://www.vagrantup.com/docs
```

#### 下载安装virtual box

```text
01 访问VirtualBox官网  https://www.virtualbox.org/​
02 选择左侧的“Downloads”​
03 选择对应的操作系统版本​
04 傻瓜式安装​
```

#### 安装出现问题
##### Mac出现问题的解决
```text
https://blog.csdn.net/Kevin_Gu6/article/details/99434168
```

##### Window 出现问题
```text
[win10中若出现]安装virtualbox快完成时立即回滚，并提示安装出现严重错误
    (1)打开服务
    (2)找到Device Install Service和Device Setup Manager，然后启动
    (3)再次尝试安装
```

#### 基于vagrant安装虚拟机
```text
## install
vagrant init ubuntu/xenial64

## config 
## TODO
## https://blog.csdn.net/feifeixiang2835/article/details/91348700
Vagrant.configure('2') do |config|
    config.vm.network = "public_network"
    
    ## 配置虚拟机信息
    config.vm.provider "virtualbox" do |vb|
        vb.memory = "2048"
        vb.cpus = 2
        vb.name = "master" 
        vb.gui = true ## 解决下面timeout的问题
end    
```

#### vagrant 启动加载顺序
```shell script
vagrant init ubuntu/xenial64 
vagrant add box ubuntu/xenial64 box_image_path 
vagrant up

vagrant status
## vagrant halt - 关机操作
vagrant suspend ## 挂起，重新唤醒vagrant up
vagrant ssh
```

#### vagrant出现timeout

```text
## 主要是指定网络的时候指定了一个固定IP，一旦切换网络，分配的IP就不可用
default: SSH auth method: private key
default: Warning: Connection reset. Retrying...
default: Warning: Remote connection disconnect. Retrying...
default: Warning: Connection reset. Retrying...
default: Warning: Connection reset. Retrying...
default: Warning: Remote connection disconnect. Retrying...
default: Warning: Connection reset. Retrying...
default: Warning: Remote connection disconnect. Retrying...
default: Warning: Connection reset. Retrying...
default: Warning: Remote connection disconnect. Retrying...
```

- 解决方案(TODO)

```text
## 设置为私网 - 待验证
config.vm.network "private network", type: "dhcp" ## 动态分配IP
```
