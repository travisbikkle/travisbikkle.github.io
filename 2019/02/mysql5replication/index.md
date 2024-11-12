# Mysql . 主从同步


大纲
====

本文参考或翻译自：
&lt;https://dev.mysql.com/doc/refman/5.7/en/replication.html&gt;

MySQL 5.7 支持多种主从复制的方法  
1.  传统方法：依赖binlog文件和文件的position保持同步
    （https://dev.mysql.com/doc/refman/5.7/en/replication-configuration.html）

2.  新方法： 依赖全局事务id即global transaction identifer（GTIDs）
    （https://dev.mysql.com/doc/refman/5.7/en/replication-gtids.html）

replication 支持不同类型的同步  
1.  异步复制（asynchronous，默认）

2.  同步复制（只有 NDB 集群才有的一种特性）

3.  半同步复制（semisynchronous，是对异步复制的一种补充）

With semisynchronous replication, a commit performed on the master
blocks before returning to the session that performed the transaction
until at least one slave acknowledges that it has received and logged
the events for the transaction; see Semisynchronous
Replication(&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-semisync.html&gt;).
MySQL 5.7 also supports delayed replication such that a slave server
deliberately lags behind the master by at least a specified amount of
time; see Section 16.3.10, Delayed
Replication(&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-delayed.html&gt;).

HOW-TO  
There are a number of solutions available for setting up replication
between servers, and the best method to use depends on the presence of
data and the engine types you are using. For more information on the
available options, see Section 16.1.2, “Setting Up Binary Log File
Position Based
Replication”(&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-howto.html&gt;).

复制格式  
1.  基于语句的(Statement Based Replication (SBR))

2.  基于行的(Row Based Replication (RBR))

3.  混合的，也就是结合以上两种(Mixed Based Replication (MBR))
    &lt;https://dev.mysql.com/doc/refman/5.7/en/replication-formats.html&gt;.

选项与变量  
Replication is controlled through a number of different options and
variables. For more information, see Section 16.1.6, “Replication and
Binary Logging Options and
Variables”(&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-options.html&gt;).

replication 的其它用途  
&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-solutions.html&gt;

原理  
&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-implementation.html&gt;

研究路线  
1.  配置 replication

    1.  基于日志位置的复制配置

    2.  基于GTIDs的复制

    3.  MySQL multi-Source replication

    4.  在上线机器上更改复制模式

    5.  复制与日志记录选项和变量

    6.  常用复制管理任务

2.  replication 实现

3.  replication 用途 ..

    1.  半同步

4.  replication notes and tips

配置 replication
================

基于 BinLog 日志文件位置 的复制
-------------------------------

master
作为数据库改变的源头，将事件（变化、更新等操作）写入到二进制日志，事件信息存储的格式根据变化的不同而不同；slave
从主机读取并执行日志。 每台 slave 都会获取到一份二进制日志（以下简称
binlog）的完整内容的副本。slave 会决定执行这个 binlog
的哪一部分。除非特别指定，否则全部执行。如果需要，你也可以配置只执行特定
database 或者 table 的相关语句。

不能配置只执行某一次特定的事件。

每台 slave 都会记录一个 binlog
的坐标：文件名称和这个文件中已经处理到什么位置。这就意味着多个 slave
可以正在执行同一个 binlog 的不同部分。因为是 slave 在控制这个过程，
slave 可以随意连接、断开 master 而不影响 master 操作。而且这意味着 slave
可以断开、重连、恢复处理。

master 和每一个 slave 都必须有一个唯一的
server-id([https://dev.mysql.com/doc/refman/5.7/en/replication-options.html\#option\_mysqld\_server-id)，并且需要通过](https://dev.mysql.com/doc/refman/5.7/en/replication-options.html#option_mysqld_server-id)，并且需要通过)
CHANGE MASTER TO 语句提供 master
主机地址、日志文件名称、日志文件位置等信息([https://dev.mysql.com/doc/refman/5.7/en/change-master-to.html)。这部分细节存储在](https://dev.mysql.com/doc/refman/5.7/en/change-master-to.html)。这部分细节存储在)
slave 的 master info repository，
可以是一个文件，也可能存储在一个表中(&lt;https://dev.mysql.com/doc/refman/5.7/en/slave-logs.html&gt;)。

首先，掌握一些基础命令，有助于后面的配置  

**控制 master 的语句** (SQL Statements for Controlling Master Servers)
（https://dev.mysql.com/doc/refman/5.7/en/replication-master-sql.html）

    SHOW BINARY LOGS
    SHOW BINLOG EVENTS
    SHOW MASTER STATUS
    SHOW SLAVE HOSTS

-   SHOW BINARY LOGS 等同于 SHOW MASTER LOGS. 需要权限.

-   支持指定文件名、位置等，见
    &lt;https://dev.mysql.com/doc/refman/5.7/en/show-binlog-events.html&gt;；
    大数据量耗费较多时间，可用 mysqlbinlog 工具代替，见
    &lt;https://dev.mysql.com/doc/refman/5.7/en/mysqlbinlog.html&gt;.

**控制 slave 的语句** (SQL Statements for Controlling Slave Servers)
（https://dev.mysql.com/doc/refman/5.7/en/replication-slave-sql.html）

    CHANGE MASTER TO
    CHANGE REPLICATION FILTER
    MASTER_POS_WAIT()
    RESET SLAVE
    SET GLOBAL sql_slave_skip_counter
    START SLAVE
    STOP SLAVE
    SHOW SLAVE STATUS and SHOW RELAYLOG EVENTS

-   内容参考
    [https://dev.mysql.com/doc/refman/5.7/en/change-master-to.html，关注点](https://dev.mysql.com/doc/refman/5.7/en/change-master-to.html，关注点)：

    1.  是否需要先 stop slave

    2.  多线程的 slave 下可能出现的间隙问题（gaps）以及
        `START SLAVE UNTIL SQL_AFTER_MTS_GAPS`

    3.  CHANGE MASTER TO .. FOR CHANNEL *channel* 的用法, 更多
        Replication Channel 参考
        &lt;https://dev.mysql.com/doc/refman/5.7/en/replication-channels.html&gt;

    4.  未指定的选项保留旧的值。

    5.  【重要】如果指定了 ‘MASTER\_HOST\` 或者
        `MASTER_PORT`，即使值没有变化，mysql 也认为 master
        主机也跟以前不一样了。这种情况下，binlog的文件名和位置就失效了，所以如果不指定
        MASTER\_LOG\_FILE 和 MASTER\_LOG\_POS，mysql默认添加上
        MASTER\_LOG\_FILE=’&#39; 且 MASTER\_LOG\_POS = 4。

    6.  ssl 相关的配置，MASTER\_SSL\_XXX 和 --ssl-XXX
        (&lt;https://dev.mysql.com/doc/refman/5.7/en/encrypted-connection-options.html&gt;)
        功能一样。

    7.  心跳检测相关的选项（比如 MASTER\_HEARTBEAT\_PERIOD
        不指定，默认是系统变量 slave\_net\_timeout 的一半；更改
        slave\_net\_timeout 也要适当更改其它关联选项否则不起作用等）

        更改默认值并检查当前连接心跳次数：

            STOP SLAVE;
            CHANGE MASTER TO MASTER_HOST=&#39;X.X.X.X&#39;, MASTER_LOG_POS=XX, MASTER_HEARTBEAT_PERIOD=10;
            START SLAVE;

            -- 一些常用的健康表
            mysql&gt; USE performance_schema;
            mysql&gt; SHOW TABLES LIKE &#34;replication%&#34;;
            &#43;---------------------------------------------&#43;
            | Tables_in_performance_schema (replication%) |
            &#43;---------------------------------------------&#43;
            | replication_applier_configuration           |
            | replication_applier_status                  |
            | replication_applier_status_by_coordinator   |
            | replication_applier_status_by_worker        |
            | replication_connection_configuration        |
            | replication_connection_status               |
            | replication_group_member_stats              |
            | replication_group_members                   |
            &#43;---------------------------------------------&#43;
            8 rows in set (0.00 sec)


            -- 检查当前配置
            SELECT * FROM performance_schema.replication_connection_configuration\G
            -- 检查当前连接心跳次数、连接状态
            SELECT * FROM performance_schema.replication_connection_status\G

    8.  MASTER\_DELAY 与 延迟复制
        &lt;https://dev.mysql.com/doc/refman/5.7/en/replication-delayed.html&gt;

    9.  MASTER\_BIND 与多网卡平面有关（可用 SHOW SLAVE STATUS 查看）

    10. 一些不可同时指定的值：

            MASTER_LOG_FILE or MASTER_LOG_POS 与 RELAY_LOG_FILE or RELAY_LOG_POS 不可同时指定。
            MASTER_LOG_FILE or MASTER_LOG_POS 与  MASTER_AUTO_POSITION = 1 不可同时指定。
            MASTER_LOG_FILE or MASTER_LOG_POS 如果不指定，使用之前的旧值。

    11. 【重要】relaylog 的删除

        In MySQL 5.7.4 and later, relay logs are preserved if at least
        one of the slave SQL thread and the slave I/O thread is running;
        if both threads are stopped, all relay log files are deleted
        unless at least one of RELAY\_LOG\_FILE or RELAY\_LOG\_POS is
        specified. 12. MASTER\_AUTO\_POSITION = 1

        When MASTER\_AUTO\_POSITION = 1 is used with CHANGE MASTER TO,
        the slave attempts to connect to the master using the GTID-based
        replication protocol. From MySQL 5.7, this option can be
        employed by CHANGE MASTER TO only if both the slave SQL and
        slave I/O threads are stopped. Both the slave and the master
        must have GTIDs enabled (GTID\_MODE=ON, ON\_PERMISSIVE, or
        OFF\_PERMISSIVE on the slave, and GTID\_MODE=ON on the master).
        Auto-positioning is used for the connection, so the coordinates
        represented by MASTER\_LOG\_FILE and MASTER\_LOG\_POS are not
        used, and the use of either or both of these options together
        with MASTER\_AUTO\_POSITION = 1 causes an error. If multi-source
        replication is enabled on the slave, you need to set the
        MASTER\_AUTO\_POSITION = 1 option for each applicable
        replication
        channel.（https://dev.mysql.com/doc/refman/5.7/en/replication-gtids-auto-positioning.html）

**Replication 与 binlog 的选项、变量**(Replication and Binary Logging
Options and Variables)
（https://dev.mysql.com/doc/refman/5.7/en/replication-options.html）

基于GTIDs的复制
---------------

MySQL multi-Source replication
------------------------------

在上线机器上更改复制模式
------------------------

复制与日志记录选项和变量
------------------------

常用复制管理任务
----------------

replication 实现
================

replication 用途
================

半同步复制
----------

&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-semisync.html&gt;

mysql 默认是异步同步，master 把操作写到 binlog 里，但是不关心 slave
是否（或者何时）收到（或者处理）这些事件。这种方式下，master
如果崩溃，可能来不及把其已经提交的事务传输给任何一个 slave。
Consequently, failover from master to slave in this case may result in
failover to a server that is missing transactions relative to the
master.

半同步可以作为异步的一种替代： 1. 当 slave 连接到主机的时候，它会提示
master，自己是否支持半同步 2. 当 master
开启了半同步，并且有至少一台开启了半同步的 slave 连接到了
master，那么任何一个执行事务的线程，就会一直等待，至少一个开启了半同步的
slave
反馈其收到了这个事务相关的全部日志（或者达到一个超时时间），然后才会
commit； 3. slave 只有在收到事件、把事件写入到 relaylog
并刷到磁盘后，才会向 master 发出这个反馈； 4. 当超时时间已经达到，master
还没有收到任何反馈，其会转成异步模式；一旦任何一个 slave
赶上（步骤3完成？），master 还会继续转回半同步模式。 5. 半同步必须在
master 和 slave 同时开启，任何一方没有开启，都是异步的模式。

### 管理界面

### 安装配置

### 监控

实战
====

如何在主从不同步的情况下，重新同步主从?
---------------------------------------

在我的测试机器上面，我在多次运行测试语句后，发现主机上有从机不存在的表。现在我想重新让两者同步，怎么办？

    -- master
    RESET MASTER;
    FLUSH TABLES WITH READ LOCK;
    SHOW MASTER STATUS;

    mysqldump -u root -p --all-databases &gt; /a/path/mysqldump.sql
    scp /a/path/mysqldump.sql TO SLAVE /b/path/mysqldump.sql
    UNLOCK TABLES;

    -- slave
    mysql -uroot -p &lt; mysqldump.sql

    RESET SLAVE;
    CHANGE MASTER TO MASTER_LOG_FILE=&#39;mysql-bin.000001&#39;, MASTER_LOG_POS=154;
    START SLAVE;
    SHOW SLAVE STATUS;

&lt;https://stackoverflow.com/questions/2366018/how-to-re-sync-the-mysql-db-if-master-and-slave-have-different-database-incase-o&gt;
&lt;https://dev.mysql.com/doc/refman/5.7/en/reset-master.html&gt;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/02/mysql5replication/  

