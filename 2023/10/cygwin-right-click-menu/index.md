# 右键添加 Cygwin


本文基于 Windows 10。

#### 效果展示
![](/images/other/20231031_cygwin_demo.png)

#### 方法
1. 重新运行 cygwin 安装程序，勾选 chere 安装包
2. 安装完成后，使用管理员身份运行 cygwin，执行以下命令
   ```bash
   chere -i -t mintty -s bash
   ```
3. 此时右键菜单应该有 &#34;Bash Prompt Here&#34; 菜单选项，如果您想更改为中文，可以在更改此处注册表
   ```text
   计算机\HKEY_LOCAL_MACHINE\SOFTWARE\Classes\Directory\background\shell\cygwin64_bash
   ```
   增加 Icon 字符串以显示菜单图标，并更改中文描述，达到上图菜单效果。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2023/10/cygwin-right-click-menu/  

