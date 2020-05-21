---
title: 分布式网络基础
category: 分布式架构设计
date: 2020-05-15 17:14:50
tags: 分布式架构设计
---



<!-- more -->

## 分布式网络基础

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/base.jpg)

在分布式服务化架构设计中,服务与服务之间通信均是基于网络底层协议来实现的,于是我们需要对网络相关基础知识有一个基本的认知,这样在我们服务与服务之间进行通信(跨进程通信)过程能够在我们的脑图形成一个基本的数据传输流程以及其中的细节问题,这样对于我们在进行网络问题的排查能够带来一定的帮助.现在开始展开网络基础相关知识的阐述.



### 网络基础知识

#### 通信协议

##### 什么是协议

协议是计算机与计算机之间通过网络通信时事先达成的一种“约定”,这种“约定”使那些由不同的厂商设备,不同的CPU以及不同的操作系统组成的计算机之间,只要遵循相同的协议就能够在网络传输中实现数据交互的通信.

##### 分组交换协议

> 分组交换

在网络传输过程中,如果传输的数据块很大,则需要将其切成多个以包为单位的数据块进行传输,而这种将数据分装为一个个以包为单元的数据块称为分组.

> 网络数据包

数据包(报文)由控制信息(称为报文首部抑或是header)以及用户数据(也称为有效负载payload),控制信息包含有效负载的传输信息,即数据包的源IP地址/目标IP地址/分组之后的序号以便于在接收端的目标IP机器能够根据序号进行拼接组成一个与源数据包一致的数据块.

一个完整的数据包组成的结构如下:

- 网络数据包的路由地址:包含发送数据包的源地址以及接收数据包的目标地址
- 错误检测与纠正: 在网络协议中执行错误检测与纠正,为避免在传输过程中发生错误,需要在网络数据包进行数据校验,奇偶检验位或者循环冗余校验.
- 跳数限制:主要是作用是告诉网络路由器数据包在网络中的时间是否超过了TTL设定的一个时间段,如果超过将会被丢弃并向发送端发起一个数据丢弃的一个报文由发送端决定是否进行重发.由于每个路由器都至少要把TTL域减一,TTL通常表示包在被丢弃前最多能经过的路由器个数.当记数到0时,路由器决定丢弃该包,并发送一个ICMP报文给最初的发送者.
- 数据包的长度
- 处理数据包的优先级
- 有效的负载,即数据包实际的数据,除了当前的结构,上述的字段信息可以在header中声明和定义.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/network_protocol.jpg)

> 分组交换协议

为了保证发送端与接收端能够正确进行通信,分组的两端必须保持报文首部和内容保持一致的约定,即房租交换协议.

#### 协议栈与网络分层模型

##### 协议栈

> 协议栈(网络堆栈)

一组可以协同工作的网络协议层,定义七个协议层的OSI参考模型通常称为栈,定义Internet上的TCP/IP协议集也称为栈.术语的堆栈式指处理协议的实际软件.

- 加载堆栈: 加载使用特定协议集所需的软件
- 绑定堆栈: 将一组网络协议链接到网络接口INC,每个INC至少绑定一个堆栈才能实现不同设备之间的数据传输.

> IP堆栈的含义

一般最普遍的网络堆栈的实现为一个互联网的协议栈,也称为IP栈,在操作系统中,IP栈提供了一套应用程序库,用于与远程设备建立和关闭链接以及在远程设备之间发送和接收数据,而这其中的一套应用程序库就是我们熟知的Socket API库,并且几乎所有提供IP堆栈的平台上的api都是一致的.TCP/IP栈提供了socket套接字用于接收和发送来自应用程序的请求,而应用程序具备选择TCP/UDP(UDP也有所属于自己的一套应用程序库来接收和发送数据)不同的协议来进行数据传输.于是在IP堆栈中提供UDP以及TCP两种协议并为两种协议提供相应的应用程序库,当我们应用程序选择TCP协议来进行网络通信的时候,此时就会将一套支持TCP协议的socket库加载到IP栈中,并将TCP协议绑定到我们的INC接口来实现与远程设备的数据传输.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/network_stack.jpg)

##### 网络分层模型

> 协议栈与OSI七层参考模型

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/osi.jpg)

- 在网络分层模型中,应用层,表示层以及会话层传输数据的单位为有效负载payload(实际数据),在TCP/UDP的传输层中传输数据的单位为片段Segment,在IP(网络层)传输数据的单位为数据包package,在数据链路层传输数据的单位为帧Frames,物理层之间的数据传输以二进制0和1的bit数据流进行传输.
- OSI参考模型每一层之间的通信都遵循特定的约定,每一层的约定对应着一套协议来负责实现该层的数据分组与组装.
- OSI参考模型每个分层都接收由它下一层所提供的服务并负责为自己的上一层提供特定服务,上下层之间进行数据通信所遵循的约定称为接口,上下层之间通过接口来实现通信.
- 根据上述可以看出,每一个分层的协议传输的数据单位不同,同时对应的处理协议也不同,我们知道不同的协议对应着一套不同的处理程序库API,也就是OSI分层模型中的每一层对应的协议栈都会有所不同,协议栈之间通过接口来实现数据通信.

> TCP/IP四层模型

- 摘录《图解TCP/IP协议》对OSI以及TCP/IP分层模型的对比如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/tcp_osi.jpg)

通过上图,我们可以很清晰地看出OSI与TCP/IP分层模型之间的联系与区分,同时每一层都对应到我们的计算机中的应用程序,操作系统和网卡硬件设备接口.

- 关于数据传输单位的术语,这里直接摘录《图解TCP/IP协议》一书:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/desc_package.jpg)

- TCP/IP传输的完整数据包描述如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/tcp_package_header.jpg)

### TCP/IP协议

#### TCP/IP协议传输流程

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/tcp_transport.jpg)

- 发送端将数据payload发送到应用程序中,经应用程序通过接口转发给传输的TCP/UDP协议,负责与远程设备建立连接,并在有效数据的数据包添加报文header(源端口以及目的端口号)
- 其次,传输层将上述带有端口号信息的数据报header以及payload传递给网络IP层,IP层接收到数据并在现有的数据添加IP的header,即源IP地址与目的IP地址.
- 接着,网络IP层将数据再次传递到数据链路层,此时在原有的数据报文上添加链路层的源mac地址以及目标mac地址,最后转发到物理层并将数据转换为二进制单位bit的数据流传输到接收端设备.

#### 握手协议

##### 三次握手

> 三次握手

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/third_tcp.jpg)

- 首先,客户端发起请求建立连接的SYN,同时发送初始序号(ISN)seq为x到服务端,此时客户端处于请求连接等待确认状态,SYN=1的报文不能携带数据,但是要消耗一个序号,此时客户端处于SYN_SEND状态.
- 其次,服务端接收到客户端的请求连接并给予SYN=1的响应ACK=1,并指定服务端初始的序号seq=y,并将客户端的seq+1作为应答,即ack=x+1发送到客户端.此时服务端处于SYN_RECV状态.
- 最后,客户端接收到SYN的报文并给予响应ACK,同时也对服务端的seq进行+1并给予响应,即ack=y+1,并将服务端发送的序号seq返回.此时客户端与服务端都彼此建立连接.

> 进行三次握手的原因

- 第一次握手确认客户端能够正常实现发送报文,而服务端能够正常接收报文;
- 第二次握手确认服务端能够发送报文,此时对于服务端而言,客户端能正常发送报文但是接收报文还不确定,而对于客户端而言,服务端可以正常接收和发送报文.
- 第三次客户端向服务端发送接收到报文的请求响应,此时对于服务端而言客户端已经具备接收和发送报文处理,于是客户端与服务端建立连接并开始进行通信.

> 半连接与全连接队列

- 半连接队列: 当服务端第一次接收到客户端发起的SYN报文请求时,服务端处于SYN_RECV的状态,此时服务端与客户端还没有建立完全连接,服务端将当前的状态的请求保存在一个队列中,即半连接队列
- 全连接队列: 服务端与客户端完成三次握手建立连接,此时会将已建立连接的请求报文添加到队列中,如果队列满则会出现丢包
- 丢包场景: 客户端发送SYN报文到服务端过程中超时会丢包,同理服务端向客户端发送ACK以及seq的响应超时也会丢包,同时根据上述可知,如果半连接队列或者是全连接队列满的话,也会出现丢包现象.

> SYN攻击(DDos攻击)

SYN攻击就是客户端在短时间内伪造大量不存在的IP地址,并向服务端不断地发送SYN报文请求,由于源IP地址不存在,于是服务为了回复确认请求的报文SYN并等待客户端的响应ACK，导致服务端需要不断重发直至超时,而这些伪造的SYN报文将长时间占用在未连接队列中,这样容易使得正常的SYN报文请求因为队列满而被丢弃,从而引起网络拥塞甚至系统瘫痪.SYN 攻击是一种典型的 DoS/DDoS 攻击.

一般通过以下命令进行排查:

```bash
netstat -n -p TCP | grep SYN_RECV
```

预防SYN攻击手段:

- 缩短超时（SYN Timeout）时间
- 增加最大半连接数
- 过滤网关防护
- SYN cookies技术

##### 四次挥手

> 四次挥手过程

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/forth_tcp.jpg)

- 客户端向服务端发起连接释放的FIN请求报文并消耗一个初始的序号(ISN)seq值为u,此时客户端不再向服务端发送数据,处于等待服务端FIN的响应,处于等待终止连接状态.
- 服务端接收到客户端的连接释放FIN报文请求并开始断开与服务端的连接,释放客户端持有的连接信息,在这个过程中连接释放需要等待其正常执行释放流程,于是服务端先向客户端给予响应ACK,并对于客户端发送的序号进行+1响应,即u+1,此时服务端消耗序号v并发送到客户端中.此时TCP连接释放处于半关闭状态,客户端需要等待服务端发送确认连接释放的报文FIN,此时客户端进入终止连接状态2.
- 这个时候服务端已经完成连接释放工作,向客户端发送连接释放确认请求FIN以及对先前客户端发起的FIN报文再次给予响应ACK,同时将客户端先前发起的连接关闭请求报文的序号seq+1返回到客户端中,即ack=seq+1,并重新初始化seq序号返回给客户端.
- 客户端接收到服务连接释放确认的报文,并响应给服务端ACK,同时将服务端携带的序号+1返回,同时也将携带服务端返回的序号响应ack一同返回给服务端,此时客户端TCP连接仍然未完全关闭,需要等待一段时间用于处理连接释放等收尾工作才会自动关闭,最后客户端与服务端都处于CLOSED状态.

> 进行四次挥手的原因

主要是服务端在接收到客户端发送连接请求的报文FIN的时候,此时服务端并不能立即关闭,因为此时可能存在未处理完的网络通信,需要等待网络通信处理完成才能向客户端发送连接关闭确认报文,但是为了避免客户端不知道情况,于是就发送一个请求响应的报文段(ACK=1,seq=v,ack=u+1)告知客户端,“你的关闭连接请求报文我收到了,但是我这边连接关闭还需要等待一段时间才能关闭,晚点我再给你发一个报文确认请求”.

#### 滑动窗口与拥塞控制

##### 滑动窗口

> 为什么TCP网络传输需要引入滑动窗口

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/split_window.jpg)

通过上述图示对比可知:

- 在没有引入滑动窗口的时候,我们的TCP网络发送数据包的时候都是一个一个地往服务端发送数据,此时如果发送的数据包需要等待的响应ACK较长的话,那么也可以清楚地知道整个网络耗时以及吞吐量都会大大地降低,严重影响到网络的性能,于是引入滑动窗口来提升我们的网络性能,即在较短的时间且允许的数据包大小范围内发送数据包无需等待响应,此时一个是可以降低网络耗时,一个是提升网络的吞吐量,相比左边逐个发送数据包性能上有所提升.
- 关于超时重发,对于没有使用滑动窗口技术来发送数据包,如果在指定的超时时间内没有收到服务端返回的响应将会进行重复;而对于使用滑动窗口技术,由于窗口数据包都有存在顺序序号,于是客户端批量发送数据包的时候,如果接收到服务端三次响应的重复确认应答,那么客户端将会进行重发,也称为高速重发机制.

> 滑动窗口与流量控制

为了防止服务端(或者称为接收端)接收到一个毫无关系的数据包并在当前的数据包进行耗时的处理而导致正常的数据包因处于高负载的情况被丢弃而重发消耗网络流量,于是就有了流量控制.

-  流量控制: TCP提供一种机制让发送端根据接收端的实际接收能力控制发送的数据量来避免无效的数据包处理导致耗时影响网络性能.
- 流量控制的窗口大小:接收端会向发送端通知自己可以接收的数据大小,于是发送端会发送不会超过这个限度的数据.该限度的大小称为窗口大小.
- 流量控制与超时: 客户端利用滑动窗口批量发送数据包,等待下一次服务端的窗口更新通知告知可以发送的数据包大小,而此时服务端接收到数据之后发现缓冲区满了,需要暂停接收数据,这个时候将会向客户端发送服务端窗口更新通知,如果此时因为超时或者更新通知数据丢包而导致无法继续通信,客户端发现过了超时时间还没有窗口更新通知,此时就会发送一个窗口探测的数据包以获取最新的窗口更新通知大小.

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/flow_control.jpg)

##### 拥塞控制

为了避免TCP一开始发送较大量的数据包而导致网络拥堵瘫痪的问题,在通信一开始时就会通过一个叫做慢启动的算法得出数值,对发送数据量的控制.

- 拥塞窗口: 为了在发送端调节所要发送的数据的量大小,在慢启动的时候会先发送一个拥塞窗口大小为1的数据段到发送端,此后每收到一次确认应答ACK,就会窗口的值就会加1,并且在发送端进行发送数据包时,会将拥塞窗口与滑动窗口通知的大小进行比较取最小值进行发送.
- 对于重发采用超时机制,那么拥塞窗口可以设置为1之后进行慢启动修正.

#### 粘包与拆包

##### 粘包与拆包问题分析

一般情况下出现粘包以及拆包现象主要是在网络传输的过程自定义协议,但是对于自定义的协议进行解析的时候存在一些换行.空格等一些特殊字符,这些特殊字符需要我们在应用层上进行手动处理来保证是一个完整的数据消息片段.

- 正常传输

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/tcp_normal.jpg)

- 粘包

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/tcp_zhanbao.jpg)

```java
// 代码表现
// 客户端发送数据
String m1 = "this is the first msg";
String m2 = "this is the second msg";

client.send(m1);
client.send(m2);

// 服务端接收数据
while(true){
  String msg = server.read();
	log.info("msg=" + msg);
}
// msg.log
// msg = this is the first msgthis is the second msg
```

- 拆包

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/tcp_chaibao.jpg)

```java
// 代码表现
// 客户端发送数据
String m1 = "this is the first msg";
String m2 = "this is the second msg";

client.send(m1);
client.send(m2);

// 服务端接收数据
while(true){
  String msg = server.read();
	log.info("msg=" + msg);
}
// msg.log
// msg = this is the first msgthis is
// msg = the second msg
```

##### 解决方案

一般需要我们进行自定义的协议,大部分都是编写框架或者是公司内部的一些基础的通用技术支持,在Java网络编程中一般底层网络利用Netty框架来帮助我们快速实现一个高性能的web服务,而Netty框架本身提供了编解码器来帮助我们解决上述的粘包与拆包问题.

- `LineBasedFrameDecoder` 可以基于换行符解决。
- `DelimiterBasedFrameDecoder`可基于分隔符解决。
- `FixedLengthFrameDecoder`可指定长度解决

后续关于Netty的编解码器再重新写一篇文章来阐述,在这里仅需知道什么是粘包与拆包即可.

### HTTP/HTTPS协议

#### HTTP协议

> http简要概述

- 用于客户端与服务端之间的通信
- 通过发起请求以及请求响应完成客户端与服务端之间的通信
- http协议是不保存状态的协议,即无状态协议,保存状态是通过cookie与session协同完成
- 通过请求的uri定位到远程资源
- 具备告知服务器端具体的意图行为,比如doGET/doPOST/doPUT/doDELETE/doPATCH等操作
- http协议是属于短连接的方式,但是可以通过“keep-alive”来持久连接,主要是为了减少每次建立连接都要进行三次握手以及退出连接进行四次挥手的过程.
- 使用管线化技术来避免每次连接都需要等待连接的响应,类似于滑动窗口原理,可以同一个时间点发起http的请求.

最后,关于上述的描述,摘录《图解HTTP》截图说明:

http方法明细如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/http_method.jpg)

http的持久连接明细如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/http_chijiuhua.jpg)

http管线化(pipeline)如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/http_pipeline.jpg)

> http报文说明

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/http_header.jpg)

- HTTP请求体结构

```reStructuredText
curl -v https://www.baidu.com

GET / HTTP/1.1                            ### 请求行(请求方法 + http协议版本)
Host: www.baidu.com
User-Agent: curl/7.54.0
Accept: */*
```

- HTTP响应体结构

```reStructuredText
curl -i https://www.baidu.com

HTTP/1.1 200 OK														### 状态行(http协议版本 + 状态码)
Accept-Ranges: bytes
Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
Connection: keep-alive
Content-Length: 2443
Content-Type: text/html
Date: Thu, 21 May 2020 06:57:08 GMT
Etag: "58860402-98b"
Last-Modified: Mon, 23 Jan 2017 13:24:18 GMT
Pragma: no-cache
Server: bfe/1.0.8.18
Set-Cookie: BDORZ=27315; max-age=86400; domain=.baidu.com; path=/

<!DOCTYPE html>
<!--STATUS OK--><html> .... </html>
```

> http常用头字段说明

- 通用的部分首部字段

```bash
## Cache-Control: 控制缓存行为
## Pragma: 报文指令
## Transfer-Encoding: 指定报文主体的传输编码方式
## Via: 代理服务器信息
## Warning: 错误通知
```

- 部分请求首部的字段

```bash
## Accept: 通知服务器用户代理可处理的媒体类型以及媒体类型的相对优先级
## Accept-Charset: 用户代理可处理的字符集,用q表示优先级
## Accept-Encoding: 用户代理可以处理的内容编码,如gzip等
## Authorization: 用户代理需要进行认证授权信息
## Referer: 告知服务器请求资源的原始uri
```

- 部分响应首部的字段

```bash
## Location: 响应状态码返回302的时候会携带Location的uri
## WWW-Authorization: 但客户端发起请求的授权认证不对的时候服务端会返回401并携带此响应信息指定认证方式
```

- 实体使用的首部字段

```bash
## Allow: 允许使用的http方法
## Content-Length: 实体主体部分的大小
## Content-Range: 数据太太,分组发送,告知当前数据发送的字节大小资源情况
```

- 缓存使用的字段

```bash
## 缓存控制,Cache-Control 与 Pragma
## Cache-Control: no-store  没有缓存
## Cache-Control: no-cache  缓存但重新验证,有缓存但未过期,响应返回304
## Cache-Control: private | public    缓存默认私有/公有(中间代理、CDN等代理中间件缓存)
## Cache-Control: max-age=31536000  | must-revalidate   缓存过期与验证

## Pragma,与Cache-Control: no-cache定义一致,但是定义Pragma以向后兼容基于HTTP/1.0的客户端,不能拿来完全替代HTTP/1.1中定义的Cache-control头

## 缓存失效时间计算公式: expirationTime = responseTime + freshnessLifetime - currentAge
## 可以查看头部信息的Cache-Control: max-age=N | expires | Last-Modified

## 缓存验证
## ETags:作为缓存的一种强校验器，ETag响应头是一个对用户代理(User Agent,即UA)不透明,如果资源请求的响应头里含有ETag,客户端可以在后续的请求的头中带上If-None-Match头来验证缓存
## Last-Modified 响应头可以作为一种弱校验器。说它弱是因为它只能精确到一秒。如果响应头里含有这个信息，客户端可以在后续的请求中带上 If-Modified-Since 来验证缓存
## 当向服务端发起缓存校验的请求时，服务端会返回 200 ok表示返回正常的结果或者 304 Not Modified(不返回body)表示浏览器可以使用本地缓存文件。304的响应头也可以同时更新缓存文档的过期时间

## Vary响应
## Vary: User-Agent,缓存服务器需要通过UA判断是否使用缓存的页面.如果需要区分移动端和桌面端的展示内容,利用这种方式就能避免在不同的终端展示错误的布局
```

- 其他首部字段

```bash
## X-FRAME-OPTIONS: 在html使用iframe标签的时候,是否允许跨域访问,有DENY/SAMEORIGN/ALL
## X-XSS-Protection: 针对跨站脚本攻击的防护机制开关,0 - XSS过滤设置无效, 1 - 设置为有效
```

> http常用的状态码说明

- 状态码的类别如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/http_status.jpg)

- 2XX:返回响应成功,204表示没有资源返回但响应正常,206表示请求部分资源
- 301: 资源的uri已经被更新,告知客户端请求资源uri需要进行更新,以后使用新的资源uri进行访问
- 302: 资源的uri被临时更新,告知客户端当前请求资源的uri需要临时更新,但是之后仍然会使用旧的资源uri.
- 304: 表示缓存,告知客户端直接使用缓存响应即可.
- 400: 客户端请求报文存在问题
- 401: 表示需要客户端请求资源的时候发起认证
- 403: 没有权限访问,就是不让客户端访问资源,不需要给出理由
- 404: 服务端没有资源
- 500: 服务端本身处理程序出现问题
- 503: 表示服务端的负载过大或者是停机维护,请稍后再试.

#### HTTPS协议

- http与https区分

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/https_http.jpg)

https协议是在http的基础上增加一层SSL/TLS协议,也就是HTTP部分接口使用SSL/TLS协议来代替进行密文加密传输数据.

- https执行流程原理

https采用非对称的公开密钥加密和对称加密的共享加密并用的混合加密机制,同时为了保证公开的密钥是可信任的,于是需要第三方的认证机构颁发证书来完成公开密钥的认证.具体的https加密执行流程如下:

![](https://raw.githubusercontent.com/xiaokunliu/xiaokunliu.github.io/feature/writing/websites/zimages/arch/network/https.jpg)

- https发起一个请求的时候,底层仍然是以TCP与远程建立连接,需要经过三次握手,此时客户端发起请求的时候,会携带以下信息: TLS/SSL协议版本 + client.random加密的对话密钥 + 加密算法 + sessionId(用于保持会话连接)
- 其次,服务端接收到信息的时候,确认SSL/TLS的版本号,会接收到客户端的加密算法,并同样生成一个对话密钥(server.random)并携带证书信息返回给客户端. 
- 客户端接收到服务端的证书信息的时候,使用第三方机构进行证书的验证,一般是根据证书管理路径/是否有效等进行验证,然后验证成功之后,将随机生成一个pre master secret,然后利用上述建立会话的基础,以client.random + server.random + pre master secret进行对称加密生成摘要,再利用证书的公钥进行加密,接着再将握手消息进行hash,利用协商好的对话密钥pre master secret进行加密(握手信息 + 握手消息的hash)发送到服务端.
- 服务端接收sessionToken的时候需要用私钥进行解密然后再利用对称加密算法进行摘要解密得到随机对话密钥,然后再将握手消息进行解密,得到握手信息,对握手消息进行hash与传输过来的hash进行验证,验证通过之后,将使用pre master secret进行加密部分握手消息传递的客户端
- 客户端使用对话密钥进行解密,验证与服务端传递的hash是否一致,如果一致握手协议至此完成,接下来所有的通信数据将由之前交互过程中生成的 pre master secret / client.random/server.random 通过随机密钥的算法得出 session Key,作为后续交互过程中的对称密钥.





