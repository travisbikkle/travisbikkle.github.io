# 端口轉發的一些方法

在《如何抓包分析 vrrp 報文是否中斷》中我們使用了 iptables 進行指定協議的報文日誌記錄功能，此處新增一個好用的命令。

我們在開發過程中，可能經常需要經過堡壘機、跳板機等中轉一次，才能訪問到處於小網中的服務，如數據庫。使用以下命令，可以在不借助第三方軟件的情況下，完成端口轉發，實現直連小網服務。

## iptables
假設 192.168.1.3 是你的小網機器，端口是 15432
```bash
iptables -t nat -A PREROUTING -p tcp --dport 15432 -j DNAT --to-destination 192.168.1.3:15432
iptables -t nat -A POSTROUTING -p tcp -m tcp --dport 15432 -j MASQUERADE
```

## nginx
```
upstream 192.168.1.3 {
    server 192.168.1.3:15432;
}
server {
    listen       15432;
    proxy_pass 192.168.1.3;
    proxy_connect_timeout 1h;
    proxy_timeout 1h;
}
```

或者用 go 自己寫了一些小工具來轉發，我自己還寫了一個自動生成 xshell 配置的工具，因爲在公司內網，不便共享。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/02/port-forward/  

