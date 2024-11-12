# Kubuntu 軟件源在哪裏修改


Kubuntu 22.04 的軟件源可以在 Discover 中的如下位置設置。
![](/images/other/20231108_discover_software_sources.png)

但是如果你的 Discover 不顯示以上界面，可以通過如下方法來修復。

1. 編輯如下文件

   ```shell
   vi /usr/share/applications/software-properties-qt.desktop
   ```

2. 將以下內容替換到源文件
   ```text
   [Desktop Entry]
   Name=Software Sources
   GenericName=Software Sources
   Comment=Configure the sources for installable software and updates
   Exec=pkexec env DISPLAY=$DISPLAY XAUTHORITY=$XAUTHORITY software-properties-qt
   Icon=applications-other
   NoDisplay=false
   Terminal=false
   X-MultipleArgs=false
   Type=Application
   Categories=System;Settings;
   MimeType=text/x-apt-sources-list;
   X-KDE-SubstituteUID=false
   X-Ubuntu-Gettext-Domain=software-properties
   ```

如此，在 Discover可以顯示軟件源的界面，在菜單中也可以搜索到在其它發行版中常見的 _軟件源_ 菜單項。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/11/kubuntu2204-software-resources-bug/  

