# Vrrp 報文抓包的方法

keepalived 的主機和備機，會每隔3s通過單播，向對方發送 vrrp 報文，當備機在 `3次*每次心跳間隔（keepalived.conf中的adver_int參數）` 都收不到主機的廣播，自己將會升主。

以下兩種可能都會存在：
1. 主機沒有發送廣播
2. 主機發送了廣播，備機沒有收到
因此，在主備都需要監控vrrp的發送（out）或接收（in）記錄。

使用以下兩種方式，都可以監控每次 vrrp 報文的內容，以便定位問題。

## iptables
1. 在主機和備機都添加監控vrrp發送和接收的iptables日誌（該規則僅抓取vrrp行爲，沒有其它副作用）
   ```bash
   iptables -t raw -A PREROUTING -p vrrp -j LOG --log-level 6    --log-prefix &#34;VRRP: &#34; 
   iptables -t raw -A OUTPUT -p vrrp -j LOG --log-level 6    --log-prefix &#34;VRRP: &#34;
   ```
2. 使用以下命令檢測是否添加成功：
   ```bash
   iptables -L -t raw -n
   ```
3. 在/var/log/messages可以查看vrrp的記錄。如果是suse或其它操作系統，該日誌中沒有vrrp記錄的話，可以在dmesg或者/var/log/的目錄中grep VRRP查看。
4. 定位完問題後，執行以下命令刪除添加的iptables規則
   ```bash
   iptables -t raw -D PREROUTING -p vrrp -j LOG --log-level 6    --log-prefix &#34;VRRP: &#34; 
   iptables -t raw -D OUTPUT -p vrrp -j LOG --log-level 6    --log-prefix &#34;VRRP: &#34;
   ```

## tcpdump wireshark
tcpdump 命令需要自行獲取，wireshark 需要自行下載，這部分僅記錄常用命令和注意點，由於在公司內網，圖片無法放上來。

1. 確定 keepalived 所經過的網卡（如eth1）後，執行以下命令
   ```
   tcpdump -i eth1 vrrp    -n                                # 打印到   console控制檯 
   tcpdump -i eth1 vrrp -n -w my_tcp_dump.   cap             # 寫到文件中 
   nohup tcpdump -i eth1 vrrp -n -w my_tcp_dump.   cap &amp;     # 後臺執行，建議執行此命令
   ```

2. 經過一段時間或問題復現後，請kill（不要kill -9，會導致tcpdump生成的文件中斷，在wireshark中打開報錯）掉tcpdump的進程。
3. 使用wireshark打開生成的my_tcp_dump.cap文件，在wireshark中可以過濾源機器、目標機器，查看發送時間，以及報文內容。
4. 如果一個節點上可能有多個keepalived，通常可以查看報文中的Virtual Rtr ID來區分，一套keepalived使用的該值通常是相同的。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2022/02/vrrp-cap/  

