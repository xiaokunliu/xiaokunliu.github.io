---
title: 分布式技术小结
category: 分布式技术小结
date: 2021-05-02 17:14:56
tags: 分布式架构设计
---

<!-- more -->

### 宏观角度 - 分布式设计目标
#### 分布式设计的目标

从宏观上思考,设计分布式系统的目标是实现系统的高性能以及高可用,而实现上述的分布式目标也会引入新的问题,即分布式技术实现的难点.
![分布式概要设计1](https://xiaokunliu.github.io/2020/05/10/distributed01/)
![分布式概要设计2](https://xiaokunliu.github.io/2020/05/14/distributed02/)


### 微观角度 - 分布式实践技术
#### 分布式基础理论
- 核心理论：CAP理论 & BASE理论
- 事务相关：ACID && 2PC & 3PC & TCC 
![分布式核心理论](https://xiaokunliu.github.io/2020/06/08/cap/)

- 分布式一致性问题
![一致性问题](https://xiaokunliu.github.io/2020/06/24/consistency/)

#### 分布式协议与算法
- 共识问题
![共识算法](https://xiaokunliu.github.io/2020/05/27/byzantine/)

- Paxos算法
![Paxos算法](https://xiaokunliu.github.io/2020/07/04/paxos/)

- Raft算法
![Raft算法简述](https://xiaokunliu.github.io/2020/07/24/raft/)
![Raft补充](https://xiaokunliu.github.io/2020/08/24/raft02/)

- 一致性Hash算法
![单节点hash算法](https://xiaokunliu.github.io/2020/10/01/single_consistence/)


#### 分布式事务


#### 分布式通信


#### 分布式服务


#### 分布式存储


#### 分布式缓存


#### 分布式计算




