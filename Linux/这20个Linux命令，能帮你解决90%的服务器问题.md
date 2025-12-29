直击运维核心痛点，极具收藏价值。

## 1.**`top` / `htop`**
实时看 CPU、内存、进程占用。这是一个非常重要且常用的工具，能够帮你快速检查CPU、内存、进程的情况，对于排查服务器问题非常有效。
top效果如下图：
![top效果](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251224150610485.png)

htop效果如下图:
![htop效果](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251224150449387.png)
使用`top -c` 能够显示完整命令；`htop` 是更友好的工具,但通常需额外安装才能用。

## 2.**`free -h`**
快速查内存使用情况。
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251224151554481.png)
重点关注 **`available`**，这是真正可用内存，别只看`used`，还要看`buff/cache`,Linux 会用缓存，不代表真没内存！

## 3.**`df -h`**
看磁盘空间使用情况。尤其要关注 `/` 和 `/var` 分区的情况。
出现报错 “No space left”时，别犹豫，第一时间用`df -h`查看一下。
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251224152327027.png)
看看`Avail`这是可用的，`Use%`这个是使用率，百分比越高，说明占用空间越多，可用空间越少。

## 4.**`vmstat`**
查看系统整体运行状况,助你一眼窥探系统瓶颈。
例如：`vmstat -w 1 5` 整体查看运行状况
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251224153947914.png)
`-w`参数宽屏列对齐显示，第一个数字是指间隔时间，第二个数字是打印输出次数。你也可以改成`vmstat -w 2 5`即是每隔2秒采样输出一次，总共采样输出5次。
几个关键指标要注意，**id**指CPU空闲时间百分比,这个越高说明CPU没那么忙。**wa**是CPU等待I/O的时间百分比,越高越说明磁盘忙不过来,如果`wa > 20%`,那你的磁盘有点不堪重负。

## 5.**`iostat`**
深度看磁盘 I/O情况。iostat是用于监控系统 CPU 使用率​ 和 磁盘 I/O 性能​ 的工具。
例如：`iostat -x 1 2` 查看磁盘的情况
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251224155849763.png)
其中-x参数显示更详细的设备统计信息。1 2参数指每间隔1秒采样输出，总共采样输出2 次​ 。
avg-cpu是显示CPU情况:
%user 指用户态进程占用 CPU 时间百分比；
%nice 在用户态下，低优先级（nice）进程的时间百分比；
%system 内核态进程占用 CPU 时间百分比；
%iowait  CPU 等待 I/O 操作完成的时间百分比
%steal   在虚拟化环境中,被其他虚拟机抢占的CPU时间；
%idle CPU 空闲时间百分比。
Device是设备IO的情况，r开头的是读取 (Read) 相关 (`r/s`, `rkB/s`, `r_await`, 等)，w开头的是写入 (Write) 相关 (`w/s`, `wkB/s`, `w_await`, 等)。
d开头是丢弃 (Discard) 相关 (d/s, 等)，通常是与 SSD 相关。f开头是刷新区 (Flush)相关 (`f/s`, 等)。
还有两个关键指标 **`aqu-sz`** 是指平均请求队列长度，即IO操作排队情况，越大越表示繁忙。**`%util`** 指设备利用率百分比。
## 6.**`netstat`** / **`ss -tulnp`**
查看网络状态，比如看那些端口开着，哪些应用在监听什么端口、哪些外部ip连在这台服务器上。
![netstat](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251224175011876.png)

![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251224175420323.png)
`ss -tulnp`是替代老旧的 `netstat`，更快，且显示更友好。很多新发行版都没有装netstat了，如果要使用需要安装，新发行版更多的可能是默认安装了ss。比如要查看80端口是否被占用可以`ss -tnp | grep :80`方式查看。

## 7.**`ping` / `traceroute`**
`ping` 测试通不通，部分云服务器和一些网站可能禁 ping，可以改用 `telnet ip port` 测试 TCP连通性。
例如：`ping baidu.com` 通常用来测试互联网访问是否可达。
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251226170310003.png)

`traceroute` 查看路由卡在哪一跳。可能部分系统要安装才可以用。
例如：`traceroute baidu.com` 来测试经过哪些路由或者路由耗时。
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251226171622652.png)
这里看到部分很多星号。甚至后面全都是星号\*是因为大部分核心服务器基于安全和减少无用负载考虑，不响应traceroute的探测包。

## 8.**`tcpdump`**
这是linux下一个常用的抓包神器！常用于排查 API 异常、丢包、协议问题。
例如：`tcpdump -i any -c 5` `-i any`是指定所有网络(网卡)。`-c 5`是指抓包5次
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251229180725767.png)

例如：`tcpdump -i eth0 port 80 -w http.pcap` 指抓取eth0网卡下端口80的数据 保存http.pcap文件中。

## 9.**`iftop -i eth0`**
iftop是一个专门实时监控和查看网络流量的工具，可以实时看流量排行，比如可以用来揪出 DDoS 或异常上传进程。
例如：`iftop -i eth0` 如下图所示
![image.png](https://cdn.jsdelivr.net/gh/xtcn92/zhpic@image/20251229181637089.png)

## 10.**`grep`**
文本搜索之王！支持正则、上下文显示。

## 11.**`tail -f`**
实时追踪日志更新，调试必备。
例如： `tail -f log | grep --color "Exception"`

## 12.**`awk` / `sed`**
`awk` 统计分析（如 Nginx 状态码），`sed` 批量替换。
例：`awk '{print $9}' access.log | sort | uniq -c`

## 13.**`find`**
按大小、时间、名称找文件。

## 14.**`du -sh * | sort -hr`**
看当前目录下各文件夹大小，快速定位“空间吞噬者”。

## 15.**`lsof -i :端口号`**
查哪个进程占用了端口（比 `netstat` 更直接）。

## 16.**`ps aux | grep [p]rocess`**
查找特定进程（方括号避免 grep 自身被匹配）。

## 17.**`kill` / `killall`**
杀死进程，先 `kill PID`（优雅退出），不行再 `kill -9`（强制终止）。
> [!note]
> 正式环境一定要谨慎使用 -9


## 18.**`systemctl`**
管理服务：`start`/`stop`/`restart`/`status`

## 19.**`journalctl -u 服务名 --since "1 hour ago"`**
查 systemd 服务的日志，比翻 `/var/log` 更集中。就是日志量大时会有点耗时。

## 20.**`history | grep "关键词"`**
快速帮你找回你曾经敲过的命令，省得重复记忆！