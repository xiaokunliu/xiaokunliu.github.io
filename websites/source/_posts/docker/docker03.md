---
title: Docker容器监控
category: Docker
date: 2021-04-18 11:44:43
tags: Docker
---

<!-- more -->

### docker 容器监控
#### 监控工具: 采集docker集群的监控（cpu/内存/机器等基础服务指标）

- CAdvisor：采集数据
- influxdb：存储数据
- grafana：展示数据

#### 基于docker-compose部署

- docker-compose.yaml文件
```yaml
version: '3.1'

volumes:
  grafana_data: {}

services:
 influxdb:
  image: tutum/influxdb:0.9
  #image: tutum/influxdb
  #image: influxdb
  restart: always
  #user: 
  environment:
    - PRE_CREATE_DB=cadvisor
  ports:
    - "8083:8083"
    - "8086:8086"
  expose:
    - "8090"
    - "8099"
  volumes:
    - ./data/influxdb:/data
 cadvisor:
  #image: google/cadvisor:v0.29.0
  image: google/cadvisor
  links:
    - influxdb:influxsrv
  command: -storage_driver=influxdb -storage_driver_db=cadvisor -storage_driver_host=influxsrv:8086
  restart: always
  ports:
    - "8080:8080"
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:rw
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro

 grafana:
  #image: grafana/grafana:2.6.0
  user: "104"
  image: grafana/grafana
  user: "104"
  #user: "472"
  restart: always
  links:
    - influxdb:influxsrv
  ports:
    - "3000:3000"
  volumes:
    - grafana_data:/var/lib/grafana
  environment:
    - HTTP_USER=admin
    - HTTP_PASS=admin
    - INFLUXDB_HOST=influxsrv
    - INFLUXDB_PORT=8086
    - INFLUXDB_NAME=cadvisor
    - INFLUXDB_USER=root
    - INFLUXDB_PASS=root
```

- 启动
```bash
docker-compose up
```

### docker的日志监控
#### docker日志配置
```text
https://docs.docker.com/config/containers/logging/configure/
```

#### 使用Graylog来收集日志
- graylog部署

```bash
## https://hub.docker.com/r/graylog/graylog
cd /opt/graylog ## 所有操作都是基于当前的目录下进行

## 创建数据目录
mkdir -p ./graylog/data

## 创建配置目录
mkdir -p ./graylog/config && cd ./graylog/config
## 直接下载官方配置的推荐文件graylog.conf以及日志配置文件log4j.xml
## https://github.com/Graylog2/graylog-docker/config/graylog.conf
## https://github.com/Graylog2/graylog-docker/config/log4j.xml

## 修改上述的graylog.conf的时区 - 中国时区
root_timezone=Etc/GMT-8
root_timezone=Asia/Shanghai

## 新建docker-compose构建完整服务(见下面的docker-compose.yaml)

## 启动服务
docker-compose up

## 访问uri && 登录 admin/admin
```

- docker-compose.yaml
```text
version: '2'
services:
  # MongoDB: https://hub.docker.com/_/mongo/
  mongodb:
    image: mongo:3
    volumes:
      - mongo_data:/data/db
  # Elasticsearch: https://www.elastic.co/guide/en/elasticsearch/reference/6.x/docker.html
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.5.1
    volumes:
      - es_data:/usr/share/elasticsearch/data
    environment:
      - http.host=0.0.0.0
      - transport.host=localhost
      - network.host=0.0.0.0
      # Disable X-Pack security: https://www.elastic.co/guide/en/elasticsearch/reference/6.x/security-settings.html#general-security-settings
      - xpack.security.enabled=false
      - xpack.watcher.enabled=false
      - xpack.monitoring.enabled=false
      - xpack.security.audit.enabled=false
      - xpack.ml.enabled=false
      - xpack.graph.enabled=false
      ## 根据机器配置对应的内存 
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    mem_limit: 256M
  # Graylog: https://hub.docker.com/r/graylog/graylog/
  graylog:
    image: graylog/graylog:2.5
    volumes:
      - graylog_journal:/usr/share/graylog/data/journal
      - ./graylog/config:/usr/share/graylog/data/config
    environment:
      # CHANGE ME!
      - GRAYLOG_PASSWORD_SECRET=somepasswordpepper
      # Password: admin
      - GRAYLOG_ROOT_PASSWORD_SHA2=8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918
      # 这个地址需要配置成你要访问的地址，比如你的容器部署在192.168.1.2，你需要配置成http://192.168.1.2:9000/api，否则访问会报错
      - GRAYLOG_WEB_ENDPOINT_URI=http://192.168.100.249:9000/api
    links:
      - mongodb:mongo
      - elasticsearch
    depends_on:
      - mongodb
      - elasticsearch
    ports:
      # Graylog web interface and REST API
      # web界面端口
      - 9000:9000
      # gelf收集日志的端口，如果需要添加graylog收集器，可以新增暴露出来的端口
      # Syslog TCP
      - 514:514
      # Syslog UDP
      - 514:514/udp
      # GELF TCP
      - 12201:12201
      # GELF UDP
      - 12201:12201/udp
# Volumes for persisting data, see https://docs.docker.com/engine/admin/volumes/volumes/
volumes:
  mongo_data:
    driver: local
  es_data:
    driver: local
  graylog_journal:
    driver: local
```

- graylog存在的问题
```text
graylog在root用户下，使用github上下载的docker-compose.yml启动时出现Operation not permitted错误
---
请参考gitlab上解决方案：  
[http://59.111.92.219/q1902-java/subject-3/issues/1](http://59.111.92.219/q1902-java/subject-3/issues/1)
```

- 配置信息收集日志

> input配置


> 创建容器并配置docker以便于收集到graylog

```bash
## 命令行启动
## docker run --log-driver=gelf --log-opt gelf -address=udp://{graylog的服务地址}:12201 --log-opt tag=<当前容器服务标签，用于graylog提供查询的分类> <IMAGE> 运行命令
docker run --log-driver=gelf --log-opt address=udp://localhost:12201 --log-opt tag="{{.ImageName}}/{{.Name}}/{{.ID}}"
```

> 使用docker-compose加入graylog日志收集

```text
## docker-compose.yaml 加入配置
version: '2'
services:
 nginx:
  image: nginx:latest
  ports:
   - "80:80"
  logging:
   driver: "gelf"
   options:
    gelf-address:"udp://localhost:12201"
    tag: front-nginx
```






