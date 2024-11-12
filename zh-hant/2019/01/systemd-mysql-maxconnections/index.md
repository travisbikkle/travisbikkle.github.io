# 使用 Systemd 託管的 Mysql 最大連接數問題

最大連接數問題

背景： mysql 最大連接數在設置爲2000的情況下，併發始終只能達到480多；
其它遇到過類似情況的項目組更改ulimit -s（stack
size）到1024可以解決問題，但是我們經過測試無效；
據說前期定位人員諮詢過mysql原廠的人，沒發現有什麼配置問題。

測試工具
========

mysqlslap -h127.0.0.1 -uroot -p123456789 --concurrency=5000
--iterations=1 --auto-generate-sql --auto-generate-sql-load-type=mixed
--auto-generate-sql-add-autoincrement --engine=innodb
--number-of-queries=1000000

show status like &#34;%Thread%&#34;&#34;;

排查過程
========

ulimit cat /proc/`pidof mysqld`/limits /etc/systemd/system.conf
/etc/systemd/user.conf systemctl edit mysql.service
/usr/lib/systemd/system/mysql.service

直接使用mysqld啓動，不用service，發現正常。最終在參照不使用service啓動的mysql
pid limits更改mysql.service所有ulimit到最大值也沒用。 systemctl show
mysql.service 發現TasksMax字段值爲512，與480比較相近。

文檔：
&lt;https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html&gt;

嘗試在/usr/lib/systemd/system/mysql.service加入以下配置

TasksMax=infinity 問題解決。

後續/深入
=========

The mappings of systemd limits to ulimit
----------------------------------------

Directive ulimit equivalent Unit LimitCPU= ulimit -t Seconds LimitFSIZE=
ulimit -f Bytes LimitDATA= ulimit -d Bytes LimitSTACK= ulimit -s Bytes
LimitCORE= ulimit -c Bytes LimitRSS= ulimit -m Bytes LimitNOFILE= ulimit
-n Number of File Descriptors LimitAS= ulimit -v Bytes LimitNPROC=
ulimit -u Number of Processes LimitMEMLOCK= ulimit -l Bytes LimitLOCKS=
ulimit -x Number of Locks LimitSIGPENDING= ulimit -i Number of Queued
Signals LimitMSGQUEUE= ulimit -q Bytes LimitNICE= ulimit -e Nice Level
LimitRTPRIO= ulimit -r Realtime Priority LimitRTTIME= No equivalent

來自
&lt;https://unix.stackexchange.com/questions/345595/how-to-set-ulimits-on-service-with-systemd&gt;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/01/systemd-mysql-maxconnections/  

