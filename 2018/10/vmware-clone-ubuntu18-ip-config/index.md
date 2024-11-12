# Vmware Clone Ubuntu. Ip 地址配置

在使用 VMware Workstation 克隆 Ubuntu Server 18.04
版本后，发现克隆前后的机器 IP 地址重复，
且无论如何更改虚拟网络设置（编辑-虚拟网络编辑器）都无效。由于 Ubuntu
18.04 采用 netplan (/etc/netplan) 而不是先前版本的
/etc/network/interfaces 管理网卡设置，因此通过如下方法，将机器 IP
地址更改为静态获取，可以解决此问题。

**1. vi /etc/netplan/50-cloud-init.yaml (此文件名可能会变化).**

    network:
        ethernets:
              ens33:
                      dhcp4: no
                      dhcp6: no
                      addresses: [192.168.44.129/24,]
                      gateway4: 192.168.44.1
                      nameservers:
                              addresses: [8.8.8.8, 8.8.4.4]

**2. 更改后，执行.**

    &gt;netplan apply
    &gt;reboot


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2018/10/vmware-clone-ubuntu18-ip-config/  

