# 免密碼 Ssh 到其它機器


背景：在配置 hadoop 的時候這樣設置會比較方便。 目標：A 機器上輸入 ssh
&lt;root@B&gt; 可以直接訪問，不需要輸入密碼

步驟：

1.  首先在 A 機器上生成密鑰對，一路回車

         ssh-keygen -t rsa

2.  在 A 機器上輸入，輸入 B 機器的密碼一次即可

         ssh-copy-id -i ~/.ssh/id_rsa.pub root@B

所以同樣的操作，B機器上可能還要再操作一遍，如果機器多了，也是很煩，因此，更懶人的做法是：

1.  準備 xshell 5

2.  打開多個機器的 ssh 會話窗口

3.  配置好各個機器的 hostname

4.  在 xshell 底部，&#34;`發送命令到所有窗口`&#34;這一行，依次輸入
    `ssh-copy-id -i ~/.ssh/id_rsa.pub root@&lt;主機名&gt;` 即可。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2018/10/ssh-without-pass/  

