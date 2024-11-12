# 一次 Mysql 死鎖問題解決


一個業務反應，環境多次出現大量服務不可使用，如app導入不響應，用戶更新超時，bpm創單超時等等。查看數據庫的processlist，發現有大量的處於Waiting
for table metadata
lock狀態的查詢，其中包含T\_APP\_INFO、TBL\_UM\_USER、T\_TICKET\_BASICINFO等表，跟故障服務一致，確定故障原因是數據庫鎖表引起；
業務自行導出所有的阻塞task，並按照阻塞時間排序，發現第一條引起阻塞的是一條來來自於localhost的
由root用戶發起的批量鎖表語句，疑似是問題根因。

上面這段是業務說的，已經排查的比較深入了，給個贊。

我之前通過直接kill掉這個query線程，他們的業務就正常走下去了，因爲忙其他事情，所以就沒有再關注。後面他們又出現了這個問題，這次必須要解決了。所以記錄一下定位過程。

定位思路
========

1.  **\[WHAT\]** &lt;root@localhost&gt; 的進程在做什麼？

    **MySQL 所有“卡住”問題，先看進程列表：.**

        show processlist;

        &#43;---------&#43;------&#43;-----------&#43;------&#43;---------&#43;------&#43;----------&#43;------------------&#43;
        | Id      | User | Host      | db   | Command | Time | State    | Info             |
        &#43;---------&#43;------&#43;-----------&#43;------&#43;---------&#43;------&#43;----------&#43;------------------&#43;
        | 3467133 | root | localhost | NULL | Query   |    320400 | Waiting for table metadata lock | LOCK TABLES `....|
        &#43;---------&#43;------&#43;-----------&#43;------&#43;---------&#43;------&#43;----------&#43;------------------&#43;

    看到 &lt;root@localhost&gt; 的用戶，有一條狀態爲 Waiting for table
    metadata lock 的查詢。查詢語句爲“LOCK TABLES……”。

    猜測：是後臺備份進程在鎖表，由於也有可能業務自己登陸後臺鎖表，所以需要證明這個確實是備份工具發起的語句。

    證明：當前時間是2月12日下午，Time
    時間顯示此語句已經等待320400s（約89小時），往前推算約爲2月9日凌晨0點。後臺備份文件夾有一個0點的文件夾，裏面備份文件爲0字節。

2.  **\[QUESTION\]** 爲什麼會導致這個問題出現

    在
    &lt;https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html#option_mysqldump_databases&gt;
    第一句就有，mysqldump 如果不加
    --single-transaction，執行mysqldump的用戶就必須有LOCK
    TABLES的權限，由此推出，這種情況（不加
    --single-transaction，當前MySQL方案）下，就會鎖表。那麼這種鎖錶行爲分不分存儲引擎呢？答案是myisam都會鎖表，而innodb會在—single-transaction的時候不鎖表，詳見官網。

    由於業務在show processlist;的時候，還看到了有另外一條語句B

    select \* from activemq\_lock for update;

    那麼即使有這條語句在執行，如果其正常提交了事務，也不會阻塞備份工具鎖表。業務由於直接用的開源activemq，所以也說不清楚這個表的作用，那麼問題出在哪裏？

3.  **\[控制變量\]** 和正常的環境來對比

    頂着業務的各種不滿（業務：我也學會kill進程了，你要解決根本問題啊，不能再kill了），直接再次kill掉這個進程，查看正常環境的狀態。發現語句B始終存在，並且show
    processlist發現該語句的Time一直在變化。說明其一直在頻繁執行。

    猜測：語句B一直在執行獲取鎖，mysqldump在備份前先 LOCK TABLES
    所有表，其它表都正常鎖住，唯有這個表獲取不到鎖，就一直等待。而其它被鎖住的表，此時是無法更新的。

    證明：當前環境再次手工執行一把備份，發現備份腳本卡住，查看processlist，發現與問題描述表現一致。此時業務果然發生了上面的問題。

解決方法
========

1.  疑問點

    早有耳聞mysqldump有—skip-lock-tables、--single-transation、--ignore-table的選項，但是由於不熟悉，所以還要自己驗證一番，看看各個參數是不是如自己所想：

    -   --skip-lock-tables
        是跳過獲取不到鎖的表，還是備份前不加鎖，還是備份的語句裏面不加鎖（曾被誤導，以爲是備份後的語句）

    -   --single-transaction
        看起來和鎖沒什麼關係，能不能達到我們的目的呢？

    -   --ignore-table
        是忽略表和視圖的意思，如果忽略這個表，那還會不會鎖住這個表呢？

2.  驗證

    帶着上面的疑問去自己的測試數據庫驗證，首先了解本例涉及到的幾種鎖以及如何構造它們：

    **我們常使用的鎖，語句一般就幾種：.**

        flush tables with read lock;
        lock tables tablename read;
        select * from table name for update;
        -- 其它 share mode 等暫不談，也不在此老生常談排他鎖、互斥鎖、只讀鎖等的概念。

    研究如下：

    &lt;table style=&#34;width:100%;&#34;&gt;
    &lt;colgroup&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;col style=&#34;width: 14%&#34; /&gt;
    &lt;/colgroup&gt;
    &lt;tbody&gt;
    &lt;tr class=&#34;odd&#34;&gt;
    &lt;td&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;說明&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;影響&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;如何定位這種鎖&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;如何釋放&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;進階&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;even&#34;&gt;
    &lt;td&gt;&lt;p&gt;flush tables with read lock;&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;全部的表都刷上read lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;執行此語句的session，在修改數據會收到報錯：ERROR1223 cant execute query because you hanve a conflicting lock.&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;其它session，在修改數據會卡住。&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1. 無法通過show open tables查看&lt;br /&gt;
    2. 無法通過information_schema.innodb_locks等表查看&lt;br /&gt;
    3. 無法通過show engine innodb status\G查看&lt;br /&gt;
    4. 其它被鎖住的session，可以通過show processlist;查看到狀態:Waiting for global read lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;session結束或者unlock tables;&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;在手工備份的時候很好用&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;odd&#34;&gt;
    &lt;td&gt;&lt;p&gt;lock tables t_test read;&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;只對某一個表刷上read lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1. 執行此語句的session，在修改數據會收到報錯：ERROR1099 table … was locked with read lock and cant be updated.&lt;br /&gt;
    2. 只能查詢鎖住的表，如果查詢其它的表，也會失敗&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;其它session，在修改數據會卡住。&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1. show open tables 可以看到表的 in_use &#43; 1&lt;br /&gt;
    2. 無法無法通過information_schema.innodb_locks等表查看&lt;br /&gt;
    3. show engine innodb status\G其中的 transactions 一列顯示 mysql tables in use 1,locked 1&lt;br /&gt;
    4. 其它被鎖住的session，可以通過show processlist;查看到狀態: Wating for table metadata lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;session結束或者unlock tables;還有其它一些場景會釋放鎖，比如alter table，詳見官網文檔；&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;釋放鎖會默認提交事務，具體詳見官網文檔&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;even&#34;&gt;
    &lt;td&gt;&lt;p&gt;select * from t_test for update;&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;對某一個表刷上排他鎖。 只能在一個事務中使用，不在事務中無效 ， 使用見附錄&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;執行此語句的session，就是爲了更改數據。&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;其它session，在修改數據會卡住。&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1. show open tables 可以看到表的 in_use &#43; 1&lt;br /&gt;
    2. 無法無法通過information_schema.innodb_locks等表查看&lt;br /&gt;
    3. show engine innodb status\G其中的 transactions 一列顯示 2 lock stucts, 2 row lock(表數據行&#43;1數量的鎖)&lt;br /&gt;
    4. 其它被鎖住的session，可以通過show processlist;查看到狀態: Wating for table metadata lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;commit; 其它未commit的異常狀態，鎖也會隨着session關閉釋放掉，具體見官網文檔&#34;&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;行鎖需要有主鍵或者索引 。本例無，所以是表鎖的效果。&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;/tbody&gt;
    &lt;/table&gt;

    經過組合 `select * from t_test for update` 和
    `lock tables t_test read;`
    重現了業務的問題。後續經過驗證，上面提到的 mysqldump
    的三個參數，都可以達到目的：

    &lt;table&gt;
    &lt;colgroup&gt;
    &lt;col style=&#34;width: 33%&#34; /&gt;
    &lt;col style=&#34;width: 33%&#34; /&gt;
    &lt;col style=&#34;width: 33%&#34; /&gt;
    &lt;/colgroup&gt;
    &lt;tbody&gt;
    &lt;tr class=&#34;odd&#34;&gt;
    &lt;td&gt;&lt;p&gt;方案&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;說明&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;缺點&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;even&#34;&gt;
    &lt;td&gt;&lt;p&gt;1. mysqldump –skip-lock-tables&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;備份前，不加鎖&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;無法保證數據一致性&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;odd&#34;&gt;
    &lt;td&gt;&lt;p&gt;2. mysqldump –single-transaction&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;備份在一個事務中進行&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;備份期間表定義變化等可能導致備份失敗（重新執行一次備份即可）&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;even&#34;&gt;
    &lt;td&gt;&lt;p&gt;3. mysqldump –ignore-table=activemq_lock&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;略過該表，不會獲取鎖&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;不備份該表&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;/tbody&gt;
    &lt;/table&gt;

附錄
====

**select…for update 的使用方法.**

    begin;
    select * from t_test for update;
    commit;

    begin;
    select * from t_test where id=1111 for update;
    commit;
    https://dev.mysql.com/doc/refman/5.7/en/select.html

**lock tables 的使用方法.**

    lock tables t_test read;

參考：
======

&lt;https://dev.mysql.com/doc/refman/5.7/en/select.html&gt;

&lt;https://dev.mysql.com/doc/refman/5.7/en/lock-tables.html&gt;

&lt;https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html#option_mysqldump_databases&gt;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/02/mysql-dead-lock-troubleshoot-case/  

