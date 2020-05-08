---
title: linux工作常用命令小结2
category: Linux技术小结
date: 2019-06-08 20:47:28
tags: Linux技术小结
---

<!-- more -->
#### linux 监控命令小结
> IO监控
```bash
# iostat 仅对系统整体情况进行分析
# 系统级别的IO监控
iostat -xdm 1

# %util:代表磁盘繁忙程度。100% 表示磁盘繁忙, 0%表示磁盘空闲。但是注意,磁盘繁忙不代表磁盘(带宽)利用率高  
# argrq-sz:提交给驱动层的IO请求大小,一般不小于4K,不大于max(readahead_kb, max_sectors_kb),可用于判断当前的IO模式,一般情况下,尤其是磁盘繁忙时, 越大代表顺序,越小代表随机
# svctm:一次IO请求的服务时间,对于单块盘,完全随机读时,基本在7ms左右,既寻道+旋转延迟时间

# 进程级IO监控
# iotop:顾名思义, io版的top
# pidstat:顾名思义, 统计进程(pid)的stat,进程的stat自然包括进程的IO状况


# 业务级IO监控
# ioprofile 命令本质上是 lsof + strace, 具体下载可见 http://code.google.com/p/maatkit/
ioprofile

# 文件级IO监控
# 文件级IO监控可以配合/补充"业务级和进程级"IO分析
# 文件级IO分析,主要针对单个文件, 回答当前哪些进程正在对某个文件进行读写操作.
# 1 lsof   或者  ls /proc/pid/fd
# 2 inodewatch.stp

# inodewatch.stp
#! /usr/bin/env stap

probe vfs.write, vfs.read
{
  # dev and ino are defined by vfs.write and vfs.read
  if (dev == MKDEV($1,$2) # major/minor device
      && ino == $3)
    printf ("%s(%d) %s 0x%x/%u\n",
      execname(), pid(), probefunc(), dev, ino)
}
```

> 查看磁盘IO
```bash
# 左右箭头：改变排序方式，默认是按IO排序。
# r：改变排序顺序。
# o：只显示有IO输出的进程。
# p：进程/线程的显示方式的切换。
# a：显示累积使用量。
# q：退出。
iotop -oP
```

> 查看进程cpu，内存负载
```bash
top -c
# 按键F，进入选择模式，根据linux上面的提示，如果选择排序则按键S，再退出即会显示按照指定的指标排序显示
# 显示内存状态 free,具体包括物理内存,虚拟内存,共享内存和系统缓存
# 显示系统进程在瞬间的运行动态,ps
```

> 虚拟内存状态
```bash
# 可以报告关于进程、内存、I/O等系统整体运行状态
# 事件间隔：状态信息刷新的时间间隔；
# 次数：显示报告的次数。
vmstat 3   # 每隔3s打印一次
```

> 磁盘分区监控
```bash
# 分区使用率大于85%:{/home:[90.7568]},搜索磁盘占用空间最大的目录并通知清理
# du命令是对文件和目录磁盘使用的空间的查看
du --max-depth=1 -BM ./

# df命令用于显示磁盘分区上的可使用的磁盘空间。默认显示单位为KB。可以利用该命令来获取硬盘被占用了多少空间，目前还剩下多少空间等信息
df -h
```

> 内存监控
```bash
# 机器内存free（含buf/cache）小于10%（7.1800）
# 查看内存占用情况
free -m

# 查看进程占用的内存,从高到低占用排序
top -c -o -%MEM
```

> 统计网络接口活动
```bash
# ifstat命令就像iostat/vmstat描述其它的系统状况一样，是一个统计网络接口活动状态的工具
apt-get install -y ifstat
ifstat -tT
```

> 查看网络状态
```bash
# netstat命令用来打印Linux中网络系统的状态信息，可让你得知整个Linux系统的网络情况
# 并不是所有的进程都能找到，没有权限的会不显示，使用 root 权限查看所有的信息
netstat -ap | grep uwsgi

# 显示网络接口列表
netstat -i

# 查看phpcgi进程数，如果接近预设值，说明不够用，需要增加
netstat -anpo | grep "php-cgi" | wc -l
```

> 全能系统信息统计工具
```bash
# dstat命令是一个用来替换vmstat、iostat、netstat、nfsstat和ifstat这些命令的工具，是一个全能系统信息统计工具
# 监控swap，process，sockets，filesystem并显示监控的时间
dstat -tsp --socket --fs
```

> 显示系统的平均负载
```bash
# uptime命令能够打印系统总共运行了多长时间和系统的平均负载。
# uptime命令可以显示的信息显示依次为：现在时间、系统已经运行了多长时间、目前有多少登陆用户、系统在过去的1分钟、5分钟和15分钟内的平均负载
uptime
```

> 系统运行状态统计工具
```bash
# sar命令是Linux下系统运行状态统计工具，它将指定的操作系统状态计数器显示到标准输出设备
# 察看内存和交换空间的使用率
# kbmemfree与kbmemused字段分别显示内存的未使用与已使用空间，后面跟着的是已使用空间的百分比（%memused字段）。
# kbbuffers与kbcached字段分别显示缓冲区与系统全域的数据存取量，单位为KB。
sar -r
```


