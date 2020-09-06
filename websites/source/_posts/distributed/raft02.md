---
title: raft算法增强与总结
category: 分布式架构设计
date: 2020-08-24 13:03:44
tags: 分布式架构设计
---

<!-- more -->

## Raft算法

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websitess/zimages/arch/raft02/title.jpg)

### Raft简述

#### Raft概要

Raft算法是一种用于管理Replicated Log的共识算法,其算法结果与效率与Multi-Paxos一致,但是在算法的设计结构上与Paxos算法是不同的,Raft算法更加便于理解和实现,主要有以下两点:

- Raft算法将leader选举,日志复制以及安全共识要素分离出来,并强化一致性,即强leader模型,集群服务节点以leader节点为主来实现实现一系列值的共识和各节点日志的一致,强leader这样做的目的是为了减少集群其他服务节点必须考虑的状态,换言之,就是如果集群服务存在leader节点,那么对非leader节点服务而言,我只考虑leader节点是存活的情况下,保证我自己当前服务节点状态不会发生变更.

- 其次,Raft算法提供成员变更机制,严格来说是引入单节点变更机制来解决集群中存在“脑裂”情况(集群出现多个leader节点)

#### Raft集群节点状态

- Follower状态: 集群服务中普通的节点,主要职责有以下方面: 一个是负责接收和处理leader节点的消息;一个是负责维持与leader节点之间的心跳检测,以感知leader节点是处于可用状态;一个是通过心跳检测获取leader节点不可用状态时,将会推荐自己作为候选节点而发起投票选举操作.
- Candidate状态: 当前集群节点为候选leader服务节点,将会向集群其他服务节点发起RPC消息的投票请求,通知其他集群服务节点来进行投票,如果超过半数投票那么当前服务节点将晋升成为leader节点.
- Leader状态: 通过选举投票成为leader节点,此时主要的工作职责有三:一个是负责接收客户端的所有写入请求,包括集群服务节点的写入请求也将会转发到leader节点来处理;一个是同步client的数据操作日志(binlog/oplog)到各个follower服务节点,以保证数据的一致性;一个是发送心跳检测以便于各个follower节点能够感知到leader节点是处于可用状态而不会发起投票选举操作.

#### Raft算法参考学习

- 分布式键值对存储系统Etcd: https://etcd.io/docs/v3.4.0/
- 基于Go语言实现的分布式注册与配置中心Consul: https://www.consul.io/docs
- Raft开源产品总览: https://raft.github.io/
- Raft学习指南: http://thesecretlivesofdata.com/raft/

### Raft算法核心原理

#### 集群强Leader选举

##### leader选举要素

- 任期Term: leader选举存在任期Term,每次完成leader选举会更新一次任期Term
- 超时Timeout: 具备两种含义,即包含leader心跳检测超时以及候选节点等待选举结果超时

##### leader选举过程

- 初始化状态

所有的键值服务节点都属于follower节点,在Raft算法中,follower节点会存在两个核心属性,即等待leader节点心跳检测的超时时间timeout以及任期编号term.在初始状态下,键值服务集群不存在leader节点,此时任期term为0,同时为了保证集群每个follower节点都能够有机会发起投票以及避免投票冲突带来的性能问题,于是采取心跳检测的超时时间作为随机数.即如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/leader_init.jpg)

- 投票请求

在上述集群的初始状态中,我们可以看到followerA节点等待leader节点的ping心跳率先超时,于是followerA节点成为候选节点Candidate,同时默认为自己的任期term进行+1的投票,此时节点A会更新当前的任期编号term为1(初始状态为0),接着再向集群服务节点B以及C发起RPC投票请求,即如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/leader_vote_req.jpg)

- 投票请求响应

当候选服务节点A发起RPC投票请求的时候,会在当前的服务节点设置一个等待选举结果的随机超时时间timeout,那么存在两种情况:

1) 如果是在timeout时间内,集群的服务节点B以及C接收到候选节点A的RPC投票请求并且此时还没有接收到其他服务节点的投票请求,那么就会更新当前的任期编号为1,同时将投票给A服务节点并给予响应,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/leader_vote_resp_success.jpg)

2) 如果在超时时间内没有获得半数投票,那么原先的选举会失效并将会重新发起投票选举,如果未超时,A服务节点接将收到其他服务节点投票响应并为自己的选票进行相应的计算增加,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/leader_vote_resp_fail.jpg)

- A节点获得半数投票成为leader节点

当候选节点A在超时的时间内获得到集群半数以上的节点给予的投票响应,于是会晋升成为leader节点,并周期性地向follower节点发起类似ping的心跳响应,并在当前任期内维持ping心跳检测的超时时间timeout,即如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/leader_vote_result.jpg)

#### 日志复制

##### 日志项Log Entry

> 日志项属性

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/log_entry.jpg)

由上图可知,日志实体(log Entry)主要包含以下几个属性:

- 日志执行指令(cmd):客户端服务发起事务请求操作并通过leader服务节点的共识模块输出的一条持久化到leader服务节点所在的状态机上的指令.
- 日志索引值(logIndex): 日志项对应的整数索引值,用来标识日志项的,是一个连续的、单调递增的整数号码,并且当前的日志项(logEntry)不会改变其在日志(log)下的位置

- 日志任期编号(term): 创建当前日志项(logEntry)的leader节点当前所处的任期编号term,主要来保证appendEnrty到日志log的一致性检查.

##### 日志项复制原理

Raft算法的日志复制是基于优化之后的2PC(减少一半的消息往返)提交来实现客户端事务操作的共识与数据的一致性,其具体过程如下:

- 客户端服务client service向Raft集群服务发起事务请求操作,将转发由Raft集群的leader节点进行事务请求的写入以此来保证Raft集群服务的共识问题,假设客户端服务发起写请求操作`set x=10`,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/log_write_req.jpg)

- leader节点接收到事务请求操作,将请求提交给leader节点服务下的共识模块进行事务操作并输出cmd指令以RPC的方式进行AppendEntries(复制日志项)到其他服务节点中,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/log_rpc_append_entry.jpg)

- 当follower节点将接收到leader节点RPC的appendEntries(日志复制)进行持久化后,将会返回给leader节点,而leader节点如果接收到大多数follower节点的RPC日志复制成功的响应,那么就会将当前的log应用到leader节点的状态机并给予客户端的响应.即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/log_rpc_append_entry_res.jpg)

- 这个时候leader节点如果有新的RPC日志项复制抑或是发起heartbeat心跳检测,`RPC-AppendEntries&Heartbeat`会携带当前最大的且即将提交的`index`到follower节点,follower节点会进行一致性检查流程并将日志项提交到本地状态机以保证与leader节点的日志数据一致.即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/log_rpc_append_entry_res_updated.jpg)

- 整个RPC日志复制流程如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/log_rpc_append_entry_final.jpg)

最后,关于Raft算法的日志复制,可以类比数据库的主从复制架构来思考,上述的提交日志可以理解为binlog或者是oplog,那么每次发起的RPC日志复制抑或是心跳检测都会携带日志最大且即将提交的索引值index,通过binlog或者是oplog将数据更新到节点的状态机上.

##### 日志的一致性检测

> 一致性检测原理

假设当前Raft服务集群节点的log以及对应的logEntry如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/log_consistence_init.jpg)

在上述图中,我们看到leader节点与follower节点的日志数据存在不一致,我们知道Raft算法是属于强leader模型,一切以“leader”为主,因此日志复制也不例外,一旦出现不一致,那么follower节点会进行一致性检查并以leader发送的RPC日志为主将不一致性的数据强制更新为与leader节点日志一致,而对于leader节点的日志是不会覆盖和删除自己的日志记录.也就是说对于raft算法的一致性检查原理主要包含以下步骤:

- leader节点通过RPC日志复制的一致性检查,查找与follower节点上与自己相同的日志项最大的索引值,这个时候follower节点的索引值之前的日志记录与leader是保持一致的,而之后的日志将产生不一致.
- leader节点找到与follower节点相同的索引值并将索引值之后的日志记录复制并通过RPC发送到follower节点,强制follower节点更新日志数据不一致的记录.

> 一致性检查流程

以上面的日志记录为例,leader节点当前的索引值为8,而follower节点A与follower节点B索引值分别为5和8,这个时候假设leader节点向集群follower节点通过RPC发送新的日志记录抑或是心跳消息.此时follower节点A与B将会进行一致性检查.

- 若为RPC复制新的日志项,其检查流程如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/log_consistence_req.jpg)

- 若为心跳检测,其检查流程如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/log_consistence_req_heartbeat.jpg)

上述不论是发送RPC复制日志项还是心跳消息,在follower节点B以及C中由于日志属性不匹配抑或是日志项不存在而被拒绝当前`index=9`或者`index=8`的日志项更新,于是对于leader节点将向后递减重新发送新的RPC复制日志到follower节点,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/log_consistence_res.jpg)

至此,在有限的时间内leader节点会将其日志复制到follower节点来保证整个Raft集群的日志数据一致性.

> 小结

- leader节点通过日志复制RPC消息,将当前最新且uncommited的日志索引值发送到Raft集群的follower节点上,假设此时的消息索引值`index=rpcIndex`

- Raft集群的follower节点接收到日志复制的RPC消息,会检查上一个日志索引值preIndex(index=rpcIndex-1)对应的日志项entry中的index,term以及cmd的属性是否匹配,如果不匹配则会拒绝当前index(rpcIndex)的entry更新并返回失败消息给leader节点,如果成功则持久化当前日志项并返回成功消息给leader节点(一致性检查).
- leader节点接收到follower节点的失败消息,于是递减当前的index`(rpcIndex-1),再次发起新的日志RPC复制消息到follower节点上
- 这个时候follower再次进行一致性检查,如果匹配那么返回成功给leader节点,此时leader节点就会知道当前follower节点的数据索引值(rpcIndex-2)与leader节点的日志是一致的.
- 最后leader几点再次发起日志复制的RPC请求,复制并更新该索引值(rpcIndex-2)之后的日志项,最终实现leader节点与follower节点的日志一致性.

#### 成员变更

##### 集群成员变更问题

- 集群中leader节点崩溃与恢复

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_cluster_down.jpg)

从上面我们可以看到在一个Raft集群服务中,如果leader节点发生不可用,那么剩下的follower节点将会重新进行选举,假设此时选举B作为leader节点,那么当原有的leader节点恢复正常的时候,此时集群存在两个“leader”节点,如何解决?

- 集群扩容

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_clutser_extend.jpg)

从上述可以看到,在实现集群服务节点的扩容时,如果新加入的服务节点刚好碰上原有的集群服务发生网络分区,导致C与B,A节点失去联系,而在C节点恢复的时候与刚加入的服务节点D&E组成一个新的Raft集群(D&E与原有的服务节点属于同一个区域内);其次如果是新加入的两个节点与原有的服务节点不属在同一个区域,那么当前raft集群扩容就存在跨区域的集群,跨区域必然会存在网络不可靠的因素,因此一旦两个区域发生网络分区错误,那此时新区域下的D&E以及原有的区域集群节点分别组成了一个Raft集群,此时就会面临双leader节点问题.

从上述的问题分析中,我们都看到集群服务可能出现多个leader节点,这就违背了Raft算法的强leader且唯一性的特征,而对于Raft算法解决这类集群成员节点变更则是通过单节点变更来解决集群的“脑裂”问题.

##### 单节点解决成员变更

在Raft算法的论文中关于集群成员节点的变更存在着一个集群的配置属性,即在了解单节点变更方式之前,我们需要对集群cluster的配置有一个基本的认知(与ES集群相似)

> 集群配置

在一个稳定的Raft集群服务中,存在着以下的一个leader以及两个follower节点,follower节点的配置需要从leader节点同步进行更新,于是在这里我们关注leader节点的配置即可,也就是整个集群的一份配置.对于ES集群而言,其配置信息如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_cluster_configuration.jpg)

上述的log/clusterName/nodes等信息组成一个集群的配置信息,也就是说Raft 算法要解决上述多leader节点的问题,其实是保障在集群配置变更时,集群能稳定运行,不会同时出现多个leader节点的集群配置.

> 引入集群配置的目的

- 其一,我们可以想到的方案是停更集群服务的执行,更改集群的配置再重启生效,但是显然在当前互联网应用中是不允许也是不被采纳的,极大影响用户体验,容易造成用户流失;
- 其二是重启集群服务的步骤存在着误操作步骤,容易导致上线服务时出现不可预知的错误.

因此Raft算法利用单节点变更的方式来实现自动化的配置更新以保证我们集群服务的强一致性.

> 单节点变更原理

- 假设现有的Raft集群服务配置为C[n1,n2,...],此时服务集群加入一个节点Xm,同时对应的配置nm,这个时候leader节点监听到有新的节点加入读取节点配置nm并更新到当前的leader配置C中,设新配置为C1,即[n1,n2,...nm],并向新节点同步数据log.
- 其次leader节点更新配置之后C1并发起日志复制的RPC请求消息到集群服务的各个节点,集群服务节点接收到RPC消息请求并将新的配置应用到本地状态机中,至此完成了单节点变更.

> 单节点如何解决成员变更

- 崩溃恢复过程

1) Raft集群的leader节点发生不可用,原先具备leader选举权的follower节点进行leader选举,最终节点B成为新一轮的leader,即如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_change_cluster_1.jpg)

此时集群的配置由[A,B,C]变更为[B,C],同时配置中的部分属性数据也会发生变更,比如任期term以及log数据

2) 这个时候如果A节点恢复并重新加入到现有的Raft集群中,那么利用单节点原理,leaderB更新配置并将log同步覆盖更新节点A,此时A节点自然作为follower节点,最后leader节点B通过日志复制的RPC向集群服务更新配置最终保证集群的强leader模型,

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_change_cluster_2.jpg)

- 扩容解决“脑裂”问题

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_change_cluster_3.jpg)

在现有的Raft集群中加入节点D,此时集群raft_cluster监听到节点D加入集群,此时leader节点更新集群的配置并向节点D同步日志数据log,最后更新集群配置之后再向Raft集群服务节点发起日志复制的RPC请求消息同步最新的集群配置信息从而保证Raft集群的强一致性.

- 集群跨区域“脑裂”问题(扩展)

> 集群脑裂

1) 初始化状态,只有华南区域部署Raft集群服务,但因业务需求原因,需要在华北地区增加机器节点,并加入到现有的Raft集群服务,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_change_cluster_region1.jpg)

2) 如果两区域网络没有发生分区错误,按照上述单节点变更原则,最终Raft集群配置以及节点状态如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_change_cluster_region2.jpg)

3) 如果两区域的网络发生分区,那么就会导致Raft集群服务在不同的区域中产生两个leader节点,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_change_cluster_region3.jpg)

4) 华南区域leader节点发生不可用,而在重新选举的过程中,区域产生分区错误,导致Raft集群服务出现两个leader节点,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_change_cluster_region4.jpg)

对于3&4问题,属于跨区域的集群的“脑裂”问题(ES集群也存在同样的问题)

> 如何解决集群脑裂问题

- 脑裂产生的原因

1) 网络发生分区错误: 机器宕机抑或是网络拥堵超时

2) 每个节点具备成为leader节点资格.

- 脑裂解决

1) 基于上述的现象,发生网络分区错误最有可能的原因就是网络超时,于是可以针对分区域的不同节点设定对应的超时时间,比如可以将华北区域的节点超时时间变长(等待心跳检查以及选举leader的超时时间)

2) 其次可以将node节点降低权限,仅作为提供数据服务,不具备竞选leader资格(即没有投票资格),假设当前集群节点个数为n,根据反证法以及投票满足大多数原则(理想状态不考虑随机超时时间)可知最小可以配置具备成为leader节点个数为`(n/2)+1`并且是在处于同一个区域下,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/mem_change_cluster_region5.jpg)

### Raft算法分析小结

#### 节点状态变化

- Raft算法集群节点服务的初始化状态均为Follower节点,并且每一个Follower节点具备成为Leader节点的资格,也就是说可以同时具备存储数据以及集群Leader的特征,其次Follower节点都拥有一个随机等待leader节点发起ping心跳检测的超时时间timeout

- 成为候选节点之后就会更新任期Term并随机设置对应的投票请求超时时间,如果在指定的超时时间内超过半数投票,那么就会晋升成为leader节点,并周期性地向集群从服务节点发起心跳检测以避免Follower节点成为候选节点发起leader投票选择.

- 主观下线,即Raft算法集群服务节点中有一个Follower节点超时没有得到leader节点的心跳检测请求,那么就会为自己进行投票,此时会晋升成为候选节点,又称为单节点的主观下线,即凭借单节点服务设置的超时时间timeout来判断leader服务是否可用.(redis的sentinel集群服务的sdown)

- 客观下线(不属于Raft算法),即集群半数以上的Follower节点没有在超时的时间内得到leader节点的心跳检测请求(或者说集群大多数节点认为leader节点不可用)而重新发起投票选举请求.(redis的sentinel集群服务的odown)

#### 节点之间的通讯

- leader投票选举请求: 通过候选节点发起RPC的投票请求,集群其他节点在指定的timeout内接收到投票请求并按照先来先得得顺序进行投票.
- 日志复制请求: 集群服务的leader节点接收到事务操作执行事务指令并发起RPC请求对事务指令复制到事务操作日志log中并同步到集群其他服务节点
- leader与follower节点心跳维持: leader节点发起RPC并携带任期Term信息的心跳检测,以此来维持集群服务leader节点与follower节点之间的健康检测状态,保证集群服务的可用性.

#### 共识与数据一致性

##### Raft算法架构

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/leader_consensus_opr.jpg)

通过上述可知,客户端client service多个节点发起事务操作提交到集群的leader节点,集群的leader服务节点通过共识模块来输出对应的数据值并记录变更日志log,同时发起日志复制到集群其他服务节点以便于同步变更操作并等待大多数节点的响应之后再持久化到本地的状态机,最后更新之后再返回给客户端响应,而集群服务的其他节点则通过新的日志复制RPC消息抑或是RPC心跳检测进行一致性校验然后更新到对应的状态机上.

##### 共识问题

- 什么是共识

简而言之,就是不同进程p对分别输入一组u的数据,通过相同的程序处理逻辑来保证其输出值最终都是v.即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/leader_consensus.jpg)

- 对于Raft集群存在哪些共识问题

1) leader的选举

当Follower节点发现Raft集群leader节点不可用时就会推荐自己选举成为leader节点并发起投票选举,这个时候Follower节点成为Candidate节点,于是会自增加自己的任期Term并向Raft集群其他服务节点发起RPC通讯的投票请求,如果赢得超过半数投票请求,那么当前Follower节点就晋升成为leader节点.

2) 事务操作

我们知道Raft算法是属于强leader模型,所有的事务请求最终都会落到leader服务节点上,那么在Raft集群服务中只有leader节点的进程输入数据并通过leader节点的共识算法模块输出状态变更的记录值,并复制共识输出的变更记录log同步到Raft集群Follower节点.

##### 数据一致性

- 什么是数据一致性

Raft集群服务对外提供数据的读取始终保证一致性,即对Raft集群之外的服务,简称为客户端服务client service,client service多个节点node向Raft集群服务发起读取请求,那么要保证client service的每个节点node读取到请求数据都是一致的.

- 如何保证一致性

当Raft算法已经选举Leader节点之后,为了保证Raft集群中的数据一致性,Raft算法采取强制的Leader策略,将客户端的写入操作更新到leader节点的日志文件中,并以RPC通讯的方式复制到Raft集群的从服务节点中,从服务节点从操作日志来增量更新数据从而保证leader与follower节点数据的一致性.

- 日志复制异常

当leader节点发起复制日志的RPC请求消息到Raft集群中各个follower节点,如果此时leader节点发生不可用,那么此时会有以下几种情况(前提是leader节点的日志已经写入成功,leader节点恢复之前):

其一follower节点大多数接收到复制日志的请求并执行成功;

其二follower节点只有少数接收到复制日志的请求并执行成功;

其三follower节点没有接收到复杂日志的请求;

不论是上述哪种情况,leader节点发生不可用的时候,必然会重新选举一个具备日志完整性的follower节点作为新的leader节点,如果当前leader节点存在原有的log的记录(1&2的情况),会重新发起复制日志的RPC请求消息到集群服务的各个节点,如果大多数集群服务节点都响应成功那么就会在当前的leader节点将日志log应用到状态机并在下一个RPC日志复制抑或是心跳检测消息通知其他follower节点更新到状态机上;如果响应失败(机器不可用除外,需要更新集群配置,重新RPC日志复制),一般是一致性检查失败,于是会根据一致性流程强制更新数据并最终保证日志数据与leader节点一致;最后是follower节点上并没有先前leader的日志记录,那么对应的数据将会丢失,因为重新选举之后,当leader节点恢复时,日志数据会根据一致性检查被强制更新为与leader节点日志一致,因此需要在客户端服务进行重试与幂等性操作保证数据不丢失.

#### leader任期Term

##### 任期特征

- Raft算法中任期Term包含时间段以及编号,并且会影响leader选举和请求的处理

- Raft算法中leader节点的任期编号具备单调性,且为正整数递增

##### 任期的更新

- 当Raft集群服务中存在follower节点发现leader节点不可用的时候,就会成为candidate节点,并为当前节点的任期自增1,然后将自己的任期编号Term以参数的形式携带在投票请求的RPC中向其他服务节点发起新一轮任期leader节点的投票选举.

##### 任期与投票选举

- Follower节点接收到Candidate节点的任期投票,若当前的follower节点任期比投票选举的任期小且是在超时的时间范围内,那么当前Follower节点就会为发起投票的请求节点进行选举并给予响应
- Follower节点接收到Candidate节点的任期投票,若当前的follower节点任期比投票选举的任期大,那么当前的Follower节点将会拒绝投票请求
- 如果Raft服务的leader节点由于发生不可用,而同时在不可用期间Raft集群已经通过选举产生新的leader节点服务,此时当原先的leader服务节点恢复健康状态时,由于会接收到leader节点的心跳检测以及任期编号等信息,发现当前的任期编号比接收到任期编号小,那么这时候原先的leader节点就会更新自己的任期并成为集群服务的follower节点(单节点变更).

#### 选举规则

##### 周期检测

leader节点会周期性向follower节点发起心跳检测(携带投票选举的Term),以避免follower节点成为候选节点进行投票选举.

##### 节点具备成为leader权利

Raft集群服务的follower节点在指定的时间内没有接收到leader节点的心跳检测消息,那么此时会认为leader节点不可用,会推荐自己成为候选节点并自增任期编号,同时发起leader选举.

##### 大多数原则

follower节点成为candidate节点发起的leader选举,如果获得集群服务大多数服务节点的投票响应,那么当前follower节点就会成为leader节点

##### 具备持续性

Raft集群通过每次选举产生新的leader,同时会对应新的Term,每一轮新Term都具备持续性,也就是说在一个任期Term内,只要是leader节点是可用的,那么就不会发生新的一轮选举,可用性是通过周期性检测来保证.

##### 一次投票&先来服务

在一次选举中,每一个服务节点最多会对一个任期编号进行投票,并且按照先来先服务原则进行投票.即在发生选举过程中,可能存在两个或者多个候选节点向集群服务发起投票,Raft集群为了避免同时发生投票的碰撞,采取随机超时时间的心跳检测机制来进行投票,这个时候对于其中一个服务节点A,先接收到服务节点B且携带任期编号为v的投票请求并给予投票,此时服务节点C也携带任期编号为v投票请求到服务节点A,由于A节点先投票给B,于是返回给节点C当前已经没有选票,投票失败的响应.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/leader_vote_request_short.jpg)

##### 满足完整性日志

日志完整性高的follower节点拒绝投票给日志完整性低的候选节点.假设现在Raft集群服务提交的日志log如下(其中包含日志索引logIndex,实体log包含任期编号term以及变更的指令):

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/raft02/leader_vote__replicated_log.jpg)

现在Raft集群中各个节点以及日志复制情况如上,那么此时如果leader节点发生不可用,followers都具备成为leader节点,但是如果是一个日志索引下标`logIndex=5`的候选节点向一个日志索引下标为`logIndex=8`的follower节点发起投票请求,这个时候follower节点将会被拒绝,主要原因是后者的`logIndex`更大,相对地,其日志完整性更高(可类比于kafka的ISR副本机制)

#### Raft算法中的timeout含义

- follower节点等待leader节点发起的心跳检测的随机超时时间
- 每次发起新的一轮选举时,候选节点发起投票请求并等待投票请求响应的随机超时时间.



