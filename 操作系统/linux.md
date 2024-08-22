# Linux

## 排查系统负载高原因

负载高->系统中活跃和不可中断进程平均数

### cpu使用率高

通过top命令查找占用CPU最高的进程PID；
通过top -Hp PID查找占用CPU最高的线程TID;
通过vmstat -n 1从系统角度查看cpu使用情况 
打印调用栈信息分析

#### 现象、原因和解决办法

* 空闲cpu低，系统占用cpu比例大于用户占用cpu，cpu资源不足，扩容
* cpu使用率很高，集中在部分代码，排查是否有死循环，优化代码
* cpu使用低，load高->存在不可中断睡眠状态进程，例如硬件设备、文件IO操作完成等，阻塞了其他进程，等待执行任务越来越多，只能恢复依赖资源或重启解决
  * 通过ps -axjf命令查看是否存在D进程
  * 通过top命令查看CPU等待IO时间，即%wa；
  * 通过iostat -dx -m 1 10查看磁盘IO情况->指标含义
  * 通过sar -n DEV 1 10查看网络IO情况；

#### 现象、原因和解决办法

* 磁盘容量小，扩大磁盘
* 内存不足，扩大内存
* 优化程序io
[Linux系统中负载较高问题排查思路与解决方法](https://www.cnblogs.com/liuyupen/p/13905967.html)
[监控工具pprof](https://studygolang.com/articles/35913?fr=sidebar)

## 授权
### chmod
<img src=".\image\11.png" alt="11" />  

## 磁盘
在Linux系统中，当磁盘空间满了，需要快速找出占用最大空间的文件或目录，以便进行清理。以下是一些常用的命令和方法来帮助你找到占用最大空间的文件或目录：

### 1. 使用 `df` 命令检查磁盘使用情况
df显示已挂载文件系统的总空间、已用空间、可用空间以及挂载点的信息，是整个文件系统使用情况，不是单个目录。
首先，使用`df`命令来查看各个磁盘分区的使用情况：

```bash
df -h
```

这个命令会显示所有挂载点的磁盘使用情况，`-h`选项表示以易读的方式（如GB、MB）显示。

### 2. 使用 `du` 命令查找特定目录的磁盘使用情况

使用`du`命令来查看特定目录下的磁盘使用情况：

```bash
du -sh /path/to/directory
```

这里，`-s`表示汇总每个指定目录的用量，`-h`表示以易读的格式显示。

### 3. 找出占用空间最大的文件或目录

要找出特定目录下占用空间最大的文件或目录，可以使用`du`命令并结合排序：

```bash
du -ah /path/to/directory | sort -rh | head -n 10
```

- `du -ah`：列出目录及其文件的磁盘使用情况。
- `sort -rh`：按磁盘使用大小逆序排序。
- `head -n 10`：显示前10条记录。

### 4. 使用 `find` 命令查找大文件

如果你想直接找出大文件，可以使用`find`命令：

```bash
find /path/to/search -type f -size +100M
```

这个命令会列出`/path/to/search`目录下所有大于100MB的文件。

### 总结

通过上述方法，你可以有效地找到并管理占用大量磁盘空间的文件或目录。定期检查磁盘使用情况并清理不必要的文件是保持系统性能的好习惯。

## oom
### 1. 检查OOM Killer日志

首先，查看OOM Killer的日志来确定哪个应用程序被杀死，使用dmesg查询系统启动以来内核的各种诊断信息，包括硬件错误、驱动程序消息和其他系统级事件。

```bash
dmesg | grep -i 'killed process'
```

或者查看`/var/log/syslog`或`/var/log/messages`（取决于你的系统配置）：

```bash
grep -i 'killed process' /var/log/syslog
```

这些日志会告诉你哪个进程被杀死，以及它在被杀死时消耗了多少内存。

### 2. 监控内存使用情况

使用`top`, `htop`或`atop`等工具实时监控内存使用情况。这些工具可以帮助你识别内存使用高的进程：

```bash
top
```

在`top`界面中，按`M`键可以按内存使用量排序。

### 3. 分析应用程序日志

查看应用程序的日志文件，可能包含有关内存问题的错误或警告信息。这些日志可能会提供关于内存泄漏或异常内存使用的线索。

### 4. 使用性能分析工具

对于特定类型的应用程序，如Java或.NET应用程序，使用专门的性能分析工具来诊断内存问题：

- **Java**: 使用`jmap`, `jstack`, `jstat`, 或`VisualVM`等工具来分析Java堆和对象的创建。
- **GoLang**: 使用`pprof`等工具来分析内存/堆使用情况。

## 运行数据
### pprof
* cpu
* 堆
* 协程
* 内存
### gdb

[Go程序内存泄露问题快速定位](https://www.hitzhangjie.pro/blog/2021-04-14-go%E7%A8%8B%E5%BA%8F%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2%E9%97%AE%E9%A2%98%E5%BF%AB%E9%80%9F%E5%AE%9A%E4%BD%8D)

## 网络抓包
### tcpdump(非linux自带)
```linux
tcpdump [option] [proto] [dir] [type]

// 抓取源ip是192.168.11.10且目的端口是22，或源ip是192.168.11.11且目的端口是80的数据包。
tcpdump -vv (src host 192.168.11.10 and port 22) or 
(src host 192.168.11.11 and port 80)
```
1. **抓取所有 TCP 数据包**:
   ```bash
   sudo tcpdump tcp
   ```
   这个命令会捕获所有 TCP 数据包。`tcp` 是一个过滤器，指定 `tcpdump` 只显示 TCP 协议的数据包。

2. **抓取所有 UDP 数据包**:
   ```bash
   sudo tcpdump udp
   ```
   类似地，使用 `udp` 过滤器可以捕获所有 UDP 数据包。

[Tcpdump 详解（抓包）](https://blog.csdn.net/mm970919/article/details/141305763)
[[Tcpdump] 网络抓包工具使用教程](https://blog.csdn.net/m0_37383484/article/details/135828598)

### 查看进程端口号
1. ss (socket statistics)是netstat的替代工具，查询出所有正在连接和建立的tcp和udp连接。以下是使用ss命令查看程序占用端口号的示例：
```bash
netstat/ss -tuln | grep <进程名或pid>
```
### 查看端口被占用进程
1. lsof（list open files）是一个列出当前系统打开文件的工具。
```bash
lsof -i tcp:<端口>
lsof -i udp:<端口>
lsof -i :<端口>
```
