# Ubuntu Vnc


本文基於 Ubuntu 20.04 編寫。

1. 安裝 vncserver 和要使用的桌面管理器 xfce
   ```bash
   apt install tightvncserver xfce4 xfce4-goodies
   ```
2. 更改密碼
   ```bash
   vncpasswd
   ```
3. 配置 vnc 以便客戶端連接時使用 xfce 桌面
   編輯  ~/.vnc/xstartup，輸入以下內容
   ```bash
   #!/bin/sh
   unset SESSION_MANAGER
   unset DBUS_SESSION_BUS_ADDRESS
   startxfce4 &amp;
   ```
4. 執行以下命令
   ```bash
   chmod &#43;x ~/.vnc/xstartup
   ```
5. 禁用休眠
   ```bash
   systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
   ```
6. 啓動服務器
   ```bash
   vncserver -localhost no -geometry 1920x1080
   ```
   
#### 參考
   &gt; https://linuxconfig.org/vnc-server-on-ubuntu-20-04-focal-fossa-linux

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/04/ubuntu-vnc/  

