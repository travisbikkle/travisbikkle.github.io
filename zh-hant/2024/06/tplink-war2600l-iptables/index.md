# TP-LINK WAR-2600L 路由器 DMZ 功能分析


## 問題

| 硬件型號   | TL-WAR2600L v2.0 企業路由器      |
|--------|-----------------------------|
| 固件版本   | 1.0.1 Build 20180313 Rel.37492               |

本地150自提了這個路由器，準備用它的多 wan 口搞搞事情，但是沒想到掉坑裏了：

1. 80, 443 端口使用寬帶撥號的外網 ip 可以訪問（不安全）
   ```txt
   在局域網內輸入外網寬帶 ip，可以訪問路由器管理界面；
   但是在真正的外網上，比如手機不連局域網，訪問寬帶 ip，實際上又不能訪問（我的 ip 是公網 ip）。 
   ```
2. 虛擬服務器轉發的端口，從外網無法基本全部不可用

## 分析過程
### 獲取 ssh 端口號和密碼，登錄到路由器後臺
1. `系統工具-設備管理-備份與導入設置`，導出一個 bin 文件。使用 7z 打開，查看如下路徑 `\backup-TP-LINK-2024-06-20\tmp\userconfig\etc\config\dropbear`
   ```bash
   config dropbear
       option PasswordAuth &#39;on&#39;
       option RootPasswordAuth &#39;on&#39;
       option Port &#39;33400&#39; # 這個就是端口
       option ssh_port_switch &#39;on&#39;
   ```
2. 在路由器管理頁面 `基本設置 - LAN設置` 中獲取 mac 地址，然後找一個 linux 命令行工具執行
   ```bash
   ## xx-xx-xx-xx-xx-xx 是你找到的 mac 地址。注意 -n
   echo -n &#34;xx-xx-xx-xx-xx-xx&#34;|sed &#34;s/-//g&#34;|md5sum|cut -c 1-8
   xxxxxxxx # 8位密碼
   ```
   登錄後我其實還沒發現（因爲我不折騰 openwrt，有羣暉和一臺2U機架服務器了，路由器我只追求穩定），
   這個路由器用的是 openwrt barrier breaker（龍媽：我是 chain breaker） 14.07。 
   後來是在與它鬥智鬥勇的過程中，才發現它的真身。

### 問題逐一分析

#### 問題一 80, 443 端口使用寬帶撥號的外網 ip 可以訪問（不安全）
由於看到了以下這兩條規則，走 `br-lan` 網卡（lan口，局域網）的流量可以訪問管理界面，走 `pe-wan1-poe`（外網）已經做了拒絕處理，因此可以認爲是安全的。
```bash
root@TP-LINK:~# iptables-save|grep remote_mngt_chain
:remote_mngt_chain - [0:0]
-A INPUT -j remote_mngt_chain
-A remote_mngt_chain -i br-lan -p tcp -m multiport --dports 80,443,23,33400 -j ACCEPT
-A remote_mngt_chain -i pe-wan1_poe -p tcp -m multiport --dports 80,443,23,33400 -j REJECT --reject-with icmp-port-unreachable
```

但我仍然決定關閉外網訪問，因爲
1. 沒有這個需求
2. 使用 `nmap x.x.x.x -p-` 是可以掃描到這個端口的，這無疑增加了攻擊者的興趣。x.x.x.x 是外網 ip
   ```bash
   yu@kde:~$ nmap x.x.x.x -p-
   Starting Nmap 7.80 ( https://nmap.org ) at xxxx-xx-xx xx:xx CST
   Nmap scan report for x.x.x.x
   Host is up (0.027s latency).
   Not shown: 65527 closed ports
   PORT      STATE    SERVICE
   1723/tcp  filtered pptp                # vpn 服務器
   1900/tcp  open     upnp                # upnp
   20002/tcp open     commtact-http       # tplink 的遠程技術支持，其實不是 commtact-http
   33400/tcp open     unknown             # ssh 端口
   ....                                   # 一些自己開的端口，略過
   Nmap done: 1 IP address (1 host up) scanned in 4.66 seconds
   ```
3. 萬一不小心執行一些命令，如 `iptables -X`，把這條 iptables 規則給刪了，在部分沒有禁用 80 端口的地區，還是可以訪問的
4. 擔心攻擊不是杞人憂天，我的任何一臺服務器，包括雲服務器和家裏面的服務器，每天都有成千上萬條各種形式的攻擊記錄，現在的攻擊手段完全自動和無差別攻擊的，在你服務器部署後的數秒鐘之內就會有各種攻擊紛至沓來

編輯配置文件 `vi /etc/config/uhttpd`
```bash
config uhttpd &#39;main&#39;
        list listen_http &#39;192.168.0.1:80&#39; # 原來是 0.0.0.0，注意有些人的 ip 是 192.168.1.1 等，而且會變，改成自己對應的ip，而且自己要記住，否則後面自己忘了，ssh也關了，路由器就很難登錄了
        list listen_http &#39;[::1]:80&#39;       # 原來是[::]:80
        list listen_https &#39;192.168.0.1:443&#39;
        list listen_https &#39;[::1]:443&#39;
```
使用 `/etc/init.d/uhttpd restart` 重啓 uhttpd，再次使用 nmap 掃描 80 端口就沒了。

#### 問題二 DMZ 轉發的端口，從外網無法基本全部不可用

1. 排除法定位問題出在哪裏    
   1.1 首先證明，我的服務是正常的
      ```absh
      局域網內的主機，無法訪問 www.zyming.cn:5000 端口
      ping www.zyming.cn 指向 180.111.103.62
      局域網內的主機，可以 ping 通 www.zyming.cn 和 180.111.103.62
      局域網內的主機，無法訪問 180.111.103.62:5000 端口
      虛擬服務器配置轉發規則 5000 端口到內網 192.168.0.100:5000
      局域網內的主機，可以訪問 192.168.0.100:5000 端口 ## 服務正常
      ```
   1.2 其次域名服務是正常的
      ``` 
      手機不連局域網，直接手機網絡上網
      手機瀏覽器可以訪問 https://www.zyming.cn:5000，可以看到羣暉報錯您所訪問的頁面不存在，證明可以連上羣暉，只是不知道哪裏有問題。手機羣暉照片應用也不可以用。
      手機可以訪問 https://180.111.103.62:5000，可以登錄羣暉，手機羣暉照片應用可以用。
      手機可以使用 RD client （遠程桌面，3389端口）遠程控制 www.zyming.cn
      手機可以使用 RD client （遠程桌面，3389端口）遠程控制 180.111.103.62
      證明域名服務商沒有封鎖我的端口，外網訪問正常。
      證明路由器沒有功能問題，DMZ 端口轉發沒有固件層面的問題。
      ```
2. 懷疑方向1（方向錯誤）   
   目前有充足的理由懷疑路由器的的 iptables 規則設置有問題。  
   當添加一條虛擬服務器規則，iptables 就會出現以下四條規則。
   ```bash
   root@TP-LINK:~# iptables-save|grep 5000
   -A postrouting_rule_vs -s 192.168.0.0/24 -d 192.168.0.100/32 -p tcp -m tcp --dport 35000 -m comment --comment home_synology -j SNAT --to-source 180.111.103.62
   -A postrouting_rule_vs -s 192.168.0.0/24 -d 192.168.0.100/32 -p udp -m udp --dport 35000 -m comment --comment home_synology -j SNAT --to-source 180.111.103.62
   -A prerouting_rule_vs -d 180.111.103.62/32 -p tcp -m tcp --dport 35000 -m comment --comment home_synology -j XDNAT --to-destination
   -A prerouting_rule_vs -d 180.111.103.62/32 -p udp -m udp --dport 35000 -m comment --comment home_synology -j XDNAT --to-destination
   root@TP-LINK:/etc# cat /proc/sys/net/ipv4/ip_forward
   ```
   這個問題應該是網管圈或折騰 openwrt 的人非常熟悉的問題，也就是 nat 不能迴流了。
   
   原理如下：
   ```text
   主機A 192.168.0.3 訪問主機B 121.237.44.219(內網192.168.0.2)，需要經過路由C 192.168.0.1
   原因：
   第一步 192.168.0.3 訪問     121.237.44.219              [src:192.168.0.3, dst:121.237.44.219]
   第二步 192.168.0.1 替換目標                              [src:192.168.0.3, dst:192.168.0.2]
   第三步 192.168.0.2 接收數據                              [src:192.168.0.3, dst:192.168.0.2]
   第四步 192.168.0.2 返回     192.168.0.3                 [src:192.168.0.2, dst:192.168.0.3]
   第五步 192.168.0.3 收到返回，但是不是來自 121.237.44.219    [src:192.168.0.2, dst:192.168.0.3]，所以丟棄掉包
   第六步 192.168.0.2 一直等不到 192.168.0.2 響應
   形成了三角訪問，而不是A-&gt;C-B然後B-&gt;C-&gt;A
      C   
      ^
    /   \
   A  &lt;-  B
   
   去 A -&gt; C -&gt; B
   回 B -&gt; A
   
   解決的關鍵：
   在於讓主機B發送數據給C，C再發送給A。
   
   第一步 192.168.0.3 訪問     121.237.44.219              [src:192.168.0.3, dst:121.237.44.219]
   第二步 192.168.0.1 替換源和目標                           [src:192.168.0.1, dst:192.168.0.2]
   第三步 192.168.0.2 接收數據                              [src:192.168.0.1, dst:192.168.0.2]
   第四步 192.168.0.2 返回     192.168.0.1                 [src:192.168.0.2, dst:192.168.0.1]
   第五步 192.168.0.1 返回     192.168.0.3                 [src:121.237.44.219, dst:192.168.0.3]
   第六步 192.168.0.3 收到數據包，機器ip正確，成功
   ```
   根據上面得出的結論，將 postrouting 相關規則的 --to-source 修改爲網關地址，但是問題沒有解決。仔細查看 iptables 規則，TP-LINK 添加了一條
   ```text
   -A postrouting_rule_multi_nat -s 192.168.0.0/24 -o pe-wan1_poe -m comment --comment NAT_LAN_WAN1 -j MASQUERADE
   ```
   該條規則應該已經考慮到了這種三角訪問的情況，所以上面的結論是錯誤的。

3. 懷疑方向2（方向錯誤）
   那麼是不是在 filter 表做了限制？
   我在 filter 表中找到相關的配置，發現並沒有任何的限制。
   ```text
   -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
   -A FORWARD -j forward
   -A input -i br-lan -j zone_lan_input
   -A input -i pe-wan1_poe -j zone_wan1_poe_input
   -A forward -i br-lan -j zone_lan_forward # zone_lan_forward 是空
   -A forward -i pe-wan1_poe -j zone_wan1_poe_forward
   ```
   那麼是不是 napt 配置有問題？

4. 懷疑方向3（方向錯誤） napt 的折騰過程
   我看到網上有人通過配置一個出接口爲 LAN 口的 napt 規則，解決了這個問題。可是我的路由器在配置 napt 的時候沒有 LAN 的選項，只有 WAN。
   嘗試手動在以下配置文件中加入一條規則，無效。
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
   # 這裏新增了一個 nat
   config rule_napt
   	option name &#39;NAT_LAN_LAN1&#39;
   	option enable &#39;on&#39;
   	option mask &#39;24&#39;
   	option ipaddr &#39;192.168.0.0&#39;
   	option interface &#39;LAN&#39;
   	option sysrule &#39;1&#39;
   ```
   是不是還有沒有添加的地方？
   死馬當活馬醫，嘗試修改下管理頁面的邏輯，放開這個路由器在配置 napt 時出接口的限制。雖然我知道這麼做大概率沒用，改了頁面，後臺沒腳本支持，也是沒用。
   但是沒其它思路，試試看吧。在路由器裏面修改不方便，不好找要修改哪些位置，所以給它用 U 盤拷下來。插拔 U 盤，確認硬盤。`cat /proc/partitions`
   ```text
   root@TP-LINK:/www/webpages/pages/userrpm# cat /proc/partitions 
   major minor  #blocks  name
   
     31        0         64 mtdblock0
     31        1        128 mtdblock1
   root@TP-LINK:/www/webpages/pages/userrpm# 
   major minor  #blocks  name
   
     31        0         64 mtdblock0
     31        1        128 mtdblock1
      8        0   60538880 sda  # 插 U 盤新增的分區，所以 U 盤是 /dev/sda
      8        1   60537824 sda1
   ```
   嘗試掛載 U 盤，發現 U 盤已經自動掛載到了 /home/ftp/volume1（害，做的都是無用功）。
   
   執行命令打包管理頁面。
   ```bash
   tar zcvf /home/ftp/volume1/www.tgz /www/
   ```
   拿到PC上改代碼，這就到了我們的強項了，快速找到幾個限制出接口的地方，給他改掉（這裏我有疑問，這些地方看起來是官方故意去掉不支持 lan 的，不知道是不是企業路由器都是這個邏輯）。
   
   | 功能        | 代碼文件                                           | 行號  |
   |-----------|------------------------------------------------|-----|
   | NAT設置     | www\webpages\pages\userrpm\napt.html           | 321 |
   | NAT-DMZ設置 | www\webpages\pages\userrpm\nat_dmz.html        | 185 |
   | 虛擬服務器設置   | www\webpages\pages\userrpm\virtual_server.html | 652 |
   
   基本都長這個樣子，把 if 判斷註釋掉。
   ```javascript
   for (i=0; i&lt;data.normal.length; i&#43;&#43;){
       // if(data.normal[i].t_name != &#34;LAN&#34;){
           interfaceItem.push({name:data.normal[i].t_name,value:data.normal[i].t_name});
   	   // }
   }
   ```
   打開前臺頁面，此時能夠在這幾個地方選擇出接口爲 LAN 了，並且嘗試添加 napt 和 虛擬服務器規則，發現後臺都能生成相應的 napt 配置或 iptables 規則。
   
   ![20240621-tplink-1.png](/images/posts/20240621-tplink-1.png)
   
   但是沒有用，沒有解決我的問題，而且看起來它生成的 LAN 口的規則還是錯的，這下沒法了，開啓長考模式——睡覺。

5. 懷疑方向4 rebind-protection（部分正確）
   第二天醒來，想到（爲什麼纔想到）既然是 openwrt，那麼爲什麼不搜 openwrt 相關的問題。果然出來很多結果，其中就有人提到這個 openwrt 默認開啓的安全相關的問題。
   在我的場景下（局域網自己用），這個配置非常愚蠢，不再描述其含義，只說操作和結果。
   首先，這個路由器沒有 openwrt 系統中的配置，沒有一個小框框，點一點就能關掉這個功能，但是因爲它是 dnsmasq 的能力，因此可以通過如下方式關閉：
   ```bash
   vi /etc/config/dhcp # 配置文件的位置
   config dnsmasq
        option domainneeded &#39;1&#39;
        option port &#39;0&#39;
        option boguspriv &#39;1&#39;
        option filterwin2k &#39;0&#39;
        option localise_queries &#39;1&#39;
        option rebind_protection &#39;0&#39; # 搜索 rebind_protection，將其從 1 修改爲 0
   /etc/init.d/dnsmasq restart # 重啓
   ```
   訪問公網ip:端口，依舊無效。
   
   焦灼。只能考慮 tcpdump 抓包了，不折騰的原則又要破壞了。但是這個路由器年限有點老，用的 barrier breaker 14.07 的源地址已經換了，下載下來 ipx 包直接本地安裝發現裝不上。

   去看看官方有沒有解決吧。找到了他們 2021 年發佈的一個 2019 的升級包，好傢伙，直接報錯無法升級，開始對 TP-LINK 心生怨念。

6. 猜猜怎麼解決的，最後聯繫官方，他們給發了一個升級固件。
   固件原理待分析。

## 備忘：iptables 四表五鏈

| raw        | mangle      | nat         | filter  | 四表 |
|------------|-------------|-------------|---------|----|
| prerouting | prerouting  | prerouting  | input   | 五鏈 |
| output     | postrouting | postrouting | forward |    |
|            | input       | output      | output  |    |
|            | output      | input       |         |    |
|            | forward     |

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2024/06/tplink-war2600l-iptables/  

