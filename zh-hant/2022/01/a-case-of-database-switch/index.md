# 一個數據庫主備頻繁切換案例


## 問題現象

某集團某局點分別部署了某數據庫的主備實例，業務發現偶爾訪問數據庫報錯。

## 定位結論

其中一個節點使用的某物理機型，網卡存在故障，發生了多次復位，致使某數據庫備機的keepalived雙機軟件接收不到對端的vrrp信號，備機在網卡復位期間短暫升主，兩臺機器都出現了vip，導致業務訪問某數據庫失敗。

需要聯繫廠商，針對以下疑問給出答覆：

1. 節點A（某品牌F）和節點B（某品牌L）使用的網卡型號一樣，但是爲什麼某品牌F的固件版本要比某品牌L的新？

2. 某品牌L的網卡信息，和某品牌L官網該型號的機器的認證信息不一致，見下表。

表1-1 某品牌F機型、某品牌L機型和某品牌L官網認證信息對比
| 項目           | 節點A                      | 節點B                   | 某品牌L官網認證信息     |
|--------------|----------------------------|-------------------------|-------------------------|
| 服務器型號     | 某品牌F FiberHome R2200 V5 | 某品牌L Inspur NF8480M5 | 某品牌L Inspur NF8480M5 |
| 網卡驅動       | ixgbe 5.1.0-k              | ixgbe 5.1.0-k           | igb.ko 5.6.0-k          |
| 網卡固件       | 0x800006da 1.1824.0        | 0x800003df              | -                       |
| 是否發生過重啓 | 否                         | 是                      | -                       |

## 定位過程
查看兩個節點的keepalived日誌

```bash
cd /xx/xx/xx/logs
zgrep Entering keepalived*
```
發現主機（節點A）上無異常，備機的keepalived在master、backup、fault三個狀態頻繁切換。

&gt; 當keepalived進入master狀態，會將vip綁定到網卡上。這樣主機和備機都有vip，業務訪問某數據庫將會出現問題。

在備機上使用如下命令可以抓到短暫出現的vip：
```bash
while true;do ip a s bond1|grep x.x.x.x;sleep 0.2;done
```

查看備機的keepalived.log，發現備機無法接收到主機的vrrp廣播。

&gt; keepalived的主備機都會每3s發送一次vrrp協議的廣播，當備機連續3次（9s）接收不到主機的廣播，就會將自己升主。

查看主機的keepalived.log，發現主機在正常發送vrrp廣播。

使用tcpdump在主機和備機同時抓包，問題復現後，發現主機確實發送了vrrp的廣播，但是備機在(15:46:14-15:46:32)有18s的時間沒有接收到任何來自主機的廣播（此處應該有圖，但是不放了）。

使用如下命令，查看備機上keepalived接收vrrp廣播的網卡情況，發現是一張由eth4和eth8組成的active-backup模式的邏輯網卡，：

```bash
cat /proc/net/bonding/bond1
```

查看/var/log/messages中兩張物理網卡的信息，發現eth4和eth8網卡發生了多次復位，復位時間和gauss-datacube出現異常的時間能夠對應的上（此處應該有圖，但是不放了）。

查看節點B的服務器型號及網卡型號、固件版本（此處應該有圖，但是不放了）。
節點B和節點A對比，發現節點A的這臺某品牌F機型，比節點B的某品牌L機型使用的網卡固件要更新：
節點B和某品牌L官網認證信息(https://www.suse.com/nbswebapp/yesBulletin.jsp?bulletinNumber=148996)對比，發現節點B節點使用的網卡driver name和認證信息不符：
除了某局點節點B的某品牌L機型節點，我們還在其它局點的appdb的節點的某品牌L機型上，也發現了網卡重啓的現象，該現象在某品牌F機型上未發現，判斷是某品牌L的這批機器網卡存在故障。
這種短暫網卡重啓的問題，由於tcp重連的特性，對於普通的應用感知不大，但是對於數據庫這樣包含雙機軟件的敏感應用，影響較大。

最終客戶聯繫廠商定位到是硬件故障。

 


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2022/01/a-case-of-database-switch/  

