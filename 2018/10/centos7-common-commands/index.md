# Centos 常用命令


如何更改为静态 IP 地址
======================

1.  `vi /etc/sysconfig/network-scripts/ifcfg-&lt;你的网卡名，如果不知道，直接 tab 自动补全&gt;`

        ## 更改并添加以下数行
        # BOOTPROTO=dfcp
        BOOTPROTO=static
        # ONBOOT=no
        ONBOOT=yes
        IPADDR=192.168.47.190 # IP 地址，先在虚拟机或路由里查看你的 IP 网段，然后在设置为你想要的值
        GATEWAY=192.168.47.2 # 网关信息，同上
        NETMASK=255.255.255.0 # 子网掩码信息，同上
        DNS1=8.8.8.8 # DNS 信息，同上

2.  `service network restart` 重启网络服务

如何更改主机名
==============

`hostnamectl set-hostname &lt;你想要的主机名&gt;`

如何关闭防火墙和SELinux
=======================

    systemctl disable firewalld.service
    systemctl stop firewalld.service

    # 编辑以下文件
    vi /etc/sysconfig/selinux
    SELINUX=disabled
    # 编辑完成后，执行
    setenforce 0
    # 重启后执行 getenforce 变成 disabled 说明更改永久生效

如何设置 NTP 时间同步
=====================

    yum install -y ntp
    systemctl enable ntpd


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2018/10/centos7-common-commands/  

