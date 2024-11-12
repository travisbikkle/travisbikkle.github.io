# Mysql 查詢鎖狀態常用命令


    show status like &#39;%lock%;

    select * from information_schema.processlist;
    select * from information_schema.processlist where state like &#34;%Waiting%&#34;;
    select * from information_schema.innodb_trx;
    SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS;

    SELECT INNODB_LOCKS.*
    FROM INNODB_LOCKS
    JOIN INNODB_LOCK_WAITS
      ON (INNODB_LOCKS.LOCK_TRX_ID = INNODB_LOCK_WAITS.BLOCKING_TRX_ID);

    SELECT * FROM INNODB_LOCKS
    WHERE LOCK_TABLE = db_name.table_name;

    SELECT TRX_ID, TRX_REQUESTED_LOCK_ID, TRX_MYSQL_THREAD_ID, TRX_QUERY
    FROM INNODB_TRX
    WHERE TRX_STATE = &#39;LOCK WAIT&#39;;

    show engine innodb status;


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/02/mysql-lock-status-commands/  

