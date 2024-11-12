# 端口转发的一些方法

在《如何抓包分析 vrrp 报文是否中断》中我们使用了 iptables 进行指定协议的报文日志记录功能，此处新增一个好用的命令。

我们在开发过程中，可能经常需要经过堡垒机、跳板机等中转一次，才能访问到处于小网中的服务，如数据库。使用以下命令，可以在不借助第三方软件的情况下，完成端口转发，实现直连小网服务。

## iptables
假设 192.168.1.3 是你的小网机器，端口是 15432
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

或者用 go 自己写了一些小工具来转发，我自己还写了一个自动生成 xshell 配置的工具，因为在公司内网，不便共享。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/02/port-forward/  

