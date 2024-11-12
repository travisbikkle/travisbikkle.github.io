# Vrrp 报文抓包的方法

keepalived 的主机和备机，会每隔3s通过单播，向对方发送 vrrp 报文，当备机在 `3次*每次心跳间隔（keepalived.conf中的adver_int参数）` 都收不到主机的广播，自己将会升主。

以下两种可能都会存在：
1. 主机没有发送广播
2. 主机发送了广播，备机没有收到
因此，在主备都需要监控vrrp的发送（out）或接收（in）记录。

使用以下两种方式，都可以监控每次 vrrp 报文的内容，以便定位问题。

## iptables
1. 在主机和备机都添加监控vrrp发送和接收的iptables日志（该规则仅抓取vrrp行为，没有其它副作用）
   ```bash
   iptables -t raw -A PREROUTING -p vrrp -j LOG --log-level 6    --log-prefix &#34;VRRP: &#34; 
   iptables -t raw -A OUTPUT -p vrrp -j LOG --log-level 6    --log-prefix &#34;VRRP: &#34;
   ```
2. 使用以下命令检测是否添加成功：
   ```bash
   iptables -L -t raw -n
   ```
3. 在/var/log/messages可以查看vrrp的记录。如果是suse或其它操作系统，该日志中没有vrrp记录的话，可以在dmesg或者/var/log/的目录中grep VRRP查看。
4. 定位完问题后，执行以下命令删除添加的iptables规则
   ```bash
   iptables -t raw -D PREROUTING -p vrrp -j LOG --log-level 6    --log-prefix &#34;VRRP: &#34; 
   iptables -t raw -D OUTPUT -p vrrp -j LOG --log-level 6    --log-prefix &#34;VRRP: &#34;
   ```

## tcpdump wireshark
tcpdump 命令需要自行获取，wireshark 需要自行下载，这部分仅记录常用命令和注意点，由于在公司内网，图片无法放上来。

1. 确定 keepalived 所经过的网卡（如eth1）后，执行以下命令
   ```
   tcpdump -i eth1 vrrp    -n                                # 打印到   console控制台 
   tcpdump -i eth1 vrrp -n -w my_tcp_dump.   cap             # 写到文件中 
   nohup tcpdump -i eth1 vrrp -n -w my_tcp_dump.   cap &amp;     # 后台执行，建议执行此命令
   ```

2. 经过一段时间或问题复现后，请kill（不要kill -9，会导致tcpdump生成的文件中断，在wireshark中打开报错）掉tcpdump的进程。
3. 使用wireshark打开生成的my_tcp_dump.cap文件，在wireshark中可以过滤源机器、目标机器，查看发送时间，以及报文内容。
4. 如果一个节点上可能有多个keepalived，通常可以查看报文中的Virtual Rtr ID来区分，一套keepalived使用的该值通常是相同的。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2022/02/vrrp-cap/  

