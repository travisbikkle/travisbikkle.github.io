# How to Add Samba Entry in /Etc/Fstab

Configure Samba On Ubuntu

#### 1. install necessary tools
```shell
sudo apt install cifs-utils
```
#### 2. prepare credentials
```shell
sudo mkdir /mnt/samba
sudo vi /root/.smb_credentials
```
#### 3. put next 2 lines into /root/.smb_credentials
```text
username=&lt;your user name&gt;
password=&lt;your password&gt;
```
#### 4. change credential file permission for security reason
```shell
chmod 600 /root/.smb_credentials
```
#### 5. configure /etc/fstab
```shell
sudo vi /etc/fstab
```
#### 6. put next line into /etc/fstab. add `noauto` to options if you don&#39;t want to let Ubuntu auto mount this path
```text
#&lt;shared path&gt;       &lt;mount point&gt; &lt;type&gt;  &lt;options&gt;
//192.168.0.3/share  /mnt/samba    cifs    credentials=/root/.smb_credentials,gid=1000,uid=1000,file_mode=0755,dir_mode=0755 0 0
```
#### 7. mount it
```shell
umount /mnt/samba -lv
mount /mnt/samba
```
本文基于 Ubuntu 22.04 LTS 编写。

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/10/fatab-samba/  

