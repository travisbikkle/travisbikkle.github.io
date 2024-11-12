# 一个数据库主备频繁切换案例


## 问题现象

某集团某局点分别部署了某数据库的主备实例，业务发现偶尔访问数据库报错。

## 定位结论

其中一个节点使用的某物理机型，网卡存在故障，发生了多次复位，致使某数据库备机的keepalived双机软件接收不到对端的vrrp信号，备机在网卡复位期间短暂升主，两台机器都出现了vip，导致业务访问某数据库失败。

需要联系厂商，针对以下疑问给出答复：

1. 节点A（某品牌F）和节点B（某品牌L）使用的网卡型号一样，但是为什么某品牌F的固件版本要比某品牌L的新？

2. 某品牌L的网卡信息，和某品牌L官网该型号的机器的认证信息不一致，见下表。

表1-1 某品牌F机型、某品牌L机型和某品牌L官网认证信息对比
| 项目           | 节点A                      | 节点B                   | 某品牌L官网认证信息     |
|--------------|----------------------------|-------------------------|-------------------------|
| 服务器型号     | 某品牌F FiberHome R2200 V5 | 某品牌L Inspur NF8480M5 | 某品牌L Inspur NF8480M5 |
| 网卡驱动       | ixgbe 5.1.0-k              | ixgbe 5.1.0-k           | igb.ko 5.6.0-k          |
| 网卡固件       | 0x800006da 1.1824.0        | 0x800003df              | -                       |
| 是否发生过重启 | 否                         | 是                      | -                       |

## 定位过程
查看两个节点的keepalived日志

```bash
cd /xx/xx/xx/logs
zgrep Entering keepalived*
```
发现主机（节点A）上无异常，备机的keepalived在master、backup、fault三个状态频繁切换。

&gt; 当keepalived进入master状态，会将vip绑定到网卡上。这样主机和备机都有vip，业务访问某数据库将会出现问题。

在备机上使用如下命令可以抓到短暂出现的vip：
```bash
while true;do ip a s bond1|grep x.x.x.x;sleep 0.2;done
```

查看备机的keepalived.log，发现备机无法接收到主机的vrrp广播。

&gt; keepalived的主备机都会每3s发送一次vrrp协议的广播，当备机连续3次（9s）接收不到主机的广播，就会将自己升主。

查看主机的keepalived.log，发现主机在正常发送vrrp广播。

使用tcpdump在主机和备机同时抓包，问题复现后，发现主机确实发送了vrrp的广播，但是备机在(15:46:14-15:46:32)有18s的时间没有接收到任何来自主机的广播（此处应该有图，但是不放了）。

使用如下命令，查看备机上keepalived接收vrrp广播的网卡情况，发现是一张由eth4和eth8组成的active-backup模式的逻辑网卡，：

```bash
cat /proc/net/bonding/bond1
```

查看/var/log/messages中两张物理网卡的信息，发现eth4和eth8网卡发生了多次复位，复位时间和gauss-datacube出现异常的时间能够对应的上（此处应该有图，但是不放了）。

查看节点B的服务器型号及网卡型号、固件版本（此处应该有图，但是不放了）。
节点B和节点A对比，发现节点A的这台某品牌F机型，比节点B的某品牌L机型使用的网卡固件要更新：
节点B和某品牌L官网认证信息(https://www.suse.com/nbswebapp/yesBulletin.jsp?bulletinNumber=148996)对比，发现节点B节点使用的网卡driver name和认证信息不符：
除了某局点节点B的某品牌L机型节点，我们还在其它局点的appdb的节点的某品牌L机型上，也发现了网卡重启的现象，该现象在某品牌F机型上未发现，判断是某品牌L的这批机器网卡存在故障。
这种短暂网卡重启的问题，由于tcp重连的特性，对于普通的应用感知不大，但是对于数据库这样包含双机软件的敏感应用，影响较大。

最终客户联系厂商定位到是硬件故障。

 


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2022/01/a-case-of-database-switch/  

