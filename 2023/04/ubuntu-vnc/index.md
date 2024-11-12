# Ubuntu Vnc


本文基于 Ubuntu 20.04 编写。

1. 安装 vncserver 和要使用的桌面管理器 xfce
   ```bash
   apt install tightvncserver xfce4 xfce4-goodies
   ```
2. 更改密码
   ```bash
   vncpasswd
   ```
3. 配置 vnc 以便客户端连接时使用 xfce 桌面
   编辑  ~/.vnc/xstartup，输入以下内容
   ```bash
   #!/bin/sh
   unset SESSION_MANAGER
   unset DBUS_SESSION_BUS_ADDRESS
   startxfce4 &amp;
   ```
4. 执行以下命令
   ```bash
   chmod &#43;x ~/.vnc/xstartup
   ```
5. 禁用休眠
   ```bash
   systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
   ```
6. 启动服务器
   ```bash
   vncserver -localhost no -geometry 1920x1080
   ```
   
#### 参考
   &gt; https://linuxconfig.org/vnc-server-on-ubuntu-20-04-focal-fossa-linux

---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/04/ubuntu-vnc/  

