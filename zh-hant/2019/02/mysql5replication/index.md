# Mysql . 主從同步


大綱
====

本文參考或翻譯自：
&lt;https://dev.mysql.com/doc/refman/5.7/en/replication.html&gt;

MySQL 5.7 支持多種主從複製的方法  
1.  傳統方法：依賴binlog文件和文件的position保持同步
    （https://dev.mysql.com/doc/refman/5.7/en/replication-configuration.html）

2.  新方法： 依賴全局事務id即global transaction identifer（GTIDs）
    （https://dev.mysql.com/doc/refman/5.7/en/replication-gtids.html）

replication 支持不同類型的同步  
1.  異步複製（asynchronous，默認）

2.  同步複製（只有 NDB 集羣纔有的一種特性）

3.  半同步複製（semisynchronous，是對異步複製的一種補充）

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

複製格式  
1.  基於語句的(Statement Based Replication (SBR))

2.  基於行的(Row Based Replication (RBR))

3.  混合的，也就是結合以上兩種(Mixed Based Replication (MBR))
    &lt;https://dev.mysql.com/doc/refman/5.7/en/replication-formats.html&gt;.

選項與變量  
Replication is controlled through a number of different options and
variables. For more information, see Section 16.1.6, “Replication and
Binary Logging Options and
Variables”(&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-options.html&gt;).

replication 的其它用途  
&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-solutions.html&gt;

原理  
&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-implementation.html&gt;

研究路線  
1.  配置 replication

    1.  基於日誌位置的複製配置

    2.  基於GTIDs的複製

    3.  MySQL multi-Source replication

    4.  在上線機器上更改複製模式

    5.  複製與日誌記錄選項和變量

    6.  常用複製管理任務

2.  replication 實現

3.  replication 用途 ..

    1.  半同步

4.  replication notes and tips

配置 replication
================

基於 BinLog 日誌文件位置 的複製
-------------------------------

master
作爲數據庫改變的源頭，將事件（變化、更新等操作）寫入到二進制日誌，事件信息存儲的格式根據變化的不同而不同；slave
從主機讀取並執行日誌。 每臺 slave 都會獲取到一份二進制日誌（以下簡稱
binlog）的完整內容的副本。slave 會決定執行這個 binlog
的哪一部分。除非特別指定，否則全部執行。如果需要，你也可以配置只執行特定
database 或者 table 的相關語句。

不能配置只執行某一次特定的事件。

每臺 slave 都會記錄一個 binlog
的座標：文件名稱和這個文件中已經處理到什麼位置。這就意味着多個 slave
可以正在執行同一個 binlog 的不同部分。因爲是 slave 在控制這個過程，
slave 可以隨意連接、斷開 master 而不影響 master 操作。而且這意味着 slave
可以斷開、重連、恢復處理。

master 和每一個 slave 都必須有一個唯一的
server-id([https://dev.mysql.com/doc/refman/5.7/en/replication-options.html\#option\_mysqld\_server-id)，並且需要通過](https://dev.mysql.com/doc/refman/5.7/en/replication-options.html#option_mysqld_server-id)，並且需要通過)
CHANGE MASTER TO 語句提供 master
主機地址、日誌文件名稱、日誌文件位置等信息([https://dev.mysql.com/doc/refman/5.7/en/change-master-to.html)。這部分細節存儲在](https://dev.mysql.com/doc/refman/5.7/en/change-master-to.html)。這部分細節存儲在)
slave 的 master info repository，
可以是一個文件，也可能存儲在一個表中(&lt;https://dev.mysql.com/doc/refman/5.7/en/slave-logs.html&gt;)。

首先，掌握一些基礎命令，有助於後面的配置  

**控制 master 的語句** (SQL Statements for Controlling Master Servers)
（https://dev.mysql.com/doc/refman/5.7/en/replication-master-sql.html）

    SHOW BINARY LOGS
    SHOW BINLOG EVENTS
    SHOW MASTER STATUS
    SHOW SLAVE HOSTS

-   SHOW BINARY LOGS 等同於 SHOW MASTER LOGS. 需要權限.

-   支持指定文件名、位置等，見
    &lt;https://dev.mysql.com/doc/refman/5.7/en/show-binlog-events.html&gt;；
    大數據量耗費較多時間，可用 mysqlbinlog 工具代替，見
    &lt;https://dev.mysql.com/doc/refman/5.7/en/mysqlbinlog.html&gt;.

**控制 slave 的語句** (SQL Statements for Controlling Slave Servers)
（https://dev.mysql.com/doc/refman/5.7/en/replication-slave-sql.html）

    CHANGE MASTER TO
    CHANGE REPLICATION FILTER
    MASTER_POS_WAIT()
    RESET SLAVE
    SET GLOBAL sql_slave_skip_counter
    START SLAVE
    STOP SLAVE
    SHOW SLAVE STATUS and SHOW RELAYLOG EVENTS

-   內容參考
    [https://dev.mysql.com/doc/refman/5.7/en/change-master-to.html，關注點](https://dev.mysql.com/doc/refman/5.7/en/change-master-to.html，關注點)：

    1.  是否需要先 stop slave

    2.  多線程的 slave 下可能出現的間隙問題（gaps）以及
        `START SLAVE UNTIL SQL_AFTER_MTS_GAPS`

    3.  CHANGE MASTER TO .. FOR CHANNEL *channel* 的用法, 更多
        Replication Channel 參考
        &lt;https://dev.mysql.com/doc/refman/5.7/en/replication-channels.html&gt;

    4.  未指定的選項保留舊的值。

    5.  【重要】如果指定了 ‘MASTER\_HOST\` 或者
        `MASTER_PORT`，即使值沒有變化，mysql 也認爲 master
        主機也跟以前不一樣了。這種情況下，binlog的文件名和位置就失效了，所以如果不指定
        MASTER\_LOG\_FILE 和 MASTER\_LOG\_POS，mysql默認添加上
        MASTER\_LOG\_FILE=’&#39; 且 MASTER\_LOG\_POS = 4。

    6.  ssl 相關的配置，MASTER\_SSL\_XXX 和 --ssl-XXX
        (&lt;https://dev.mysql.com/doc/refman/5.7/en/encrypted-connection-options.html&gt;)
        功能一樣。

    7.  心跳檢測相關的選項（比如 MASTER\_HEARTBEAT\_PERIOD
        不指定，默認是系統變量 slave\_net\_timeout 的一半；更改
        slave\_net\_timeout 也要適當更改其它關聯選項否則不起作用等）

        更改默認值並檢查當前連接心跳次數：

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


            -- 檢查當前配置
            SELECT * FROM performance_schema.replication_connection_configuration\G
            -- 檢查當前連接心跳次數、連接狀態
            SELECT * FROM performance_schema.replication_connection_status\G

    8.  MASTER\_DELAY 與 延遲複製
        &lt;https://dev.mysql.com/doc/refman/5.7/en/replication-delayed.html&gt;

    9.  MASTER\_BIND 與多網卡平面有關（可用 SHOW SLAVE STATUS 查看）

    10. 一些不可同時指定的值：

            MASTER_LOG_FILE or MASTER_LOG_POS 與 RELAY_LOG_FILE or RELAY_LOG_POS 不可同時指定。
            MASTER_LOG_FILE or MASTER_LOG_POS 與  MASTER_AUTO_POSITION = 1 不可同時指定。
            MASTER_LOG_FILE or MASTER_LOG_POS 如果不指定，使用之前的舊值。

    11. 【重要】relaylog 的刪除

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

**Replication 與 binlog 的選項、變量**(Replication and Binary Logging
Options and Variables)
（https://dev.mysql.com/doc/refman/5.7/en/replication-options.html）

基於GTIDs的複製
---------------

MySQL multi-Source replication
------------------------------

在上線機器上更改複製模式
------------------------

複製與日誌記錄選項和變量
------------------------

常用複製管理任務
----------------

replication 實現
================

replication 用途
================

半同步複製
----------

&lt;https://dev.mysql.com/doc/refman/5.7/en/replication-semisync.html&gt;

mysql 默認是異步同步，master 把操作寫到 binlog 裏，但是不關心 slave
是否（或者何時）收到（或者處理）這些事件。這種方式下，master
如果崩潰，可能來不及把其已經提交的事務傳輸給任何一個 slave。
Consequently, failover from master to slave in this case may result in
failover to a server that is missing transactions relative to the
master.

半同步可以作爲異步的一種替代： 1. 當 slave 連接到主機的時候，它會提示
master，自己是否支持半同步 2. 當 master
開啓了半同步，並且有至少一臺開啓了半同步的 slave 連接到了
master，那麼任何一個執行事務的線程，就會一直等待，至少一個開啓了半同步的
slave
反饋其收到了這個事務相關的全部日誌（或者達到一個超時時間），然後纔會
commit； 3. slave 只有在收到事件、把事件寫入到 relaylog
並刷到磁盤後，纔會向 master 發出這個反饋； 4. 當超時時間已經達到，master
還沒有收到任何反饋，其會轉成異步模式；一旦任何一個 slave
趕上（步驟3完成？），master 還會繼續轉回半同步模式。 5. 半同步必須在
master 和 slave 同時開啓，任何一方沒有開啓，都是異步的模式。

### 管理界面

### 安裝配置

### 監控

實戰
====

如何在主從不同步的情況下，重新同步主從?
---------------------------------------

在我的測試機器上面，我在多次運行測試語句後，發現主機上有從機不存在的表。現在我想重新讓兩者同步，怎麼辦？

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
> URL: https://travisbikkle.github.io/zh-hant/2019/02/mysql5replication/  

