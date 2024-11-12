# Mysql 原厂检查清单


1\) MySQL配置文件 （my.cnf 或 my.ini）

2\) MySQL完整的错误日志文件 （error log
file）.（如果文件太大，可以压缩后上传）

3\) MySQL的 slow query log 文件 （如果已经配置收集的话）.

4\)
生成下面的mysql\_output.txt文本文件（请在查询性能低、响应慢时运行）：

    （请使用具有SUPER权限的MySQL用户（如root）登录MySQL命令行客户端并运行）

    TEE mysql_output0416.txt;
    select now(),@@version,@@version_comment,@@hostname,@@port,@@basedir,@@datadir,@@tmpdir,@@log_error,
    @@slow_query_log_file,user(),current_user(),/*!50600 @@server_uuid,*/@@server_id\G
    SHOW GLOBAL VARIABLES;
    SHOW GLOBAL STATUS;
    SHOW ENGINES\G
    SHOW PLUGINS\G
    select benchmark(50000000,(1234*5678/37485-1298&#43;8596^2)); #should take less than 20 seconds
    SELECT ENGINE, COUNT(*), SUM(DATA_LENGTH), SUM(INDEX_LENGTH) FROM information_schema.TABLES GROUP BY ENGINE;
    SHOW ENGINE INNODB STATUS;
    /*!50503 SHOW ENGINE performance_schema STATUS */;
    /*!50503 SELECT * FROM performance_schema.setup_instruments WHERE name LIKE &#39;wait/sync%&#39; AND (enabled=&#39;yes&#39; OR timed=&#39;yes&#39;)*/;
    -- Info on transactions and locks
    SELECT r.trx_id waiting_trx_id, r.trx_mysql_thread_id waiting_thread, r.trx_query waiting_query,
    b.trx_id blocking_trx_id, b.trx_mysql_thread_id blocking_thread, b.trx_query blocking_query,
    bl.lock_id blocking_lock_id, bl.lock_mode blocking_lock_mode, bl.lock_type blocking_lock_type,
    bl.lock_table blocking_lock_table, bl.lock_index blocking_lock_index,
    rl.lock_id waiting_lock_id, rl.lock_mode waiting_lock_mode, rl.lock_type waiting_lock_type,
    rl.lock_table waiting_lock_table, rl.lock_index waiting_lock_index
    FROM information_schema.INNODB_LOCK_WAITS w
    INNER JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
    INNER JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id
    INNER JOIN information_schema.INNODB_LOCKS bl ON bl.lock_id = w.blocking_lock_id
    INNER JOIN information_schema.INNODB_LOCKS rl ON rl.lock_id = w.requested_lock_id\G
    SHOW FULL PROCESSLIST;
    /*!50503 SELECT * FROM information_schema.innodb_trx */;
    /*!50503 SELECT * FROM performance_schema.threads */;
    /*!50708 SELECT * FROM sys.session */;
    SHOW OPEN TABLES;
    SHOW MASTER STATUS\G
    SHOW SLAVE STATUS\G
    /*!50602 SELECT * FROM MYSQL.SLAVE_MASTER_INFO */;
    /*!50602 SELECT * FROM MYSQL.SLAVE_RELAY_LOG_INFO */;
    /*!50602 SELECT * FROM MYSQL.SLAVE_WORKER_INFO */;
    SHOW MASTER LOGS;
    SELECT SLEEP(300);
    SHOW GLOBAL STATUS;
    SHOW ENGINE INNODB STATUS;
    /*!50503 SHOW ENGINE performance_schema STATUS */;
    /*!50503 SELECT * FROM performance_schema.setup_instruments WHERE name LIKE &#39;wait/sync%&#39; AND (enabled=&#39;yes&#39; OR timed=&#39;yes&#39;)*/;
    -- Info on transactions and locks
    SELECT r.trx_id waiting_trx_id, r.trx_mysql_thread_id waiting_thread, r.trx_query waiting_query,
    b.trx_id blocking_trx_id, b.trx_mysql_thread_id blocking_thread, b.trx_query blocking_query,
    bl.lock_id blocking_lock_id, bl.lock_mode blocking_lock_mode, bl.lock_type blocking_lock_type,
    bl.lock_table blocking_lock_table, bl.lock_index blocking_lock_index,
    rl.lock_id waiting_lock_id, rl.lock_mode waiting_lock_mode, rl.lock_type waiting_lock_type,
    rl.lock_table waiting_lock_table, rl.lock_index waiting_lock_index
    FROM information_schema.INNODB_LOCK_WAITS w
    INNER JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
    INNER JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id
    INNER JOIN information_schema.INNODB_LOCKS bl ON bl.lock_id = w.blocking_lock_id
    INNER JOIN information_schema.INNODB_LOCKS rl ON rl.lock_id = w.requested_lock_id\G
    SHOW FULL PROCESSLIST;
    /*!50503 SELECT * FROM information_schema.innodb_trx */;
    /*!50503 SELECT * FROM performance_schema.threads */;
    /*!50708 SELECT * FROM sys.session */;
    SHOW OPEN TABLES;
    SHOW MASTER STATUS\G
    SHOW SLAVE STATUS\G
    /*!50602 SELECT * FROM MYSQL.SLAVE_MASTER_INFO */;
    /*!50602 SELECT * FROM MYSQL.SLAVE_RELAY_LOG_INFO */;
    /*!50602 SELECT * FROM MYSQL.SLAVE_WORKER_INFO */;
    select * from information_schema.innodb_trx;
    select * from information_schema.innodb_locks;
    select * from information_schema.innodb_lock_waits;
    select * from performance_schema.events_waits_history;
    SHOW MASTER LOGS;

    SELECT
    t.TABLE_SCHEMA, t.TABLE_NAME, s.TABLE_NAME
    FROM
    information_schema.tables t
    LEFT OUTER JOIN
    information_schema.statistics s ON t.TABLE_SCHEMA = s.TABLE_SCHEMA
    AND t.TABLE_NAME = s.TABLE_NAME
    AND s.INDEX_NAME = &#39;PRIMARY&#39;
    WHERE
    s.TABLE_NAME IS NULL
    AND t.TABLE_SCHEMA not in (&#39;information_schema&#39;,&#39;mysql&#39;,&#39;performance_schema&#39;)
    AND t.TABLE_TYPE = &#39;BASE TABLE&#39;;

    \s
    NOTEE;

    （注意：SELECT SLEEP(300)会休眠300秒，请勿中断运行！）

5\) 生成下面的query.txt文本文件（请在查询性能低、响应慢时运行）：

    TEE query.txt;

    EXPLAIN EXTENDED select count(1) from t_intg_dm_0863;
    SHOW WARNINGS;

    SHOW CREATE TABLE t_intg_dm_0863\G
    SHOW INDEXES FROM t_intg_dm_0863;
    SHOW TABLE STATUS LIKE &#39;t_intg_dm_0863&#39;\G

    SET PROFILING=1;
    SHOW SESSION STATUS;
    select count(1) from t_intg_dm_0863;
    select sleep(1);
    select count(1) from t_intg_dm_0863;
    SHOW SESSION STATUS;
    SHOW PROFILE ALL FOR QUERY 2;
    SHOW PROFILE ALL FOR QUERY 4;
    SELECT *
    FROM INFORMATION_SCHEMA.PROFILING
    WHERE QUERY_ID = 2 OR QUERY_ID = 4 ORDER BY SEQ;
    SET PROFILING=0;

    SET optimizer_trace=&#34;enabled=on&#34;;
    select count(1) from t_intg_dm_0863;
    SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;
    SET optimizer_trace=&#34;enabled=off&#34;;
    NOTEE;

6\)
OS的状态信息，生成文件linuxdiags.txt（使用root用户运行，请在查询性能低、响应慢时运行）：
(注意：先单独运行 &#34;script&#34;命令，然后再运行其他命令）

    script /tmp/linuxdiags.txt

    set -x
    id
    uptime
    uname -a
    free -m
    cat /proc/cpuinfo
    cat /proc/mounts
    mount
    ls -lrt /dev/mapper
    pvdisplay
    vgdisplay
    lvdisplay
    df -h
    df -i
    top -b -d 10 -n 6
    iostat -x 10 6
    vmstat 10 6
    numactl -H
    numastat -m
    numastat -n
    ps -ef | grep -i mysql
    ls -al /etc/init.d/ | grep -i mysql
    for PID in `ps -ef | awk &#39;/mysqld[^_[]/{print $2}&#39;`; do
    echo &#34;PID=$PID&#34;;
    cat /proc/$PID/limits;
    done
    ps auxfww | grep mysql
    dmesg
    egrep -i &#34;err|fault|mysql|oom|kill|warn|fail&#34; /var/log/*
    exit


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/04/mysql-check-list/  

