# 右鍵添加 Cygwin


本文基於 Windows 10。

#### 效果展示
![](/images/other/20231031_cygwin_demo.png)

#### 方法
1. 重新運行 cygwin 安裝程序，勾選 chere 安裝包
2. 安裝完成後，使用管理員身份運行 cygwin，執行以下命令
   ```bash
   chere -i -t mintty -s bash
   ```
3. 此時右鍵菜單應該有 &#34;Bash Prompt Here&#34; 菜單選項，如果您想更改爲中文，可以在更改此處註冊表
   ```text
   計算機\HKEY_LOCAL_MACHINE\SOFTWARE\Classes\Directory\background\shell\cygwin64_bash
   ```
   增加 Icon 字符串以顯示菜單圖標，並更改中文描述，達到上圖菜單效果。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2023/10/cygwin-right-click-menu/  

