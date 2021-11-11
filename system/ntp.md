# 配置 NTP

NTP（Network Time Protocol）是利用网络进行时间同步的协议。一些系统服务的正确运行需要依赖各个节点之间的时间同步（例如 MUNGE），因此必须要配置集群的 NTP 服务。如果仅仅是购买单个服务器，由于系统已经开启了网络对时功能，无需进行配置。

### 配置​方案

集群​节点进行时间同步需要访问外网的时间同步服务器，故先使用 `login01` 同步互联网时间，集群的其它节点再和 `login01` 进行同步。也就是说，在互联网层面上，`login01` 是一个客户端，而在集群内部它却是一个服务端。

{% hint style="info" %}
假设​配置网络时已经配置了 [NAT](https://ryanbernx.gitbook.io/cluster-admin/system/network#nat)，原则上所有的计算节点和存储节点也都可以直接访问互联网上的时间同步服务器，因此集群所有节点作为客户端进行网络对时也是一种同步时间的做法。而将 login01 作为集群内部的 NTP 服务端的好处是：1) 没有配置 NAT 时也能使用；2) 使用局域网同步的误差更小。
{% endhint %}

### 安装​ chrony

Ce​ntOS 7 使用的 NTP 软件为 chrony。这个软件一般已经随系统安装好了，如果没有，可使用如下命令安装：

```bash
yu​m install chrony
```

{% hint style="danger" %}
一些​过时的文档会使用 ntp 这个软件进行同步，CentOS 7 已经使用了 chrony 进行替代。显然同时安装两种 NTP 软件时就会造成冲突，如果一定要使用 ntp，则必须关掉 chrony。
{% endhint %}

### 配置​服务端

在​集群中，NTP 服务端指的是 `login01` 节点。找到 `/etc/chrony.conf` 文件后，按照如下方式修改：

{% code title="/etc/chrony.conf" %}
```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
allow 192.168.1.0/24

# Serve time even if not synchronized to a time source.
#local stratum 10

# Specify file containing keys for NTP authentication.
#keyfile /etc/chrony.keys

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking
```
{% endcode %}

其实​就是取消了 26 行的注释。这样做的目的是允许局域网中的节点到本节点查询。`192.168.1.0/24` 正是我们在配置网络时使用的网段。

修改​完毕后，使用 `systemctl restart chronyd` 重启 chrony。

### 配置​客户端

在​集群中，NTP 客户端指的是 除`login01` 节点外的其它节点，在本测试集群中为 `io01, c[01-02]`。找到 `/etc/chrony.conf` 文件后，按照如下方式修改（每个节点都要修改）：

{% code title="/etc/chrony.conf" %}
```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server login01 iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
#allow 192.168.0.0/16

# Serve time even if not synchronized to a time source.
#local stratum 10

# Specify file containing keys for NTP authentication.
#keyfile /etc/chrony.keys

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking
```
{% endcode %}

即​在默认脚本基础上，将第 3 - 6 行的内容注释，并添加第 7 行的内容。这样做的目的是将 login01 设置为该节点的 NTP 时间同步服务器。

修改​完毕后，使用 `systemctl restart chronyd` 重启 chrony。如果想要测试配置是否成功，可以使用 `chronyc tracking` 这个命令检查同步状态，正常会有如下的输出：

```
[root@io01 ~]# chronyc tracking
Reference ID    : C0A80164 (login01)
Stratum         : 3
Ref time (UTC)  : Thu Nov 11 13:15:00 2021
System time     : 0.000006488 seconds slow of NTP time
Last offset     : -0.000009639 seconds
RMS offset      : 0.000009639 seconds
Frequency       : 7.595 ppm slow
Residual freq   : +8.013 ppm
Skew            : 2.182 ppm
Root delay      : 0.014799952 seconds
Root dispersion : 0.000187714 seconds
Update interval : 2.0 seconds
Leap status     : Normal
```

### 总结​

这里​介绍了如何在集群上配置时间同步，最终目的是让集群所有节点的系统时间相同。

* 采用​的具体方案是登录节点从互联网获取时间，并将其共享给集群其它节点。
* 使用​的软件为 chrony，其是 ntp 的一个替代。
* 集群​中 NTP 的服务端和客户端共用一套程序和配置文件，分别修改 `/etc/chrony.conf` 即可达到效果。
