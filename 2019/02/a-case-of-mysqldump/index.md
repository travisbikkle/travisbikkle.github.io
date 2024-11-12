# Mysqldump 使用案例


背景
====

客户N在使用H部门提供的MySQL遇到部分性能问题后，未得到H部门的及时支撑。机缘巧合，我们的服务化MySQL刚刚发布第一版，客户N有意切换我们的MySQL。由于部门策略调整，我们准备由原来的社区MySQL切换为部门R的商业版MySQL，其间对接问题不提，客户提出的首要问题是前期尝试通过mysqldump备份数据，发现有报错并且很慢，我们的策略是
为拓展业务先把锅接下来吧 答应先提供数据迁移方案供客户评估。

机器、数据、应用情况
====================

1.  源机器cpu核心数16，内存32G；

2.  两台机器，一个是master，一个是slave；未配置互为主备；

3.  开启了基于GTID的主从复制；

4.  从镜像库来看，数据量3800W左右，实际生产环境每天还会增加约不到100w；

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
    &lt;td&gt;&lt;p&gt;0-1w&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1w-10w&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;10w-50w&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;50w-100w&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;100w-1000w&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;&amp;gt;1000w&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;tr class=&#34;even&#34;&gt;
    &lt;td&gt;&lt;p&gt;表数量约&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;2105&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;83&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;28&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;5&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;6&lt;/p&gt;&lt;/td&gt;
    &lt;td&gt;&lt;p&gt;1&lt;/p&gt;&lt;/td&gt;
    &lt;/tr&gt;
    &lt;/tbody&gt;
    &lt;/table&gt;

5.  MySQL为社区版5.7.23，所有表均为INNODB引擎；

6.  据客户N的业务人员反馈，他们尝试使用mysqldump可能会报错。

一些准备工作
============

为了能够顺滑的开展后期工作，我习惯先整理一些常用的命令，以备随时复制粘贴…

    -- 查询所有业务数据库的表名，数据库，存储引擎信息
    select table_name,table_schema,engine from information_schema.tables where engine=&#39;innodb&#39; and table_schema not in(&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);

    -- 查询所有业务数据库的表的数量
    select count(*) from information_schema.tables where engine=&#39;innodb&#39; and table_schema not in(&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);

    -- 查询所有表的数据量
    SELECT CONCAT(TABLE_SCHEMA,&#39;.&#39;,TABLE_NAME) AS table_name, IFNULL(TABLE_ROWS,0) as table_rows FROM information_schema.tables WHERE TABLE_SCHEMA NOT IN (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;) ORDER BY 2;


    -- 查询所有业务数据库的视图数量
    select table_name,table_schema from information_schema.views where table_schema not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);
    select count(*) from information_schema.views where table_schema not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);

    -- 查询所有routines(存储过程和函数)的数量
    select * from mysql.proc where db not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;)\G

    -- 查询所有触发器的数量
    SELECT * FROM information_schema.triggers where TRIGGER_SCHEMA not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;)\G

    -- 查询所有事件的数量
    SELECT * FROM information_schema.EVENTS where EVENT_SCHEMA not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;)\G

    -- 查询所有用户数量
    select user,host from mysql.user;

    -- 查看磁盘IO信息
    iostat -x -p /dev/mapper/vg02-lv02 1 10 -m
    iostat -x -p 1 10 -m

首先尝试使用原生mysqldump
=========================

业务诚不欺我，果然有坑，报错如下(安全需要，隐藏关键信息)。

mysqldump: Couldn’t execute *SHOW FIELDS FROM `XX`*: View *XX.XX*
references invalid table(s) or column(s) or function(s) or
definer/invoker of view lack rights to use them (1356)

报错信息很明显了，本次实践中，主要是视图引用创建语句中子查询的列不存在，select
都会报错，这个我们只能让业务自己去审视，决策是否删除或者修复。

由于通过mysqldump来发现那些视图有问题非常不效率，所有写了一个简单的脚本：

**搜集所有有问题的视图.**

    #!/usr/bin/env bash

    function usage {
      echo &#34;Usage: $0 [-u USER_NAME] [-p PASSWORD] [-d WORKDIR] [-D:DROP ERROR VIEWS]&#34;
      echo &#34;Do not support -uroot, using -u root please.&#34;
      # too 2
      exit 2
    }

    function set_variable {
      local varname=$1
      shift
      if [[ -z &#34;${!varname}&#34; ]]; then
        eval &#34;$varname=\&#34;$@\&#34;&#34;
      else
        echo &#34;Error: $varname already set&#34;
        usage
      fi
    }

    function execMysqlCommand {
        mysql -u${USER_NAME} -p${PASSWORD} --skip-column-names -e &#34;$1&#34;
    }

    function checkView {
        viewName=&#34;$1&#34;
        # too slow
        # mysql -u${USER_NAME} -p${PASSWORD} -e &#34;select 1 from ${viewName} limit 1&#34; &gt;/dev/null 2&gt;&gt;/home/mysql/temp/view_error
        # not good either
        # mysql -u${USER_NAME} -p${PASSWORD} -e &#34;update ${viewName} set thisIsANotExistCol=123;&#34; &gt;/dev/null 2&gt;&gt;/home/mysql/temp/view_error
        #
        execMysqlCommand &#34;show fields from ${viewName};&#34; &gt;/dev/null 2&gt;&gt;/home/mysql/temp/view_error
        return $?
    }

    function checkAllViewsAndGetErrorViews {
        echo &#34;&#34;&gt;/home/mysql/temp/view_error
        i=1
        for view in ${views[@]};do
            echo -n &#34;checking $view ...$i/${#views[@]}&#34; &#34;...&#34;
            checkView ${view}
            result=$?
            [[ ${result} -ne 0 ]] &amp;&amp; echo &#34;bad&#34;
            [[ ${result} -ne 0 ]] &amp;&amp; echo &#34;pass&#34;
            ((i&#43;&#43;))
        done;
        cat /home/mysql/temp/view_error|grep &#34;1356&#34;|awk -F&#34;&#39;&#34; &#39;{print $2}&#39;&gt;/home/mysql/temp/error_list
        rm /home/mysql/temp/view_error -rf
        error_views=(`cat /home/mysql/temp/error_list`)
    }

    function printIgnoreMsg {
        [[ ${#error_views[@]} -gt 0 ]] &amp;&amp; echo &#34;You can add these statements to mysqldump to ignore those error views:&#34;
        for view in ${error_views[@]};do
            echo -n &#34; --ignore-table=${view}&#34;
        done
        echo &#34;&#34;
    }

    function backupErrorViewsSql {
        echo &#34;Backing up create statement of error views to ${WORKDIR}...&#34;
        echo &#34;&#34; &gt; /home/mysql/temp/backup_create_view_sql -rf
        for view in ${error_views[@]};do
            execMysqlCommand &#34;show create view $view;&#34; &gt;&gt;/home/mysql/temp/backup_create_view_sql 2&gt;/dev/null
        done
        cat /home/mysql/temp/backup_create_view_sql|awk -F&#39;\t&#39; &#39;{print $2&#34;;&#34;}&#39;|grep -v &#39;Create View;&#39;&gt;&gt;/home/mysql/temp/backup_create_view
        rm -rf /home/mysql/temp/backup_create_view_sql
        echo &#34;Done backing up create statement of error views.&#34;

    }

    function deleteErrorViews {
        echo &#34;Dropping error views...&#34;
        for view in ${error_views[@]};do
            while [[ &#34;X&#34; == &#34;X${confirm}&#34; ]];do
                read -p &#34;please confirm to delete ${view}:(y/n)&#34; confirm
            done
            if [[ &#34;Xy&#34; == &#34;X${confirm}&#34; ]];then
                execMysqlCommand &#34;drop view $view;&#34; 2&gt;/dev/null
            fi
        done
        echo &#34;Done dropping error views.&#34;
    }

    init() {
        unset DELETE_VIEWS USER_NAME PASSWORD WORKDIR

        while getopts &#39;u:p:d:D?h&#39; option
        do
          case ${option} in
            d) set_variable WORKDIR $OPTARG ;;
            D) set_variable DELETE_VIEWS true ;;
            u) set_variable USER_NAME $OPTARG ;;
            p) set_variable PASSWORD $OPTARG ;;
            h|?) usage ;; esac
        done

        [[ -z &#34;${USER_NAME}&#34; ]] &amp;&amp; usage
        [[ -z &#34;${PASSWORD}&#34; ]] &amp;&amp; usage
        [[ -z &#34;${WORKDIR}&#34; ]] &amp;&amp; set_variable WORKDIR &#34;/home/mysql/temp&#34; &amp;&amp; mkdir -p ${WORKDIR}

        echo &#34;Using directory ${WORKDIR} as temp dir.&#34;
    }

    getAllViews() {
        echo &#34;Getting all views from schema...&#34;
        views=(`execMysqlCommand &#34;select concat(table_schema,&#39;.&#39;,table_name) from information_schema.views where table_schema not in (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);&#34; 2&gt;/dev/null`)
    }

    init $@
    getAllViews
    checkAllViewsAndGetErrorViews
    printIgnoreMsg

    [[ X&#34;true&#34; == X&#34;${DELETE_VIEWS}&#34; &amp;&amp;  ${#error_views[@]} -gt 0 ]] &amp;&amp; backupErrorViewsSql &amp;&amp; deleteErrorViews

命令优化x
=========

具体方案之前，先加上一些基本的备份对象

    --hex-blob --single-transaction --quick --routines --triggers

方案一 160分钟
--------------

单线程直接执行mysqldump，大概160分钟

    &gt; /data01/chroot/usr/local/mysql5.7.23/bin/mysqldump -udbXXXX -pXXXX --all-databases --hex-blob --ignore-table=netcxx.xxxxx --ignore-table=netxxx.rxxx(此处很多忽略的视图) | gzip &gt; /temp/back0129.sql.gz

方案二 90分钟
-------------

考虑一个表一个文件，10个线程，大概90分钟；TODO 测试增加线程

**multidump.sh\[lines=25..55\].**

    multidump() {
        rm -rf ${WORKDIR}/backup
        mkdir -p ${WORKDIR}/backup

        COMMIT_COUNT=0
        COMMIT_LIMIT=10
        error_views_file=&#34;${WORKDIR}/error_list&#34;
        DBTBS=(`cat ${WORKDIR}/listOfTables`)
        i=1
        for DBTB in ${DBTBS[@]};do
            echo &#34;processing $i/${#DBTBS[@]}&#34;
            ((i&#43;&#43;))
            DB=`echo ${DBTB} | sed &#39;s/\./ /g&#39; | awk &#39;{print $1}&#39;`
            TB=`echo ${DBTB} | sed &#39;s/\./ /g&#39; | awk &#39;{print $2}&#39;`
            if [[ &#34;X&#34;`grep -w ${DBTB} ${error_views_file}` != X&#34;&#34; ]];then
                echo skip &#34;${DBTB}&#34;
                continue
            fi
            dumpIt ${DB} ${TB}
            (( COMMIT_COUNT&#43;&#43; ))
            if [[ ${COMMIT_COUNT} -eq ${COMMIT_LIMIT} ]]
            then
                COMMIT_COUNT=0
                wait
            fi
        done
        if [[ ${COMMIT_COUNT} -gt 0 ]]
        then
            wait
        fi
    }

方案三 15-22分钟
----------------

mysqlpump 是 mysql 提供的工具，文档和网上教程一大堆，这里只谈使用。
可以很直观的看到执行到哪个表，剩余多少行；注意：mysqlpump遇到错误会停止继续，比如命令不正确、数据结构有问题。而且这个数据库开启GTID，所以如果你的数据库没有此选项，要把命令中的—set-gtid-purged=ON去掉。

两种压缩格式的时间差距还是很明显：

mysqlpump -u*username* -p*password* --compress-output=ZLIB
--default-parallelism=100 --set-gtid-purged=ON --hex-blob
--add-drop-database --add-drop-table --add-drop-user --users |gzip &amp;gt;
/temp/test.sql.gz

Dump progress: 0/xx tables, xx/xxxxxxxxx rows Dump completed in xxxxxx
milliseconds

mysqlpump -u*username* -p*password* --compress-output=LZ4
--default-parallelism=100 --set-gtid-purged=ON --hex-blob
--add-drop-database --add-drop-table --add-drop-user --users &amp;gt;
/temp/testlz4.lz4

方案四
------

mysqlpump
可以针对database进行多线程导出，但是有时候数据分布不均匀，90%的数据可能都在一个表内，这种情况下mysqlpump显得无能为力。有没有可以对单个大表继续进行分拆的工具呢？
[mydumper](https://github.com/maxbube/mydumper/releases) 可以做这件事。

### 首先统计表的分布

    totalSql=&#34;SELECT IFNULL(SUM(TABLE_ROWS),0) as t_rows_sum FROM information_schema.tables WHERE TABLE_SCHEMA NOT IN (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;);&#34;
    eachTableSql=&#34;SELECT CONCAT(TABLE_SCHEMA,&#39;.&#39;,TABLE_NAME) AS table_name, IFNULL(TABLE_ROWS,0) as table_rows FROM information_schema.tables WHERE TABLE_SCHEMA NOT IN (&#39;mysql&#39;,&#39;information_schema&#39;,&#39;performance_schema&#39;,&#39;sys&#39;) ORDER BY 2;&#34;

### 验证mydumper导出database的效率

1.  只导出netcare，17分钟-20分钟左右

mydumper -u *username* -p *password* -v 3 -B *databaseName* --triggers
--events --routines --rows=500000 --compress-protocol -c -t
*threadNum.ie.100* --trx-consistency-only --outputdir /temp/mydumper

### 验证mydumper导出某一个大表的效率

&lt;https://dev.mysql.com/doc/mysql-enterprise-backup/8.0/en/&gt;

    方案五 优化前：21分钟
    unset tempdir
    tempdir=/data03/backup_`date &#39;&#43;%y%m%d%H%M%S&#39;`
    mkdir ${tempdir}
    ~/mysqlbackup -udbAdmin -pabcd1234 --backup-dir=${tempdir} --compress backup
    echo &#34;Successfully backing up data to ${tempdir}&#34;

    # 有待优化 https://dev.mysql.com/doc/mysql-enterprise-backup/4.1/en/backup-capacity-options.html
    --limit-memory=MB （default 100）
    --read-threads=num_threads （default 1）
    --process-threads=num_threads （default 6）
    --write-threads=num_threads （default 1）
    ~/mysqlbackup -udbAdmin -pabcd1234 --backup-dir=${tempdir} --compress backup

    #优化后： 15分钟
    # 整库备份到单个文件
    ~/mysqlbackup -udbAdmin -pabcd1234 --compress --compress-level=5 --limit-memory=1024 --read-threads=10 --process-threads=15 --write-threads=10 --backup-dir=${tempdir} --backup-image=/data03/`basename ${tempdir}`.bin backup-to-image

    #直接备份到目标机器：
    ~/mysqlbackup -udbAdmin -pabcd1234 --compress --compress-level=5 --limit-memory=1024 --read-threads=10 --process-threads=15 --write-threads=10 --backup-dir=${tempdir} --backup-image=- backup-to-image | ssh root@10.15.32.73 &#39;cat &gt; /opt/temp_for_restore/my_backup.bin&#39;

具体备份恢复使用见另一篇文章 mysql gtid 主从复制数据迁移(物理备份)


---

> : [Travis Bikkle](https://github.com/travisbikkle)  
> URL: https://travisbikkle.github.io/2019/02/a-case-of-mysqldump/  

