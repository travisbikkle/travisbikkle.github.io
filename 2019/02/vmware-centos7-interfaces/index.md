# Vmware Centos 双网卡平面设置


    [root@host1 ~]# ip a
    1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: ens33: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 00:0c:29:7d:ba:48 brd ff:ff:ff:ff:ff:ff
        inet 192.168.17.101/24 brd 192.168.17.255 scope global noprefixroute ens33
           valid_lft forever preferred_lft forever
        inet6 fe80::64ee:1323:6aaa:61da/64 scope link noprefixroute
           valid_lft forever preferred_lft forever
    3: ens37: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 00:0c:29:7d:ba:52 brd ff:ff:ff:ff:ff:ff
        inet 192.168.44.129/24 brd 192.168.44.255 scope global noprefixroute dynamic ens37
           valid_lft 1083sec preferred_lft 1083sec
        inet6 fe80::6ccf:c498:99ff:1910/64 scope link noprefixroute
           valid_lft forever preferred_lft forever
    [root@host1 ~]# cat /etc/sysconfig/network-scripts/ifcfg-ens33
    TYPE=Ethernet
    PROXY_METHOD=none
    BROWSER_ONLY=no
    BOOTPROTO=static
    IPADDR=192.168.17.101
    GATEWAY=192.168.17.1
    NETMASK=255.255.255.0
    DNS1=8.8.8.8

    DEFROUTE=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=yes
    IPV6_AUTOCONF=yes
    IPV6_DEFROUTE=yes
    IPV6_FAILURE_FATAL=no
    IPV6_ADDR_GEN_MODE=stable-privacy
    NAME=ens33
    UUID=fe4b3f1d-a5da-47e6-bacf-a4341d936b2f
    DEVICE=ens33
    ONBOOT=yes
    [root@host1 ~]#


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/02/vmware-centos7-interfaces/  

