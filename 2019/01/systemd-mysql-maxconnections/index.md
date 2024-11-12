# 使用 Systemd 托管的 Mysql 最大连接数问题

最大连接数问题

背景： mysql 最大连接数在设置为2000的情况下，并发始终只能达到480多；
其它遇到过类似情况的项目组更改ulimit -s（stack
size）到1024可以解决问题，但是我们经过测试无效；
据说前期定位人员咨询过mysql原厂的人，没发现有什么配置问题。

测试工具
========

mysqlslap -h127.0.0.1 -uroot -p123456789 --concurrency=5000
--iterations=1 --auto-generate-sql --auto-generate-sql-load-type=mixed
--auto-generate-sql-add-autoincrement --engine=innodb
--number-of-queries=1000000

show status like &#34;%Thread%&#34;&#34;;

排查过程
========

ulimit cat /proc/`pidof mysqld`/limits /etc/systemd/system.conf
/etc/systemd/user.conf systemctl edit mysql.service
/usr/lib/systemd/system/mysql.service

直接使用mysqld启动，不用service，发现正常。最终在参照不使用service启动的mysql
pid limits更改mysql.service所有ulimit到最大值也没用。 systemctl show
mysql.service 发现TasksMax字段值为512，与480比较相近。

文档：
&lt;https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html&gt;

尝试在/usr/lib/systemd/system/mysql.service加入以下配置

TasksMax=infinity 问题解决。

后续/深入
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

来自
&lt;https://unix.stackexchange.com/questions/345595/how-to-set-ulimits-on-service-with-systemd&gt;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/01/systemd-mysql-maxconnections/  

