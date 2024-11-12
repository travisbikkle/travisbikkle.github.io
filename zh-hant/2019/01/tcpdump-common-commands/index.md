# Tcpdump 常用命令


1.  man tcpdump

2.  &lt;https://en.wikipedia.org/wiki/Multicast_address&gt;

3.  &lt;https://en.wikipedia.org/wiki/Subnetwork&gt;

4.  \[ip address classes\] (&lt;http://www.vlsm-calc.net/ipclasses.php&gt;)  
    &lt;http://vod.sjtu.edu.cn/help/Article_Print.asp?ArticleID=631&gt;

tcpdump usage
=============

\`\`\` tcpdump \# print number like ip and port tcpdump -n tcpdump -c 4
tcpdump -i eth1 tcpdump -i any tcpdump host 100.107.166.116 tcpdump src
host 100.107.166.116 tcpdump -n -i any dst port 3306 or dst port 22
tcpdump -n -i any *dst port 3306 || dst port 22* tcpdump -n -i any *(dst
port 3306 || dst port 22) and dst host 100.107.166.116* \`\`\`


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/01/tcpdump-common-commands/  

