# Linux性能监控

[toc]

## 体系结构



![a3119cdccbe38046ae41e5a8f7a9b52](https://img.zimei.fun/202405092157095.jpg)

## 入门级

### 1.top（内存，CPU）

#### 基本命令

```shell
top #回车后进入面板

进入面板后

1 # 查看所有cpu
c # 查看进程执行目录
q # 退出面板
```

> running运行进程数量
>
> sleeping睡眠进程
>
> zombil僵尸进程

#### 使用场景

功能齐全

### 3.vmstat（内存，swap虚拟内存）

#### 基本命令

```shell
vmstat 1 10 # 1秒一刷,刷10次
```

> memory内存 
>
> swap虚拟内存
>
> si 进 so出
>
> cache缓存

#### 使用场景

主要用来看各种内存

### 4.iostat（磁盘）

#### 基本命令

```shell
iostat -d -x -k 1 10 # 1秒一刷,刷10次
```

> %util 繁忙率（越高越忙）

#### 使用场景

例如，服务器挂了2个1T的盘，正在大量读写。同时服务器上面有提供数据库服务。

此时担心磁盘由于读写操作达到IO瓶颈等，导致服务器崩溃。需要通过该命令查看磁盘忙不忙（查看的是IO使用率）。

ps： `df -h  df -Th`也可以看磁盘

## 中级

### 5.mpstat（cpu）

#### 基本命令

```shell
mpstat 1 10 # 1秒一刷,刷10次
```

> %usr 使用 
>
> %idle 空闲
>
> %iowait 等待

#### 使用场景

专门看cpu忙不忙。

可以看服务器上所有cpu。例如，云平台上面租用的服务器是4核的，全部都能查看。

### 6.pidstat（某个进程磁盘读写）

#### 基本命令

```shell
pidstat -d 1 10 # -d参数看盘，1秒一刷,刷10次 
```

#### 使用场景

可以看哪个进程在做读写，谁在用磁盘。

并不是所有进程都用磁盘，有的进程只是单纯使用内存。

而有的进程，例如不断写日志文件，就需要一直占用磁盘读写。

### 7.free（内存）

#### 基本命令

```shell
free -h # -h更加直观
```

#### 使用场景

看内存

### 8.netstat（网络）

#### 基本命令

```shell
看服务端口占用情况
netstat -anptu # 更加全面
or
ss -ntl # 更加简洁
```

#### 使用场景

看端口，看用户访问量

### 9.sar（性能分析）

#### 基本命令

```shell
sar 1 10 # 默认监听cpu，和mpstat一样
sar -d 1 10 # 加参数-d，添加磁盘
sar -q 1 10 # 加参数-q，显示负载
```

> sar -q 
>
> ldavg-1 ldavg-5 ldavg-15看整个服务器是否繁忙。
>
> 例如，在跑项目时，虽然某个cpu100%使用率，但是另一个cpu0%，整个项目实际上是没有问题的。用该命令看整体负载。
>
> cpu核数为1，负载是0.4表示正常，负载长期1以上服务器存在问题

#### 使用场景

更加详细的全面分析，有点像万能胶水，可以复用别的命令的功能。

## 高级

### 10.[系统性能分析工具：perf - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/186208907)

### 11.[systemtap从入门到放弃（一） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/347313289)

### 12.[Linux神器strace的使用方法及实践 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/180053751)

### 13.[Linux 全能系统监控工具dstat的实例详解-腾讯云](https://cloud.tencent.com/developer/article/1722033)

## 参考🧲

> [Linux 懒人运维:**vmstat,iostat,mpstat,sat,pidstat** 等_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1kX4y1Q7XR/?spm_id_from=333.337.search-card.all.click&vd_source=ac53754f6533097757863a1d248f5406)
>
> [Linux 懒人运维:wget,curl,watch,**top**,uptime 等_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1rD4y137Tx/?spm_id_from=pageDriver&vd_source=ac53754f6533097757863a1d248f5406)
>
> [Linux 懒人运维:strace,**netstat**,mount,lsblk,history 等_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1L24y1872e/?p=137&spm_id_from=pageDriver&vd_source=ac53754f6533097757863a1d248f5406)
>
> [Linux 懒人运维:常用命令ls,cd,mkdir,cat,**free**,df._哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1wY4y1U7KV/?spm_id_from=pageDriver&vd_source=ac53754f6533097757863a1d248f5406)