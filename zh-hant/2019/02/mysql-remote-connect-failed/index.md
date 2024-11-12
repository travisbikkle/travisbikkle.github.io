# Mysql 遠程無法連接問題定位記錄


某業務通過Hibernate訪問mysql，後臺報錯 Access denied for user
*matexxx*@*xxxx* (using password: YES);
一般搞過開發的人都知道，這種問題不是密碼錯了，就是遠程連接未打開，這兩者其實都屬於一個問題，就是用戶的grant權限問題，但是此業務情況稍特殊。定位過程如下。

查看用戶
========

    SELECT USER,HOST FROM MYSQL.USER;

發現用戶matexxx對應的host爲
%，說明遠程連接已經打開；詢問業務是否更改過密碼，引出問題背景：
業務曾重裝過mysql，使用mysqldump將舊庫數據備份，並且只在新庫的master上執行了一次恢復操作。

查看主從複製的狀態
==================

    SHOW SLAVE STATUS\G

發現互爲主備的mysql機器，其中一臺的slave
io狀態爲connecting，Last\_IO\_Error 顯示覆制用戶 replicator
禁止登錄。既然複製用戶和業務用戶都無法登錄，懷疑點聚焦在用戶的grant語句方面，原因可能是其備份恢復過程中出現錯誤操作，其要求緊急恢復，原因就暫不深挖。

【解決】 主從複製的問題要先解決。錯誤產生的原因很可能是其使用mysqldump
--all-databases備份，然後在配置好主從的機器上直接恢復，導致兩邊的機器replicator主從複製用戶的ip並不正確（實際應該配置對方ip）。恢復方法：

**請將下面語句中的變量替換爲實際的值.**

    GRANT REPLICATION SLAVE ON *.* TO &#39;${repl_user_name}&#39;@&#39;${IP}&#39; IDENTIFIED BY &#39;${repl_user_pwd}&#39;;
    FLUSH PRIVILEGES;

    SHOW MASTER LOGS; --在master(互爲主備的機器，master就是你要複製的機器，請自行理解)上執行
    -- 記錄上面執行語句的結果，例如
    -- Log_name：mysql-bin.000002
    -- File_size：483

    STOP SLAVE; --在出錯的機器上，執行
    CHANGE MASTER TO MASTER_HOST=&#39;${master_ip}&#39;,MASTER_PORT=&#39;3306&#39;,MASTER_LOG_FILE=&#39;mysql-bin.000002&#39;,MASTER_LOG_POS=483;
    START SLAVE;

回到主要問題
============

重啓業務應用（反正已經壞了）發現仍然無法登錄，查看進程列表，發現大量連接狀態都爲
` Waiting in connection_control plugin `，而且在另一臺機器C上面使用matexxx登錄一直卡住，而使用root卻沒有問題，證明此用戶登錄失敗，被拒絕後觸發了
connection\_control 的機制。

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

此處解釋一下 connection\_control 的作用
=======================================

一句話，防暴力破解。

官網曾有人對此插件發表過疑問，官方表示不應該在開放遠程連接的外網機器上配置此插件，而且在生產環境上也不應存在host爲%的用戶。
&lt;https://bugs.mysql.com/bug.php?id=89155&gt;

此插件的作用是，多次登錄失敗，服務器增加對客戶端的響應延遲，以增加暴力破解的時間；少量的失敗登錄對用戶的正常登錄沒有影響，如果存在大量的失敗登錄（被暴力破解時）則用戶正常登錄時耗時會增加。

**查看插件是否啓用.**

    show plugins;
    select plugin_name,plugin_library,load_option from information_schema.plugins;
    show variables like &#34;%connection_control%&#34;;

    connection_control_failed_connections_threshold: 3 
    connection_control_max_connection_dely: 214xxxxxxxx 
    connection_control_min_connection_dely: 1000 

-   在機制生效前允許的失敗次數

-   允許延長到的最大時間

-   最小時間，單位ms

暫時規避插件的作用，簡化問題
============================

註釋掉/etc/my.cnf的connection\_control相關的行，重新啓動兩臺機器。使用機器C重新登錄兩臺機器，發現其中一臺遠程可以登錄了，
但是另一臺開始很快反饋報錯信息。懷疑在有問題的機器上，用戶密碼被錯誤的修改過。

【解決】在無法登陸的機器上，重新運行grant語句並指定密碼

    GRANT ALL PRIVILEGES ON *.* TO &#34;matexxx&#34;@&#34;%&#34; IDENTIFIED BY &#34;${userPWD}&#34;;
    FLUSH PRIVILEGES；

重啓應用後問題消失。打開插件。

正確的恢復方法
==============

由於前期人員備份腳本使用的
--all-databases，會一起導出用戶信息。所以正確的恢復方法，大體是：

    在舊的主機和備機上各自執行備份命令。
    在安裝好的新的機器上，各自登錄並執行
    STOP SLAVE;
    各自導入備份的文件，查看show master logs並在兩臺機器上重新配置指定binlog文件：
    CHANGE MASTER TO MASTER_LOG_FILE=&#39;XXXX.bin0000001&#39;, MASTER_LOG_POS=123;
    START SLAVE;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/02/mysql-remote-connect-failed/  

