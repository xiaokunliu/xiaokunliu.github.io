---
title: Raft算法简述
category: 分布式协议与算法
date: 2020-07-24 03:31:43
tags: 分布式协议与算法
---


<!-- more -->

### Raft算法

#### Raft简述

##### Raft概要

基于Multi-Paxos算法的基础上做了一些限制与简化,Raft算法属于强一致性模型的共识算法模型,在集群服务节点中仅有一个leader节点服务来负责所有请求的写入操作.Raft算法主要解决服务节点之间的leader选举,各个服务节点之间的日志同步以及服务节点成员中leader节点服务发生宕机时如何变更来达成最终的共识问题.从本质上是属于强leader模型,一切以leader为准的方式来实现各个服务节点一系列值的共识以及节点之间日志的一致性.

##### Raft算法三种状态

- Follower状态: 集群服务中普通的节点,主要职责有以下方面: 一个是负责接收和处理leader节点的消息;一个是负责维持与leader节点之间的心跳检测,以感知leader节点是处于可用状态;一个是通过心跳检测获取leader节点不可用状态时,将会推荐自己作为候选节点而发起投票选举操作.
- Candidate状态: 此时已经感知到leader节点不可用,那么这个时候将会向集群服务节点发起RPC消息的投票请求,通知其他集群服务节点来进行投票,如果超过半数投票那么将当前服务节点晋升为leader节点.
- Leader状态: 通过选举投票成为leader节点,此时主要的工作职责有三:一个是负责接收客户端的所有写入请求,包括集群服务节点的写入请求也将会转发到leader节点来处理;一个是同步数据日志到各个follower服务节点,以保证数据的一致性;一个是发送心跳检测以便于各个follower节点能够感知到leader节点是处于可用状态而不会发起投票选举操作.

##### Raft算法开源产品

- 分布式键值对存储系统Etcd: https://etcd.io/docs/v3.4.0/
- 基于Go语言实现的分布式注册与配置中心Consul: https://www.consul.io/docs
- Raft开源产品总览: https://raft.github.io/
- Raft学习指南: http://thesecretlivesofdata.com/raft/

#### Raft算法解决分布式共识问题

##### 单节点实现的共识问题

假设现在有一个数据存储服务的节点Node用于存储key-value的键值数据,这个时候客户端向键值服务节点Node发起写操作请求,那么此时对于Node执行写请求达到的共识操作要么是成功,要么是失败,即如下:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/single_node.jpg)

##### 多节点实现共识问题

当上述的键值服务节点增加副本时,这个时候便构成分布式共识问题,即一个client发起写请求操作到键值服务集群中,那么对于键值服务集群节点需要根据客户端client发起的写请求操作来达到共识.基于上述的Raft算法理解,我们知道raft算法是属于强leader模型来达到一致性,于是需要在键值服务节点中选举leader节点作为接收client的写请求,并且将由leader节点向其他follower节点发起执行命令来达到整个服务键值节点的一致性.因此整个过程包含leader选举策略以及实现的leader集群主从节点日志复制.即从宏观上看如下:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/multi_node.jpg)

##### Leader选举策略

- 在初始状态下,所有的键值服务节点都属于follower节点,在Raft算法中,follower节点会存在两个核心属性,即等待leader节点心跳检测的超时时间timeout以及任期编号number(也可以理解为武侠小说里面的选举丐帮帮主的任期,意为第几任丐帮帮主).在初始状态下,键值服务集群不存在leader节点,此时任期number为0,同时为了保证集群每个follower节点都能够有机会发起投票以及避免投票冲突带来的性能问题,于是采取心跳检测的超时时间作为随机数.即如下:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/leader_init.jpg)

- 在上述可以看到,follower节点A等待超时间为三个键值服务节点最小,于是将会由follower节点A向其他服务节点发起投票RPC消息请求成为leader节点,于是这个时候follower节点A成为候选节点A,更新当前节点的任期number为1并为自己投票获得为1的选票,即:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/leader_candidated.jpg)

- 这个时候其他键值服务B和C节点接收到候选节点A发起的RPC请求投票信息,这个时候B和C节点由于还没有接收到其他投票请求,那么就会更新当前的任期编号为1,并且将投票给A服务节点给予响应,同时A服务节点在发起RPC请求投票的时候会设置一个等待选举结果的随机超时时间timeout,如果在超时时间内没有获得半数投票,那么原先的选举会失效并将会重新发起投票选举,如果未超时,A服务节点接将收到其他服务节点投票响应并为自己的选票进行相应的计算增加,即:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/leader_candidated_response.jpg)

- 此时服务节点A获得到选票为3,在Raft算法中,集群服务节点超过半数即可由候选节点升级为leader节点,于是服务A节点获取到的票数为3已超过服务集群节点的半数agreement成为leader节点.即:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/leader_elected.jpg)

- 这个时候集群服务已经成功将服务节点A选举为leader节点,我们知道在Raft算法中,follower节点是有机会通过自身发起RPC投票消息成为leader节点的,那么当集群服务已经存在leader节点的时候,我们并不希望follower再次发起投票选举的动作,于是对于follower节点而言需要知道当前集群服务leader节点的存活情况,于是leader节点需要向follower节点周期性地发起心跳信息的检测以告知当前follower节点leader节点是可用状态,同时follower节点进行心跳检测超时时间是随机的,以避免超时发起投票选举冲突带来的网络开销性能,即:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/leader_ping.jpg)

至此,我们完成了一个键值服务集群的leader选举过程,接下来我们来看下客户端向集群服务发起写入请求操作的处理过程.

##### 主从节点日志复制

Raft实现主从日志复制其实是可以理解为实现集群服务节点对客户端client发起的写请求达成共识的问题,其主要实现是基于上述的leader选举与2PC提交协议来实现客户端client的写入操作,也就是当一个客户端client向集群服务发起一个写请求操作的时候,集群服务节点需要针对客户端的写入请求操作采取一致的变更行为.

- 首先是客户端向键值服务集群发起一个`set x= 10`的写入请求,此时服务集群接收写请求过程,一种情况是发送的写请求直接被leader节点接收,另一种是发送的写请求到folloer节点,由follower节点转发给leader节点,如下:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/leader_write_req.jpg)

- Leader节点接收到客户端发起的写请求操作之后,会将当前的`set x=10`指令更新到undo日志中,并向集群服务节点发起RPC日志复制更新操作,即如下:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/leader_write_log.jpg)

- Follower节点接收到Leader节点的RPC复制日志消息请求,根据日志的任期,命令以及索引值更新到本地状态机中,并给予leader消息的响应.即如下:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/leader_write_log_replicated.jpg)

- Leader服务节点接收到集群Follower服务节点的RPC日志复制消息的响应,此时约定一个投票规则,即超过半数节点服务返回响应为success,那么此时leader节点就可以将复制日志log的entry进行提交并告知当前leader节点已经完成复制日志的提交,此时follower节点接收到通知之后提交复制日志log的entry以与leader节点达成共识.即如下:

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/leader_write_commit.jpg)

- 最后键值集群服务对客户端client的写请求`set x= 10`达成共识之后并返回成功响应给客户端.在这个过程中客户端一直等待leader节点完成写入请求,处于阻塞状态.

![](/Users/keithl/docker/dev/data/xiaokunliu.github.io/websites/zimages/arch/raft/leader_write_log_response.jpg)

至此,一个键值服务集群基于Raft算法的强Leader模型来完成客户端发起写入请求的共识问题.

##### 解决成员变更

在上述的Raft集群模型中,只是一个正常运作的过程,如果键值集群服务节点发生扩容,那么此时将会面临集群分裂问题,即集群服务存在多个Leader节点;如果是当前的集群Leader节点发生宕机,那么此时将会面临Leader重新选举以及数据崩溃恢复问题.可以看到上述两个问题均为集群服务节点发生变更的问题,那么Raft算法是如何解决成员节点变更问题呢?

> 集群成员节点的变更本质

假设现在有一个基于Raft算法构成的集群服务,在上述的日志复制以及Leader选举之后,现在服务集群的节点A为Leader节点,节点B以及节点C作为Follower节点,当Raft算法服务集群发生以下缩容抑或是扩容之后,集群成员的变更情况如下

- 集群服务节点Leader发生宕机导致集群服务节点成员的变更







- 集群扩容产生的集群“脑裂”问题的演变(ES集群称为脑裂)





> 单节点变更解决集群成员节点变更问题









#### Raft算法的一般性

##### Leader选举策略





##### 日志复制





##### 成员变更问题













