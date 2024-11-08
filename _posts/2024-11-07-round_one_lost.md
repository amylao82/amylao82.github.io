---
layout: post
title: 第一回合:被墙打败
tags:  netlib 
image: netlib.png
---

在中国,要想无缝访问整个互联网,没点翻墙本领傍身可不行.

{{ more }}

然而这段时间,与墙交手,失败了一次.
借用格斗游戏里的一句话:

```
round 1;
ready;
go;

...;

your lost

```

### 事情的来龙去脉

首先,一直自诩对操作系统和网络还是比较了解.

因此操作系统一直使用archlinux,就是看中其采用激进的升级方式,可以让自己一直接触到最新的技术.

而在网络方面,实现了基本科学上网后就认为完胜高墙.

当然,这里说的基本科学上网,有一点点曲折.因为公司有海外IP访问权.在家里使用的是科学,是借用公司的网络实现的.
要通过VPN,还要跟网管打下交道,因此使用自己的隧道,转到公司再出海.

然而高墙的这一轮操作,让我初尝败绩.

### 问题出现

问题的出现 ,最初跟科学上网一点关系都没有.

原因是我在家里的服务器里部署了一个nextcloud服务.

刚开始时,这个服务都是以源码部署的.
然后某一天发现,archlinux的软件源里把nextcloud收录了.
脑袋一激灵:使用软件源里的软件包,岂不是可以一直保持着使用最新的版本?

说干就干,一通捣鼓,把服务切换过去了.

爽了一段时间发现,因为archlinux的激进升级系统,如果要安装一个新软件,而你的操作系统没有升级的话,会无法下载到软件包.
因此,每次要安装一个新的软件时,都需要先执行一下:

```
pacman -Syu
```

而每次升级系统,nextcloud也会跟着升级.升级后需要执行

```
sudo -u nextcloud php /usr/share/webapp/nextcloud/occ upgrade
sudo -u nextcloud php /usr/share/webapp/nextcloud/occ maintenance:mode --off

```

有时候升级系统后,忘记输入上面的二条命令,服务不可用,而在手机上的app又没有提示.还不清楚是什么问题,需要在电脑浏览器里访问一次,才知道:哦,原来是还没有退出维护模式.
一次二次还好,经常这样搞,有点不胜其扰.

我就只想要一个自动备份手机相册的服务而已,其实我并不需要最新的nextcloud.


### 不幸的事情出现

既然不想要一直更新nextcloud,那想到的办法就是使用docker方式来部署.把nextcloud与archlinux的升级切割开来.

然而,因为某种原因,docker已经全面在中国大地上被禁止.
docker.io域名被墙,IP地址不通.
所有的镜像站都被关闭.
所有的网络教程上提到的方式都不再可用.

更致命的,我对docker的工作机制还保留在初级水平.


想通过公司的魔法渠道下载到镜像,在`/etc/docker/daemon.conf`里配置镜像源,网上的教程配置的是http_proxy, https_proxy, socks5_proxy.
都配置了,而且我可以确定socks5是正常工作的.

> 浏览器通过switch_omega,使用同一个socks5,可以科学上网.

网上有说配置proxy方式已经行不通.
众说纷纭,无所适从.


### 结尾

此一回合,我承认我被墙打败了.

自学的最大问题,应该是没有正向反馈.
在整个docker的配置过程中,不管如何捣鼓,所有的结果都是同一个结果,感觉不到变化.这就是没有正向反馈.


接下来,需要学习的不仅仅是网络技术,更多的是docker的相关工作机制.
下一回合,必定要再一次把墙踩在脚下.

### 另外的思考

感觉互联网世界正在对普通人关闭大门.以前只要你想学,都能在网络上找到对应的资料.
而现在,对于新技术的学习,可能更多的保存在高校里.
我相信,这次对docker的封禁,以及之前对google的封禁,应该只对应于普罗大众,对高校,也就是教育网,还是开放的.
如果对高校封禁,则所有的学术研究都无法进行.

比如,github的封禁,造成所有研发公司都无法正常运作,最后不得不放开.

在教育网内开放,而在社会封禁,会造成几个结果:

1. 学阀现象加深.高校里的当前既得利益者,乐见其成.而更进一步推波助澜.
2. 认知割裂.从高校毕业出来的学子,当时学习,研究,如鱼得水.然而出到社会后,可能会突然觉得自己怎么能力水得一B?

学习是一生的事情,并不是只在学校里那短短的二十年.
而如果把大部分人的学习权利剥夺,整个社会的进程应该会慢下来吧.

胡思乱想,胡言乱语.杞人忧天.不值一哂.一笑置之.



