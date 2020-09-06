---
title: 分布式理论基础
category: 分布式架构设计
date: 2020-06-08 15:16:16
tags: 分布式架构设计
---



<!-- more -->

### 分布式理论基础

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/title.jpg)

在这一篇中主要讲述分布式基础理论知识,其中包含CAP定理,ACID以,BASE理论以及一致性协议分析.有了CAP定理的基础,能够帮助我们在根据业务特点进行分区容错一致性模型设计中提供解决问题的方向以及架构设计方案的设计与落地实现.同时需要区分数据库ACID的AC与我们的分布式AC存在的联系与差异,其次,在分布式网络中,为避免节点故障抑或是网络延迟等问题导致系统服务出现大量的不可用问题,那么对于BASE理论实现的技术方案有哪些.最后讲述分布式系统中数据的一致性问题.



#### CAP定理

##### 网络分区容错(Partition tolerance)

- 网络连通性

> 服务节点之间的网络通信正常

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/network_connected.jpg)

在上述可以看到,服务A集群与冗余服务A1与A2节点形成一个对外闭环的集群,同理服务B也构成一个闭环集群.此时发起一个请求操作,需要通过服务A与服务B进行协作,在服务B节点正常运作情况下,这个时候的分布式网络是处于连通状态,服务A与服务B之间能够进行正常网络通信完成数据协作.

- 网络分区

> 服务节点之间的网络通信发生中断或者是延迟响应出现短时间的中断.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/network_partition.jpg)

在上述的图中,一个请求操作需要通过服务节点A与服务节点B完成协作,但是如果服务B没有做集群部署,此时服务节点B发生故障或者是网络延迟,那么这个时候服务节点A与服务节点B之间将无法进行通信,此时服务A将与服务B失去联系,即服务节点之间发生网络通信中断而产生网络分区,此时服务A节点与服务B节点被划分为彼此独立互不相连的节点服务,即如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/network_parition2.jpg)

- 分区容错

> 服务节点之间的网络通信能够通过容错处理手段来保证服务节点之间互通.

为了保证上述的服务A与服务B能够正常通信,即使服务B节点发生故障,也可以通过集群服务B的其他节点B1或者是B2来继续为服务A提供服务,以保证服务A节点能够正常完成请求操作的处理.而对于这种情况称为分区容错,在分布式环境中,节点故障与网络延迟是无法避免的,因此为了保证我们的分布式系统服务能够正常运作,那么在进行架构设计的时候就需要做到满足CAP定理中的分区容错(Partition Tolerance)

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/network_tolenrace.jpg)

##### 一致性(Consistency)

> 客户端每次发起的读取请求,不论是访问分布式集群中的哪个节点,都能够读取到最新的一份写入数据.

分布式一致性的理解如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/consistency.jpg)

在上述发起一个事务操作请求的时候,集群的服务A接收事务请求操作`v=v1`的处理,此时服务A需要并未将数据同步到冗余服务节点A1以及A2,而此时向集群A服务发起读取请求操作,此时由于负责集群服务的均衡处理将请求转发到服务A1,但是此时存储数据v的服务节点A1并未从服务A同步到最新的数据v,从而导致发起读请求操作的客户端读取到数据v结果不一致.这个时候就需要要求服务节点A接收到数据状态变更也需要向集群服务中的冗余服务节点发起数据同步操作,保证其他冗余服务节点与服务节点A数据的一致性.也就是说分布式系统的一致性需要保证在有状态的服务节点下数据的读取是最新写入的.即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/consistency2.jpg)

而对于数据在分布式环境进行同步操作的时候,会存在以下两个核心问题:

- 节点发生故障如何保证读取的数据是一致性的,即存在节点故障问题
- 节点之间需要通过网络进行数据同步,即存在网络延迟问题

##### 可用性(Availability)

> 要实现可用性,需要牺牲数据的一致性,也就是无法保证数据看到都是最新的一份.

我们先来思考分布式系统中哪些地方会存在不可用的情况:

- 单节点故障导致服务的不可用.
- 服务与服务之间调用依赖,被调用方服务发生故障,调用方服务也会发生故障导致不可用.
- 当发起一个事务请求操作通过服务A来调用服务B的时候,此时服务节点B需要同步数据到其他冗余服务节点B1以及B2,如果此时有读取请求的操作来访问服务节点A,为了保证看到的数据是最新的,这个时候由于B1或者B2节点未完成数据同步但是要保证数据读取最新这个时候会导致服务A出现短暂的不可用,或者称为同步等待数据返回.

通过上述我们看到,节点故障我们可以通过分区容错解决,也就说在分布式系统设计中分区容错与可用性是可以并存的,而服务调用方发生故障导致不可用,也可以认为是分区容错的一种手段,只不过是在应用层次通过过载保护或者是降级进行控制;然而对于后者,我们看到要保证数据的一致性,读取请求需要进行同步等待服务节点完成数据的同步操作之后再进行返回,那么在这个等待的时间窗口内,对于客户端而言,读取请求资源的操作出现短时间内的不可用.

由此可知,在我们的分布式环境下,为了保证节点与节点之间的正常通信,即保证分布式系统服务节点的网络连通性,我们必须要满足分区容错这一特性,但是对于RDBMS系统而言,比如MySQL或者是Oracle的关系型数据库,这类传统的关系型数据库由于设计的初衷都不是考虑分布式系统设计,而是保证一致性以及系统可用,尤其是事务操作的一致性,读写实时性抑或是复杂SQL的多表查询,放到分布式系统中实现要更为复杂.于是才有了分布式数据库,比如MongoDB满足CP原则,而CouchDB满足AP原则.同时对于关系型数据库,比如MySQL,我们会通过冗余服务复制来满足分布式系统设计的AP抑或是通过CP + AP的方式实现分库分表的集群.

CAP定理应用如下图所示:



![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/cap.png)

因此,当我们在思考分布式系统设计的时候,需要基于CAP定理从业务数据层面去思考我们的架构设计方案.

#### ACID理论

##### 事务的ACID

- 原子性(Atomicity)

是指一个数据库操作的事务不可分割的工作单位,要么操作成功,要么操作失败.

- 一致性(Consistency)

一致性是指事务开启之前以及开启之后数据库的完整性约束没有被破坏.

- 隔离性(Isolation)

多个事务并发访问的时候,两个事务之间彼此相互独立并且互不影响,一个事务不应该影响其他事务运行效果.

- 持久化(Durability)

意味着在事务完成以后，该事务对数据库所作的更改便持久的保存在数据库之中，并不会被回滚

- 分布式系统如何保证事务的ACID

当我们了解到一个操作具备ACID特性的时候,我们基本上可以认为完成了一个事务操作,而在分布式系统中,实现一个满足ACID的请求操作需要考虑到网络以及节点故障问题等问题,于是就需要通过分布式事务协议来保证完成一个具备ACID特性的请求操作.

##### 2PC事务协议

- 工作原理

分为投票阶段以及提交阶段,集群服务节点扮演角色有协调者(Coordinator)以及事务的参与者(Cohort).

其工作流程如下:

> 投票阶段

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/2pc_vote01.jpg)

首先向集群服务节点发起一个事务请求操作,即`v=v1`,这个时候集群服务选举一个协调者来负责接收事务请求的操作,并由协调者统一指挥相应的事务操作.这个时候接收到事务请求之后协调者将向集群服务各个参与事务的节点发起`v=v1`操作请求,询问是否允许进行操作.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/2pc_vote02.jpg)

这个时候服务集群参与者的节点接收到协调者的操作请求,并在执行当前的操作请求的时候进行undo以及redo的日志记录,通过undo日志实现回滚,redo日志实现持久化,最后将响应结果返回给协调者.

> 提交阶段 - 成功提交

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/2pc_commit01.jpg)

协调者接收到参与者节点进行事务执行操作准备的响应,根据参与者节点的响应结果,为了保证事务的ACID特性,如果所有的参与者节点都允许进行事务`v=v1`的请求操作时,那么这个时候协调者就会向各个参与者节点服务发起commit的请求并等待各个参与者节点事务操作结果的响应.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/2pc_commit02.jpg)

参与者接收到协调者的提交请求之后,这个时候每个参与者节点就可以正常完成单点机器的事务操作,即满足事务ACID的特性,完成事务操作之后每个参与者节点都会向协调者发起事务操作结果的响应.这个时候协调者接收到事务操作响应的结果并向客户端给予事务操作成功的响应.

> 提交阶段 - 失败提交

- 参与者3因网络原因导致超时未响应
- 参与者3拒绝`v=v1`的请求操作

对于实现分布式事务的ACID,要保证数据的强一致性,因此只要其中有一个参与者节点在进行事务投票阶段发生上述问题,这个时候协调者会视为abort,则协调者下一步将会发起abort的回滚操作,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/2pc_rollback01.jpg)

协调者根据参与者节点给予的响应结果作出abort的回滚策略,此时向各个参与者节点发起abort的回滚操作,即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/2pc_rollback01-1.jpg)

这个时候参与者节点接收到事务回滚操作,将原先保存的undo数据进行恢复,然后将回滚的操作结果响应给协调者,协调者接收到所有残月者节点回滚的响应结果,向客户端发起事务操作失败的响应结果.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/2pc_rollback02.jpg)

> 2PC产生阻塞

- 协调者发起请求的时候会等待参与者的响应结果
- 参与者接收到协调者发起的请求并给予响应此时也处于等待状态,等待提交redo或者回滚undo
- 此时参与者接收到应答指令执行相应的操作之后,协调者会继续处于等待参与者处理的结果响应
- 同时,既然是分布式系统,必然离不开网络服务之间的通讯以及机器节点的故障问题,因而如果协调者产生不可用,此时所有的参与者将会一直处于阻塞状态

> 2PC产生数据不一致

- 当协调者向参与者发起提交或者是回滚操作的时候,其中有一个参与者服务节点产生不可用的情况,这个时候参与者节点将无法接收到提交或者回滚信息,那么这个时候就会产生数据不一致.

> 2PC的整体流程总结

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/2pc_flow.jpg)

##### 3PC事务协议

在实际应用场景中,3PC的使用场景并不多,大部分是基于2pc的实现来完成分布式事务,甚至是为了保证数据的强一致性会采取TCC的事务协议来完成,对于3PC现简单阐述如下.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/3pc_flow.jpg)

> 3PC提交过程说明

- 协调者服务节点发起事务请求给到参与者节点询问是否允许事务请求提交
- 参与者节点接收到事务请求提交之后将当前数据记录到undo日志中,以便于后续请求的超时或者是无响应进行事务的回滚,这个时候记录undo日志之后返回给协调者告知请求提交的结果
- 协调者此时如果没有接收到参与者的请求提交响应回复抑或是拒绝请求提交将终止当前操作不再继续下一步;如果收到允许请求提交操作,则会向参与者节点发起预提交请求到参与者节点中

- 参与者节点如果没有接收到预提交的请求抑或是网络延迟中断,那么就会将上次的undo日志进行数据回复并丢弃当前的事务操作;如果能正确接收到预提交的请求操作,那么这个时候会将更新的数据记录到redo日志中,以便于后续进行持久化.
- 对于协调者而言,如果正确接收到预提交请求的ACK响应,那么这个时候将会执行请求提交到参与者节点;如果没有接收到ACK的响应抑或是网络超时问题,将会直接丢弃当前的事务操作.
- 而对于第三阶段的确认提交,参与者节点不论是网络中断还是超时抑或是正常接收到doCommit的提交请求操作,那么这个时候都会执行redo日志进行持久化操作并响应给协调者节点告知当前事务操作完成并且已经提交.

> 3PC存在的数据不一致问题

通过上述的流程分析,我们很容易得出,3PC的事务协议提交阶段存在的数据不一致主要体现在:

在协调者进行预提交阶段,协调者服务节点向参与者节点发起预提交请求,其中向参与者集群服务节点已成功发起预提交请求,但是这个时候协调者服务节点发生故障导致不可用,那么就会导致参与者节点服务部分没有收到预提交的请求,这个时候这部分参与者节点由于没有收到preCommit请求而进行undo日志的回滚.

##### TCC事务协议

> TCC定义

TCC,即Try(预留资源)-Confirm(确认操作)-Cancel(撤销操作),其核心流程如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/tcc_flow.jpg)

> TCC原理分析

- 在主服务节点注册一个确认以及撤销的操作,然后在主服务节点开启事务执行资源的锁定并预留可用的资源,然后提交消息分发到各个子服务节点中
- 子服务节点接收到主服务节点的事务操作消息,这个时候子服务也将自己对应的资源进行锁定,同时也将可用的资源预留出来以便于提升并发操作的性能,然后向主服务节点推送操作结果的消息为Success或者是Fail
- 主服务节点根据各个子服务节点操作结果的反馈,如果存在一个子服务节点的Fail反馈,那么就执行撤销回滚操作并将锁定的资源回收到资源池中来保证整个分布式事务的ACID,如果返回都是Success,那么就执行确认操作释放锁定资源.最后将操作结果以消息的形式分发到各个子服务节点上
- 子服务节点接收到主服务节点的事务确认或者回滚消息,也将执行相应的确认或者是回滚操作,如果是回滚操作,那么同样也将锁定的资源回收到资源池中,如果是执行确认操作,那么就释放资源.

可以看到TCC是建立在业务基础上来保证分散的服务节点的事务一致性,实现相对比2PC更为复杂些.

> TCC各个服务节点运作

以一个送礼下单为例展开,现有一个订单服务,礼物服务以及用户积分服务.如下图所示:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/tcc_tx.jpg)

订单服务可以认为是一个主服务,主要负责订单状态的更新,而礼物服务以及积分服务作为一个子服务,负责更新礼物库存以及积分信息.于是对于一个TCC的操作流程可以分解为如下几个步骤:

- 订单服务接收到送礼下单的请求,那么这个时候订单服务将会创建一个订单并设置当前的订单状态为PENDING以便于提升并发性能查询而并非出于创建订单的阻塞等待状态,同时分发消息到礼物服务以及积分服务.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/tcc_s1.jpg)

- 礼物服务以及积分服务分别收到订单服务发出的事务操作请求命令,这个时候礼物服务以及积分服务分别记录redo以及undo日志,redo可以理解为可用资源提供外界查询以实现并发访问的性能,undo锁定使用的资源以便于回滚,这里的redo可以是对应的数据库变更数据的记录,undo日志记录变更的数据以及当前为锁定状态.然后将操作结果返回给订单服务节点

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/tcc_s2.jpg)

- 订单服务节点更新返回的子服务消息结果,如果都是操作成功,那么这个时候就执行确认操作,订单服务将订单状态更新为FINISHED,并将消息分发到各个服务节点,如果存在一个节点操作失败,则执行更新状态为REJECTED,并下发回滚操作的消息.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/base/tcc_s3.jpg)

基于上述的认知之后,我们要实现一个基于TCC的分布式事务,如果在没有使用开源产品的场景下,最简单的方式是可以基于MQ的方式来实现可靠消息的生产与消费从而实现我们的分布式事务协议TCC.

#### BASE理论

> BASE理论定义

BASE是Basically Available(基本可用),Soft Sate(软状态)以及Eventual Consistency(最终一致性)三个短语的缩写.

- 基本可用:可能是部分功能不可用或者是因网络原因响应时间延迟导致延迟的时间段内不可用
- 软状态:集群服务节点在进行数据同步期间数据的过渡状态.比如一个事务操作前为v1,进行事务操作后为v2,节点服务之间数据由v1转变为v2的状态为过渡状态.
- 最终一致性:经过系统内部服务之间的运行协作,最终服务节点表现在业务上抑或是集群leader选举或者是数据上等最终保持一致性

> 数据一致性模型

- 线性一致性(强一致性): 不论是在集群服务哪个节点,看到的数据最终都是一致性.
- 顺序一致性(弱一致性): 集群服务节点的数据变动以及操作顺序保持一致.
- 最终一致性(弱一致性): 集群的所有服务节点最终都会呈现数据的一致性

> 基本可用策略

- 流量削峰:应对高并发量冲击,可以从以下几个方面来考虑问题,一个是错开时间段;一个是限流+错开时间段,比如预约抢购;一个是在边缘节点抑或是负责均衡器做流控;一个是在应用程序中基于MQ消息队列来做缓冲;一个是在底层数据库表采取分区表,不够再进行分库分表设计.
- 延迟响应: 比如在进行购票的时候,我们会提交购票请求然后需要进入系统的排队等候,等待请求被处理.
- 降级: 一个是当被调用的服务依赖不可用时,为了服务调用者节点时可用的,需要加入异常处理的降级操作;又或者是在负载均衡器中发现后端服务不可用的时候可以采取降级返回错误提示等
- 过载保护: 在抢购或者是秒杀系统中,可以考虑在nginx中进行限流然后将超出的流量直接放回抢购失败;抑或是在应用服务中的线程池中将任务添加到阻塞队列中,如果队列满了可以考虑直接丢弃任务策略.

  #### 参考文档

```reStructuredText
## 有状态与无状态服务
https://nordicapis.com/defining-stateful-vs-stateless-web-services/

## CAP定理
https://dzone.com/articles/understanding-the-cap-theorem
https://dzone.com/articles/quick-notes-what-cap-theorem
```

最后关于一致性的详细分析留到下一篇文章中.













