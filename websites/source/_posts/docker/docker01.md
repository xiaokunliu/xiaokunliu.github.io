---
title: Docker使用小结
category: Docker
date: 2021-04-18 11:44:43
tags: Docker
---

<!-- more -->

### docker 相关命令小结
#### 查看镜像地址 
- docker info
```text
Client:
 Context:    default
 Debug Mode: false
 Plugins:
  app: Docker App (Docker Inc., v0.9.1-beta3)
  buildx: Build with BuildKit (Docker Inc., v0.5.1-docker)
  scan: Docker Scan (Docker Inc., v0.6.0)

Server:
 Containers: 1
  Running: 0
  Paused: 0
  Stopped: 1
 Images: 1
 Server Version: 20.10.5
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 1
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 io.containerd.runtime.v1.linux runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 05f951a3781f4f2c1911b05e61c160e9c30eaa8e
 runc version: 12644e614e25b05da6fd08a38ffa0cfe1903fdec
 init version: de40ad0
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 5.10.25-linuxkit
 Operating System: Docker Desktop
 OSType: linux
 Architecture: x86_64
 CPUs: 4
 Total Memory: 1.941GiB
 Name: docker-desktop
 ID: CVY7:NOBR:D337:VCRN:D24O:ZFH2:DJTN:TBXE:QO6K:ZOXP:A444:XIZN
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 HTTP Proxy: http.docker.internal:3128
 HTTPS Proxy: http.docker.internal:3128
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
```

#### docker 镜像操作
- docker pull
```bash
## docker pull keithl20160828/dev:1.0
## docker pull ubuntu:lastest
```

- docker基于镜像创建容器实例
```bash
## docker run -d -p 3306:3306 -p 32005:80 -p 32006:22 --name dev -v $PWD/data:/opt/data keithl20160828/dev:1.0
## -d 后台执行
## -p 端口映射
## --name 创建的容器名称
## -v docker容器数据挂载在本地磁盘
## 镜像以及版本
```

- 列出镜像
```bash
## 列出镜像
docker image ls

## 查看容器的详细信息
docker inspect name/Id
```

- 悬虚镜像
```bash
## 仓库名称以及镜像标签版本为none的镜像
## 删除悬虚镜像
docker image prune
```

- 删除镜像
```bash
## 删除一个镜像
docker image rm imageName/Id
## 删除所有redis镜像
docker image rm $(docker image ls -q redis)
## 删除redis:2.0版本之前的版本镜像
docker imgage rm $(docker imgae ls -q -f before=redis:2.0)

docker rmi imageId
```

- 基于已有的容器进行创建
```bash
docker [container] commit -m "commit msg" -a "author" containerId newImage:tag 
```

#### 容器操作
- 启动与关闭
```bash
docker [container] start containerName or docker start containerName
docker [container] stop containerName or docker stop containerName
```

- 第一次运行的容器
```bash
docker run -d -p 80:8080 --name nginx-dev nginx:1.0
## -i 标准输入打开，默认为false
## -t 是否分配一个终端，默认为false
docker run -d -it -p 80:8080 --name nginx-dev nginx:1.0 
```

- 查看启容器日志
```bash
docker logs [OPTIONS] CONTAINER
```

- 查看容器
```bash
## 查看所有容器
docker ps -a
## 查看运行的容器
docker ps
## 查看容器的详细信息
docker inspect name/Id
```

- 进入容器
```bash
## docker 进入容器
docker exec -it containerName/Id /bin/bash
```

- 导入与导出容器
```bash
## 导出
docker export containerName/Id container.tar

## 从已有的容器文件导入
cat container.tar | docker import keithl20160828/container:tag
## 从uri导入
docker import uri
```

- 暂停与重启容器
```bash
## 暂停容器
docker [container] pause containerName/Id
## 重启
docker [container] restart containerName/Id 
```

- 删除容器
```bash
## 删除一个终止的容器
docker rm containerName/Id
## 删除所有终止的容器
docker container prune
```

- 容器文件复制
```bash
## container cp命令支持在容器和主机之间复制文件
## 将本地的data文件复制到dev容器下的/tmp
docker [container] cp data dev:/tmp
```

- 查看文件系统变更
```bash
## docker [container] diff containerName/Id
docker diff dev
```

- 查看端口映射
```bash
## docker [container] port containerName/Id 
docker port dev
```

- 更新配置
```bash
## container update命令可以更新容器的一些运行时配置，主要是一些资源限制份
docker update --help 查看可以更新容器运行时的配置
```

#### 定制镜像
- Dockerfile文件示例
```text
FROM ubuntu:16.04
MAINTAINER keithl@1248237617.com

RUN mv /etc/apt/sources.list /etc/apt/sources.list-backup
COPY $PWD/config/sources.list /etc/apt/sources.list
COPY $PWD/config/.vimrc /root/.vimrc

RUN apt-get update && apt-get install -y openssh-server
RUN mkdir /var/run/sshd
RUN echo 'root:root123' | chpasswd
RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# SSH login fix. Otherwise user is kicked off after login
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

#ENV NOTVISIBLE "in users profile"
RUN echo "export VISIBLE=now" >> /etc/profile

## install nginx
##RUN apt-get install nginx

## install mysql
##RUN apt-get install mysql-server-5.7

## install vim
RUN apt-get install -y vim

## use netstat
RUN apt-get install -y net-tools

EXPOSE 22 80 3306 50000
CMD ["/usr/sbin/sshd", "-D"]
```

- 基于Dockerfile构建镜像
```bash
# cd /path/dockerfilepath (ls -al # have Dockerfile file)
docker build -t imageName:tag .
```

- 基于容器定制镜像
```bash
docker commit containerName keithl20160828/dev:1.0
docker images
docker push keithl20160828/dev:1.0
```

- 基于已有的镜像构建为自己的私有库镜像
```bash
docker tag session-web:lastest keithl20160828/session-web:1.0
docker push keithl20160828/session-web:1.0
```

- 保存镜像并拷贝到新机器重建镜像
```bash
## 在机器A上保存
docker save nginx | gzip > nginx-lastest.tar.gz
## 在机器B上加载nginx镜像
docker load -i nginx-lastest.tar.gz 
docker load < nginx-lastest.tar.gz 
```

#### 数据持久化
- docker数据管理方式

```bash
## 使用-v以及--mount方式来挂载数据卷(创建docker的数据卷会持久化到操作系统上的文件系统),同时还可以通过指定--mount type=bind挂载在当前宿主机器上的文件目录，
## 也可以通过--mount bind=tmpfs来挂载到内存,而使用-v是挂载到数据卷上，即使持久化到docker创建的指定数据卷上
## --mount or -v 挂载到数据卷，生产环境
## --mount type=bind,挂载到主机目录，需要使用主机目录的决定路径，用于测试
## --mount type=tmpfs，挂载到内存中
## 于是使用docker进行挂载使用--mount能够更为明确自己做的事情
## 用户需要在多个容器之间共享一些持续更新的数据,需要使用数据卷进行挂载
```

- 挂载形式
```text
--mount选项支持三种类型的数据卷，
包括：
- volume：普通数据卷，映射到主机/var/lib/docker/volumes路径下；
- bind：绑定数据卷，映射到主机指定路径下；
- tmpfs：临时数据卷，只存在于内存中,还可以使用tmpfs类型的数据卷,其中数据只存在于内存中,容器退出后自动删除.
```

- 创建数据卷
```bash
## 创建数据卷
docker volume create -d volumeName
## 存储在本地主机目录
/var/lib/docker/volumes
```

- 查看数据卷
```bash
## 列出数据卷
docker volumn ls
## 查看指定数据卷信息
docker inspect volumn volumeName
```

- 删除悬虚数据卷
```bash
## 删除悬虚数据卷 
docker volume prune

## 删除数据卷
docker volume rm volumeName
```

- 数据卷挂载
```bash
## 使用持久化
## 创建一个数据卷并基于数据进行挂载
docker volume create myVolume
docker run -d -p 8080:80 --name nginx-dev -v myVolume:/etc/nginx nginx:1.0 
docker run -d -p 8080:80 --name nginx-dev --mount source=myVolume,target=/etc/nginx nginx:1.0

## 数据卷挂载在主机文件夹上
docker run -d -p 8080:80 --name nginx-dev -v /opt/nginx/:/etc/nginx nginx:1.0  ## 本地目录不存在会创建
docker run -d -p 8080:80 --name nginx-dev --mount type=bind,source=/opt/nginx,target=/etc/nginx nginx:1.0   ## 本地目录不存在会报错

## 挂载文件
docker run -d -p 8080:80 --name nginx-dev -v /opt/nginx/nginx.conf:/etc/nginx/nginx.conf nginx:1.0
docker run -d -p 8080:80 --name nginx-dev --mount type=bind,source=/opt/nginx/nginx.conf,target=/etc/nginx/nginx.conf nginx:1.0 

## Docker挂载数据卷的默认权限是读写（rw），用户也可以通过ro指定为只读
docker run -d -p --name web -v /webapp:/opt/webapp:ro trainning/webapp python app.py
```

- 数据卷共享
```text
## 多个容器之间共享数据卷 - 数据卷容器
## 创建数据卷容器，仅用于多个容器进行共享
## 此时进入容器根目录即可看到dbdata目录,容器最好命名与共享的文件夹名称一致，一眼就知道共享的文件夹名称
docker run -it -d -v /dbdata --name db ubuntu:latest

## 其他容器进行共享
## 使用--volumes-from参数所挂载数据卷的容器自身并不需要保持在运行状态
## 如果删除了挂载的容器（包括db、db1和db2），数据卷并不会被自动删除。
## 如果要删除一个数据卷，必须在删除最后一个还挂载着它的容器时显式使用docker rm -v命令来指定同时删除关联的容器
docker run -it -d --volumes-from db --name db1 ubuntu:latest
docker run -it -d --volumes-from db --name db2 ubuntu:latest

## 此时在进入上述容器db/db1/db2的根目录下的dbdata为三个容器共享，此时其中一个容器在此目录进行写变化，则其他容器也能接收到最新的数据变化
```

- 数据备份与恢复
```bash
## 挂载宿主机器并备份dbdata数据目录
docker run --volumes-from db -v $(pwd):/backup --name backup-docker ubuntu:lastest tar cvf /backup/backup.tar /dbdata

## 恢复
## 先创建一个数据卷
docker run -it -d -v /dbdata2 --name dbdata2 ubuntu:lastest
docker run -it -d --volumes-from dbdata2 -v $(pwd):/backup restore-docker ubuntu:lastest tar xvf /backup/backup.tar
```

#### docker网络
- 宿主机器与docker容器之间的互通
```bash
## -P, 随机分配端口
docker run -d -it -P --name dev keithl20160828/dev:1.0

## -p 指定端口
docker run -d -it -p 8080:80 --name dev keithl20160828/dev:1.0

## 映射指定地址的任意端口
## 将容器的80端口映射到192.168.10.9地址下的随机分配的端口
docker run -d -it -p 192.168.10.9::80 --name dev keithl20160828/dev:1.0 
```

- 查看绑定的端口
```bash
## 查看容器端口绑定到对应的地址与端口
docker [container] port containerName/Id containerPort

## 查看容器的网络配置
docker inspect containerName/Id 
```

- 容器之间的服务互通
```bash
## 创建一个db的容器并绑定数据卷
docker volume create db
docker run -it -d -p 3306:3306 --name mysql-db -v db:/opt/mysql/data mysql:1.0

## 实现web容器与db互联
## --link 实现互联
## --link linkedContainerName:linkedContainerAlias
docker run -it -d -p 80:80 --name web -v $PWD/go:/opt/web --link mysql-db:mysql web:1.0 go run main.go
```

- Docker通过两种方式公开连接信息
```bash
## 1）环境变量env
## 查看环境变量
docker exec -it dev /bin/bash
env

docker run -it -d --name dev ubuntu:1.0 env

## 2)更新/etc/hosts文件
docker exec -it dev /bin/bash
cat /etc/hosts
```

- 网络相关的技术
```text
SDN（软件定义网络）或NFV（网络功能虚拟化）的相关技术
```

#### docker使用相关的细节
- source.list配置
- 参考：https://www.oreilly.com/library/view/robot-operating-system/9781783987443/0b841e6d-1ac3-4b5d-adba-482c92119e64.xhtml
```text
deb http://mirrors.163.com/ubuntu/ trusty main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ trusty-security main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ trusty-updates main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ trusty-backports main restricted universe multiverse
```

#### Dockerfile使用
##### 组成部分
- 基础镜像信息 
- 维护者信息 
- 镜像操作指令
- 容器启动时执行指令


##### 基础镜像信息
```text
FROM openjdk:8
```

##### 维护者信息
```text
MAINTAINER xiaokunliu
```

##### 镜像操作指令
```text
VOLUME /tmp
ADD ./dubbo-admin/dubbo-admin/target/dubbo-admin-0.0.1-SNAPSHOT.jar
```

##### 容器启动时执行指令
```text
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/app.jar]
```

##### 指令操作说明
```text
https://docs.docker.com/engine/reference/builder/
```

#### Docker Client 指令
`https://docs.docker.com/engine/reference/commandline/docker/`