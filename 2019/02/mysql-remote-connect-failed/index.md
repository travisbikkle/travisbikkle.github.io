# Mysql 远程无法连接问题定位记录


某业务通过Hibernate访问mysql，后台报错 Access denied for user
*matexxx*@*xxxx* (using password: YES);
一般搞过开发的人都知道，这种问题不是密码错了，就是远程连接未打开，这两者其实都属于一个问题，就是用户的grant权限问题，但是此业务情况稍特殊。定位过程如下。

查看用户
========

    SELECT USER,HOST FROM MYSQL.USER;

发现用户matexxx对应的host为
%，说明远程连接已经打开；询问业务是否更改过密码，引出问题背景：
业务曾重装过mysql，使用mysqldump将旧库数据备份，并且只在新库的master上执行了一次恢复操作。

查看主从复制的状态
==================

    SHOW SLAVE STATUS\G

发现互为主备的mysql机器，其中一台的slave
io状态为connecting，Last\_IO\_Error 显示复制用户 replicator
禁止登录。既然复制用户和业务用户都无法登录，怀疑点聚焦在用户的grant语句方面，原因可能是其备份恢复过程中出现错误操作，其要求紧急恢复，原因就暂不深挖。

【解决】 主从复制的问题要先解决。错误产生的原因很可能是其使用mysqldump
--all-databases备份，然后在配置好主从的机器上直接恢复，导致两边的机器replicator主从复制用户的ip并不正确（实际应该配置对方ip）。恢复方法：

**请将下面语句中的变量替换为实际的值.**

    GRANT REPLICATION SLAVE ON *.* TO &#39;${repl_user_name}&#39;@&#39;${IP}&#39; IDENTIFIED BY &#39;${repl_user_pwd}&#39;;
    FLUSH PRIVILEGES;

    SHOW MASTER LOGS; --在master(互为主备的机器，master就是你要复制的机器，请自行理解)上执行
    -- 记录上面执行语句的结果，例如
    -- Log_name：mysql-bin.000002
    -- File_size：483

    STOP SLAVE; --在出错的机器上，执行
    CHANGE MASTER TO MASTER_HOST=&#39;${master_ip}&#39;,MASTER_PORT=&#39;3306&#39;,MASTER_LOG_FILE=&#39;mysql-bin.000002&#39;,MASTER_LOG_POS=483;
    START SLAVE;

回到主要问题
============

重启业务应用（反正已经坏了）发现仍然无法登录，查看进程列表，发现大量连接状态都为
` Waiting in connection_control plugin `，而且在另一台机器C上面使用matexxx登录一直卡住，而使用root却没有问题，证明此用户登录失败，被拒绝后触发了
connection\_control 的机制。

    show processlist;
    &#43;----&#43;------&#43;-----------&#43;------&#43;---------&#43;------&#43;--------------------------------------&#43;------------------&#43;
    | Id | User | Host      | db   | Command | Time | State                                | Info             |
    &#43;----&#43;------&#43;-----------&#43;------&#43;---------&#43;------&#43;--------------------------------------&#43;------------------&#43;
    |  3 | mmmm | x.x.x.x | NULL | Query   |    0 | init                                 | show processlist |
    | 32 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 33 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 34 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 35 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 36 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 37 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 38 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 39 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 40 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 41 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 42 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 43 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 44 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 45 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |
    | 46 | mmmm | x.x.x.x | NULL | Connect | NULL | Waiting in connection_control plugin | NULL             |

此处解释一下 connection\_control 的作用
=======================================

一句话，防暴力破解。

官网曾有人对此插件发表过疑问，官方表示不应该在开放远程连接的外网机器上配置此插件，而且在生产环境上也不应存在host为%的用户。
&lt;https://bugs.mysql.com/bug.php?id=89155&gt;

此插件的作用是，多次登录失败，服务器增加对客户端的响应延迟，以增加暴力破解的时间；少量的失败登录对用户的正常登录没有影响，如果存在大量的失败登录（被暴力破解时）则用户正常登录时耗时会增加。

**查看插件是否启用.**

    show plugins;
    select plugin_name,plugin_library,load_option from information_schema.plugins;
    show variables like &#34;%connection_control%&#34;;

    connection_control_failed_connections_threshold: 3 
    connection_control_max_connection_dely: 214xxxxxxxx 
    connection_control_min_connection_dely: 1000 

-   在机制生效前允许的失败次数

-   允许延长到的最大时间

-   最小时间，单位ms

暂时规避插件的作用，简化问题
============================

注释掉/etc/my.cnf的connection\_control相关的行，重新启动两台机器。使用机器C重新登录两台机器，发现其中一台远程可以登录了，
但是另一台开始很快反馈报错信息。怀疑在有问题的机器上，用户密码被错误的修改过。

【解决】在无法登陆的机器上，重新运行grant语句并指定密码

    GRANT ALL PRIVILEGES ON *.* TO &#34;matexxx&#34;@&#34;%&#34; IDENTIFIED BY &#34;${userPWD}&#34;;
    FLUSH PRIVILEGES；

重启应用后问题消失。打开插件。

正确的恢复方法
==============

由于前期人员备份脚本使用的
--all-databases，会一起导出用户信息。所以正确的恢复方法，大体是：

    在旧的主机和备机上各自执行备份命令。
    在安装好的新的机器上，各自登录并执行
    STOP SLAVE;
    各自导入备份的文件，查看show master logs并在两台机器上重新配置指定binlog文件：
    CHANGE MASTER TO MASTER_LOG_FILE=&#39;XXXX.bin0000001&#39;, MASTER_LOG_POS=123;
    START SLAVE;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/02/mysql-remote-connect-failed/  

