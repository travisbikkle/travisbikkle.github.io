# Mysql 主从复制异常恢复


检查版本
========

如果版本较老，基于binlog而不是gtid的版本， 数据量较小，可以参考
基于binlog的老版本。

基于binlog的老版本
==================

    1. 登录主机数据库，
    mysql --login-path=local
    执行
    mysql&gt; stop slave;
    mysql&gt; show master status;
    记录以上master status的 log_file 和 log_pos 信息
    mysql&gt; exit;

    2. 进入备份脚本目录（根据版本不同，可能在以下位置）
    cd /opt/backup/mysql/backup_script
    或者
    cd /opt/backup/mysql/backup_script

    执行
    ./backupandDelete.sh

    3. 找到最新备份的文件，如
    cd /opt/backup/mysql/backup_data/xxxxx
    或者
    cd /opt/backup/mysql/backup_data/xxxxx

    找到备份文件
    xxxxx.sql.gz

    将其拷贝到另外一台机器
    scp .. ..

    4. 在另外一台机器，解压
    gunzip -d xxxx.sql.gz
    得到如 /root/xxxx.sql 的文件

    5. 登录MySQL
    mysql --login-path=local
    执行(注意替换路径，和ip，端口，密码等信息)
    mysql&gt; set sql_log_bin=0;
    mysql&gt; source /root/xxxx.sql
    mysql&gt; select user,host from mysql.user;
    mysql&gt; update mysql.user set host=&#34;另外一台机器ip&#34; where user=&#34;replicator&#34;;
    mysql&gt; flush privileges;
    mysql&gt; set sql_log_bin=1;
    mysql&gt; change master to master_host=&#34;另外一台机器ip&#34;, master_port=13307, master_log_file=&#34;上面记录的log_file&#34;, master_log_pos=&#39;上面记录的log_pos&#39;;
    mysql&gt; start slave;

    6. 查看主从状态是否正常
    show slave status\G

基于gtid的版本
==============

    1. 登录主机数据库，
    mysql --login-path=local
    执行
    mysql&gt; stop slave;
    mysql&gt; exit;

    2. 进入备份脚本目录（根据版本不同，可能在以下位置）
    cd /opt/backup/mysql/backup_script
    或者
    cd /opt/backup/mysql/backup_script

    执行
    ./backupandDelete.sh

    3. 找到最新备份的文件，如
    cd /opt/backup/mysql/backup_data/xxxxx
    或者
    cd /opt/backup/mysql/backup_data/xxxxx

    找到备份文件
    xxxxx.sql.bin（如果不是bin格式的文件，说明版本不对，此时不是商业版，或者没有做使用mysqlbackup备份恢复的需求，后者直接拷贝最新版backupandDelete.sh使用即可，或者使用binlog的方案）
    将其拷贝到另外一台机器
    scp .. ..

    在另一台机器上，将备份文件的权限更改为mysql的属组
    chown mysql: xxxx.sql.bin

    4. 在要恢复的机器上，执行以下检查：

    4.1 检查是否有mysqlbackup程序
    mysqlbackup --version
    4.2 检查/opt/mysql/.bashrc
    是否有ip_cluster_a的字样，其中ip_cluster_a或者ip_cluster_b中，一个是本机ip，一个是对端ip

    如果以上条件满足，进行第5_a步，否则执行5_b步骤。

    5_a. 进行恢复
    以下命令中的false或者true代表是否备份机器的/opt/mysql/data/目录，请根据机器磁盘剩余空间选择
    /opt/mysql/dataRecover.sh xxxx.sql.bin false
    或者
    /opt/mysql/dataRecover.sh xxxx.sql.bin

    5_b. 如果在第4步检查通过，此步可以跳过。否则，如果bashrc里没有ip_cluster_a或者ip_cluster_b，说明版本较老。

       方法一：可以取该版本对应的资料，按照资料进行操作（四个步骤：
           1.关闭sql_log_bin，更改mysql.user表的用户ip为对端ip
           2 清空binlog和relaylog信息
           stop slave; reset slave; reset master;
           3. 设置主从
           STOP SLAVE;CHANGE MASTER TO MASTER_HOST=&#39;${another_ip}&#39;, MASTER_AUTO_POSITION=1 FOR CHANNEL &#39;rpl1&#39;;START SLAVE

       方法二：也可以将两个ip手动写入该文件，并且拷贝最新版本的dataRecover.sh，然后执行步骤5_a
           注意替换值为实际ip
           ip_cluster_a=${master_ip}
           ip_cluster_b=${slave_ip}

    6. 恢复完成后，登录两台机器查看主从复制状态。


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/04/recover-mysql-from-replication-error/  

