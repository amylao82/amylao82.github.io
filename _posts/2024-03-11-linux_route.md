---
layout: post
title:  把电脑配置为路由器
tags:  netlib 
image: netlib.png
---

在日常的开发中,总会遇到服务器端或客户端不由你控制的情况.如果服务器端与客户端在同一局域网还好,如果两端通信需要通过路由器,那无形中数据抓包就会更为复杂.

考虑这样的一种情况,你要开发一个客户端,服务器端已经使用了已有方案,该方案由第三方提供,同时,提供方也提供了一个参考客户端,参考客户端为X86平台的程序.然而你需要在嵌入式端实现.

{{ more }}

在这样的参考方案下,因为服务器端, 路由器和客户端都无法控制,想要抓包分析极为不便.
如果要实现抓包,只能在客户端运行的电脑上,一股脑把所有通过网卡的数据包抓下来.这样的数据量就有点大了.

更进一步,如果客户端是一个嵌入式设备,只能在里面使用命令行工具`tcpdump`来抓包.
再更进一步,嵌入式设备是一个不开放的系统,你甚至无法登录进去,那只能束手无策了.

### 配置电脑作为路由器

从上面的例子中,可以得出如下网络拓扑:

![](/img/post/route_topo.png)

在这样的拓扑下,服务器端和客户端不能登录,此时,我们可以在数据的通道上做扩展,只要把路由器变为一个可以由我们控制的设备,就可以在这个路由器上,把通过的数据抓下来.


要配置电脑作为路由器,前提条件是:你的电脑必须要有二个网卡.一个网卡用于接收上级路由分配的IP地址,另外一个网卡用于组建子网.

在我的电脑里,一共有三个物理网卡, eno1用于接入上级路由器,作为一个子网设备,上级路由器会通过DHCP服务分配一个IP到该网卡.
其余二个网卡,可以用于组建子网.

```
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp2s0f0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 90:1b:0e:31:c6:02 brd ff:ff:ff:ff:ff:ff
3: enp2s0f1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 90:1b:0e:31:c6:03 brd ff:ff:ff:ff:ff:ff
4: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 18:c0:4d:df:2c:c2 brd ff:ff:ff:ff:ff:ff
    altname enp0s31f6
    inet 10.66.30.46/23 brd 10.66.31.255 scope global dynamic noprefixroute eno1
       valid_lft 14249sec preferred_lft 12449sec

```


> 如果只有一个网卡,在接入上级路由后,只能作为DHCP客户端工作,无法作为DHCP服务器提供服务
>
> 有一种观点,可以在一个物理网卡上创建虚拟网卡,以实现一个DHCP客户端,一个DHCP服务端.
>
> 但现在还不清楚如何做. 

### 路由器需要提供的服务

市面上供应的路由器,大多基于linux系统开发,通过裁剪一些不必要的部件,从而提供一个专用于网络路由的系统.

既然市面上的arm平台路由器能实现,那基于X86的linux,肯定也能实现路由器的功能.

而要想在linux电脑上实现一个路由器,就先要了解一般意义上的路由器需要提供哪此服务.

大概整理一下,普通路由器需要提供的基本功能,包括如下4个:

1. 路由功能
2. NAT转换
3. IP地址分配
4. DNS服务

#### 路由功能
路由器路由器,最重要的功能当然是路由功能了.路由功能需要对连接到该路由器下的所有设备提供路由功能,让它们之间可以相互通信.

比如,连接到路由器下的设备有`192.168.1.2`和`192.168.1.3`, 这二个设备间的相互通信,就需要路由器`192.168.1.1`来转发数据包.

#### NAT转换
在实现子网内的设备相互通过的功能同时,也需要提供网络地址转换(NAT)功能,以实现子网与上一级网络的通信.

#### IP地址分配
为了快速组网,路由器必须对接收的设备提供自动分配IP地址功能.
如果IP地址分配功能缺失,就需要用户每接入一个设备,就需要手动分配一次IP地址.

#### DNS服务
在万維网内通信,使用的是域名访问系统,路由器需要提供DNS服务,以实现万维网的访问.

### 有线网络的路由
要实现上面列出的几个功能,在archlinux的发行版里,提供有几个软件包可以实现:

1. kea
2. dnsmasq

配置方式都差不多.下面使用dnsmasq来说明如何配置.

#### 配置监听网络接口和ip地址池

dnsmasq的配置文件,保存在`/etc/dnsmasq.conf`里.列出我的配置文件里面的内容

```
#监听的网络接口,配置文件里说,如果要监听多个接口,可以写多个interface行.
#但是在写入多个时有点问题.
interface=enp2s0f1

#分配的IP地址池取值范围.
dhcp-range=192.168.0.50,192.168.0.150,12h

```

在上面这个配置文件里, 只配置了二项,一个是监听的接口,另一个是IP地址的取值范围.
配置好文件,启动dnsmasq服务

```
systemctl start dnsmasq
```

dnsmasq服务将会自动给连接到这个网络接口的设备分配IP地址.

但是在启动dnsmasq服务前,需要手动给网络接口分配一个IP地址.

```
ip addr add 192.168.0.1/24 dev enp2s0f1
```

如果不分配地址,dnsmasq无法分配IP地址.

> 相比kea,其配置文件`kea-dhcp4.conf`里, 配置监听的网络接口时,需要写上对应的IP地址.
> ```
> "Dhcp4": {
>     // Add names of your network interfaces to listen on.
>     "interfaces-config": {
>         // See section 8.2.4 for more details. You probably want to add just
>         "interfaces": [ "enp2s0f1/192.168.0.1" ]
> ```


*我也一直在研究不需要配置网络接口IP地址的方法,但是还没有进展.*

配置完成后,只是完成了任务1(路由功能),任务3(IP地址分配),而NAT转换,还需要进一步配置.

#### 配置转发功能

要在linux里设置ip转发功能.可以通过命令 `sysctl net.ipv4.ip_forward=1` 把转发功能打开.

而转发数据包转发到哪里呢? 那就是把enp2s0f1的数据,转发到eno1网卡上,把eno1的数据,转发到enp2s0f1上,这样就实现了数据的交换.

上面通过命令打开ip_forward功能,其最终修改的文件是 `/proc/sys/net/ipv4/ip_forward`.

可以通过在配置文件 `/etc/sysctl.d/99-sysctl.conf` 写上如下行, 开机后就会自动设置这个ip_forward值

```
net.ipv4.ip_forward=1
```

> 在archlinux里, 使用linux_zen内核,这个值默认是打开的.
>
> 而在ubuntu 22.04里,默认关闭.
>
> 配置时因地制宜,不一而足.


开启转发功能后,还需要在iptables里,把子网发过来的数据包,转发到连接上一级网络的网络接口上.

因为我配置的子网IP地址在`192.168.0.0/24`段,因此把从这个网段过来的数据,转发到eno1网络接口上,如果配置的网段不是此网段,对命令作相对应的修改.

```
 iptables -P INPUT ACCEPT
 iptables -P FORWARD ACCEPT
 iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eno1 -j MASQUERADE
```


#### 配置DNS

DNS的配置使用默认即可正常工作.但是如果有特殊的需求,可以在`dnsmasq.conf`里配置,在里面搜索关键字`resolv`, 里面有提示如何配置,让连接子设备生成`/etc/resolv.conf`文件.通过这个文件里的配置,可以提供域名服务.

通常地,DNS服务器地址就是路由器的地址,所以不用做特殊的配置. 但是如果在一个复杂的网络里,DNS服务器会部署在不同的硬件服务器里,这时就要对resolv配置项进行配置.



### 配置时遇到的问题

在配置的过程中,碰到几个问题,除了上面说到的需要预先给监听网卡一个IP地址外,还有另外一个问题:我想同时监听二个网络接口,类似到路由器一个wan口,多个lan口的功能.

但是在配置了多个监听网络接口后,因为需要预先分配IP地址,难道这二个网络接口预先分配的IP地址相同吗?
经过测试,分配了相同的IP地址后,无法为子网连接设备分配IP地址.

但是如果每个网络接口分配不同的IP地址,假设一个是`192.168.0.1`, 另一个是`192.168.0.2`,那么,在C类网络地址里,岂不是在路由器里占用一半的网络地址,给子网分配的地址点另外一半?这样带机量不是减少一半?

再者,在这样的网络内,连接到不同的网络接口的子设备,如何相互通信?

因此,我怀疑路由器都是只有一个子网口,家庭路由器可能是一个路由器和一个交换机结合的混合设备.
如图:

![](/img/post/SOHO_ROUTE.png)

不知是否如此,既然我在本机上无法配置多个网络接口分配为同一个网段的IP地址,那就只好把每个接口都配置为一个独立的子网了.


### 无线网络的路由

如果要配置无线路由器,功能与上面所列内容并无二致.唯无线网卡默认都工作在sta模式,需要把无线网卡切换为softap模式,才能对外提供IP分配服务.

要切换无线网卡的工作模式,可以通过`hostapd`服务:
```
pacman -S hostapd
```

hostapd的配置文件如下,需要配置热点名称,加密协议,密码.


```
##### hostapd configuration file ##############################################
# Empty lines and lines starting with # are ignored

# AP netdevice name (without 'ap' postfix, i.e., wlan0 uses wlan0ap for
# management frames with the Host AP driver); wlan0 with many nl80211 drivers
# Note: This attribute can be overridden by the values supplied with the '-i'
# command line parameter.
##interface=wlp0s20f0u3
interface=wlp0s20f0u11u1

# 要在 IEEE 802.11 管理框架中使用的 SSID（即热点名）
ssid=amy_ac1200
# 驱动接口类型（hostap/wired/none/nl80211/bsd）
driver=nl80211
# 国家或地区代码（ISO/IEC 3166-1）
country_code=US

# 工作模式 (a = IEEE 802.11a (5 GHz), b = IEEE 802.11b (2.4 GHz)
hw_mode=g
# 使用信道
channel=7
# 允许最大连接数
max_num_sta=5

# Bit 字段：bit0 = WPA, bit1 = WPA2
wpa=2
# Bit 字段：1=wpa, 2=wep, 3=both
auth_algs=1

# 加密协议；禁用不安全的 TKIP
wpa_pairwise=CCMP
# 加密算法
wpa_key_mgmt=WPA-PSK
wpa_passphrase=amy_test

# hostapd日志设置
logger_syslog=-1
logger_syslog_level=2
logger_stdout=-1
logger_stdout_level=2
debug=4

# 802.11n设备请取消注释并修改以下部分
## 启用802.11n支持
ieee80211n=1
## QoS 支持
#wmm_enabled=1
## 请使用“iw list”查看设备信息并相应地修改 ht_capab
#ht_capab=[HT40+][SHORT-GI-40][TX-STBC][RX-STBC1][DSSS_CCK-40]

```


> 一直以来,我都以为在linux里安装新硬件是不需要安装驱动的.
>
> 但是这个无线网卡,才让我认识到我的认知错误,之所以平时我们的硬件一连接上Linux就可以正常工作,是因为linux里购置了超多的硬件驱动程序.
>
> 使用无线网卡,芯片是rtl8812,在linux下,无法正常工作,原因是没有对应的驱动.
>
> 从网上找到这个芯片的驱动 https://github.com/morrownr ,下载编译安装后,该无线网卡才能正常工作.


### 简单配置流程

1. 安装dnsmasq, `pacman -S dnsmasq`
2. 编辑配置文件.
3. 增加IP地址.`ip addr add 192.168.0.1 dev enp2s0f1`
4. 启动服务 `systemctl start dnsmasq`
5. 配置转发





