# 免密码 Ssh 到其它机器


背景：在配置 hadoop 的时候这样设置会比较方便。 目标：A 机器上输入 ssh
&lt;root@B&gt; 可以直接访问，不需要输入密码

步骤：

1.  首先在 A 机器上生成密钥对，一路回车

         ssh-keygen -t rsa

2.  在 A 机器上输入，输入 B 机器的密码一次即可

         ssh-copy-id -i ~/.ssh/id_rsa.pub root@B

所以同样的操作，B机器上可能还要再操作一遍，如果机器多了，也是很烦，因此，更懒人的做法是：

1.  准备 xshell 5

2.  打开多个机器的 ssh 会话窗口

3.  配置好各个机器的 hostname

4.  在 xshell 底部，&#34;`发送命令到所有窗口`&#34;这一行，依次输入
    `ssh-copy-id -i ~/.ssh/id_rsa.pub root@&lt;主机名&gt;` 即可。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2018/10/ssh-without-pass/  

