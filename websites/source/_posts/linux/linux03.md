---
title: linux工作常用命令小结3
category: Linux技术小结
date: 2020-04-02 20:47:28
tags: Linux技术小结
---

<!-- more -->

#### linux 工作常用命令小结(1)
> 参考链接
```text
## 参考链接：
http://man.linuxde.net/
http://linuxtools-rst.readthedocs.io/zh_CN/latest/index.html
```

> 检查机器配置
```bash
# linux CPU大小,对于双核的cpu，在cpuinfo中会看到两个cpu,通过Physical Processor ID来区分单核和双核
cat /proc/cpuinfo |grep "model name" && cat /proc/cpuinfo |grep "physical id"

# 内存大小
cat /proc/meminfo |grep MemTotal

# 硬盘大小
fdisk -l | grep Disk

# 查看内核/操作系统/CPU信息的linux系统信息命令
uname -a

# 查看操作系统版本,是数字1不是字母L
head -n 1 /etc/issue

# 查看CPU信息的linux系统信息命令
cat /proc/cpuinfo

# 列出加载的内核模块
lsmod 

# 查看环境变量资源
env

# 查看内存使用量和交换区使用量
free -m

# 查看各分区使用情况
df -h 

# 查看指定目录的大小
du -sh *

# 查看系统运行时间、用户数、负载
uptime

# 查看系统负载磁盘和分区
cat /proc/loadavg

# 查看挂载的分区状态
mount | column -t 

# 查看所有分区
fdisk -l

# 查看所有网络接口的属性
ifconfig

# 查看防火墙设置
iptables -L

# 查看路由表
route -n 

# 查看所有监听端口
netstat -lntp 

# 查看所有已经建立的连接
netstat -antp 

# 查看网络统计信息进程
netstat -s 

# 查看所有进程
ps -ef 

# 查看系统所有用户
cut -d: -f1 /etc/passwd

# 查看系统所有组
cut -d: -f1 /etc/group

# 查看当前用户的计划任务服务
crontab -l
```

> 日志查看
```bash
more /path/logfiles
less /path/logfiles
tail -f /path/logfiles
```

> 机器监控命令
```bash
## 查看系统进程/cpu/内存/线程等信息
top
```

>  文本处理操作
```bash
# 批量的文件转换为linux的换行符号
find dir/ -name "*.py" |xargs sed -i 's/\r//'

# 单个文件
sed -i 's/\r//'  filename
```

> 查看当前服务IP
```bash
# 出口IP
curl http://members.3322.org/dyndns/getip
# 内网IP 以及 服务的网卡信息
ifconfig
```

> 查看端口占用的进程
```bash
lsof –i:port
```

> 域名操作工具
```bash
# dig命令是常用的域名查询工具，可以用来测试域名系统工作是否正常
dig www.baidu.com

# nslookup，是常用域名查询工具，就是查DNS信息用的命令
nslookup www.baidu.com

# host命令是常用的分析域名查询工具，可以用来测试域名系统工作是否正常
host www.baidu.com
```

> 追踪网络数据包信息
```bash
# traceroute命令用于追踪数据包在网络上的传输时的全部路径，它默认发送的数据包大小是40字节
traceroute www.baidu.com

# tcpdump，可以打印所有经过网络接口的数据包的头信息，也可以使用-w选项将数据包保存到文件中，方便以后分析
tcpdump tcp port 80     # 监控本机器的80端口数据包流量

# 抓包命令
ngrep -d any -W byline port port_num | head
```
