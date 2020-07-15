---
title: paxos算法
category: 分布式协议与算法
date: 2020-07-04 03:34:31
tags: 分布式协议与算法
---

<!-- more -->

### Paxos算法

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/title.jpg)

在讲述分布式的一致性之前,先对基本的分布式协议算法有一个初步的认知,其实再分析分布式环境常见问题,最后再回到一致性问题来进行阐述.本文主要讲述分布式共识算法之Paxos算法,分别从朴素的算法说明,流程原理以及最终实现的原理逐一展开阐述说明.最后说明一点,在这里不会去花时间证明Paxos算法,有兴趣可以查看Lamport的Paxos论文证明实现.



#### 朴素的Paxos算法简述

##### 共识问题描述

假设现在有三个服务节点能够进行提案操作,那么Paxos的共识算法就是确保上述服务节点之一的提案数据值能够被选中,也就是说达成共识的安全要求需满足以下三个条件:

- 只有被提案的数据值才具备被选中的资格
- 最终仅有一个提案值能够被选中
- 除非提案的数据值最终被选中，否则进程将无法学习到该提案的数据值

考虑三个服务节点都是属于独立部署且需要通过网络进行异步消息通讯,此时建立起一个异步通信且非拜占庭将军问题模型如下:

- 服务节点以任意的运作速度运行,除非出现宕机或者是重启,如果在服务节点发生故障之前选择一个提案数据值能够持久化到可靠的存储系统中,那么当服务节点重启恢复的时候就能够将故障之前的提案数据值进行恢复,否则不存在解决方案.
- 服务节点之间可以花费任意长的时间进行通信,可以复制,也可以丢失但是不能被破坏.

##### Paxos参与的角色

- 提案者（proposers）:负责发起一个提案值的写入操作,其中包含有序自增长的提案编号以及提案值数据值v.
- 接收者（acceptors）:负责接收提案者的写入操作,通过共识算法来实现选择最终的提案数据值v,并将最终选中的提案值写入到学习者,写入学习者节点后的数据将不能发生变更.
- 学习者（learner）:存储接收者达成共识的提案数据值,仅作为最终数据的存储并为其他学习者提供学习最新选中的数据值v.

##### 如何确定值

> 准备阶段

基于上述的异步非拜占庭模型基础上,在保证服务节点正常运作以及消息不丢失的情况,Paxos算法确定一个值需要满足以下条件:

- 接受者必须接收第一个发起提议的请求.
- 如果选择了一个值为v的提议请求,那么每一个被选择的更高编号的提议者都存在提案值为v,即如下:

```reStructuredText
1) 如果选择一个值为v的提议请求，则被任意一个接收者所接收的每一个较高编号的提议者都存在提议值v.
2) 如果选择一个值为v的提议请求，那么任何提议者发出的每一个高编号的提案都存在提议值为v.
3) 对于任意的数据v和n,这个时候向接收者服务集群现在发起一个提议数据(携带编号以及数据值)[n,v],这个时候集群中的接收者服务节点将会出现两种情况:一是丢弃编号小于n的提议请求;二是接收的数据值v为小于n的最高编号的提议服务节点.
```

基于上述的分析,我们可以梳理并总结提议者向接收者发起提议请求(n,v)之后响应返回的算法如下:

- 接收者接收到提议请求(n,v)并承诺不再接收小于n的编号提议请求,如果之前没有提议过将直接返回;如果已经提议过但还没确定那么直接丢弃小于编号n的提议请求;
- 接收者接收到提议请求(n,v)时,发现已经选择并确定一个数据值为v',那么这个时候将选择v'小于n的最高编号n'作为新的响应数据并携带确定的数据v'到提议者.

上述的操作阶段称为prepare准备阶段.

> 接收阶段

基于上述准备阶段算法,此时接收阶段算法处理如下:

- 对于提议者而言,其接收到准备阶段响应的请求,这个时候会有两种情况:一个是返回的响应为准备阶段发起的编号n表示当前携带提议编号n以及v可以正常发起一个接收请求到接收服务节点;一个是返回的响应为接收服务节点携带小于编号的最高编号n'以及对应确定的值v',这个是提议者将当前最大的编号n以及对应确定的值v'向接收服务节点发起接收请求.
- 对于接收服务节点而言,如果还没有响应一个编号大于n的提议请求,那么将会接收[n,v]或者是[n, v']的请求提议并将数据持久化到学习者服务节点中.

##### 对齐(learn)策略

首先,我们先考虑以下几个方面:

- 接收者服务节点面临多个提议服务请求操作,那么如何保证确定值的数据安全.(一旦确定并写入,后续将不能进行更改)
- 对于提议者而言,如何保证提议者节点发起的提议编号是大于确定值的最高编号.

通过上述问题,我们可以反推思考,当接收服务节点确定值之后将数据更新到leaner节点,那么后续的提议操作都可以通过leaner节点获取最终确定值以及对应的最高编号以保证接收服务节点;其次对于提议者服务节点而言,当向接收服务节点发起提议请求的时候,提议者也可以通过learner服务节点获取确定值的最高编号再次发起提议请求操作,这样避免多次无效的请求操作被丢弃.

#### Baisc-Paxos算法

假设服务集群中有三个提议者节点clientA,clientB以及clientC,同时对应着接收者节点acceptorA,acceptorB以及acceptorC,以及leaner服务节点leaner

##### 准备阶段

主要是描述的是多节点之间如何就某个值（提案 Value）达成共识,换而言之就是单个数据值在集群服务节点中达成共识.

> 准备阶段 - 发起提议请求

提案者节点A向接收节点A,B,C分别发起编号为[1, ]的准备请求,如下所示:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/basic01.jpg)

- 提议者是如何确定当前的编号值?

根据上述确定值的算法方案,提议者发起的编号为整数且具备单调递增的特性,于是我们需要将每个提议者发起的提案编号进行记录并存储起来,于是需要有一张本地变量表来存储提议者每次发起提议的编号,当重新发起提议的时候将按照单调性自增长产生对应的提议编号.

- 不同的提议者节点如何避免提议编号的冲突?

在上述提议者节点A发起提议编号为[1, ]准备请求,那么如果提议者B或者C节点也发起提议编号为[1, ]的时候,要如何避免编号冲突?首先,在分析提议编号冲突之前,我们先分析下如果提议者节点B以及C节点同时发起提议编号均为[1, ]准备请求,那么这个时候对于接收者的服务节点,我们根据上述确定一个值的策略,这个时候三个节点提议相同的编号最有可能给予的响应都是成功,如果响应为成功,那么会出现三种情况,一种是都提议相同的最终值v,一种至少有半数节点提议相同值v,那么可以通过半数投票进行确定,一种是提议节点发起数据值v的请求写入是混乱无序的,这个时候对于接收者服务节点是无法进行选择的.对于Paxos共识算法而言,为了避免上述的问题,将采取提议编号自增长的策略,那么提议者节点从哪里获取当前的编号为整个集群服务更高的编号呢?在Paxos算法中,存在这样的一个learn角色,我们可以将我们每个提议者节点获取提议编号的时候更新到我们的learner节点服务来维护我们集群服务提议者编号数据,保持提议者编号的单调性,这样每个提议者节点就可以从learn节点获取一份更高的编号发起准备请求.即

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/basic02.jpg)

这个时候提议者节点A以及B节点分别从learner学习者服务节点获取更高的编号并记录下来,然后向接收者服务节点分别发起编号为[1,]以及[2,]的准备请求.

> 准备阶段 - 接收响应请求

接收者服务节点A,B,C分别接收到提议者A以及B节点的服务请求,这个时候接收者A以及B先后接收到提议者A,B的准备请求,也就是说acceptorA以及acceptorB先接收到准备请求携带的提议编号[1, ],根据上述确定值的策略,接收者节点A以及B此时并没有确认提案,于是将返回一个“尚无其他提议”请求的响应给到提议者节点A以及B.同时承诺只能接收大于编号为1的提案请求.即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/basic03.jpg)

这个时候提议者节点A接收到acceptorA以及acceptorB节点尚无其他提议的响应,于是准备向接收者服务节点发起确认提交的请求.同样地,对于提议者节点B发起准备请求[2,]的时候,也接收者服务A以及B节点并没有确认提案,于是也向提议者A以及B节点发起尚无提案的响应请求,同时承诺只能接收大于编号为2的提案请求.即

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/basic04.jpg)

这个时候接收者节点A以及B考虑到分布式环境存在网络原因,于是对于提议者节点B发起准备请求阶段,导致接收者节点C先接收到提议者B节点的准备请求,于是对于接收者节点C而言是先接收到提议者B的准备请求[2, ],于是对于提议者A发起的准备请求[1, ]将会丢弃并承诺只能接收大于编号为2的提案请求.此时将会向提议者节点B给予响应,而对于提议者节点A的准备请求将直接丢弃并不做响应即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/basic05.jpg)

##### 接收阶段

> 接收阶段 - 提议者作出决策

对于提议者节点,将按照最大的提议编号作为向接收者服务节点A,B以及C发起数据值确认接收的请求.由于在上述的提议者节点之前准备阶段接收到“尚无其他提案”的请求响应,于是对于提议者节点A将会向接收者服务节点A,B,C节点发起[1, v1]的接收请求,而对于提议者节点B将会向接收者服务节点发起[2, v2]的接收请求,即

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/basic06.jpg)

> 接收阶段 - 接收者接收提案

根据上述选择值的策略,接收者服务节点由于在准备阶段已经承诺不再接收小于当前最高编号的提案,这个时候对于接收者服务节点A,B而言,其在准备阶段已经受理并承诺不再接收小于编号为2的提议者,于是这个时候对于接收请求[1, v1]将会直接丢弃而接收请求[2, v2].即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/basic07.jpg)

这个时候接收者将接收确定值为v2,并将最终提案值v2存储到所有的学习者learner节点服务中.

##### 问题补充

当集群服务节点这个时候已经确定提案值v2并将其写入到学习者learner节点中,那么假设这个存在提议者节点C节点,这个时候再发起一个提议的请求操作[7, v7],如下所述:

- 准备阶段

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/basic08.jpg)

对于提议者服务节点而言,提议者节点从学习者节点learner获取当前最大的提案编号,假设此时为7,这个时候分别向接收者服务节点发起准备请求[7, ];

对于接收者服务节点而言,接收者服务节点接收到[7, ]的准备请求,此前接收者服务节点已经确认提案委[2, v2],此时7>2,但是由于已经存在确认提案值[2, v2],于是向提议者服务节点发起[2, v2]的请求响应.即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/basic09.jpg)

- 接收阶段

这个时候提议者接收到准备阶段响应的请求数据[2, v2],对于提议者节点而言此时知道当前集群服务节点已经存在确定提案值v2,于是选择最大的编号max(2, 7),并丢弃修改的提案值v7,然后向接收者服务节点发起接收请求[7, v2],即

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/basic10.jpg)

而对于接收者服务节点而言,此时会接收当前的请求[7, v2]并将最新的编号数据更新到学习者learner节点中,此时完成了提议者节点C的提议请求操作.

#### Multi-Paxos思想实现

主要是描述的是执行多个Basic Paxos 实例，就一系列值达成共识.换而言之就是在集群服务中一系列的数据值写入达成共识问题.

##### 基于leader的选举策略

实现一系列数据值的修改来达成共识,可以利用leader节点设计思路来实现Multi-Paxos思想,即在集群服务节点中选举一个主节点作为leader节点,负责接收客户端的读写操作,也就是说所有的客户端服务向leader服务节点发起写入操作的时候由leader服务节点将写入操作直接转发到接收者服务节点,而客户端发起读取操作的时候,直接由leader服务节点读取本地数据,即从宏观上看的读写流程如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/leader01.jpg)

通过上述的读写流程,我们可以思考以下几个问题:

- 提议者为什么在这里可以只需要发起一次提议的写入请求就可以达到共识,Multi-Paxos算法是如何达到共识?

首先,在上述的分布式集群服务中,对于接收者服务节点而言,它主要承担着决策主导的作用,也就是不论是哪个客户端发起的写入请求到集群服务中,都将会把写入请求转发到leader节点负责执行写入以及分发写入到acceptor节点来达到共识,我们可以对比在Basic-Paxos算法中,之所以需要有prepare阶段主要目的是为了解决多个提议者同时发起提议以及提议冲突问题.而leader节点正如前文所述负责将执行读写操作,不存在多个提议者服务,减少提议者发起请求的冲突,提升性能(因为冲突需要通过分布式加锁解决).

- 提议者即然可以发起一次写入操作请求,那么写入的操作请求中是否还需要携带提议者编号呢?

提议者是发起写入操作的时候是需要携带提议编号的.有两个方面:一个是假设没有携带提议者编号,从我们应用服务而言,我们无法感知到数据变化的版本,比如现有接收者服务节点存储的数据为v1,之后提议者B发起一个写入提议者的请求为v2,同时提议者C此时又发起一个写入请求v1,最终对于业务服务而言是看不到数据变更的,但是我们知道这份数据是发生变化的,如果存储的数据为栈或者链表等结构存储,那么有可能造成数据丢失,即“ABA”问题;另一个是可以理解为写入操作的“决策”,即这个时候如果有提议者A发起一个写入请求[4, 5]以及一个提议者B发起一个提议请求为[9, 10],假设此时接收者服务节点已经存储一份本地数据表[8, 12]的数据,那么对于提议者A此时由于编号小于现有存储的数据,于是将会把提议者A的写入操作丢弃而接收提议者B的写入操作以保证数据的正确性.因为在Basic-Paxos算法中提议编号是具备单调性.在这里Multi-Paxos算法也是建立在Basic-Paxos算法的基础上引入leader节点的思想来解决多值共识问题.

- 引入leader节点存在的问题

根据上述分析,我们很容易想到引入leader节点的时候,leader节点承担集群服务的写入操作,写入分发以及写入的响应操作,因此存在淡点故障以及写入的性能瓶颈.

- 如何让提议者具备高可用

在paxos算法中,我们知道存在三个角色,即提议者,学习者以及接收者,那么让提议者服务具备高可用特性,相信第一反应就是做成集群冗余,那么对于集群冗余而言,我们就可以考虑到故障转移以及数据恢复问题,根据上述的Paxos算法参与的角色,我们可以将提议者发起的请求记录在存储服务,其他冗余服务可以主节点服务的存储服务中学习并获取数据进行更新.即如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/leader02.jpg)

从上面可以看到当前的leader节点服务扮演着提议者以及学习者的角色来实现一个基于Multi-Paxos的leader策略的高可用方案.

- 如何让提议者提升性能呢?

通过上述的分析,我们知道引入leader节点来执行写入操作,仍然存在单点写入的性能瓶颈,那么在分布式架构设计中要如何提升写入性能呢?单从架构设计上来看,分布式设计最重要的一个设计思想就是“分而治之”思想,同理,在这里我们要提升性能就是将写入的请求进行分流到不同的节点中,此时采取的方案可以类似于redis-cluster的集群方案,同时如果需要保证高可用的话,就可以需要分别针对单点的写入进行高可用的部署,即满足上述的高可用方案.因此,对于一个分散的写入集群可以如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/learn03.jpg)

这里的方案和上述高可用的方案存在一些区别,主要如下:

- 在上述的图示中,两个leader节点都具备发起提议写入操作,但是既然存在多个提议者允许写入操作,那么就会存在提议编号冲突问题,对于冲突问题可以考虑不相交的编号集合策略来选择对应的最高编号,同时保证单调性,在上述中两个节点中简单采用奇偶正整数来划分编号并保证编号的单调性,这样就能够避免提议编号的冲突.
- 基于上述的策略,我们考虑这样的一个情况,那就是假如leader1节点发起[3, 7]的提议写入请求,leader2发起一个[4,9]的提议请求,而此时对于接收者服务节点存储的数据的最大编号为2,如果leader1节点由于网络原因导致leader2先将[4.9]提议者写入,那么当leader1发起的[3.7]请求到接收者的时候由于编号较小将会丢弃请求,这样会导致数据丢失.对于此类情况可以考虑重新发起提议请求选择更大的编号来重新执行写入操作.

在实际分布式集群中,leader扮演着proposer+learner角色,而follower节点扮演着acceptor+learner角色,即leader与follower节点简要如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/learn04.jpg)

##### 基于状态机策略

通过上述的分析,对于多值数据的确定,可以考虑引入leader以及leader节点分散写入的方式来提升性能,但是对于实现有序的多值确定要如何保证呢?这个时候在分布式系统架构中会引入一个“状态机”设计,根据状态机理论,如果初始状态一致,输入的状态一致,那么最终输出的结果也将具备一致性,从而达到共识的目的,利用状态机来存储每一次提案编号以及确定的值,即实例,因此对于分布式系统架构设计引入状态机的目的就是要保证有序且一致性.现在来看下状态机服务的演变过程如下:

- 初始状态的服务,状态机服务为单点服务,按照先来后到的顺序接收多个业务客户端发起的command命令,通过状态机服务来将客户端发起的command记录并存储起来,并记录对应command的状态机序号(保持单调性),然后由server负责处理command并将输出结果存储到状态机中,产生状态转移,最后由状态机将数据输出到客户端.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/state01.jpg)

- 考虑到状态机服务的单点故障问题,于是引入状态机集群服务,即如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/state02.jpg)

- 这个时候我们要思考如何保证状态机服务集群对客户端发起的command具备有序性?假设现在server1接收到command1,server2接收到command2,server3同时接收到command3以及command4,而此时command1-command4是具备按照先后次序发起请求,那么这个时候状态的转移也应当保证有序,即对应command1-command4的输出结果.那么对于server1-server3就需要达到共识目的,引入上述的Paxos的共识算法,同时每个server节点具备三个角色(proposer/acceptor/learner),也就是说每个server都能通过自身发起提议请求,同时确定一个选中的值然后传递到状态机作为输入,状态机记录提议编号以及确定值,每个server服务通过learner角色来学习获得对应的提议编号以及选择的确定值以保证服务server发起的提议编号为当前状态机最高编号从而避免冲突.即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/state03.jpg)

server服务之间通过learner学习并获取对应的instance实例,这里的实例可以理解为每次发起提议请求产生确定值的一次操作.server具备leaner角色,于是每个server之间可以通过学习获取到最新的实例信息.即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/state04.jpg)

由于第一次command1落到服务集群的server1中,于是在server1中将当前确定记录为一个instance并存储到存储层中,同样command2落到集群服务server2中,由于server1已经存在一个记录,这个时候server2从server1学习到instance1信息并更新,然后再以更高的编号发起提议并确定command2的请求操作,同理server3也是从server2学习并最终发起提议以及确定值,于是我们知道在集群服务中,无论是客户端发起的请求是落地到哪个server服务中,都将会从其他节点学习到最新的instance信息并以当前确定值更高的编号发起新的一轮提议.可以看到当上述4个instance确定之后,server服务集群之一才能发起新的一轮实例提议请求.

- 状态机服务的leader选举策略

在上述我们看到如果没有leader节点服务,每次提交的command请求都会随机落到其中的某一个server中,而每一个server发起提议请求时都需要从learner服务中获取当前一个更高的编号来发起提议请求,于是产生冲突概率会增加,导致server服务都需要从其他server服务节点中进行学习并更新确定的instance以保证发起的编号是更高且没有冲突的.虽然最后可以保证共识,但是在分布式环境每次都需要从节点学习会带来额外的网络开销,影响服务的性能,于是会考虑引入leader服务节点,将上述的server服务选举出一个leader节点,由leader节点负责接收数据的提议写入操作并同步到其他server节点中.即:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/paxos/state-leader.jpg)

- 状态机记录最大编号的作用

状态机保存的最高编号时刻与Paxos算法实例保持同步,也就是说通过Paxos算法达成共识的最高编号以及对应的确定值最终都会作为状态机的输入,状态机输入记录的数据要与server的实例保持同步.但是如果发生机器宕机的时候,状态机可能因为写入磁盘数据失败导致回滚而造成数据丢失,在这里我们不采用fsync强制的约束写盘方式,而是通过状态机记录编号来进行重启回放,也就是服务启动的时候,状态机会和server服务的最高编号进行比较,比如现在状态机检查到的最高编号为n,而Paxos算法server的最高编号为m,且满足n < m,这个时候如果存在(n, m]的确定值,那么状态机就能够从paxos服务server拉取n - m之间的编号以及对应的确定值作为状态机的输入记录,那么对外的业务集群服务仍然能够获取到确定的最高编号为m以及对应的确定值的数据,这也称为重启回放.关于状态机的实现,比如Mysql的binlog也可以理解为状态机等.

#### Paxos算法参考

```reStructuredText
https://mp.weixin.qq.com/s?__biz=MzI4NDMyNTU2Mw==&mid=2247483695&idx=1&sn=91ea422913fc62579e020e941d1d059e#rd
https://en.wikipedia.org/wiki/Paxos_(computer_science)#Assumptions
```
