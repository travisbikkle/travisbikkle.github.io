# Mysql 主從複製異常恢復


檢查版本
========

如果版本較老，基於binlog而不是gtid的版本， 數據量較小，可以參考
基於binlog的老版本。

基於binlog的老版本
==================

    1. 登錄主機數據庫，
    mysql --login-path=local
    執行
    mysql&gt; stop slave;
    mysql&gt; show master status;
    記錄以上master status的 log_file 和 log_pos 信息
    mysql&gt; exit;

    2. 進入備份腳本目錄（根據版本不同，可能在以下位置）
    cd /opt/backup/mysql/backup_script
    或者
    cd /opt/backup/mysql/backup_script

    執行
    ./backupandDelete.sh

    3. 找到最新備份的文件，如
    cd /opt/backup/mysql/backup_data/xxxxx
    或者
    cd /opt/backup/mysql/backup_data/xxxxx

    找到備份文件
    xxxxx.sql.gz

    將其拷貝到另外一臺機器
    scp .. ..

    4. 在另外一臺機器，解壓
    gunzip -d xxxx.sql.gz
    得到如 /root/xxxx.sql 的文件

    5. 登錄MySQL
    mysql --login-path=local
    執行(注意替換路徑，和ip，端口，密碼等信息)
    mysql&gt; set sql_log_bin=0;
    mysql&gt; source /root/xxxx.sql
    mysql&gt; select user,host from mysql.user;
    mysql&gt; update mysql.user set host=&#34;另外一臺機器ip&#34; where user=&#34;replicator&#34;;
    mysql&gt; flush privileges;
    mysql&gt; set sql_log_bin=1;
    mysql&gt; change master to master_host=&#34;另外一臺機器ip&#34;, master_port=13307, master_log_file=&#34;上面記錄的log_file&#34;, master_log_pos=&#39;上面記錄的log_pos&#39;;
    mysql&gt; start slave;

    6. 查看主從狀態是否正常
    show slave status\G

基於gtid的版本
==============

    1. 登錄主機數據庫，
    mysql --login-path=local
    執行
    mysql&gt; stop slave;
    mysql&gt; exit;

    2. 進入備份腳本目錄（根據版本不同，可能在以下位置）
    cd /opt/backup/mysql/backup_script
    或者
    cd /opt/backup/mysql/backup_script

    執行
    ./backupandDelete.sh

    3. 找到最新備份的文件，如
    cd /opt/backup/mysql/backup_data/xxxxx
    或者
    cd /opt/backup/mysql/backup_data/xxxxx

    找到備份文件
    xxxxx.sql.bin（如果不是bin格式的文件，說明版本不對，此時不是商業版，或者沒有做使用mysqlbackup備份恢復的需求，後者直接拷貝最新版backupandDelete.sh使用即可，或者使用binlog的方案）
    將其拷貝到另外一臺機器
    scp .. ..

    在另一臺機器上，將備份文件的權限更改爲mysql的屬組
    chown mysql: xxxx.sql.bin

    4. 在要恢復的機器上，執行以下檢查：

    4.1 檢查是否有mysqlbackup程序
    mysqlbackup --version
    4.2 檢查/opt/mysql/.bashrc
    是否有ip_cluster_a的字樣，其中ip_cluster_a或者ip_cluster_b中，一個是本機ip，一個是對端ip

    如果以上條件滿足，進行第5_a步，否則執行5_b步驟。

    5_a. 進行恢復
    以下命令中的false或者true代表是否備份機器的/opt/mysql/data/目錄，請根據機器磁盤剩餘空間選擇
    /opt/mysql/dataRecover.sh xxxx.sql.bin false
    或者
    /opt/mysql/dataRecover.sh xxxx.sql.bin

    5_b. 如果在第4步檢查通過，此步可以跳過。否則，如果bashrc裏沒有ip_cluster_a或者ip_cluster_b，說明版本較老。

       方法一：可以取該版本對應的資料，按照資料進行操作（四個步驟：
           1.關閉sql_log_bin，更改mysql.user表的用戶ip爲對端ip
           2 清空binlog和relaylog信息
           stop slave; reset slave; reset master;
           3. 設置主從
           STOP SLAVE;CHANGE MASTER TO MASTER_HOST=&#39;${another_ip}&#39;, MASTER_AUTO_POSITION=1 FOR CHANNEL &#39;rpl1&#39;;START SLAVE

       方法二：也可以將兩個ip手動寫入該文件，並且拷貝最新版本的dataRecover.sh，然後執行步驟5_a
           注意替換值爲實際ip
           ip_cluster_a=${master_ip}
           ip_cluster_b=${slave_ip}

    6. 恢復完成後，登錄兩臺機器查看主從複製狀態。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/zh-hant/2019/04/recover-mysql-from-replication-error/  

