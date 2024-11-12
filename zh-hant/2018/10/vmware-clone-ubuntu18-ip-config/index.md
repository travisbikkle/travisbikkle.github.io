# Vmware Clone Ubuntu. Ip 地址配置

在使用 VMware Workstation 克隆 Ubuntu Server 18.04
版本後，發現克隆前後的機器 IP 地址重複，
且無論如何更改虛擬網絡設置（編輯-虛擬網絡編輯器）都無效。由於 Ubuntu
18.04 採用 netplan (/etc/netplan) 而不是先前版本的
/etc/network/interfaces 管理網卡設置，因此通過如下方法，將機器 IP
地址更改爲靜態獲取，可以解決此問題。

**1. vi /etc/netplan/50-cloud-init.yaml (此文件名可能會變化).**

    network:
        ethernets:
              ens33:
                      dhcp4: no
                      dhcp6: no
                      addresses: [192.168.44.129/24,]
                      gateway4: 192.168.44.1
                      nameservers:
                              addresses: [8.8.8.8, 8.8.4.4]

**2. 更改後，執行.**

    &gt;netplan apply
    &gt;reboot


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2018/10/vmware-clone-ubuntu18-ip-config/  

