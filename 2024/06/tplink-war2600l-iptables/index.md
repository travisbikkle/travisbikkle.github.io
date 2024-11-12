# TP-LINK WAR-2600L 路由器 DMZ 功能分析


## 问题

| 硬件型号   | TL-WAR2600L v2.0 企业路由器      |
|--------|-----------------------------|
| 固件版本   | 1.0.1 Build 20180313 Rel.37492               |

本地150自提了这个路由器，准备用它的多 wan 口搞搞事情，但是没想到掉坑里了：

1. 80, 443 端口使用宽带拨号的外网 ip 可以访问（不安全）
   ```txt
   在局域网内输入外网宽带 ip，可以访问路由器管理界面；
   但是在真正的外网上，比如手机不连局域网，访问宽带 ip，实际上又不能访问（我的 ip 是公网 ip）。 
   ```
2. 虚拟服务器转发的端口，从外网无法基本全部不可用

## 分析过程
### 获取 ssh 端口号和密码，登录到路由器后台
1. `系统工具-设备管理-备份与导入设置`，导出一个 bin 文件。使用 7z 打开，查看如下路径 `\backup-TP-LINK-2024-06-20\tmp\userconfig\etc\config\dropbear`
   ```bash
   config dropbear
       option PasswordAuth &#39;on&#39;
       option RootPasswordAuth &#39;on&#39;
       option Port &#39;33400&#39; # 这个就是端口
       option ssh_port_switch &#39;on&#39;
   ```
2. 在路由器管理页面 `基本设置 - LAN设置` 中获取 mac 地址，然后找一个 linux 命令行工具执行
   ```bash
   ## xx-xx-xx-xx-xx-xx 是你找到的 mac 地址。注意 -n
   echo -n &#34;xx-xx-xx-xx-xx-xx&#34;|sed &#34;s/-//g&#34;|md5sum|cut -c 1-8
   xxxxxxxx # 8位密码
   ```
   登录后我其实还没发现（因为我不折腾 openwrt，有群晖和一台2U机架服务器了，路由器我只追求稳定），
   这个路由器用的是 openwrt barrier breaker（龙妈：我是 chain breaker） 14.07。 
   后来是在与它斗智斗勇的过程中，才发现它的真身。

### 问题逐一分析

#### 问题一 80, 443 端口使用宽带拨号的外网 ip 可以访问（不安全）
由于看到了以下这两条规则，走 `br-lan` 网卡（lan口，局域网）的流量可以访问管理界面，走 `pe-wan1-poe`（外网）已经做了拒绝处理，因此可以认为是安全的。
```bash
root@TP-LINK:~# iptables-save|grep remote_mngt_chain
:remote_mngt_chain - [0:0]
-A INPUT -j remote_mngt_chain
-A remote_mngt_chain -i br-lan -p tcp -m multiport --dports 80,443,23,33400 -j ACCEPT
-A remote_mngt_chain -i pe-wan1_poe -p tcp -m multiport --dports 80,443,23,33400 -j REJECT --reject-with icmp-port-unreachable
```

但我仍然决定关闭外网访问，因为
1. 没有这个需求
2. 使用 `nmap x.x.x.x -p-` 是可以扫描到这个端口的，这无疑增加了攻击者的兴趣。x.x.x.x 是外网 ip
   ```bash
   yu@kde:~$ nmap x.x.x.x -p-
   Starting Nmap 7.80 ( https://nmap.org ) at xxxx-xx-xx xx:xx CST
   Nmap scan report for x.x.x.x
   Host is up (0.027s latency).
   Not shown: 65527 closed ports
   PORT      STATE    SERVICE
   1723/tcp  filtered pptp                # vpn 服务器
   1900/tcp  open     upnp                # upnp
   20002/tcp open     commtact-http       # tplink 的远程技术支持，其实不是 commtact-http
   33400/tcp open     unknown             # ssh 端口
   ....                                   # 一些自己开的端口，略过
   Nmap done: 1 IP address (1 host up) scanned in 4.66 seconds
   ```
3. 万一不小心执行一些命令，如 `iptables -X`，把这条 iptables 规则给删了，在部分没有禁用 80 端口的地区，还是可以访问的
4. 担心攻击不是杞人忧天，我的任何一台服务器，包括云服务器和家里面的服务器，每天都有成千上万条各种形式的攻击记录，现在的攻击手段完全自动和无差别攻击的，在你服务器部署后的数秒钟之内就会有各种攻击纷至沓来

编辑配置文件 `vi /etc/config/uhttpd`
```bash
config uhttpd &#39;main&#39;
        list listen_http &#39;192.168.0.1:80&#39; # 原来是 0.0.0.0，注意有些人的 ip 是 192.168.1.1 等，而且会变，改成自己对应的ip，而且自己要记住，否则后面自己忘了，ssh也关了，路由器就很难登录了
        list listen_http &#39;[::1]:80&#39;       # 原来是[::]:80
        list listen_https &#39;192.168.0.1:443&#39;
        list listen_https &#39;[::1]:443&#39;
```
使用 `/etc/init.d/uhttpd restart` 重启 uhttpd，再次使用 nmap 扫描 80 端口就没了。

#### 问题二 DMZ 转发的端口，从外网无法基本全部不可用

1. 排除法定位问题出在哪里    
   1.1 首先证明，我的服务是正常的
      ```absh
      局域网内的主机，无法访问 www.zyming.cn:5000 端口
      ping www.zyming.cn 指向 180.111.103.62
      局域网内的主机，可以 ping 通 www.zyming.cn 和 180.111.103.62
      局域网内的主机，无法访问 180.111.103.62:5000 端口
      虚拟服务器配置转发规则 5000 端口到内网 192.168.0.100:5000
      局域网内的主机，可以访问 192.168.0.100:5000 端口 ## 服务正常
      ```
   1.2 其次域名服务是正常的
      ``` 
      手机不连局域网，直接手机网络上网
      手机浏览器可以访问 https://www.zyming.cn:5000，可以看到群晖报错您所访问的页面不存在，证明可以连上群晖，只是不知道哪里有问题。手机群晖照片应用也不可以用。
      手机可以访问 https://180.111.103.62:5000，可以登录群晖，手机群晖照片应用可以用。
      手机可以使用 RD client （远程桌面，3389端口）远程控制 www.zyming.cn
      手机可以使用 RD client （远程桌面，3389端口）远程控制 180.111.103.62
      证明域名服务商没有封锁我的端口，外网访问正常。
      证明路由器没有功能问题，DMZ 端口转发没有固件层面的问题。
      ```
2. 怀疑方向1（方向错误）   
   目前有充足的理由怀疑路由器的的 iptables 规则设置有问题。  
   当添加一条虚拟服务器规则，iptables 就会出现以下四条规则。
   ```bash
   root@TP-LINK:~# iptables-save|grep 5000
   -A postrouting_rule_vs -s 192.168.0.0/24 -d 192.168.0.100/32 -p tcp -m tcp --dport 35000 -m comment --comment home_synology -j SNAT --to-source 180.111.103.62
   -A postrouting_rule_vs -s 192.168.0.0/24 -d 192.168.0.100/32 -p udp -m udp --dport 35000 -m comment --comment home_synology -j SNAT --to-source 180.111.103.62
   -A prerouting_rule_vs -d 180.111.103.62/32 -p tcp -m tcp --dport 35000 -m comment --comment home_synology -j XDNAT --to-destination
   -A prerouting_rule_vs -d 180.111.103.62/32 -p udp -m udp --dport 35000 -m comment --comment home_synology -j XDNAT --to-destination
   root@TP-LINK:/etc# cat /proc/sys/net/ipv4/ip_forward
   ```
   这个问题应该是网管圈或折腾 openwrt 的人非常熟悉的问题，也就是 nat 不能回流了。
   
   原理如下：
   ```text
   主机A 192.168.0.3 访问主机B 121.237.44.219(内网192.168.0.2)，需要经过路由C 192.168.0.1
   原因：
   第一步 192.168.0.3 访问     121.237.44.219              [src:192.168.0.3, dst:121.237.44.219]
   第二步 192.168.0.1 替换目标                              [src:192.168.0.3, dst:192.168.0.2]
   第三步 192.168.0.2 接收数据                              [src:192.168.0.3, dst:192.168.0.2]
   第四步 192.168.0.2 返回     192.168.0.3                 [src:192.168.0.2, dst:192.168.0.3]
   第五步 192.168.0.3 收到返回，但是不是来自 121.237.44.219    [src:192.168.0.2, dst:192.168.0.3]，所以丢弃掉包
   第六步 192.168.0.2 一直等不到 192.168.0.2 响应
   形成了三角访问，而不是A-&gt;C-B然后B-&gt;C-&gt;A
      C   
      ^
    /   \
   A  &lt;-  B
   
   去 A -&gt; C -&gt; B
   回 B -&gt; A
   
   解决的关键：
   在于让主机B发送数据给C，C再发送给A。
   
   第一步 192.168.0.3 访问     121.237.44.219              [src:192.168.0.3, dst:121.237.44.219]
   第二步 192.168.0.1 替换源和目标                           [src:192.168.0.1, dst:192.168.0.2]
   第三步 192.168.0.2 接收数据                              [src:192.168.0.1, dst:192.168.0.2]
   第四步 192.168.0.2 返回     192.168.0.1                 [src:192.168.0.2, dst:192.168.0.1]
   第五步 192.168.0.1 返回     192.168.0.3                 [src:121.237.44.219, dst:192.168.0.3]
   第六步 192.168.0.3 收到数据包，机器ip正确，成功
   ```
   根据上面得出的结论，将 postrouting 相关规则的 --to-source 修改为网关地址，但是问题没有解决。仔细查看 iptables 规则，TP-LINK 添加了一条
   ```text
   -A postrouting_rule_multi_nat -s 192.168.0.0/24 -o pe-wan1_poe -m comment --comment NAT_LAN_WAN1 -j MASQUERADE
   ```
   该条规则应该已经考虑到了这种三角访问的情况，所以上面的结论是错误的。

3. 怀疑方向2（方向错误）
   那么是不是在 filter 表做了限制？
   我在 filter 表中找到相关的配置，发现并没有任何的限制。
   ```text
   -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
   -A FORWARD -j forward
   -A input -i br-lan -j zone_lan_input
   -A input -i pe-wan1_poe -j zone_wan1_poe_input
   -A forward -i br-lan -j zone_lan_forward # zone_lan_forward 是空
   -A forward -i pe-wan1_poe -j zone_wan1_poe_forward
   ```
   那么是不是 napt 配置有问题？

4. 怀疑方向3（方向错误） napt 的折腾过程
   我看到网上有人通过配置一个出接口为 LAN 口的 napt 规则，解决了这个问题。可是我的路由器在配置 napt 的时候没有 LAN 的选项，只有 WAN。
   尝试手动在以下配置文件中加入一条规则，无效。
   ```text
   root@TP-LINK:/etc/config# cat nat
   
   config default &#39;default&#39;
   	option zones &#39;LAN WAN1&#39;
   	option prerouting &#39;vs one_dmz dmz&#39;
   	option postrouting &#39;one_nat multi_nat vs&#39;
   	list network &#39;LAN-LAN&#39;
   	list network &#39;WAN1-WAN1&#39;
   
   config nat_global &#39;global&#39;
   	option enable &#39;on&#39;
   
   config nat_alg
   	option pptp &#39;on&#39;
   	option l2tp &#39;off&#39;
   	option tftp &#39;off&#39;
   	option ftp &#39;on&#39;
   	option h323 &#39;on&#39;
   	option ipsec &#39;on&#39;
   	option sip &#39;off&#39;
   
   config rule_napt
   	option name &#39;NAT_LAN_WAN1&#39;
   	option enable &#39;on&#39;
   	option mask &#39;24&#39;
   	option interface &#39;WAN1&#39;
   	option sysrule &#39;1&#39;
   	option ipaddr &#39;192.168.0.0&#39;
   
   config rule_vs
   	option name &#39;home_synology&#39;
   	option protocol &#39;ALL&#39;
   	option enable &#39;on&#39;
   	option external_port &#39;35000&#39;
   	option ipaddr &#39;192.168.0.100&#39;
   	option interface &#39;WAN1&#39;
   	option internal_port &#39;35000&#39;
   # 这里新增了一个 nat
   config rule_napt
   	option name &#39;NAT_LAN_LAN1&#39;
   	option enable &#39;on&#39;
   	option mask &#39;24&#39;
   	option ipaddr &#39;192.168.0.0&#39;
   	option interface &#39;LAN&#39;
   	option sysrule &#39;1&#39;
   ```
   是不是还有没有添加的地方？
   死马当活马医，尝试修改下管理页面的逻辑，放开这个路由器在配置 napt 时出接口的限制。虽然我知道这么做大概率没用，改了页面，后台没脚本支持，也是没用。
   但是没其它思路，试试看吧。在路由器里面修改不方便，不好找要修改哪些位置，所以给它用 U 盘拷下来。插拔 U 盘，确认硬盘。`cat /proc/partitions`
   ```text
   root@TP-LINK:/www/webpages/pages/userrpm# cat /proc/partitions 
   major minor  #blocks  name
   
     31        0         64 mtdblock0
     31        1        128 mtdblock1
   root@TP-LINK:/www/webpages/pages/userrpm# 
   major minor  #blocks  name
   
     31        0         64 mtdblock0
     31        1        128 mtdblock1
      8        0   60538880 sda  # 插 U 盘新增的分区，所以 U 盘是 /dev/sda
      8        1   60537824 sda1
   ```
   尝试挂载 U 盘，发现 U 盘已经自动挂载到了 /home/ftp/volume1（害，做的都是无用功）。
   
   执行命令打包管理页面。
   ```bash
   tar zcvf /home/ftp/volume1/www.tgz /www/
   ```
   拿到PC上改代码，这就到了我们的强项了，快速找到几个限制出接口的地方，给他改掉（这里我有疑问，这些地方看起来是官方故意去掉不支持 lan 的，不知道是不是企业路由器都是这个逻辑）。
   
   | 功能        | 代码文件                                           | 行号  |
   |-----------|------------------------------------------------|-----|
   | NAT设置     | www\webpages\pages\userrpm\napt.html           | 321 |
   | NAT-DMZ设置 | www\webpages\pages\userrpm\nat_dmz.html        | 185 |
   | 虚拟服务器设置   | www\webpages\pages\userrpm\virtual_server.html | 652 |
   
   基本都长这个样子，把 if 判断注释掉。
   ```javascript
   for (i=0; i&lt;data.normal.length; i&#43;&#43;){
       // if(data.normal[i].t_name != &#34;LAN&#34;){
           interfaceItem.push({name:data.normal[i].t_name,value:data.normal[i].t_name});
   	   // }
   }
   ```
   打开前台页面，此时能够在这几个地方选择出接口为 LAN 了，并且尝试添加 napt 和 虚拟服务器规则，发现后台都能生成相应的 napt 配置或 iptables 规则。
   
   ![20240621-tplink-1.png](/images/posts/20240621-tplink-1.png)
   
   但是没有用，没有解决我的问题，而且看起来它生成的 LAN 口的规则还是错的，这下没法了，开启长考模式——睡觉。

5. 怀疑方向4 rebind-protection（部分正确）
   第二天醒来，想到（为什么才想到）既然是 openwrt，那么为什么不搜 openwrt 相关的问题。果然出来很多结果，其中就有人提到这个 openwrt 默认开启的安全相关的问题。
   在我的场景下（局域网自己用），这个配置非常愚蠢，不再描述其含义，只说操作和结果。
   首先，这个路由器没有 openwrt 系统中的配置，没有一个小框框，点一点就能关掉这个功能，但是因为它是 dnsmasq 的能力，因此可以通过如下方式关闭：
   ```bash
   vi /etc/config/dhcp # 配置文件的位置
   config dnsmasq
        option domainneeded &#39;1&#39;
        option port &#39;0&#39;
        option boguspriv &#39;1&#39;
        option filterwin2k &#39;0&#39;
        option localise_queries &#39;1&#39;
        option rebind_protection &#39;0&#39; # 搜索 rebind_protection，将其从 1 修改为 0
   /etc/init.d/dnsmasq restart # 重启
   ```
   访问公网ip:端口，依旧无效。
   
   焦灼。只能考虑 tcpdump 抓包了，不折腾的原则又要破坏了。但是这个路由器年限有点老，用的 barrier breaker 14.07 的源地址已经换了，下载下来 ipx 包直接本地安装发现装不上。

   去看看官方有没有解决吧。找到了他们 2021 年发布的一个 2019 的升级包，好家伙，直接报错无法升级，开始对 TP-LINK 心生怨念。

6. 猜猜怎么解决的，最后联系官方，他们给发了一个升级固件。
   固件原理待分析。

## 备忘：iptables 四表五链

| raw        | mangle      | nat         | filter  | 四表 |
|------------|-------------|-------------|---------|----|
| prerouting | prerouting  | prerouting  | input   | 五链 |
| output     | postrouting | postrouting | forward |    |
|            | input       | output      | output  |    |
|            | output      | input       |         |    |
|            | forward     |

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2024/06/tplink-war2600l-iptables/  

