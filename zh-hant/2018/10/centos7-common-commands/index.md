# Centos 常用命令


如何更改爲靜態 IP 地址
======================

1.  `vi /etc/sysconfig/network-scripts/ifcfg-&lt;你的網卡名，如果不知道，直接 tab 自動補全&gt;`

        ## 更改並添加以下數行
        # BOOTPROTO=dfcp
        BOOTPROTO=static
        # ONBOOT=no
        ONBOOT=yes
        IPADDR=192.168.47.190 # IP 地址，先在虛擬機或路由裏查看你的 IP 網段，然後在設置爲你想要的值
        GATEWAY=192.168.47.2 # 網關信息，同上
        NETMASK=255.255.255.0 # 子網掩碼信息，同上
        DNS1=8.8.8.8 # DNS 信息，同上

2.  `service network restart` 重啓網絡服務

如何更改主機名
==============

`hostnamectl set-hostname &lt;你想要的主機名&gt;`

如何關閉防火牆和SELinux
=======================

    systemctl disable firewalld.service
    systemctl stop firewalld.service

    # 編輯以下文件
    vi /etc/sysconfig/selinux
    SELINUX=disabled
    # 編輯完成後，執行
    setenforce 0
    # 重啓後執行 getenforce 變成 disabled 說明更改永久生效

如何設置 NTP 時間同步
=====================

    yum install -y ntp
    systemctl enable ntpd


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2018/10/centos7-common-commands/  

